# Your Kubernetes manifests

> _Is my deployment descriptor complete?_

Your Kubernetes manifests act like a contract for Kubernetes.

A complete manifest helps Kubernetes schedule, run, update, and protect your workload predictably.

## Runtime contract

These checks cover the parts of the manifest Kubernetes needs before it can run the Pod safely: health probes, resource requests and limits, and bounded local disk usage.

They influence scheduling, restart decisions, traffic routing, and node pressure handling.

### Readiness, liveness, and startup probes are defined

[Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) tell Kubernetes how to check the health signal exposed by the application.

**A probe tells Kubernetes what health signal to monitor, how often to check it, and what to do if the check fails.**

There are three types of probes, each with its own purpose.

**A startup probe** checks if the application has finished starting.

This is important for workloads that take a long time to start.

When you set up a startup probe, Kubernetes waits for it to succeed before running liveness or readiness checks.

This prevents the container from being marked unhealthy while it is still starting.

**A readiness probe** checks if the container should get traffic right now.

If it fails, Kubernetes does not restart the container; instead, it removes the Pod from the Service endpoints.

This makes readiness useful in cases where the app is running but temporarily can’t handle requests, such as during warm-up, a slow dependency, or overload.

**A liveness probe** checks if the container is stuck and needs Kubernetes to restart it.

This helps in cases like deadlocks, where a process runs but does no useful work.

**Be careful with liveness probes: if set too strictly, they might restart containers that could have fixed themselves, worsening the problem.**

How you set up the probe is as important as the type you choose.

Settings like `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, and `failureThreshold` control when probing starts, how often it runs, how long Kubernetes waits for a response, and how many failures are allowed before it acts.

### Resource requests and limits are set

**Every container should specify the CPU and memory resources it needs.**

When a Pod is created, the scheduler uses [requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) to pick a node with enough free resources to run it.

> See [setting CPU and memory limits and requests](https://learnkube.com/setting-cpu-memory-limits-requests) for a deeper walkthrough.

**Requests and limits affect different parts of Kubernetes.**

Requests are a scheduling input: the scheduler uses them to decide where a Pod can fit before the Pod starts.

It does not use the app’s future real usage, and it does not enforce the request after the Pod is running.

**Limits are enforced at runtime by the kubelet, container runtime, and kernel.**

A limit sets the maximum amount a container can use while running on a node.

CPU and memory limits work differently: if a container uses too much CPU, it usually just slows down.

But if it goes over the memory limit, the system might kill it.

**Because of this, every container should have CPU and memory requests, as well as a memory limit.**

[Setting CPU limits depends on your workload and cluster setup.](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)

Some details are easy to miss.

**If you set a limit but not a request for a resource, Kubernetes might use the [limit](https://kubernetes.io/docs/concepts/policy/limit-range/) as the request unless another default is set.**

Requests and limits also affect how the Pod behaves when node resources are tight.

Kubernetes assigns each Pod a quality of service class based on the requests and limits set for its containers.

A Pod can be `Guaranteed`, `Burstable`, or `BestEffort`. These [classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/) influence which Pods kubelet evicts first when a node runs low on resources.

For example, a Pod is `Guaranteed` only when every container has both CPU and memory requests and limits set, and the request equals the limit for each resource.

QoS is not the same as priority.

`PriorityClass` is a separate scheduling and eviction signal: the scheduler can preempt lower-priority Pods to make room for a higher-priority Pod, and kubelet also considers priority during eviction.

### Ephemeral storage usage is bounded

CPU and memory are not the only resources that can strain a node.

**Kubernetes also tracks [local ephemeral storage](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage), which includes the container writable layer, container logs, and disk-backed `emptyDir` volumes.**

This matters for apps that write temporary files, cache data, buffer uploads, unpack archives, or create large logs.

**If this usage grows too much, kubelet can remove Pods when the node runs low on local storage.**

Set `ephemeral-storage` requests and limits for containers that use meaningful local disk space:

```yaml
resources:
  requests:
    ephemeral-storage: 1Gi
  limits:
    ephemeral-storage: 2Gi
```

If the app needs a writable folder like `/tmp`, cache, or upload space, mount an `emptyDir` there and set `sizeLimit` if you know the max size:

```yaml
volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 1Gi
```

This works well with `readOnlyRootFilesystem: true`: the image filesystem stays read-only, and only the few paths that need to be writable are clearly defined and limited.

## Rollouts and configuration

These checks cover what happens when the workload changes.

A production manifest should make rollouts predictable, tolerate old and new Pods running together, and define how configuration changes reach running Pods.

### Rolling update settings are explicit

A Deployment uses a rolling update by default.

**Kubernetes creates new Pods, waits for them to become available, and gradually removes old Pods.**

The defaults work for simple workloads, but production manifests should clearly set how rollouts happen.

The main fields are:

- `maxUnavailable`: how many replicas can be unavailable during the rollout.
- `maxSurge`: how many extra replicas Kubernetes can create above the desired replica count.
- `minReadySeconds`: how long a newly created Pod must be ready without any container crashing before the Deployment counts it as available.
- `progressDeadlineSeconds`: how long Kubernetes waits before marking the rollout as failed.
- `revisionHistoryLimit`: how many old ReplicaSets are kept for rollback.

For example, `maxUnavailable: 0` keeps capacity from dropping during a rollout, but requires enough extra capacity for surge Pods.

`minReadySeconds` makes the Deployment wait until a newly created Pod has been ready without container crashes for a minimum time before counting it as available.

It affects rollout progress and old Pod scale-down; it does not delay traffic once the readiness probe passes.

These settings don’t replace readiness probes.

**Instead, they rely on readiness probes to signal to Kubernetes when a new Pod can accept traffic.**

See the [Kubernetes rollback guide](https://learnkube.com/kubernetes-rollbacks) for a deeper explanation of how Deployments create ReplicaSets, perform rolling updates, and keep old revisions for rollback.

### The workload tolerates old and new Pods running together

**During a rolling update, Kubernetes can run both the old and new versions of a workload simultaneously behind the same Service.**

If old and new versions don’t work well together, the rollout can fail even if every Pod is healthy.

For example, a new Pod might write data that an old Pod can’t read, or a new API might send requests that the old backend doesn’t understand.

**The Kubernetes rule is simple: if you use `RollingUpdate`, the workload must handle mixed versions during the rollout.**

Deployments support two strategy types.

`RollingUpdate` is the default.

Kubernetes creates Pods for the new version while some Pods from the old version are still running.

The Service can send traffic to both versions during the rollout, depending on which Pods are Ready.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

`Recreate` works differently.

Kubernetes stops the old Pods first, then starts the new Pods.

**This avoids mixed versions but usually causes downtime unless another service handles traffic.**

```yaml
strategy:
  type: Recreate
```

If old and new Pods can’t safely run together, the Deployment strategy should show that.

**`Recreate` can be safer for workloads that need only one version at a time, but it sacrifices availability to avoid mixed versions, so it should be used with care.**

### ConfigMap and Secret updates have a reload strategy

**Kubernetes gives `ConfigMap` and `Secret` data to containers in different ways, and each method has different update behavior.**

Environment variables are read when the container starts.

If a `ConfigMap` or `Secret` used this way changes, the running container won’t see the new value: the Pod must be restarted.

Mounted volumes work differently.

Kubelet can update `ConfigMap` and `Secret` files in a running Pod, but the updates occur with some delay rather than instantly.

The delay depends on the kubelet’s sync and cache.

Also, volumes mounted with `subPath` don’t get updates.

Choose the behavior you want:

- **If the app reads configuration only at startup,** changing it should trigger a new rollout. Common Kubernetes approaches to this are using versioned `ConfigMap` names or changing a Pod template annotation so that the Deployment creates a new ReplicaSet.
- **If the app reloads files while running,** mount the configuration as files and make sure the watcher handles Kubernetes volume updates properly.
- **If the configuration should never change,** use an [immutable ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/#immutable-configmaps) or immutable Secret and create a new object for each change.

There is a tricky file-watching issue here.

Kubernetes updates projected files using symlinks, so normal file watchers can miss changes.

Ahmet explains this in [Pitfalls reloading files from Kubernetes Secret & ConfigMap volumes](https://ahmet.im/blog/kubernetes-inotify/).

## Placement and disruption

These checks cover how the workload is isolated and where replicas are placed.

They reduce the chance that a privilege mistake, node drain, or zone failure takes down the application.

### Non-root, read-only filesystem, and dropped capabilities are configured

Containers should run with the least privileges needed.

These settings are set in the Pod or container [security context](https://learnkube.com/security-contexts).

In Kubernetes, setting `runAsNonRoot: true` tells kubelet not to start the container if it would run as user ID 0.

You can also set `runAsUser` to a non-zero value to be clear.

Without [Pod user namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/), UID 0 inside the container maps to UID 0 on the node.

**With user namespaces enabled through `spec.hostUsers: false`, root inside the container can be mapped to an unprivileged user on the host, which reduces the impact of a container escape.**

That is a useful extra isolation layer, but it is not a reason to run normal applications as root by default.

Use non-root containers unless the workload has a real need for root inside the container.

If you rely on user namespaces for that exception, make it explicit in the manifest.

Next, make the root filesystem read-only.

**When you set `readOnlyRootFilesystem: true`, the application can’t write to its image filesystem.**

If it needs a writable path, like `/tmp`, mount a small writable volume such as `emptyDir` just for that location.

Then, remove any privileges the process does not need.

`allowPrivilegeEscalation: false` stops the process from gaining extra privileges while running.

`capabilities.drop: ["ALL"]` removes extra Linux capabilities most containers don’t need. If your container needs a specific capability, you can add it back.

You should also limit system calls.

When you use `seccompProfile.type: RuntimeDefault`, the container uses the runtime’s default seccomp profile. This blocks system calls that most regular apps don’t need.

### A PodDisruptionBudget is defined

When a node is drained for maintenance, upgrade, or cluster scale-down, Kubernetes removes the Pods running on it.

**Without a declared tolerance for [disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/), too many replicas of the same workload can go down at once.**

Kubernetes separates voluntary disruptions, such as draining or removing a node, from involuntary ones, such as a node crash.

A [PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) only applies to voluntary disruptions.

**A PDB sets how many Pods must stay available during a voluntary disruption.**

For example, `minAvailable: 2` means at least 2 matching Pods stay running. `maxUnavailable` sets the limit the other way and can be easier to understand.

A PDB is helpful for workloads that must stay available while nodes are drained.

**But it is a best-effort protection for voluntary evictions, not a hard availability guarantee.**

It cannot prevent a node from crashing, and it does not stop a Deployment or HorizontalPodAutoscaler from lowering the number of replicas.

It can also block maintenance if it is too strict.

For example, `minAvailable` equal to the replica count leaves no room for a node drain.

Choose a value that preserves enough healthy replicas while still allowing voluntary disruptions to make progress.

### Pods are spread across nodes and zones

**Don’t run all replicas of a workload in the same failure domain.**

If every replica is on a single node, a single failure can take them all down.

The same risk exists at the zone level.

To improve availability, spread replicas across different nodes and, if possible, across zones.

In production, use [`topologySpreadConstraints`](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) to set this up for each workload or as a cluster default:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: my-app
```

`topologyKey` decides how Pods are spread, and `maxSkew: 1` asks the scheduler to keep matching Pods as balanced as possible.

**`whenUnsatisfiable: ScheduleAnyway` lets the scheduler place the Pod even if a perfect balance isn’t possible, but it still prefers nodes that reduce the skew.**

Topology spread constraints are powerful, but they are not harmless defaults.

The scheduler only works with the topology domains it can see from existing eligible nodes, and multiple constraints are combined together.

**A strict `DoNotSchedule` rule can leave Pods Pending if one zone has no capacity, even when other zones could run them.**

Autoscaled node pools, taints, node affinity, and missing topology labels can all change the result.

Two details are worth checking every time: the `topologyKey` must exist consistently on eligible nodes, and the `labelSelector` must match the Pod template labels for the workload.

If either one is wrong, placement might look reasonable while the scheduler is balancing against the wrong set of Pods or domains.

Use topology spread constraints deliberately and test them under failure and scale-out scenarios.

The [KubeFM episode on pod topology spread constraints](https://kube.fm/pod-topology-martin) is a good discussion of how a reasonable-looking configuration can cause surprising scheduling behavior in production.

## Secrets and metadata

These checks cover the operational details that make workloads safer to manage at scale: how secrets are delivered, how resources are labelled, and whether manifests use APIs supported by the cluster versions you run.

### Secrets are mounted as volumes, not passed as environment variables

Kubernetes can give a Secret to your app as environment variables or as files.

For application credentials, it’s usually better to use mounted files.

**Environment variables can leak secrets: they might show up in debug output, process listings, or crash dumps.**

Once a secret is in the process environment, it can end up where it shouldn’t.

Mounted [Secret volumes](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod) are also easier to manage.

Kubelet can update mounted Secret files in a running Pod, while environment variables don’t update after container start.

### Resources have meaningful labels

**[Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) make it easier to understand Kubernetes resources as your cluster grows.**

Kubernetes recommends a common set of [`app.kubernetes.io/*`](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) labels so resources can be identified consistently across tools and operations.

**You don’t have to use these labels, but they make resources easier to manage, search, and understand.**

They should be applied to top-level resources such as Deployments, StatefulSets, and Services, as well as to the pod template within workload objects.

This helps you see what a resource belongs to, which instance it’s part of, which version is running, and which tool manages it.

Common labels include `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`, and `app.kubernetes.io/managed-by`.

**It’s also helpful to add a few custom labels for things your organization cares about, like owner, environment, or cost center.**

### Manifests use supported Kubernetes API versions

Every Kubernetes manifest starts with an `apiVersion` and a `kind`.

**The API server uses those fields to decode, validate, admit, and store the object.**

> See [what happens inside the Kubernetes API server](https://learnkube.com/kubernetes-api-explained) for the full request path.

Those API versions change over time.

A manifest that works on one cluster version can fail after an upgrade if it still uses an API version that has been deprecated or removed.

This affects Deployments, Ingresses, CronJobs, PodDisruptionBudgets, HPAs, and many other resources.

**It also affects Helm charts and live Helm releases, because the stored release metadata might still contain old API versions.**

Before going to production, check that your manifests use API versions supported by the Kubernetes versions you run now and plan to upgrade to next.

Useful checks include:

- Compare your manifests with the official [Kubernetes API deprecation guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/).
- Scan manifests, Helm charts, and live releases with tools such as [Pluto](https://github.com/FairwindsOps/pluto), [kube-no-trouble](https://github.com/doitintl/kube-no-trouble), or [KubePug](https://github.com/kubepug/kubepug).
- If a Helm release is already stored with removed API versions, use [helm-mapkubeapis](https://github.com/helm/helm-mapkubeapis) before upgrading the cluster.

**Deprecated APIs are a deployment risk because Kubernetes can reject the object before your app even starts.**

> See [validating Kubernetes YAML](https://learnkube.com/validating-kubernetes-yaml) for tools that can catch manifest issues before they reach your cluster.
