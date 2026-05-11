# Your application

> _Does my app behave well in a container?_

Kubernetes can control your app only if it behaves well in a container.

**Kubernetes might stop a Pod, start a replacement on another node, or increase or decrease the number of replicas at any time.**

Your app should handle these changes smoothly.

This section covers both the application behavior and the container image.

## Application behavior

These checks cover the runtime contract your application must satisfy inside a container: logging, configuration, shutdown, health signals, local state, and connection handling.

If these behaviours are wrong, Kubernetes can still start the Pod, but updates, replacements, and scaling events will be fragile.

### The application logs to `stdout` and `stderr`

There are two main ways to handle logging: passive and active.

In passive logging, the application writes logs to `stdout` and `stderr`.

**The application doesn’t need to know where logs are stored, how they’re processed, or which system collects them.**

Kubernetes reads container logs and displays them using built-in tools such as [`kubectl logs`](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

In this setup, your app only needs to produce logs and the platform handles collecting and delivering them.

As long as your application writes to standard output, the logs can be:

- collected by node-level agents
- enriched with metadata such as pod name and namespace
- correlated with metrics and traces
- forwarded to any logging backend

**Your app does not need to change if the logging system changes.**

This approach also follows the [twelve-factor app principle](https://12factor.net/logs), treating logs as continuous event streams instead of files.

`kubectl logs` is useful for debugging but not for long-term log storage.

In production, ensure logs are collected and stored in a cluster-level logging system managed by the platform.

**Active logging works differently.**

In this model, the application sends logs directly to external systems such as Elasticsearch or third-party services.

This also creates more chances for problems: if the logging system stops working, it could affect your app as well.

Because of this, active logging is usually harder to move between systems and should be avoided unless you have a good reason to use it.

**Make sure your logs are structured.**

This means you should write logs in a clear, consistent format that computers can easily read, rather than plain text.

For example, instead of writing:

```text
User login failed for id 42
```

You write:

```json
{ "event": "login_failed", "user_id": 42 }
```

**Structured logs are easier to search, filter, and analyze.**

They’re also easier to connect with metrics and traces.

For errors, Kubernetes also supports [termination messages](https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/#customizing-the-termination-message), which let you save a short final error summary along with the regular logs.

### Configuration is read from environment variables or files

**Keep settings separate from your app’s code.**

This lets you can change the configuration without rebuilding the app image.

The same app can run in different environments with different settings.

Kubernetes is designed for this model through the [`ConfigMap`](https://kubernetes.io/docs/concepts/configuration/configmap/) API, which lets Pods consume non-confidential configuration as environment variables, command-line arguments, or files in a volume.

Think of a `ConfigMap` as the source of non-sensitive configuration.

**Environment variables and mounted files are the delivery mechanisms.**

The first common delivery mechanism is environment variables.

This method is simple and works well for small values like flags, hostnames, ports, and feature switches.

A `ConfigMap` can populate one variable at a time with `configMapKeyRef`, or many variables at once with `envFrom`.

For example, one key can become one environment variable:

```yaml
env:
  - name: FEATURE_FLAG
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: featureFlag
```

Or every key in the `ConfigMap` can become an environment variable:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

**The second common delivery mechanism is mounted files.**

This is usually better if your app already reads a file format, such as YAML, JSON, TOML, or properties. A `ConfigMap` mounted as a volume exposes each key as a file inside the container.

For example:

```yaml
volumeMounts:
  - name: config
    mountPath: /etc/my-app
    readOnly: true
volumes:
  - name: config
    configMap:
      name: app-config
```

Only keep non-sensitive settings in a `ConfigMap`.

**Use environment variables for simple scalar values.**

Use mounted files when the app expects a config file, when the value is structured, or when you want the app to reload file-based configuration.

If configuration should not change after creation, Kubernetes also supports [immutable ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/#immutable-configmaps).

For sensitive configuration, Kubernetes also provides the [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) object, which can be consumed through environment variables or volume mounts in a similar way.

**Do not treat `ConfigMap` as general-purpose file storage.**

Kubernetes stores objects through the API server and etcd, and a single object such as a `ConfigMap` or `Secret` is limited to 1 MiB when serialized.

If your configuration is getting close to that size, it probably belongs somewhere else.

See [why etcd breaks at scale in Kubernetes](https://learnkube.com/etcd-breaks-at-scale) for the reasoning behind those limits.

### The application handles SIGTERM and shuts down gracefully

**When a Pod is terminated, your application should shut down gracefully rather than exit abruptly.**

Kubernetes starts stopping a Pod by sending a stop signal to the main process in the container and then waits for the set [`terminationGracePeriodSeconds`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) before forcefully stopping any remaining processes.

Your app should handle `SIGTERM` properly.

After receiving `SIGTERM`, your app should stop accepting new requests, finish any ongoing work, properly close long-lasting or keep-alive connections, and exit before the grace period ends.

This matters because traffic might still reach the Pod briefly while Kubernetes is shutting it down.

Kubernetes tracks terminating backends through [`EndpointSlice`](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) conditions such as `ready`, `serving`, and `terminating`.

For some traffic types, Kubernetes can still send traffic to Pods that are stopping to allow smooth connection closing, especially for Services using [`externalTrafficPolicy: Local`](https://kubernetes.io/docs/reference/networking/virtual-ips/#traffic-to-terminating-endpoints).

The right shutdown steps are:

1. Receive `SIGTERM`
1. Stop accepting new requests
1. continue serving in-flight requests for a short drain period
1. Close idle and keep-alive connections cleanly
1. Exit before [`terminationGracePeriodSeconds`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) expires

**[Graceful shutdown](https://learnkube.com/graceful-shutdown) only works if the stop signal reaches the app process.**

Kubernetes sends the stop signal to PID 1 inside the container.

That’s why the container entrypoint should start the app so signals are passed on correctly.

In Dockerfiles, instead of:

```dockerfile
CMD node server.js
```

Whenever you can, use the exec form of `CMD` or `ENTRYPOINT`.

```dockerfile
CMD ["node", "server.js"]
```

If you use a wrapper script, make sure it passes signals correctly and ends by replacing itself with the app process using `exec`.

**Kubernetes also supports a [`preStop`](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks) hook.**

This can help with small, predictable shutdown actions, but it does not replace proper signal handling in your app.

The `preStop` hook runs before the TERM signal is sent, and its time counts against the same shutdown grace period.

Set `terminationGracePeriodSeconds` to give your app enough time to shut down cleanly.

### The application exposes health signals

Kubernetes can’t decide on its own what "healthy" means for your app.

Your app needs to give this information in a way the kubelet can check.

**That is why your app should expose health signals, such as a small HTTP endpoint, a TCP listener, a command that can be run inside the container, or, for gRPC services, an implementation of the [gRPC Health Checking Protocol](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/).**

For most apps, the easiest way is to provide a small HTTP endpoint.

When Kubernetes uses an HTTP probe, it checks the HTTP status code.

Any code from 200 up to, but not including, 400 means success.

The response body doesn't matter, so keep these endpoints simple and focused on returning the correct status.

That is the same pattern Kubernetes uses for its own API server [health endpoints](https://kubernetes.io/docs/reference/using-api/health-checks/), such as `/livez` and `/readyz`, while the older `/healthz` endpoint has been deprecated since Kubernetes v1.16.

For `httpGet` probes, kubelet stops reading the response body after 10 KiB, while probe success is still determined solely by the HTTP status code.

**The health signal should match the decision Kubernetes needs to make.**

- A readiness signal should answer whether this container should receive traffic right now.
- A liveness signal should answer whether the application is stuck and cannot recover on its own.
- A startup signal should answer whether the application has finished initializing.

### The application doesn't store state on the local disk

A container has a local disk you can write to, but this storage is only temporary.

**When the container stops, is replaced, or the Pod moves, data stored only in the container’s disk is not a reliable place for permanent app data.**

Kubernetes calls this [`ephemeral local storage`](https://kubernetes.io/docs/concepts/storage/ephemeral-storage/).

That is why you should not keep app data only on the container’s local disk.

Local storage within the Pod is fine for temporary data such as scratch files, caches, or working data that can be recreated.

Keep permanent data outside the container’s life cycle.

In Kubernetes, that usually means using a [`PersistentVolume`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) through a volume claim, or using an external managed storage system such as a database or object store.

**This is especially important if your app runs multiple copies.**

If each copy keeps its own local data, it will diverge over time, and behavior will change depending on which Pod handles a request.

If the application is truly stateful and requires stable storage or identity, Kubernetes provides the [`StatefulSet`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) controller.

This rule also applies to connections.

When a Pod is replaced, any open TCP connections it had are also lost.

### The application handles long-lived connections correctly

**Some app protocols keep connections open for a long time.**

This is common with gRPC, WebSockets, HTTP keep-alive, HTTP/2, and database connection pools.

In Kubernetes, a [`Service`](https://kubernetes.io/docs/concepts/services-networking/service/) sends traffic to backend Pods, but a long-lasting connection can stay connected to the same Pod until it closes.

This means increasing the number of Pods will not automatically spread out work that is already using existing connections.

Your app should handle [long-lived connections](https://learnkube.com/kubernetes-long-lived-connections) directly.

**Clients should be able to reconnect easily, and servers should close connections smoothly during shutdown.**

This is especially important when a Pod is stopping.

**Kubernetes starts removing the stopped Pod from traffic, but closing connections does not happen right away.**

Endpoint state is shared through [`EndpointSlice`](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/), and some traffic can still reach stopping Pods while connections are closing.

- Clients should handle disconnects and reconnect safely.
- Servers should support graceful connection draining during shutdown.
- Long-lived requests and streams should be allowed to finish when possible before the process exits.

**See [long-lived connections in Kubernetes](https://learnkube.com/kubernetes-long-lived-connections) for a full walkthrough.**

## Container image

These checks cover the image artifact Kubernetes pulls and runs.

The image should be small, predictable, and traceable so production rollouts and rollbacks use exactly the version you intended.

### The container image contains only what is needed to run the application

**Your container image should include only the files, libraries, and tools needed to run the app in real use.**

[A smaller image](https://learnkube.com/blog/smaller-docker-images) is easier to share and usually has less extra software.

The usual way to do this is with a [multi-stage build](https://docs.docker.com/build/building/multi-stage/). Build tools and dependencies stay in earlier stages, and only the final runtime files are copied into the last stage.

The runtime image should not include build tools, package managers, test files, or anything else not needed after the app starts.

### Image tags are stable and `:latest` is avoided

Image references should always be clear and stable.

Kubernetes recommends not using the [`:latest`](https://kubernetes.io/docs/concepts/containers/images/) tag in production.

Using it makes it harder to know which version is running and harder to go back if needed.

Use a clear version tag: if you need a fixed reference, use the full `sha256` digest (for example, `myimage@sha256:abc123...`).
