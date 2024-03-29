# Components

The following are a detailed description of each of the components that make up the solution.

## RDMA Hardware Daemon Set
The RDMA Harware Daemon Set is a [Kubernetes Daemon Set](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) that has two tasks. The first task is to initialize the RDMA SRIOV enabled interfaces on a node and the second task is to create a RESTful endpoint for providing metadata about the PF's and their associated VF's that exist on a given node that are part of the Kubernetes cluster. Further detail about each of the containers within the RDMA Hardware Daemon set can be seen below:

1. Init Container - the init container is a privileged container that runs as a [Kubernetes Init Container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/), in simplified terms this means it is run before any other container within the pod. The reason that the container needs to be run as a privileged container is because it will be working on a nodes network devices, which means it needs special access in order to configure them. The init container does two things, the first is to scan all the available interfaces on the node and determine if it is an RDMA device and whether SRIOV is enabled. For all interfaces that meet those two requirements, the init container configures each interfaces VF's to be available and ready to run.
2. Server Container - the server container is an unprivileged container that scans through all available interfaces when starting up. It then makes a list of all interfaces that are SRIOV enabled and upon a RESTful get request from a user will return associated metadata about the container. The server container only scans the interfaces at startup because SRIOV devices can only change configurations upon a machine restart, which would rerun the init container and then the server container. The RESTful endpoint serves data in a JSON formatted list of PF's and each PF has an internal list of VF's, more info can be found [here](https://github.com/rit-k8s-rdma/rit-k8s-rdma-ds). The RESTful endpoint that the server container sets up is bound to the host network.

Both the init container and the server container run under the same RDMA Hardware Daemon Set pod. When configuring how to install this pod please look at [daemon set install instructions](install.md#daemon_set_install_instructions) for more detail.


## Scheduler Extension

The scheduler extension is a software component that is responsible for ensuring that RDMA-enabled pods are deployed onto nodes with enough RDMA resources (bandwidth and virtual functions) to support them. The scheduler extension runs alongside the kube-scheduler process within a Kubernetes cluster, and listens for HTTP requests on a TCP port (8888 by default). The extension is then registered within the configuration file for kube-scheduler (see the [install page](install.md#scheduler_extension_install_instructions) for details). Every time a new pod is being scheduled, kube-scheduler will make an HTTP request to the scheduler extenstion that contains the details of the new pod as well as a list of nodes within the cluster that kube-scheduler believe to be eligible to deploy the pod onto. The scheduler extension's job is to filter down that list based on whether the nodes in it have enough free RDMA VFs and bandwidth to support the pods requirements or not.

In order to accomplish this, the scheduler extension contacts the RDMA hardware daemon set running on each of the potential nodes in the cluster to gather information about the current allocation of RDMA resources on that node. The requests to each daemon set are made in parallel, with a timeout for cases where a node's daemon set doesn't respond. Once the information has been gathered from a node, the scheduler extension calculates whether or not the node's interface and reserved bandwidth requirements can be met by the remaining resources on that node. If they can be, the node is added to a list of those that are elligible to deploy the pod onto. Once all of the nodes in the list passed in by kube-scheduler have been contacted or have timed out, this list is returned to kube-scheduler in an HTTP response. From here, kube-scheduler will limit its choice of where to place the new pod to one of the nodes in the returned list. If the returned list is empty, the pod will not be scheduled, and the output of `kubectl describe` for the pod will show the reasons given by the scheduler extension as to why the nodes in the list that was passed in were not eligible to host the pod.

For help installing the scheduler extension and registering it with kube-scheduler, see the [scheduler extension](install.md#scheduler_extension_install_instructions) section of the install page.

More information about the Kubernetes API for scheduler extensions can be found on the Kubernetes [website](https://kubernetes.io/docs/concepts/extend-kubernetes/extend-cluster/#scheduler-extensions) and [github](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md).

## CNI Plugin

The container network interface, or CNI, is a specification that governs the way network access is provided to containers and containerized applications. Kubernetes utilizes CNI as its standard for allocating IP addresses and network interfaces to pods that have been deployed in a cluster. A CNI plugin is a piece of software that performs that allocation, along with any other additional setup. Since RDMA network interfaces require specific additional setup to configure bandwidth limits and reservations, as well as to select virtual functions such that a pod's bandwidth requirements are satisfied, a CNI plugin is necessary for handling RDMA-enabled pods.

The CNI plugin developed for this project is a fork of the existing [Mellanox CNI plugin](https://github.com/Mellanox/sriov-cni), which was limited in the fact that it always allocated one interface to each pod and didn't support bandwidth reservation or limitation. The CNI plugin for this project improves upon this by adding support for an arbitrary number of RDMA interfaces per pod, including the ability to allocate no RDMA interfaces to pods that do not need any.

The CNI plugin is executed each time a pod is created or destroyed. It runs only once per pod for each of these actions, and must allocate or deallocate all of the interfaces for a pod at one time. This is done by executing an algorithm that finds a mapping of requested pod interfaces onto virtual functions that allows a pod's requirements to be satisfied. This is similar to one of the steps performed by the scheduler extension, though the scheduler extension need only determine whether a pod's requirements can be satisfied by a node, rather than what the exact final allocation of VFs to that pod will be to satisfy it.

The tasks performed by the CNI plugin during pod setup are the following:

 1. Determine a placement of RDMA virtual functions that will satisfy the pod's requirements.
 2. Move each of the needed virtual functions into the pod's network namespace.
 3. Rename each of the virtual functions that have been added to the pod's network namespace.
 4. Set the bandwidth limits and reservations on each RDMA VF, if necessary.
 5. Allocate IP addresses to each of the VFs that have been allocated to the pod.

The tasks performed by the CNI plugin during pod teardown are the following:

 1. Rename each of the virtual functions in the pod's network namespace.
 2. Move each of the virtual functions from the pod's network namespace back to the host's network namespace.
 3. Remove any bandwidth reservations or limitations set on the deallocated virtual functions.


## Dummy Device Plugin
The Dummy Device Plugin is a stop gap measure for the current system. The directory `/dev/infiniband` is needed within any pod that requires RDMA. In order to access devices in this directory a container either needs to be run in privileged mode or a [Device Plugin](https://kubernetes.io/docas/concepts/extend-kubernetes/compute-storage-net/device-plugins/) can return a directory that a container will have privileged access to. For obvious reasons we opted to create a Device Plugin for the sole purpose that it will give **any container** that requests an RDMA device the proper access to `/dev/infiniband`. This Dummy Device Plugin does not act like any other Device Plugin where it is meant to manage a specialized resources; that is done by the RDMA Hardware Daemon Set, Scheduler Extender, and CNI components. When configuring how to install this pod please look at [dummy device plugin installation](install.md#dummy_device_plugin_installation) for more detail.


