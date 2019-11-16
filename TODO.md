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

Common strategies are to use the Git commit hash or the CI build ID as part of the image tag. Can be combined with semantic versioning.

For example: `v1.0.1-bfeda01f`

**Avoid the `latest` tag**

* * *

## Deploying applications

### Use an Ingress for routing traffic to your app

Even for simple applications.

Requires installation of an [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

### Set a PodDisruptionBudgets for your applications

TODO: integrate with current "Set pod disruption budgets" in "Fault tolerance" (application development)

### All services that don't need to be accessed from outside the cluster should be ClusterIP

### Secure Ingress endpoints with TLS

* * *

## User management

### Use an external identity system for user management

For example, Azure Active Directory, AWS IAM.

Authentication with bearer token and API server validates token with the external service

* * *

## Additional services

### Run your own container registry

TODO: is this a best practice?

### If you use Helm, run your own Helm chart registry

Chartmuseum, Artifactory

* * *

## Pod networking

TODO: integrate with current "Network policies"

### Use NetworkPolicy to restrict the communication between Pods

### Create a deny-all network policy in all namespaces

### Avoid inter-Pod communication across namespaces

* * *

## Pod security

TODO: integrate with current "Pod security policies" (governance)

### Use PodSecurityPolicy to enforce security features in all Pods

- Using PodSecurityPolicies requires the PodSecurityPolicy admission controller to be enabled, but in most Kubernetes deployments it is not enabled. As soon as the PodSecurityPolicy admission controller is enabled, you need appropriate PodSecurityPolicy resources to allow any Pods to be created.
  - You also need to grant "use" access to the created PodSecurityPolicies to the service account of the workload or the controller of the workload (you can use  `system:serviceaccounts` group which compromises all controller service accounts).

* * *

## General policies

TODO: integrate with current "Custom policies" (governance)

### Use Open Policy Agent (OPA) and Gatekeeper to enfore custom policies on all resources

Only allow compliant Kubernetes resources (of any kind) to be applied to the cluster (compliant with the defined policies).

- Open Policy Agent (OPA): policy engine
- Gatekeeper
  - Validating admission control webhook
  - Kubernetes Operator for installing, configuring and managing Open Policy Agent policies

Example policies that can be implemented with Gatekeeper:

- Services must not be exposed publicly on the internet
- Allow containers only from trusted container registries
- All containers must have resource limits
- Ingress hostnames must not overlap
- Ingresses must use only HTTPs

* * *

## Stateful applications

### Avoid managing state in Kubernetes, if possible

Use storage services outside the cluster (e.g. Amazon DynamodDB, Amazon S3).

### Create a default StorageClass named "default"

Because this name is often the default in Helm charts.

### Prefer operators for running stateful applications rather than managing it yourself

Many stateful applications (e.g. databases) have operators that make it easier to run and manage them reliably.

* * *

## Admission controllers

### Use the officially recommended set of admission controllers

Recommended set of admission controllers to be enabled: `NamespaceLifecycle, LimitRanger, ServiceAccount, DefaultStorageClass, DefaultTolerationSeconds, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, Priority, ResourceQuota, PodSecurityPolicy`

### If you use multiple mutating admission webhook, make sure that they don't modify the same fields of a resource

They would undo each other's actions. Furthermore, the order in which admission webhooks run is undefined.

### If you use create a mutating admission webhook, also create a validating admission webhook that validates the mutations

### Restrict the request to be sent to an admission webhook to the minimum

### Scope admission webhooks to specific namespaces

Using the `namespaceSelector` field in the MutatingWebhookConfiguration/ValidatingWebhookConfiguration resource.

### Always exclude the kube-system namespace from the scope of a custom admission webhook

### Restrict RBAC rules for creating admission webhook configuration

Restrict "create" on MutatingWebhookConfiguration/ValidatingWebhookConfiguration resources.

* * * 

## Managing YAML resource manifests

### If you deploy to multiple environments, use a templating system

Do not manually maintain copies of the same YAML files for the different environments. Use a templating system like Helm, kustomize, Kapitan, ytt.

Works by defining the common structure of the YAML files once and defining values to substitute for each environment.

* * *

## Logging

TODO: integrate with current "Logging" (application development) and "Logging setup" (cluster configuration).

### Applications write logs to stdout or stderr rather than to files

This allows for the use of node agent based log aggregation systems rather than sidecar container based ones.

Everything a container writes to stdout or stderr is saved by the container runtime at a known location. This means that an agent running on each worker node can collect the logs of all the containers on that node and send them to the central log store.

If the container writes the logs to a file instead, the log file is local to the container and can't be accessed by a node agent. In this case, each Pod would require a sidecar container that collects these logs and sends them to the central log store.

A node agent (for example, deployed as a DaemonSet) is more efficient and easier to manage than a sidecar container for every Pod.

### Use a log aggregation system

Cluster components and applications log to their containers, which means that logs are distributed all across the cluster. This makes it difficult to inspect the logs (probably of multiple components) to troubleshoot a problem.

A log aggregation system collects all these logs and saves them at a central place in the cluster so that you can inspect them at a single place.

Some log aggregation tools include: EFK stack (Elasticsearch, Fluentd, Kibana), Loki, DataDog, Sumo Logic, Sysdig, GCP Stackdriver, Azure Monitor, AWS CloudWatch

### Use managed log aggregation system rather than a self-hosted one (if possible)

The operational overhead of running your own log aggregation system can be come quite large. This is because for running a log aggregation system, you need to take care of things like persistent storage, backups, archival, log rotation, etc.

With a hosted log aggregation solution, all these things are taken care of for you and there are fewer things that can go wrong.

Some hosted log aggregation systems include: DataDog, Sumo Logic, Sysdig, GCP Stackdriver, Azure Monitor, AWS CloudWatch

### Define a log retention and archival strategy

How long to retain logs in the log store and what to do with them afterwards?

As a general rule, retaining logs for 30-45 day in the log store is a reasonable value. After that, if the logs should still be available, you can move them to a cost-efficient archiving storage (e.g. [Amazon Glacier](https://aws.amazon.com/glacier/)).

Take into account any regulatory and internal compliance in your organisation.

### Collect logs from both cluster components and applications

Your applications are not the only components that produce logs in your cluster — all the cluster components do as well, and you should collect their logs too.

Here are some of the cluster components whose logs you should also have in your log aggregation system:

- All nodes: kubelet, container runtime
- Master nodes: API server, scheduler, controller mananger
- (Kubernetes auditing (all requests to the API server))

* * *

## Monitoring

### Set up extensive monitoring in your cluster

TODO: what kind of advice to give in this item?

Monitoring is the collection and aggregation of measurements from different components of your cluster. It is extremely important for gaining insight into the internals of your cluster, assessing its health, and detecting and troubleshooting problems, and even preventing them before they occur.

Monitoring consists of two parts:

1. Components make measurements and expose them as metrics
2. The monitoring system periodically collects these metrics

Some monitoring systems include:

- Self-hosted: Prometheus
- Managed: DataDog, Sumo Logic, Sysdig, Google Stackdriver, Azure Monitor, Azure Monitor for Containers, AWS CloudWatch, AWS Container Insights

_What should you monitor?_

Exactly what metrics to collect depends on the components in your cluster (i.e. what metrics they expose). 

There are some general guidelines as to the _types_ of metrics to collect:

- Infrastructure (e.g. nodes): USE metrics — Usage, Saturation, Errors
- Applications: RED  metrics — Rate, Errors, Duration

### If you use a self-hosted monitoring system, run it in a dedicated management cluster

You could run the monitoring system in the production cluster itself. However, if there is a problem with the cluster, the monitoring system might be affected as well (and you might _need_ the monitoring system to troubleshoot the problem with the cluster).

To avoid this, you can create a dedicated cluster that only runs management tools (such as the monitoring system) for all your other clusters.

* * * 

## Alerting

### Only alert on events that require immediate human intervention

Omit alerts that don't require a human to take action right away. Too many unimportant alerts cause "alert fatigue" and cause important alerts to be ignored.

### Focus on alerts that affect your service level objectives (SLOs)

The core of your alerts should be on incidents that negatively affect the service level that you promised to your customers. 

### Automate remediation of all non-critical alerts

All incidents that don't merit an alert (because they don't require immediate human intervention, or they don't affect the customer experience) should be handled automatically (invest in automation).

* * *

## Configuration and secrets

TODO: integrate with current "Configuration and secrets"

### Separate all configuration from the application code

Configuration should be maintained outside the application code.

This has several benefits. First, changing the configuration does not require recompiling the application. Second, the configuration can be updated when the application is running. Third, the same code can be used in different environments.

In Kubernetes, the configuration can be saved in ConfigMaps, which can then be mounted into containers as volumes are passed in as environment variables.

Save only non-sensitive configuration in ConfigMaps. For sensitive information (such as credentials), use the Secret resource.

### Save non-critical configuration in ConfigMaps and critical configuratio in Secrets

Secrets are similar to ConfigMaps but have some special semantics to protect their content (e.g. content of Secrets is not displayed in some kubectl outputs)

### Mount Secrets as volumes, not environment variables

The content of Secret resources should be mounted into containers as volumes rather than passed in as environment variables.

This is to prevent that the secret values appear in the command that was used to start the container, which may be inspected by individuals that shouldn't have access to the secret values.

- Injected environment variables are always present and may become artifacts in logs for the entire system.
- Secret-based environment variables should be mounted as a volume (not environment variables). In this way, they're only available to the desired process/container. Not the whole pod.

See [tweet](https://twitter.com/jeyfelbrandauer/status/1194752211366088704)

### Use PodPresets to automatically mount ConfigMaps or Secrets into containers

[PodPresets](https://kubernetes.io/docs/concepts/workloads/pods/podpreset/)

### Include version in ConfigMap and Secret to ensure configuration changes are reloaded

For example, instead of naming a ConfigMap just `config` name it `config-v1`. Whenever you update the configuration in the ConfigMap, also update the version (for example, `config-v2`). Then, update all references to the ConfigMap (for example, in a Deployment) to the new version (e.g. `config-v2`).

This causes all Pods to be restarted, whih ensures that the new configuration is indeed loaded into the Pods, regardless of whether the ConfigMap is mounted as a volume or as environment variables, and, in the latter case, regardless of whether the app watches the configuration file for changes.

You can automate this process with a CD system.

If you don't delete the previous versions of the ConfigMap (e.g. `config-v1`), you can easily roll back to a previous configuration.

* * *

## Role-based access control (RBAC)

TODO: integrate with current "Role-Based Access Control (RBAC) policies" (governance)

### Follow the "least privilege" principle for RBAC roles

### Don't use "catch all" service accounts for Pods

If a Pod needs to access the Kubernetes API, tailor an RBAC role that allows exactly those operatios that the Pod has to do (and nothing more), assign it to a new service account, and assign this service account to the Pod.

Don't use an exising service account for the Pod that might have an associated role with more permissions than the Pod needs.

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
  labels:**
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

* * *

## Amazon Web Services (AWS)

### Use kube2iam to integrate Kubernetes permissions with AWS IAM

As a best practice to use this, I would recommend using [kube2iam](https://github.com/jtblin/kube2iam) -- this allows you to control which IAM roles a pod is allowed to assume based on namespace, but also allows the pod to see itself as having the role vs requiring it to assume such a role.

In our setup we use kube2iam and assign nodes their IAM role - this IAM role is allowed to assume other roles. With kube2iam, we assign the role through a pod annotation, so the pod sees itself as having that role. Additionally we use namespace restrictions to control access to which roles a pod is allowed to assume.

See #7
