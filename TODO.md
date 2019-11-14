### Reserve priority and space for Kubernetes resources

Kubelet can be instructed to reserve a certain amount of resources for the system and for Kubernetes components (kubelet itself and Docker etc).

Reserved resources are subtracted from the node's allocatable resources. This improves scheduling and makes resource allocation/usage more transparent.

You can [explore how to reserve resources on the official documentation](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable).

### Handling long lived connections

<https://itnext.io/on-grpc-load-balancing-683257c5b7b3>
<https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/>

* * *


## Building container images

### Use only base images from trusted image providers

### Minimise image sizes/build "scratch" images

### Image tags are immutable

If the content of an image changes, the image tag must change too.

Common strategies are to use the Git commit hash or the CI build ID as part of the image tag.


* * *

## Configuration and secrets

TODO: integrate with current "Configuration and secrets"

### Use PodPresets to automatically mount ConfigMaps or Secrets into containers

### Include version in ConfigMap and Secret to ensure configuration changes are reloaded

For example, instead of naming a ConfigMap just `config` name it `config-v1`. Whenever you update the configuration in the ConfigMap, also update the version (for example, `config-v2`). Then, update all references to the ConfigMap (for example, in a Deployment) to the new version (e.g. `config-v2`).

This causes all Pods to be restarted, whih ensures that the new configuration is indeed loaded into the Pods, regardless of whether the ConfigMap is mounted as a volume or as environment variables, and, in the latter case, regardless of whether the app watches the configuration file for changes.

You can automate this process with a CD system.

### Mount Secrets as volumes, not environment variables

TODO: integrate with existing "Mount Secrets as volumes, not environment variables".

- Injected environment variables are always present and may become artifacts in logs for the entire system.
- Secret-based environment variables should be mounted as a volume (not environment variables). In this way, they're only available to the desired process/container. Not the whole pod.

See #5


* * *


## Labelling resources

TODO: integrate with current "Tagging resources"

### All resources have a common set recommended labels

**What?**

The [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) recommends a set of common labels to be applied to all resource objects.

**Why?**

Having a set of common labels allows third-party tools to interoperate with the resources in your cluster. Furthermore, a common set of labels facilitates and standardises manual management of the resources.

**How?**

The recommended labels are:

- `app.kubernetes.io/name`: the name of the application
- `app.kubernetes.io/instance`: a unique name identifying the instance of an application
- `app.kubernetes.io/version`: the current version of the application (e.g., a semantic version, revision hash, etc.)
- `app.kubernetes.io/component`: the component within the architecture
- `app.kubernetes.io/part-of`: the name of a higher level application this one is part of
- `app.kubernetes.io/managed-by`: the tool being used to manage the operation of an application

Here is an example of applying these labels to a StatefulSet resource:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-abcxyz
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
spec:
  # ...
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mysql
        app.kubernetes.io/instance: mysql-abcxzy
        app.kubernetes.io/version: "5.7.21"
        app.kubernetes.io/component: database
        app.kubernetes.io/part-of: wordpress
        app.kubernetes.io/managed-by: helm
    spec:
      # ...
```

Note that depending on the application, only some of these labels may be defined (for example, `app.kubernetes.io/name` and `app.kubernetes.io/instance`), but they should be applied to _all_ resource objects.

**References**

- [Kubernetes documentation: Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)

### Use independent versions for containers, Pods, and apps

E.g. if Pod specification changes, update only the Pod and app version, but not the container image version, etc.

### Maintain a "release" for top-level resources

Resources that compromise an entire app (e.g. Deployment, StatefulSet) should have a `release` label.

This label should change every time the application is deployed (the release should be updated even if the version of the app stays the same).


* * *


## Resource management

TODO: integrate with current "Resource utilisation"

### Define resource requests for all containers

**What?**

You can set a resource request (CPU and memory) for each container of a Pod in the Pod specification.

**Why?**

The **request** for a resource is the minimum amount of resource the container needs for running. It influences the scheduling decision of the Kubernetes scheduler (the scheduler schedules a pod only to a node that has enough free resources to accommodate the requests of all the containers of the pod).

By default, no resource requests are set, which means that the scheduler schedules a pod to any node, no matter how many free resources it has.

Setting requests for memory and CPU for each container of a Pod ensures that the Pod is able to run properly on the node it is scheduled to.

**How?**

Set the [`pod.spec.containers[].resources.requests`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#resourcerequirements-v1-core) field of a Pod specification:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
spec:
  # ...
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          requests:
            memory: 100Mi
            cpu: 100m
```

**References**

- [Kubernetes documentation: Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)

### Define resource limits according to the desired Quality of Service (QoS) class

The **limit** for a resource is the maximum amount of the resource that the container is allowed to use (you can think of it as an upper cap for resource usage bursts). What happens when a container reaches the limit, depends on the type of resource:

- For memory, the container is killed and restarted
- For CPU, the container is throttled (TODO: what does "throttled" mean exactly?)

By default, no resource limits are set, which means that there's no limit of how much resources a pod may use on a node.

Setting a resource limits prevents Pods from monopolising all the resources on node.

Given a value for the resource request, the value for the resoruce limit determines the Quality of Service (QoS) class of the Pod:

- Best-effort: no requests and limits, lowest priority, Pod is killed first
- Burstable: limits are higher than requests, middle-priority, killed if no Best-effort Pods exist
- Guaranteed: limits equal requests, highest priority, killed only if no Best-effort and Burstable Pods exist

### Don't define CPU limits

TODO: how does this affect the QoS class? Where's the source that this is a best practice?

### Create a LimitRange for each namespace

### Create a ResourceQuota for each namespace

### Use PriorityClass to define the order in which Pods are scheduled and evicted


* * *


## Health checks

TODO: integrate with current "Health checks"

### Set initialDelaySeconds to appropriate value for all liveness and readiness probes


* * *


## Advanced scheduling

### Use Pod Affinity if certain Pods should be colocated

### Use Pod Anti-affinity if certain Pods should not be colocated

### Use Node Affinity or Node Selector if a Pod should run only on a subset of nodes

### Use Taints and Tolerations to reserve nodes for certain Pods


* * *


## Application lifecycle

TODO: integrate with current "Graceful shutdown" (rename it "Application lifecycle")

### Container listens for SIGTERM signal and shuts down gracefully

### Container implements preStop lifecycle hook to shut down gracefully

This is an alternative to listening for the SIGTERM signal.

### Set the termination grace period if gracefully shutting down takes very long

### Container implements postStart lifecycle hook to do any start-up tasks

### Only use the exec method for the postStart lifecycle hook
