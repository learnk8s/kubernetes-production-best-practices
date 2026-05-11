# Going live

> _When something breaks, will I know? Can I recover?_

Going live is about what to do once the workload is running in production.

When you start running your system live, it works nonstop.

To keep it running well, set up ways to watch it, record activity, send alerts, deliver updates, fix problems, and control costs from the beginning.

## Visibility

These checks cover whether the team can see what the workload and Kubernetes control plane are doing.

Metrics, logs, traces, and Events should be available before users notice a problem.

### The current health of the application is visible

Kubernetes observability has three main layers.

**Metrics are numbers recorded over time.**

They are easy and inexpensive to maintain and monitor, and they show how your system is performing.

[Prometheus](https://prometheus.io/) is the common choice, but any system that works with OpenTelemetry will do.

**Logs record specific events.**

They produce more data than metrics and help you understand what happened when a metric shows an issue.

**Traces show how long requests take as they pass through different parts of the system.**

They help you identify where delays occur when things are slow, and the reason is unclear.

Start with metrics. The two main types cover most needs:

- **USE metrics for infrastructure**, which stands for Utilization, Saturation, Errors. Nodes, Pods, and containers are a good fit.
- **RED metrics for services**, which stand for Rate, Errors, and Duration. This is what customer-facing workloads care about.

Logs also need a place to be stored.

**By default, Kubernetes keeps container logs on the node that ran them, but those logs are lost if the node fails.**

A small program running on each node, like [Fluent Bit](https://fluentbit.io/), [Vector](https://vector.dev/), [Grafana Alloy](https://grafana.com/docs/alloy/latest/), or a cloud provider’s tool, sends container logs to a central storage where you can search and keep them.

Collect both application logs and cluster logs (kubelet, API server, scheduler, controller manager), and turn on the [Kubernetes audit log](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) early.

It is much easier to enable before you need it.

### Kubernetes Events are collected for the workload

**Kubernetes Events explain what the control plane tried to do with your workload.**

They are often the fastest way to understand why a Pod is not running, not Ready, or not being updated.

Events can show:

- Failed scheduling because requests are too high, node selectors do not match, or taints are not tolerated.
- Image pull failures such as `ImagePullBackOff` and `ErrImagePull`.
- Probe failures that remove a Pod from Service endpoints or restart the container.
- Failed volume mounts for ConfigMaps, Secrets, PVCs, or CSI volumes.
- Evictions caused by memory pressure, disk pressure, node drains, or preemption.
- Rollout stalls where new Pods cannot become available.

**The problem is that Events do not last long.**

The API server keeps events for only 1 hour by default, and many managed clusters keep them for even shorter periods.

**If no one checks during the problem, the most useful information might be lost.**

Forward Kubernetes Events to the same place you search logs and metrics.

Common options include [kubernetes-event-exporter](https://github.com/resmoio/kubernetes-event-exporter), cloud-provider event integrations, or an observability agent that already watches the Kubernetes API.

For production workloads, the runbook should say where to find recent Events for the namespace, Deployment, ReplicaSet, Pod, HPA, KEDA's `ScaledObject`, and related Services or Ingresses.

## Recovery

These checks cover how the team reacts when a deployment or infrastructure failure happens.

Decide the recovery posture up front and test Pod, node, and rollout failure paths before production.

### A rollback vs. roll-forward posture has been decided

**If an update fails, you can undo it (rollback) or fix it with a new update (roll-forward).**

Decide which way to handle this before a problem happens, not during it.

Rolling back is fast and safe for simple changes.

You can fix a config mistake or code bug without changing the database by using `kubectl rollout undo` in seconds, or a Git revert if you use GitOps.

Rolling forward is the only option if the broken version did something you cannot undo, such as changing the database, saving data in a new way, or using a message queue.

A few practical notes:

- [`kubectl rollout undo deployment/my-app`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment) walks the Deployment back one revision. Revision history is kept according to `.spec.revisionHistoryLimit`, which defaults to 10.
- A running workload should be traceable back to the source revision that produced it. Use image tags that include the commit SHA, an annotation such as `app.kubernetes.io/version`, or GitOps revision tracking.
- Under GitOps with Argo CD or Flux, rollback is a Git revert, and the cluster converges on its own. This tends to be the easiest model to reason about in practice.

See the [Kubernetes rollback guide](https://learnkube.com/kubernetes-rollbacks) for a deeper walkthrough.

### You know what happens when a Pod crashes or a node dies

In production, Pods can crash, and nodes can fail.

Sometimes a process encounters an unexpected error, a node runs out of memory, or a cloud region experiences issues.

The important thing is whether your system keeps working and if you notice the problem before your customers do.

Most safety measures must be tested with real failures, not just described in manifests.

The app should stop on serious errors and let the kubelet restart it.

The workload should have health checks, a PodDisruptionBudget, and placement rules that spread replicas across nodes and zones.

Scale-down should drain traffic properly.

A few things to verify before going live:

- More than one replica is running, and `topologySpreadConstraints` actually places them on different nodes and, if possible, different zones. A three-replica workload that happens to land on the same node provides no more protection than a single replica.
- The PodDisruptionBudget is tight enough that a node drain cannot take the workload below its minimum, and loose enough that a drain can still make progress.
- When a Pod is `OOMKilled` or exits non-zero, Kubernetes restarts it, and the restart is visible in metrics and logs. A silent crash loop is the worst kind of incident.
- When a whole node is lost, the Pods that were on it are rescheduled within a few minutes. The readiness probe controls when they start receiving traffic again; graceful-shutdown behavior controls what happens to in-flight requests during the failure.

**The best way to be sure is to test failures in a non-production setup.**

Try deleting a pod while it is busy, draining a node, or blocking a zone.

If your system keeps working, the recovery path is ready. If not, fix the weakest assumption and test the failure again.

## Runbooks and cost

These checks cover the operating loop after launch.

The team should know how to troubleshoot common failures and revisit resource sizing once real production usage and cost data exist.

### A troubleshooting runbook exists

If a Pod enters `CrashLoopBackOff`, the on-call person should follow the runbook: identify the error, review the logs, and apply the documented fixes.

**Do not depend on searching the web for answers: a written process helps you fix problems faster.**

A minimum runbook covers:

- The common Pod states (`Pending`, `CrashLoopBackOff`, `ImagePullBackOff`, `OOMKilled`, `Error`, `Completed`), and what to check for each.
- Where to find logs for the current Pod and for the previous terminated container, using `kubectl logs --previous`.
- How to describe a Pod and where in the output to look first — events at the bottom usually explain the most recent problem, and the centralized event store should preserve them after Kubernetes drops them.
- How to get a shell into a running Pod with `kubectl exec`, and how to debug an image that does not even start with [`kubectl debug`](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/).
- How to roll back or roll forward, depending on the team's recovery posture.
- Escalation: who to call and how.

The [Kubernetes troubleshooting flowchart](https://learnkube.com/troubleshooting-deployments) is a practical starting template and is easier to adapt than to write from scratch.

### Cost has been reviewed, and the workload is right-sized

Running in production does not always mean running efficiently.

**Teams new to Kubernetes often over-provision by setting high requests, high limits, and extra replicas.**

Under-provisioning causes clear problems, but over-provisioning usually only shows up when you get the bill.

After your workload has been running for a week or two, take a look at how it is performing:

- **Compare actual CPU and memory use to requests.** If the use is about 20% of the request, the request is too high, and the workload is paying for capacity it does not need.
- **Compare peak traffic to the number of replicas.** If the lowest replica count is three and the average CPU use is 15%, two replicas with a higher use target probably do the same job.
- **Look at node use across the cluster.** When it stays under 50% most of the time, something is usually using more resources than needed, either in the workload or the node group.

A few tools help.

**The [Kubernetes instance calculator](https://learnkube.com/kubernetes-instance-calculator) is useful for sizing nodes to workloads.**

VPA in `Off` mode gives continuous right-sizing recommendations without acting on them.

FinOps tools such as [OpenCost](https://www.opencost.io/), or the cloud vendor's cost explorer, read Kubernetes metrics and turn them into dollar figures.
