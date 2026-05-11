# Your security

> _Will this pass review?_

Your workload should be safe to run alongside other workloads and ready for regulated environments.

Everything listed here is required for production.

Security reviews will check these points, and policy engines will block your deployment if you miss any of them.

## Runtime access controls

These checks cover the runtime boundaries around the workload: which Pod security profile applies, which Kubernetes identity the Pod uses, and which network paths are allowed.

They limit what the workload can do if it is misconfigured or compromised.

### Pod Security Standards are enforced at the namespace level

**[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) define three built-in security profiles for Kubernetes workloads,** which are enforced by [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/), which replaced PodSecurityPolicy after PSPs were removed in Kubernetes 1.25.

The three profiles go from permissive to strict:

- **`privileged` places no restrictions on the Pod.** It is useful for system components that genuinely need host access, such as CNI plugins and node exporters.
- **`baseline` blocks known ways to gain extra privileges.** Privileged containers, host namespaces, and host paths are not allowed, and a few risky capabilities are blocked. Most existing applications run under `baseline` without changes.
- **`restricted` is the current strict security profile for pods.** It requires non-root users, a read-only root filesystem, a seccomp profile, and all extra permissions removed. This is the profile most production application namespaces should aim for, matching the least-privilege runtime settings in the workload manifest.

Profiles are turned on by labeling a namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

There are three modes: `enforce`, `audit`, and `warn`, and you can use them together.

It is best to start with `warn` and `audit` set to `restricted` while keeping `enforce` at `baseline`.

Fix any problems found in the audit, then change `enforce` to `restricted`: this helps avoid unexpected deployment blocks in production.

### Each workload has a dedicated ServiceAccount with minimal RBAC

**Kubernetes gives every Pod a [ServiceAccount](https://kubernetes.io/docs/concepts/security/service-accounts/).**

If none is set, the Pod uses the namespace's `default` ServiceAccount, which is shared by everything else in that namespace without its own.

To keep things organized, follow these two rules.

First, every workload should have its own named ServiceAccount, declared in its manifest and owned by that workload.

Second, give that ServiceAccount only the permissions it needs.

Most application Pods do not need to talk to the Kubernetes API at all.

**For those, a ServiceAccount with no RoleBindings is the right answer.**

For Pods that need API access, like operators, controllers, or sidecars that read ConfigMaps, it is safer to start with no permissions and add them one by one until the workload works.

_Starting with wide permissions and planning to reduce them later rarely works well._

See [RBAC in Kubernetes](https://learnkube.com/rbac-kubernetes) for a full walkthrough of roles, bindings, and common patterns.

**Default ServiceAccount tokens are not auto-mounted.**

By default, Kubernetes puts the ServiceAccount's JWT token into every Pod at `/var/run/secrets/kubernetes.io/serviceaccount/`.

**If your workload does not need to call the API server, this token just adds extra risk.**

The mount can be disabled on the ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
automountServiceAccountToken: false
```

It can also be disabled on an individual Pod:

```yaml
spec:
  automountServiceAccountToken: false
```

The official guide on [configuring service accounts for pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) covers the mount behavior in more detail.

### Network access is restricted with NetworkPolicy

By default, Pods can usually communicate with every other Pod in the cluster, and every Pod can usually receive traffic from any other Pod.

This flat network is convenient for development, but it is too open for production.

[NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) lets you limit traffic at the Pod level.

The important production question is not just "do we have a NetworkPolicy?" It is: "Who can connect to this workload, and what can this workload connect to?"

For each workload, define the expected traffic:

- **Ingress**: which namespaces, Pods, or controllers are allowed to send traffic to this workload.
- **Egress**: which in-cluster services, databases, queues, DNS servers, or external IP ranges this workload is allowed to reach.
- **Default deny**: whether the namespace or workload starts closed and only opens the paths it needs.

NetworkPolicies add up.

**Once a Pod is chosen by a policy for incoming or outgoing traffic, traffic in that direction is blocked unless another rule allows it.**

This makes default-deny policies strong, but also easy to break if you forget needed paths like DNS.

Two caveats matter in production:

1. NetworkPolicy only works if the cluster's CNI plugin enforces it.
1. The built-in Kubernetes API works with Pod selectors, namespace selectors, and IP blocks. It does not understand domain names.

## Supply chain and admission control

These checks cover what happens before Kubernetes runs the workload.

Images and manifests should be scanned, trusted, and validated at admission so unsafe deployments are blocked before they start.

### Container images are scanned and pulled from a trusted registry

You should scan every image before it leaves the build process, and keep scanning it regularly while it is in the registry.

**An image with no CVEs today might have some next week, since new security problems are always found.**

Common scanners include [Trivy](https://trivy.dev/), which covers images, filesystems, Git repositories, and Kubernetes resources in a single binary, and [Grype](https://github.com/anchore/grype).

**Cloud-vendor scanners built into GCR, Artifact Registry, ECR, and ACR work well as a backstop.**

Team rules are just as important as the scanning tool.

Decide ahead of time which CVE severity will stop a release, which will create a ticket, and who is responsible for fixing those tickets.

Scanning tells you if an image is safe, but you also need to know if the image comes from a trusted source.

**Production clusters should get images only from registries your team controls or has approved, like a private registry, a copy of trusted sources, or an approved list.**

Admission policy is usually where you enforce this when the workload is accepted.

### Admission policies validate every manifest

RBAC decides who can do what. Pod Security Standards decide what a Pod can do while running. Neither of them can answer questions like:

- Images must come from our private registry.
- Every Deployment must carry an `owner` label.
- Ingress hostnames must not collide.
- Production workloads must reference a signed image digest rather than a tag.

For many of these checks, you do not need an external policy engine.

**Kubernetes has native [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/) resources that evaluate [CEL](https://kubernetes.io/docs/reference/using-api/cel/) expressions during admission.**

CEL policies are a good first choice for object-local validation, such as:

- requiring labels and annotations;
- rejecting images outside approved registries;
- requiring image references to use digests instead of mutable tags;
- enforcing fields such as `runAsNonRoot`, `readOnlyRootFilesystem`, or `resources.requests`;
- restricting `Service` types, `hostPath`, `hostNetwork`, or privileged containers.

**Use native admission policies first when the rule can be expressed from the object being admitted and a small set of parameters.**

General-purpose policy engines are still useful when you need more than native validation, such as mutation, resource generation, image signature verification, external data lookups, complex cross-resource checks, or policy reuse outside Kubernetes.

Two projects lead that space:

- **[Kyverno](https://kyverno.io/) uses policies written in YAML** that look like Kubernetes manifests. There is no separate policy language to learn. Kyverno supports validation, mutation, generation, and image verification from a single tool, making it the lower-friction choice for teams adopting policy-as-code for the first time.
- **[OPA](https://www.openpolicyagent.org/) with [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) uses [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/), a policy language.** Rego is harder to learn, but it is better for complex rules, and the same engine can be used outside Kubernetes for APIs, CI/CD, or Terraform.

For example, CEL can require an image digest, but it cannot verify the image signature by itself.

CEL can check an Ingress hostname format, but a uniqueness check across existing Ingress objects is usually a webhook or policy-engine problem.

> See [Kubernetes policies](https://learnkube.com/kubernetes-policies) for a deeper walkthrough of CEL, Kyverno, OPA, and the admission control model.

## Cloud access and secrets

These checks cover production credentials outside Kubernetes itself.

Workloads should use short-lived cloud identity and retrieve secrets from a dedicated secret manager instead of carrying long-lived keys in the cluster.

### Workloads use workload identity for cloud resources

**If a Pod needs to access S3, a managed database, or any cloud API, do not give it a fixed access key.**

Fixed credentials are easy to leak, hard to change, and cannot be limited to just one Pod.

Workload identity is the right method.

Each cloud has a Kubernetes-aware version that swaps a Pod's ServiceAccount for a short-lived cloud credential, limited to exactly what that workload should be able to do:

- **AWS provides [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)** and, more recently, EKS Pod Identity.
- **GCP provides [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity).**
- **Azure provides [Microsoft Entra Workload ID](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview).**

The Pod gets a token that expires in a few minutes.

It is linked to a specific IAM role and ServiceAccount in a certain namespace.

If the token leaks, the risk only lasts a short time.

> See [authentication in Kubernetes](https://learnkube.com/authentication-kubernetes) for how ServiceAccount tokens, OIDC, and workload identity fit together.

### Secrets live in an external secret store

Kubernetes [`Secret`](https://kubernetes.io/docs/concepts/configuration/secret/) objects are a delivery mechanism for Pods, and not a replacement for a full secret-management system.

The base64 encoding in a Secret is only a serialization format: it exists so arbitrary bytes can be stored safely in YAML and JSON.

The real question is where the secret should live as the source of truth.

**If Kubernetes is the source of truth, the secret lifecycle is tied to the cluster: access control, audit trails, rotation workflows, backups, and replication all have to be solved around Kubernetes and etcd.**

Anyone with `get secrets` permissions in the namespace can read the value through the API, and the value is stored in etcd unless the object is short-lived or mounted from somewhere else.

[Encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) protects stored data, but the API server still decrypts it when authorized clients read it.

For production credentials such as database passwords, API keys, TLS private keys, or signing keys, the usual approach is to keep the source of truth in a dedicated secret manager:

- Store secrets in a dedicated secret manager such as HashiCorp Vault, AWS Secrets Manager, Google Secret Manager, Azure Key Vault, or an equivalent.
- Use the secret manager for ownership, audit logs, access policies, versioning, replication, and rotation.
- Let Kubernetes consume those secrets through a bridge such as [External Secrets Operator](https://external-secrets.io/) or the [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/).

Those two bridges make different trade-offs.

**External Secrets Operator syncs values from the external store into Kubernetes `Secret` objects.**

That works well with existing Pods, controllers, and applications that already expect normal Kubernetes Secrets, but the secret value still exists in the cluster after sync.

**Secrets Store CSI Driver mounts values from the external store into the Pod filesystem.**

It can avoid creating a Kubernetes `Secret` object at all, which is useful when you want the value mounted directly from the external provider.

The workload uses its workload identity to log in to the secret manager.

This means you do not store fixed credentials in Kubernetes or have to change them manually.

**The benefit is that the secret lifecycle stays centralized in the system built to manage secrets, while Kubernetes receives only the values each workload needs.**

These controls define what Kubernetes allows at admission and during runtime.
