# Kubernetes Patterns

O'Reilly, 2019.

The book is a collection of "patterns" for container-based cloud-native environments and applications, and it shows how Kubernetes implements these patterns.

You can learn how to optimally use the tools that Kubernetes provides to you to make the best usage of Kubernetes' implementation of these patterns. 

## Chapter 2: Predictable Demands

Containers should know and declare their requirements. This allows the platform (Kubernetes) to put Pods to the right place in the cluster and treat them appropriately.

**Types of requirements:**

- Runtime dependencies
  - Persistent storage, host port, configuration (ConfigMaps, Secrets): defined in Pod spec.
- Resource requirements
  - Requests
  - Limits
  - **QoS** of Pods (defined by requests and limits): defines in what order Pods are killed by the **kubelet**
    - Best-effort: no requests and limits, lowest priority, Pod is killed first
    - Burstable: limits are higher than requests, middle-priority, killed if no Best-effort Pods exist
    - Guaranteed: limits equal requests, highest priority, killed only if no Best-effort and Burstable Pods exist

[**Pod Priority and Preemption**](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/):

[PriorityClass](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#priorityclass-v1-scheduling-k8s-io): priority of a Pod relative to other Pods, used by the **scheduler**

- Defines in what order Pods are scheduled (if multiple Pods are pending)
- If there's no place for a pending Pod, the scheduler may evict running Pods with a lower priority

PriorityClass and QoS are unrelated: **QoS used by kubelet** for killing, **PriorityClass used by scheduler** for scheduling (and evicting)

**Best practices:**

- Make tests to discover resource requirements (CPU and memory) for all containers and set requests and limits in Pod specs
- Use _Guaranteed_ QoS class for critical Pods in production (in dev, you can use _Best-effort_ and _Burstable_)
- If using _Burstable_ QoS, ensure a low limit/request ratio in the LimitRanges
  - The higher the limit/request ratio, the higher the risk that the node runs out of resources and Pods must be killed (if many containers are close to their limits at the same time)
- Lock down access to PriorityClass with RBAC, limit PriorityClass in ResourceQuota for each namespace

## Chapter 3: Declarative Deployment

## Chapter 4: Health Probe

A way for the platform (Kubernetes) to learn about the internal state of the containerised apps (which otherwise are black boxes for the platform) and take appropriate action.

The app should implement these interfaces to provide the maximum amount of information about its internal state.

Kubernetes (kubelet) performs three types of periodic health checks:

1. **Process:** is main process of container still running? → restart container
1. **Liveness:** implemented by app, is app running? → restart container
1. **Readiness:** implemented by app, is app ready to serve requests? → remove Pod from Service

**Best practices:**

- Define liveness and readiness probes for all containers
- Set initalDelaySeconds for all liveness and readiness probes (make tests to find out how big the value should be)

## Chapter 5: Managed Lifecycle

Signals that the platform emits to the application about the lifecycle of the application (which is managed by the platform).

The app should listen to these signals to learn about its own lifecycle (which it otherwise has no access to as it is fully managed by the platform) and react to it accordingly.

Signals:

- **SIGTERM:** emitted to a container when the platform decides to shut down a container (for whatever reason). The container should listen to this and gracefully shut down (i.e. cleaning up and exiting the main container process).
- **SIGKILL:** only emitted after SIGTERM when the container process is still running after `terminationGracePeriodSeconds`. Kills the container (hard).
- **postStart:** emitted immediately after creating a container. The implementation must be provided by the app (in the same way as liveness/readiness probes, i.e. _exec_, _httpGet_, _tcpSocket_). Happens independently from starting the container main process, i.e. calling the postStart hook and starting the container process happens at the same time.
- **preStop:** emitted when the platform decides to shut down a container. Emitted before SIGTERM and SIGTERM will be only executed after the preStop handler completes. Can be used as an alternative to reacting to SIGTERM.

**Best practices:**

- App should listen to either SIGTERM or implement a preStop hook and gracefully exit
  - Gracefully exiting should be quick, at least within the 30 seconds default `terminationGracePeriodSeconds`
- If gracefully exiting takes very long, set the `terminationGracePeriodSeconds` in the Pod spec
- Implement a postStart hook if there are any start-up tasks to do for a container
- Only use the _exec_ method for the postStart hook, because it is not guaranteed that the container process is already running when the postStart hook is called (so _httpGet_ and _tcpSocket_ could or could not be executed in a race-condition fashion)

## Chapter 6: Automated Placement
