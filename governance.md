# Governance

Best practices for creating, managing and administering namespaces.

## Namespace limits

When you decide to segregate your cluster in namespaces, you should protect against misuses in resources.

You shouldn't allow your user to use more resources than what you agreed in advance.

Cluster administrators can set constraints to limit the number of objects or amount of computing resources that are used in your project with quotas and limit ranges.

You should check out the official documentation if you need a refresher on [limit ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)

### Namespaces have LimitRange

Containers without limits can lead to resource contention with other containers and unoptimized consumption of computing resources.

Kubernetes has two features for constraining resource utilisation: ResourceQuota and LimitRange.

With the LimitRange object, you can define default values for resource requests and limits for individual containers inside namespaces.

Any container created inside that namespace, without request and limit values explicitly specified, is assigned the default values.

You should check out the official documentation if you need a refresher on [resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/).

### Namespaces have ResourceQuotas

With ResourceQuotas, you can limit the total resource consumption of all containers inside a Namespace.

Defining a resource quota for a namespace limits the total amount of CPU, memory or storage resources that can be consumed by all containers belonging to that namespace.

You can also set quotas for other Kubernetes objects such as the number of Pods in the current namespace.

If you're thinking that someone could exploit your cluster and create 20000 ConfigMaps, using the LimitRange is how you can prevent that.

## Pod security policies

When a Pod is deployed into the cluster, you should guard against:

- the container being compromised
- the container using resources on the node that are not allowed such as process, network or file system

More in general, you should restrict what the Pod can do to the bare minimum.

### Enable Pod Security Policies

For example, you can use Kubernetes Pod security policies for restricting:

- Access the host process or network namespace
- Running privileged containers
- The user that the container is running as
- Access the host filesystem
- Linux capabilities, Seccomp or SELinux profiles

Choosing the right policy depends on the nature of your cluster.

The following article explains some of the [Kubernetes Pod Security Policy best practices](https://resources.whitesourcesoftware.com/blog-whitesource/kubernetes-pod-security-policy)

### Disable privileged containers

In a Pod, containers can run in "privileged" mode and have almost unrestricted access to resources on the host system.

While there are specific use cases where this level of access is necessary, in general, it's a security risk to let your containers do this.

Valid uses cases for privileged Pods include using hardware on the node such as GPUs.

You can [learn more about security contexts and privileges containers from this article](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).

### Use a read-only filesystem in containers

Running a read-only file system in your containers forces your containers to be immutable.

Not only does this mitigate some old (and risky) practices such as hot patching, but also helps you prevent the risks of malicious processes storing or manipulating data inside a container.

Running containers with a read-only file system might sound straightforward, but it might come with some complexity.

_What if you need to write logs or store files in a temporary folder?_

You can learn about the trade-offs in this article on [running containers securely in production](https://medium.com/@axbaretto/running-docker-containers-securely-in-production-98b8104ef68).

### Prevent containers from running as root

A process running in a container is no different from any other process on the host, except it has a small piece of metadata that declares that it's in a container.

Hence, root in a container is the same root (uid 0) as on the host machine.

If a user manages to break out of an application running as root in a container, they may be able to gain access to the host with the same root user.

Configuring containers to use unprivileged users, is the best way to prevent privilege escalation attacks.

If you wish to learn more, the follow [article offers some detailed explanation examples of what happens when you run your containers as root](https://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b).

### Limit capabilities

Linux capabilities give processes the ability to do some of the many privileged operations only the root user can do by default.

For example, `CAP_CHOWN` allows a process to "make arbitrary changes to file UIDs and GIDs".

Even if your process doesn't run as `root`, there's a chance that a process could use those root-like features by escalating privileges.

In other words, you should enable only the capabilities that you need if you don't want to be compromised.

_But what capabilities should be enabled and why?_

The following two articles dive into the theory and practical best-practices about capabilities in the Linux Kernel:

- [Linux Capabilities: Why They Exist and How They Work](https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work)
- [Linux Capabilities In Practice](https://blog.container-solutions.com/linux-capabilities-in-practice)

### Prevent privilege escalation

You should run your container with privilege escalation turned off to prevent escalating privileges using `setuid` or `setgid` binaries.

## Network policies

A Kubernetes network must adhere to three basic rules:

1. **containers can talk to any other container in the network**, and there's no translation of addresses in the process — i.e. no NAT is involved
1. **nodes in the cluster can talk to any other container in the network and vice-versa**. Even in this case, there's no translation of addresses — i.e. no NAT
1. **a container's IP address is always the same**, independently if seen from another container or itself.

The first rule isn't helping if you plan to segregate your cluster in smaller chunks and have isolation between namespaces.

_Imagine if a user in your cluster were able to use any other service in the cluster._

Now, _imagine if a malicious user in the cluster were to obtain access to the cluster_ — they could make requests to the whole cluster.

To fix that, you can define how Pods should be allowed to communicate in the current namespace and cross-namespace using Network Policies.

### Enable network policies

Kubernetes network policies specify the access permissions for groups of pods, much like security groups in the cloud are used to control access to VM instances.

In other words, it creates firewalls between pods running on a Kubernetes cluster.

If you are not familiar with Network Policies, you can read [Securing Kubernetes Cluster Networking](https://ahmet.im/blog/kubernetes-network-policy/).

### There's a conservative NetworkPolicy in every namespace

This repository contains various use cases of Kubernetes Network Policies and samples YAML files to leverage in your setup. If you ever wondered [how to drop/restrict traffic to applications running on Kubernetes](https://github.com/ahmetb/kubernetes-network-policy-recipes), read on.

## Role-Based Access Control (RBAC) policies

Role-Based Access Control (RBAC) allows you to define policies on how to access resources in your cluster.

It's common practice to give away the least permission needed, _but what is practical and how do you quantify the least privilege?_

Fine-grained policies provide greater security but require more effort to administrate.

Broader grants can give unnecessary API access to service accounts but are easier to controls.

_Should you create a single policy per namespace and share it?_

_Or perhaps it's better to have them on a more granular basis?_

There's no one-size-fits-all approach, and you should judge your requirements case by case.

_But where do you start?_

If you start with a Role with empty rules, you can add all the resources that you need one by one and still be sure that you're not giving away too much.

### Disable auto-mounting of the default ServiceAccount

Please note that [the default ServiceAccount is automatically mounted into the file system of all Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server).

You might want to disable that and provide more granular policies.

### RBAC policies are set to the least amount of privileges necessary

It's challenging to find good advice on how to set up your RBAC rules. In [3 realistic approaches to Kubernetes RBAC](https://thenewstack.io/three-realistic-approaches-to-kubernetes-rbac/), you can find three practical scenarios and practical advice on how to get started.

### RBAC policies are granular and not shared

Zalando has a concise policy to define roles and ServiceAccounts.

First, they describe their requirements:

- Users should be able to deploy, but they shouldn't be allowed to read Secrets for example
- Admins should get full access to all resources
- Applications should not gain write access to the Kubernetes API by default
- It should be possible to write to the Kubernetes API for some uses.

The four requirements translate into five separate Roles:

- ReadOnly
- PowerUser
- Operator
- Controller
- Admin

You can read about [their decision in this link](https://kubernetes-on-aws.readthedocs.io/en/latest/dev-guide/arch/access-control/adr-004-roles-and-service-accounts.html).

## Custom policies

Even if you're able to assign policies in your cluster to resources such as Secrets and Pods, there are some cases where Pod Security Policies (PSPs), Role-Based Access Control (RBAC), and Network Policies fall short.

As an example, you might want to avoid downloading containers from the public internet and prefer to approve those containers first.

Perhaps you have an internal registry, and only the images in this registry can be deployed in your cluster.

_How do you enforce that only **trusted containers** can be deployed in the cluster?_

There's no RBAC policy for that.

Network policies won't work.

_What should you do?_

You could use the [Admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) to vet resources that are submitted to the cluster.

### Allow deploying containers only from known registries

One of the most common custom policies that you might want to consider is to restrict the images that can be deployed in your cluster.

[The following tutorial explains how you can use the Open Policy Agent to restrict not approved images](https://blog.openpolicyagent.org/securing-the-kubernetes-api-with-open-policy-agent-ce93af0552c3#3c6e).

### Enforce uniqueness in Ingress hostnames

When a user creates an Ingress manifest, they can use any hostname in it.

```yaml|highlight=7|title=ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: first.example.com
      http:
        paths:
          - backend:
              serviceName: service
              servicePort: 80
```

However, you might want to prevent users using **the same hostname multiple times** and overriding each other.

The official documentation for the Open Policy Agent has [a tutorial on how to check Ingress resources as part of the validation webhook](https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/#4-define-a-policy-and-load-it-into-opa-via-kubernetes).

### Only use approved domain names in the Ingress hostnames

When a user creates an Ingress manifest, they can use any hostname in it.
a

```yaml|highlight=7|title=ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: first.example.com
      http:
        paths:
          - backend:
              serviceName: service
              servicePort: 80
```

However, you might want to prevent users using **invalid hostnames**.

The official documentation for the Open Policy Agent has [a tutorial on how to check Ingress resources as part of the validation webhook](https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/#4-define-a-policy-and-load-it-into-opa-via-kubernetes).
