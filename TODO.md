## Reserve priority and space for Kubernetes resources

Kubelet can be instructed to reserve a certain amount of resources for the system and for Kubernetes components (kubelet itself and Docker etc).

Reserved resources are subtracted from the node's allocatable resources. This improves scheduling and makes resource allocation/usage more transparent.

You can [explore how to reserve resources on the official documentation](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable).

## Handling long lived connections

<https://itnext.io/on-grpc-load-balancing-683257c5b7b3>
<https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/>
