# Cluster configuration

Cluster configuration best practices.

## Approved Kubernetes configuration

Kubernetes is flexible and can be configured in several different ways.

But how do you know what's the recommended configuration for your cluster?

The best option is to compare your cluster with a standard reference.

In the case of Kubernetes, the reference is the Centre for Internet Security (CIS) benchmark.

### The cluster passes the CIS benchmark

The Center for Internet Security provides several guidelines and benchmark tests for best practices in securing your code.

They also maintain a benchmark for Kubernetes which you can [download from the official website](https://www.cisecurity.org/benchmark/kubernetes/).

While you can read the lengthy guide and manually check if your cluster is compliant, an easier way is to download and execute [`kube-bench`](https://github.com/aquasecurity/kube-bench).

[`kube-bench`](https://github.com/aquasecurity/kube-bench) is a tool designed to automate the CIS Kubernetes benchmark and report on misconfigurations in your cluster.

Example output:

```terminal|title=bash
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
[WARN] 1.1.1 Ensure that the --anonymous-auth argument is set to false (Not Scored)
[PASS] 1.1.2 Ensure that the --basic-auth-file argument is not set (Scored)
[PASS] 1.1.3 Ensure that the --insecure-allow-any-token argument is not set (Not Scored)
[PASS] 1.1.4 Ensure that the --kubelet-https argument is set to true (Scored)
[PASS] 1.1.5 Ensure that the --insecure-bind-address argument is not set (Scored)
[PASS] 1.1.6 Ensure that the --insecure-port argument is set to 0 (Scored)
[PASS] 1.1.7 Ensure that the --secure-port argument is not set to 0 (Scored)
[FAIL] 1.1.8 Ensure that the --profiling argument is set to false (Scored)
```

> Please note that it is not possible to inspect the master nodes of managed clusters such as GKE, EKS and AKS, using `kube-bench`. The master nodes are controlled and managed by the cloud provider.

### Disable metadata cloud providers metada API

Cloud platforms (AWS, Azure, GCE, etc.) often expose metadata services locally to instances.

By default, these APIs are accessible by pods running on an instance and can contain cloud credentials for that node, or provisioning data such as kubelet credentials.

These credentials can be used to escalate within the cluster or to other cloud services under the same account.

### Restrict access to alpha or beta features

Alpha and beta Kubernetes features are in active development and may have limitations or bugs that result in security vulnerabilities.

Always assess the value an alpha or beta feature may provide against the possible risk to your security posture.

When in doubt, disable features you do not use.

## Authentication

When you use `kubectl`, you authenticate yourself against the kube-api server component.

Kubernetes supports different authentication strategies:

- **Static Tokens**: are difficult to invalidate and should be avoided
- **Bootstrap Tokens**: same as static tokens above
- **Basic Authentication** transmits credentials over the network in cleartext
- **X509 client certs** requires renewing and redistributing client certs regularly
- **Service Account Tokens** are the preferred authentication strategy for applications and workloads running in the cluster
- **OpenID Connect (OIDC) Tokens**: best authentication strategy for end-users as OIDC integrates with your identity provider such as AD, AWS IAM, GCP IAM, etc.

You can learn about the strategies in more detail [in the official documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).

### Use OpenID (OIDC) tokens as a user authentication strategy

Kubernetes supports various authentication methods, including OpenID Connect (OIDC).

OpenID Connect allows single sign-on (SSO) such as your Google Identity to connect to a Kubernetes cluster and other development tools.

You don't need to remember or manage credentials separately.

You could have several clusters connect to the same OpenID provider.

You can [learn more about the OpenID connect in Kubernetes](https://thenewstack.io/kubernetes-single-sign-one-less-identity/) in this article.

## Role-Based Access Control (RBAC)

Role-Based Access Control (RBAC) allows you to define policies on how to access resources in your cluster.

### ServiceAccount tokens are for applications and controllers **only**

Service Account Tokens should not be used for end-users trying to interact with Kubernetes clusters, but they are the preferred authentication strategy for applications and workloads running on Kubernetes.

## Logging setup

You should collect and centrally store logs from all the workloads running in the cluster and from the cluster components themselves.

### There's a retention and archival strategy for logs

You should retain 30-45 days of historical logs.

### Logs are collected from Nodes, Control Plane, Auditing

What to collect logs from:

- Nodes (kubelet, container runtime)
- Control plane (API server, scheduler, controller manager)
- Kubernetes auditing (all requests to the API server)

What you should collect:

- Application name. Retrieved from metadata labels.
- Application instance. Retrieved from metadata labels.
- Application version. Retrieved from metadata labels.
- Cluster ID. Retrieved from Kubernetes cluster.
- Container name. Retrieved from Kubernetes API.
- Cluster node running this container. Retrieved from Kubernetes cluster.
- Pod name running the container. Retrieved from Kubernetes cluster.
- The namespace. Retrieved from Kubernetes cluster.

### Prefer a daemon on each node to collect the logs instead of sidecars

Applications should log to stdout rather than to files.

[A daemon on each node can collect the logs from the container runtime](https://rclayton.silvrback.com/container-services-logging-with-docker#effective-logging-infrastructure) (if logging to files, a sidecar container for each pod might be necessary).

### Provision a log aggregation tool

Use a log aggregation tool such as EFK stack (Elasticsearch, Fluentd, Kibana), DataDog, Sumo Logic, Sysdig, GCP Stackdriver, Azure Monitor, AWS CloudWatch.
