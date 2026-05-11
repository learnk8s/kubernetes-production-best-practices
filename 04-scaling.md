# Scaling

> _How does my app handle real-world load?_

When discussing Kubernetes scaling, tools such as HPA, VPA, KEDA, Cluster Autoscaler, and Karpenter often come up.

Cluster Autoscaler and Karpenter mostly handle the platform by adding or removing nodes.

HPA, VPA, and KEDA focus on the workload by changing the number of Pods or the size of those Pods.

For those managing a workload, scaling means understanding how the app behaves under pressure and providing Kubernetes with the right information to make decisions.

Two changes from late 2025 matter for vertical scaling.

- **[In-place Pod resize](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga/) became generally available in Kubernetes 1.35 (December 2025).** You can now change CPU and memory settings on a running Pod without restarting it, making vertical scaling much smoother.
- **[Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 1.4+ added an `InPlaceOrRecreate` update mode that uses in-place Pod resize.** This mode is still in beta in Kubernetes 1.35, but it means VPA can now be used for workloads sensitive to interruptions.

## Scaling model

These checks cover how the workload should grow or shrink under load.

Choose horizontal, event-driven, or vertical scaling based on application behaviour, and set autoscaler limits so scaling does not create new failures.

### The application can scale horizontally

Horizontal scaling means running multiple copies of the same Pod and is usually the best option for apps that don’t keep state.

**The [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) helps with this, as do tools like [KEDA](https://keda.sh/) for event-driven scaling based on queue length or Kafka lag.**

A horizontally scaled app must avoid per-Pod state and tolerate replicas being added or removed.

Check these conditions before increasing the replica count:

- The application does not store state on the local disk.
- The application handles long-lived connections so that new replicas see traffic.
- The application shuts down gracefully so that scale-down does not drop in-flight requests.

If any requirements are missing, adding more replicas can create additional problems rather than solve them.

For stateless services, HPA works well when CPU, memory, or request rate closely match the load.

For event-driven workloads, [KEDA](https://keda.sh/) usually works better than plain HPA.

**Queue length, Kafka lag, Pub/Sub backlog, waiting jobs, and scheduled traffic often provide better signals than CPU usage.**

KEDA also supports scaling down to zero, which HPA cannot do on its own.

### Autoscaler bounds and scale-down behavior are explicit

An autoscaler should in place and its minimum, maximum, and scale-down settings should be carefully chosen.

For HPA, set clear bounds:

- `minReplicas` protects baseline availability and cold-start latency.
- `maxReplicas` protects downstream dependencies, node capacity, and cost.
- `behavior.scaleDown` controls how quickly Kubernetes removes replicas after load drops.
- `behavior.scaleUp` controls how aggressively Kubernetes adds replicas when load rises.

For example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

**The scale-down window is important because it stops the autoscaler from removing capacity right after a brief drop in traffic.**

Without it, the workload can change too quickly: scaling up during a spike, scaling down too soon, then scaling up again with the next burst.

For `ScaledObject` workloads, KEDA creates and manages an HPA behind the scenes: the KEDA object still needs `minReplicaCount`, `maxReplicaCount`, `pollingInterval`, and `cooldownPeriod` values that match the workload.

For example:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-worker
spec:
  scaleTargetRef:
    name: my-worker
  minReplicaCount: 1
  maxReplicaCount: 50
  pollingInterval: 30
  cooldownPeriod: 300
```

If you want more control, KEDA lets you adjust the HPA behavior using `advanced.horizontalPodAutoscalerConfig.behavior`.

### Vertical scaling is understood as an option, not a default

Vertical scaling means making each Pod larger, and it’s the right choice in some specific cases:

- **When horizontal scaling does not help,** for example, a single-replica stateful database, a JVM that benefits from a bigger heap, or an ML model that needs more memory to load.
- **When the workload was over-provisioned,** it should shrink without hand-editing the manifest.
- **When load patterns change over time,** continuous right-sizing is more practical than periodic tuning.

Vertical scaling was difficult because VPA had to delete and recreate a Pod to change its size.

**This caused problems for apps sensitive to delays, so most teams only used VPA to get recommendations and made changes by hand.**

In-place Pod resize fixes this.

**Now you can update CPU and memory on a running Pod without restarting it.**

With VPA 1.4+ in `InPlaceOrRecreate` mode (beta in Kubernetes 1.35), VPA can keep adjusting your Pods without usually interrupting your app.

One important rule remains: VPA and HPA should not control the same metric for the same workload.

If both react to the CPU, they can conflict.

For example, VPA raises CPU requests, which changes how HPA measures usage and can cause problems.

The safest way is to use VPA for memory and HPA for CPU or another custom metric.

## Resource pressure

These checks cover what happens when resources are limited.

Requests should be based on real usage, and priority should make it clear which workloads survive contention.

### Resource requests are based on real usage data

Initial resource requests are estimates, which is acceptable.

The key is to update them with real usage data as soon as it becomes available.

The workflow is straightforward:

1. Deploy with the best available guess for CPU and memory requests, and a memory limit.
1. Run the workload under a representative load.
1. Look at actual CPU and memory usage over a few days. Useful sources include Prometheus, a managed monitoring tool such as Datadog, `kubectl top`, or VPA in `Off` mode, which generates recommendations without applying them.
1. Update the manifest, or let VPA update it in place.

**Focus on the range of usage, not just the average.**

For example, if your app’s CPU averages 200m but peaks at 800m, set your request and limit for the peak, not the average.

Memory is less forgiving than the CPU.

**If a Pod averages 200 MiB but spikes to 500 MiB once an hour, it needs a 500 MiB request, or it will eventually be killed for running out of memory.**

> See [setting CPU and memory limits and requests](https://learnkube.com/setting-cpu-memory-limits-requests) for a deeper walkthrough of sizing decisions.

### Priority classes express what should survive resource pressure

When a node is overcommitted, the kubelet evicts Pods to free up resources.

By default, it uses QoS class and the time since Pods started, which usually isn’t what a production operator wants.

**[`PriorityClass`](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) is how that is expressed explicitly.**

A few classes cover most setups:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 1000000
globalDefault: false
description: 'Customer-facing production workloads.'
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 100
globalDefault: false
description: 'Background batch jobs; evict first'
```

A Pod spec then references the right class:

```yaml
spec:
  priorityClassName: production-critical
```

When resources are limited, lower-priority Pods are removed to make room for higher-priority ones.

They can also be stopped during scheduling.

**On a shared cluster running batch jobs, ML training, or CI runners alongside production traffic, priority classes help make sure batch jobs don’t take resources from customer-facing ones.**

## Traffic and validation

These checks cover whether scaling actually works for users.

Autoscaling, scale-down, and load spikes should be tested so added or removed replicas do not cause hidden errors or dropped requests.

### Scale-down drains traffic cleanly

When HPA removes replicas, Kubernetes picks a Pod and terminates it.

**This works the same way as a node drain or a rolling update: the Pod receives a `SIGTERM`, has a grace period, and is then killed.**

Graceful shutdown is very important here.

With steady traffic, a faulty `SIGTERM` handler might drop a few requests per restart without notice.

**But during an HPA scale-down in a traffic spike, many Pods stop quickly, and any that don’t drain properly will drop active requests.**

Check these things during scale-down:

- The Pod continues to serve traffic for a short grace period after `SIGTERM`, so endpoint state has time to propagate.
- Long-lived connections are closed cleanly rather than being dropped abruptly.
- `terminationGracePeriodSeconds` is long enough for the slowest in-flight request to finish.
- Do not rely on a PodDisruptionBudget to slow down HPA scale-down. PDBs protect voluntary evictions, such as node drains, not replica count changes from a Deployment or HorizontalPodAutoscaler.

> See [graceful shutdown](https://learnkube.com/graceful-shutdown) for the full pattern.

### The scaling path has been load-tested

An autoscaler works as a control loop, and control loops have some delay.

HPA’s default check interval is 15 seconds, metrics take time to reach the server, and a new Pod needs time to start and be ready.

**From when a new load arrives to when a new Pod serves traffic, it often takes 30 to 90 seconds.**

If traffic grows faster than that, autoscaling alone won’t save your app.

Existing Pods will return errors while the autoscaler catches up, and users will notice.

**It’s much better to find this out in a load test than in production.**

A useful load test:

- Ramps traffic from zero to the target peak in a realistic time, rather than five minutes of flat load.
- Measures p99 latency and error rate through the entire ramp.
- Confirms that scaling actually happens by watching `kubectl get hpa` or the relevant metrics dashboard.
- Confirms that scale-down afterward drains traffic cleanly, including graceful termination and long-lived connection handling.

When traffic grows faster than the autoscaler can respond, options include pre-scaling before a known spike, lowering the HPA target utilization to allow more room, or using a faster signal, like KEDA on queue length, which can react more quickly than CPU-based scaling.

> See [autoscaling apps on Kubernetes](https://learnkube.com/autoscaling-apps-kubernetes) and [Kubernetes autoscaling strategies](https://learnkube.com/kubernetes-autoscaling-strategies) for the full picture.
