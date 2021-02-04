# Kubernetes Best Practices

O'Reilly, 2019.

## Chapter 1. Setting Up a Basic Service

### Managing resource manifests

_Resource manifests are the declarative state of an application._

- Store resource manifests in Git (version control)
- Manage Git repository on GitHub (code review)
- Use a single top-level directory for all manifests of an application
- Use subdirectories for subcomponents (e.g. services) of the application
- Use GitOps: ensure that contents of cluster match content of Git repository
  - Deploy to production only from a specific Git branch using some automation (e.g. a CI/CD pipeline)
- Use CI/CD from the beginning (difficult to retrofit into an existing application)

### Managing container images

- Base images on well-known and trusted image providers
  - Alternatively, build all images "from scratch" (e.g. with Go)
- The tag of an image is immutable (i.e. images with different content have different tags
  - Combine semantic versioning with the hash of the corrsponding Git commit (e.g. _v1.0.1-bfeda01f_)
- Avoid the _latest_ tag (is not immutable)

### Deploying applications

- Set resource requests equal to limits: provides predictability at the expense of maximising the resource utilisation (the app can't make use of excess idle resources)
  - Set requests and limits to different values only when you have more experience
- Always use an Ingress for exposing services, even for simple applications (for production, you don't need to use an Ingress for experimentation)
- Manage configuration data that is likely to be updated during runtime separately from the code in a ConfigMap
- Put a version number in the name of each ConfigMap (e.g. `myconfig-v1`). When you make an update to the configuration, create a new ConfigMap with an increased version number (e.g. `myconfig-v2`), and then update the application to use the new ConfigMap. This ensure that the new configuration is loaded into the application, no matter how the ConfigMap is mounted in the app (as an env var or a file, and in the latter case, if the app watches the file for changes)
  - Don't delete the previous ConfigMap (e.g. `myconfig-v1`). This allows to roll back to a previous configuration at any time.
- If deploying an application to multiple environments, use a template system (don't maintain multiple copies of resource manifest directories)
  - Helm, kustomize, Kapitan, ...

## Chapter 2. Developer Workflows

_Creating a development cluster (where developers can deploy and test the applications they are working on)._

- One cluster per organisation or team (10-20 people)
- One namespace for each developer
- Use an external identity systems for cluster user management (Azure Active Directory, AWS IAM)
  - Authentication with bearer token and API server validates token with the external service
- Grant developers the `edit` ClusterRole for their namespace with a RoleBinding (not ClusterRoleBinding)
- Grant developers the `view` ClusterRole for the entire cluster with a ClusterRoleBinding (not RoleBinding)
- Assign a ResourceQuota to each namespace
- Make developer namespaces transient by assigning them a time to live (TTL) after which they are automatically deleted
  - So that no unused resources accumulate in the cluster
- Assign the following annotations to each namespace: TTL, assignee, resource quotas, team, purpose
  - You could define a CRD that creates a namespace with all this metadata

## Chapter 3. Monitoring and Logging in Kubernetes

### Monitoring

- Run monitoring system in a dedicated "utility cluster" (to avoid problems with the target cluster affecting the monitoring system)

### Logging

_Collect and centrally store logs from all the workloads running in the cluster and from the cluster components themselves._

- Implement a retention and archival strategy for logs (retain 30-45 days of historical logs)
- What to collect logs from:
  - Nodes (kubelet, container runtime)
  - Control plane (API server, scheduler, controller mananger)
  - Kubernetes auditing (all requests to the API server)
- Applications should log to stdout rather than to files
  - Allows a daemon on each node to collect the logs from the container runtime (if logging to files, a sidecar container for each pod might be necessary)
- Some log aggregation tools: EFK stack (Elasticsearch, Fluentd, Kibana), DataDog, Sumo Logic, Sysdig, GCP Stackdriver, Azure Monitor, AWS CloudWatch
- Use a hosted logging solution (e.g. DataDog, Stackdriver) rather than a self-hosted one (e.g. EFK stack)

### Alerting

- Only alert on events that affect service level objectives (SLO)
- Only alert on events that require immediate human intervention
- Automate remediation of events that don't require immediate human intervention
- Include relevant information in the alert notification (e.g. link to troubleshooting playbook, context information)

## Chapter 4. Configuration, Secrets, and RBAC

### ConfigMaps and Secrets

- Use ConfigMaps and Secrets to inject configuration into pods
- PodPresets: automatically mount a ConfigMap or Secret to a pod based on annotations
- In the application, watch the configuration file for changes, so that the configuration can be changed at runtime by updating the ConfigMap or Secret
- When using values from a ConfigMap/Secret as environment variables, the environment variables in the containers are NOT updated when updating the ConfigMap/Secret
- Use CI/CD pipeline that restarts pods whenever a ConfigMap/Secret is updated (this ensures that the new data is being used by the pods, even if the application does not watch the configuration file for changes or if the configuration data is mounted as environment variables)
  - Alternatively, include a version name in the name of the ConfigMap and when configuration changes, create a new ConfigMap and update applications to use the new ConfigMap (see Chapter 1).
- Always mount Secrets as volumes (files), never as env vars
- Avoid stateful applications in Kubernetes
  - Use SaaS/cloud service offerings for stateful services
  - If running on premises and public SaaS is not an option, have a dedicated team that provides internal stateful SaaS to the rest of the organisation

### RBAC

- Use specific service accounts for all "users" of the Kubernetes API that are assigned tailored roles with the least amount of privileges to do the job

## Chapter 5. Continuous Integration, Testing, and Deployment

_Common steps of a CI/CD pipeline: (1) push code to Git repository, (2) build entire application code, (3) running tests against the built code, (4) building the container images, (5) push the container images to a container registry, (6) deploy the application to Kubernetes (use one of various deployment strategies, such as rolling update, blue/green deployment, canary deployment, or A/B deployment), (7) run tests against the deployed application (e.g. a chaos experiment)_

- Keep production code in the master branch
- Keep container images sizes small (use scratch images with multistage builds, distroless base images, or optimised base images, e.g. Alpine, Debian Slim)
- Use an image tagging strategy: each image that is built by the CI system should have a unique tag (image tags should be immutable, that is, if two images have differing content, they can't have the same tag, see Chapter 1)
  - Use the build ID as part of the tag
  - Use the Git commit hash as part of the tag
- Minimise CI build times
- Include extensive tests in CI (build should fail if any test fails)
- Set up extensive monitoring in the production environment

## Chapter 6. Versioning, Releases, and Rollouts

_The true declarative nature of Kubernetes really shines when planning the proper use of labels._

_By properly identifying the operational and development states by the means of labels in the resource manifests, it becomes possible to tie in tooling and automation to more easily manage the complex processes of upgrades, rollouts, and rollbacks._

- _Version: increments when the code specification changes_
- _Release: increments when the applicatoin is (re)-deployed (even if it's the same version of the app)_
- _Rollout: how a replicated app is put into production (this is taken care of automatically by the Deployment resource when there are changes to the `deployment.spec.template` field)_
- _Rollback: revert an application to the state of a previous release_

Best practices:

- Label each resource with at least: `app`, `version`, `environment`
  - Pods can additionally be labelled with `tier` and top-level objects like Deployments or Jobs should be labelled with `release` and `release-number`
- Use independent versions for container images, Pods, and Deployments
  - E.g. if Pod specification changes, update only the Pod and Deployment version, but not the container image version
- Use a `release` (e.g. `frontend-stable`, `frontend-alpha`) and `release-number` (e.g. `34e57f01`) label for top-level objects (e.g. Deployment, StatefulSet, Job)
  - If the same version of the app is deployed again, it results in a new release number
  - The release number is created by the CI/CD tool that deploys the application

_Compare this with the officially [recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) in the Kubernetes documentation._

- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`
- `app.kubernetes.io/version`
- `app.kubernetes.io/component`
- `app.kubernetes.io/part-of`
- `app.kubernetes.io/managed-by`

## Chapter 7. Worldwide Application Distribution and Staging

_Deploying app in multiple regions around the world (for scaling, reduced latency, etc.)._

_Distributing container images, load balancing, canary regions, testing..._

## Chapter 8. Resource Management

### Advanced scheduling

- Pod affinity
- Pod anti-affinity
- Node selector
- Taints and tolerations

### Pod resource management

- Resource requests
- Resource limits
- Quality of Service (automatically determined by values for requests and limits)
- PodDisruptionBudget
- ResourceQuota
- LimitRange
- Cluster autoscaler
- Horizontal pod autoscaler
- Vertical pod autoscaler

## Chapter 9. Networking, Network Security, and Service Mesh

### Services and Ingresses

_Service:_

- ClusterIP (headless service has no label selector but an explicitly assigned Endpoint; is not managed by kube-proxy; has no ClusterIP address but creates a DNS entry for every Pod in the Endpoint)
- NodePort
- ExternalName
- LoadBalancer

_Ingress:_

Provides HTTP application-level routing in contrast to level 3/4 of services.

Ingress controller enables the use of Ingress resources (all of them are third-party)jjj

- All services that don't need to be accessed from outside the cluster should be ClusterIP
- Use Ingress for external-facing HTTP services and choose appropriate ingress controller

### NetworkPolicy

_Defines how pods within the cluster are allowed to communicate with each other._

- _Requires a CNI that suppports NetworkPolicy (Calico, Cilium, Weave Net)_
- Start with restricting ingress, then restrict egress if needed
- Create a deny-all policy in all namespaces
- Try to restrict inter-pod communication to within namespaces (avoid cross-namespace communication)

### Service meshes

_Manage traffic between services of an application (or multiple applications)._

- Most probably only needed for large deployments with hundreds of services and thousands of endpoints

## Chapter 10. Pod and Container Security

### PodSecurityPolicy

_Centrally enforce security-sensitive fields in pod specifications._

_Many fields of PodSecurityPolicy match those of securityContext in the Pod specifications._

- Using PodSecurityPolicies requires the PodSecurityPolicy admission controller to be enabled, but in most Kubernetes deployments it is not enabled. As soon as the PodSecurityPolicy admission controller is enabled, you need appropriate PodSecurityPolicy resources to allow any Pods to be created.
  - You also need to grant "use" access to the created PodSecurityPolicies to the service account of the workload or the controller of the workload (you can use  `system:serviceaccounts` group which compromises all controller service accounts).
- Use <https://github.com/sysdiglabs/kube-psp-advisor> to generate PodSecurityPolicies automatically based on exisiting Pods

### RuntimeClass

_Allow to specify which container runtime to use for a Pod (if there are multiple ones configured) based on the amount of isolation between containers that is required for this pod._

- Set the `runtimeClassName` field in the pod specification
- Only use it if you have workloads that require different amounts of workload isolation on the host (for security or compliance)

### Other

- Use DenyExecOnPrivileged or DenyEscalatingExec admission controllers as an easier alternative to PodSecurityPolicies -> **however this is not a best practice as these are deprecated and it is recommended to use PodSecurityPolicies**
- Use Falco to enforce security policies within the container runtime

## Chapter 11. Policy and Governance for Your Cluster

Explanation:

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

## Chapter 12. Managing Multiple Clusters

_How to manange multiple clusters, making application in different clusters interact with each other, deploying applications to multiple clusters at once, Kubernetes Federation..._

## Chapter 13. Integrating External Services and Kubernetes

- _Application in Kubernetes consuming a service from outside the cluster_
- _Application outside the cluster consuming a service in Kubernetes_
- _Application in Kubernetes consuming a service in another Kubernetes cluster_

## Chapter 14. Running Machine Learning in Kubernetes

_Apparently, Kubernetes is "perfect environment toenable the machine learning workflow and lifecycle"._

## Chapter 15. Building Higher-Level Application Patterns on Top of Kubernetes

_Develop higher-level abstractions in order to provide more developer-friendly primitives on topof Kubernetes._

## Chapter 16. Managing State and Stateful Applications

### Basic volumes

_Mounting directories from the host into containers._

- Use `emptyDir` for sharing data between containers in the same pod
- Use `hostPath` if the data needs to be accessed also by agents running on the node

### Storage managed by Kubernetes

_Kubernetes support for managing persistent storage._

- _PersistentVolume: a "disk" that exists independently from any nodes in the cluster and has its own Kubernetes resource_
- _PersistentVolumeClaim: a request for a PersistentVolume referenced from a Pod spec. This exists to prevent that specific PersistentVolumes must be referenced from a Pod spec (making the Pod spec non-portable) by referencing the generic PersistentVolumeClaim instead._
- _StorageClass: defines a provisioner to create the disk backing a PersistentVolume to automate the creation of PersistentVolumes. A StorageClass name is referenced from PersistenVolumeClaim._
- _Default StorageClass: used by any PersistentVolumeClaim that doesn't explicitly define a StorageClass name. Requires the [DefaultStorageClass](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) admission controller to be enabled._

Best practices:

- Avoid managing state in the cluster if you can: use an outside service for persisting state
  - Even if it involves modifying the app to become stateless
- Define a default StorageClass named `default` (because this is often used by default in Helm charts)
- If cluster is distributed across multiple availability zones, ensure that PersistentVolumes and Pod using them are in the same availability zone
  - By properly labelling all objects and using node affinity, etc.

### Running stateful applications

- Check if an operator exists for the type of application, and if yes, use it

## Chapter 17. Admission Control and Authorization

### Admission control

- Recommended set of admission controllers to enabled: `NamespaceLifecycle, LimitRanger, ServiceAccount, DefaultStorageClass, DefaultTolerationSeconds, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, Priority, ResourceQuota, PodSecurityPolicy`
- If you use multiple mutating admission control webhooks, don't modify the same fields of the same resources (the order in which admission control webhook are called is undefined)
- If you use mutating admission webhooks, also create a validating admission webhook that verifies that the resources have been modified in the way you expected
- Define the least amount of requests to be sent to an admission webhook (avoid `resources: [*]`, etc.)
- Always use the `namespaceSelector` field in MutatingWebhookConfiguration/ValidatingWebhookConfiguration, which causes the admission control webhook to be only applied in certain namespaces. Select the least amount of namespaces that are necessary.
- Always exclude the `kube-system` namespace from the scope of an admisson control webhook by the means of the `namespaceSelector` field
- Don't give anyone RBAC rules to create MutatingWebhookConfiguration/ValidatingWebhookConfiguration unless it's really needed
- Don't send Secret resources to an admission control webhook if it's not needed (scope the requests that are passed to the webhook to the bare minimum)

### Authorization

- Only use the default RBAC mode (there's also ABAC and webhook, but don't use them)
- For RBAC best practices, see Chapter 4

## Table of contents

- [1. Setting Up a Basic Service](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#setting_up_a_basic_service)
  1. [Application Overview](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561250104)
  1. [Managing Configuration Files](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561246824)
  1. [Creating a Replicated Service Using Deployments](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561241144)
     1. [Best Practices for Image Management](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561237176)
     1. [Creating a Replicated Application](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561242120)
  1. [Setting Up an External Ingress for HTTP Traffic](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561243528)
  1. [Configuring an Application with ConfigMaps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561197272)
  1. [Managing Authentication with Secrets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561203320)
  1. [Deploying a Simple Stateful Database](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561204264)
  1. [Creating a TCP Load Balancer by Using Service](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561173288)
  1. [Using Ingress to Route Traffic to a Static File Server](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561167912)
  1. [Parameterizing Your Application by Using Helm](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116560979624)
  1. [Deploying Services Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116560979048)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116560953208)
- [2. Developer Workflows](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#developer_workflows)
  1. [Goals](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560933368)
  1. [Building a Development Cluster](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560937720)
  1. [Setting Up a Shared Cluster for Multiple Developers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560926536)
     1. [Onboarding Users](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560944488)
     1. [Creating and Securing a Namespace](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560934536)
     1. [Managing Namespaces](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560895832)
     1. [Cluster-Level Services](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560895576)
  1. [Enabling Developer Workflows](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560904456)
  1. [Initial Setup](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560891160)
  1. [Enabling Active Development](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560881816)
  1. [Enabling Testing and Debugging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560881560)
  1. [Setting Up a Development Environment Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560869960)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560857912)
- [3. Monitoring and Logging in Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#monitoring_and_logging_in_kubernetes)
  1. [Metrics versus Logs](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560853272)
  1. [Monitoring Techniques](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560844312)
  1. [Monitoring Patterns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560837624)
  1. [Kubernetes Metrics Overview](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560837048)
     1. [cAdvisor](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560817480)
     1. [Metrics Server](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560823304)
     1. [Kube-State-Metrics](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560816504)
  1. [What Metrics Do I Monitor?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560793912)
  1. [Monitoring Tools](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560793608)
  1. [Monitoring Kubernetes by Using Prometheus](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560770200)
  1. [Logging Overview](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560747672)
  1. [Tools for Logging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560703128)
  1. [Logging by Using an EFK Stack](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560673528)
  1. [Alerting](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560671768)
  1. [Best Practices for Monitoring, Logging, and Alerting](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560628088)
     1. [Monitoring](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560624744)
     1. [Logging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560635928)
     1. [Alerting](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560631672)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560616104)
- [4. Configuration, Secrets, and RBAC](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#configuration_secrets_and_rbac)
  1. [Configuration through ConfigMaps and Secrets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560611272)
     1. [ConfigMaps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560599272)
     1. [Secrets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560602904)
  1. [RBAC](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560540264)
     1. [RBAC Primer](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560526056)
     1. [RBAC Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560515272)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560499352)
- [5. Continuous Integration, Testing, and Deployment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#continuous_integration_testing_and_deployment)
  1. [Version Control](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560474808)
  1. [Continuous Integration](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560473352)
  1. [Testing](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560471640)
  1. [Container Builds](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560459496)
  1. [Container Image Tagging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560451208)
  1. [Continuous Deployment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560452440)
  1. [Deployment Strategies](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560434776)
  1. [Testing in Production](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560401128)
  1. [Setting Up a Pipeline and Performing a Chaos Experiment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560433704)
     1. [Setting Up CI](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560374392)
     1. [Setting Up CD](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560378168)
     1. [Performing a Rolling Upgrade](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560341736)
     1. [A Simple Chaos Experiment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560336632)
  1. [Best Practices for CI/CD](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560323736)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560313288)
- [6. Versioning, Releases, and Rollouts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#versioning_releases_and_rollouts)
  1. [Versioning](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560292760)
  1. [Releases](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560308088)
  1. [Rollouts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560301944)
  1. [Putting It All Together](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560280584)
     1. [Best Practices for Versioning, Releases, and Rollouts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560299720)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560295256)
- [7. Worldwide Application Distribution and Staging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#worldwide_application_distribution_and_staging)
  1. [Distributing Your Image](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560261480)
  1. [Parameterizing Your Deployment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560248616)
  1. [Load Balancing Traffic Around the World](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560242456)
  1. [Reliably Rolling Out Software Around the World](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560238584)
     1. [Pre-Rollout Validation](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560222216)
     1. [Canary Region](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560209464)
     1. [Identifying Region Types](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560214664)
     1. [Constructing a Global Rollout](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560203320)
  1. [When Something Goes Wrong](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560240056)
  1. [Worldwide Rollout Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560239752)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560206392)
- [8. Resource Management](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#resource_management)
  1. [Kubernetes Scheduler](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560193816)
     1. [Predicates](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560181704)
     1. [Priorities](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560186200)
  1. [Advanced Scheduling Techniques](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560175928)
     1. [Pod Affinity and Anti-Affinity](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560177992)
     1. [nodeSelector](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560172360)
     1. [Taints and Tolerations](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560162200)
  1. [Pod Resource Management](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560179080)
     1. [Resource Request](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560134984)
     1. [Resource Limits and Pod Quality of Service](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560134408)
     1. [Pod Disruption Budgets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560129736)
     1. [Managing Resources by Using Namespaces](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560096152)
     1. [ResourceQuota](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560093512)
     1. [LimitRange](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560077128)
     1. [Cluster Scaling](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560047784)
     1. [Application Scaling](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560031400)
     1. [HPA with Custom Metrics](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560028392)
     1. [HPA with Custom Metrics](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560015352)
  1. [Resource Management Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560142104)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116559974184)
- [9. Networking, Network Security, and Service Mesh](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#networking_network_security_and_service_mesh)
  1. [Kubernetes Network Principles](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559966824)
  1. [Network Plug-ins](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559961016)
     1. [Kubenet](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559949016)
  1. [Kubenet Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559947592)
     1. [The CNI Plug-in](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559943624)
  1. [CNI Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559953512)
  1. [Services in Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559921624)
     1. [Service Type ClusterIP](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559916888)
     1. [Service Type NodePort](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559911912)
     1. [Service Type ExternalName](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559929032)
     1. [Service Type LoadBalancer](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559924568)
     1. [Ingress and Ingress Controllers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559901736)
  1. [Services and Ingress Controllers Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559892984)
  1. [Network Security Policy](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559881640)
     1. [Network Policy Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559861992)
  1. [Service Meshes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559877352)
  1. [Service Meshe Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559834840)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559826904)
- [10. Pod and Container Security](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#pod_and_container_security)
  1. [Pod Security Policies](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559815192)
     1. [Enabling PodSecurityPolicy](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559832184)
     1. [Anatomy of a PodSecurityPolicy](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559833528)
     1. [PodSecurityPolicy challenges](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559797704)
  1. [PodSecurityPolicy Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559739752)
  1. [Pod Security Policy Next Steps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559728008)
  1. [Workload Isolation and RuntimeClass](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559716552)
     1. [Using the RuntimeClass](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559707480)
     1. [Runtime Implementations](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559704744)
  1. [Workload Isolation and RuntimeClass Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559729672)
  1. [Other Pod and Container Security Considerations](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559687128)
     1. [Admission Controllers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559681304)
     1. [Intrusion and Anomaly Detection Tooling](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559682856)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559677992)
- [11. Policy and Governance for Your Cluster](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#policy_and_governance_for_your_cluster)
  1. [Why Policy and Governance Is Important](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559674616)
  1. [How Is This Policy Different?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559662808)
  1. [Cloud-Native Policy Engine](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559661528)
  1. [Introducing Gatekeeper](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559671928)
  1. [Example Policies](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559670312)
  1. [Gatekeeper Terminology](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559651288)
     1. [Constraint](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559647656)
     1. [Rego](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559639944)
     1. [Constraint Template](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559638552)
  1. [Defining Constraint Templates](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559644296)
  1. [Defining Constraints](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559629816)
  1. [Data Replication](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559610936)
  1. [UX](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559607128)
  1. [Audit](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559600456)
  1. [Becoming Familiar with Gatekeeper](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559595192)
  1. [Gatekeeper Next Steps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559587256)
  1. [Policy and Governance Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559583208)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559574696)
- [12. Managing Multiple Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#managing_multiple_clusters)
  1. [Why Multiple Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559565848)
  1. [Multicluster Design Concerns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559565432)
  1. [Managing Multiple Cluster Deployments](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559539240)
     1. [Deployment and Management Patterns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559523784)
  1. [The GitOps Approach to Managing Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559538984)
  1. [Multicluster Management Tools](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559505816)
  1. [Kubernetes Federation](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559470712)
  1. [Managing Multiple Clusters Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559470136)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559432024)
- [13. Integrating External Services and Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#integrating_external_services_and_kubernetes)
  1. [Importing Services into Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559434680)
     1. [Selector-Less Services for Stable IP Addresses](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559433048)
     1. [CNAME-Based Services for Stable DNS Names](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559413272)
     1. [Active Controller-Based Approaches](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559402936)
  1. [Exporting Services from Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559395672)
     1. [Exporting Services by Using Internal Load Balancers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559380584)
     1. [Exporting Services on NodePorts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559387944)
     1. [Integrating External Machines and Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559399400)
  1. [Sharing Services Between Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559372280)
  1. [Third-Party Tools](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559370024)
  1. [Connecting Cluster and External Services Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559358280)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559340328)
- [14. Running Machine Learning in Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#running_machine_learning_in_kubernetes)
  1. [Why Is Kubernetes Great for Machine Learning?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559349160)
  1. [Machine Learning Workflow](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559351480)
  1. [Machine Learning for Kubernetes Cluster Admins](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559325096)
  1. [Model Training on Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559317832)
     1. [Training Your First Model on Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559323528)
  1. [Distributed Training on Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559299512)
     1. [Resource Constraints](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559296552)
     1. [Specialized Hardware](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559293624)
     1. [Libraries, Drivers, and Kernel Modules](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559289560)
     1. [Storage](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559274488)
     1. [Networking](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559271336)
     1. [Specialized Protocols](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559283944)
  1. [Data Scientist Concerns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559259240)
  1. [Machine Leaning on Kubernetes Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559255368)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559247176)
- [15. Building Higher-Level Application Patterns on Top of Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#building_higher_level_application_patterns_on_kubernetes)
  1. [Approaches to Developing Higher-Level Abstractions](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559230936)
  1. [Extending Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559224888)
     1. [Extending Kubernetes Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559228840)
     1. [Extending the Kubernetes User Experience](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559228584)
  1. [Design Considerations When Building Platforms](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559204344)
     1. [Support Exporting to a Container Image](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559210232)
     1. [Support Existing Mechanisms for Service and Service Discovery](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559209816)
  1. [Building Application Platforms Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559199784)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559195672)
- [16. Managing State and Stateful Applications](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#managing_state_and_stateful_application)
  1. [Volumes and Volume Mounts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559179528)
     1. [Volume Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559171176)
  1. [Kubernetes Storage](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559166920)
     1. [Persistent Volumes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559176808)
     1. [Persistent Volume Claim](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559162232)
     1. [Storage Classes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559152200)
     1. [Kubernetes Storage Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559141272)
  1. [Stateful Applications](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559166616)
     1. [StatefulSets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559116024)
     1. [Operators](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559100040)
  1. [StatefulSet and Operator Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559132584)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559076840)
- [17. Admission Control and Authorization](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#admission_control_and_authorization)
  1. [Admission Control](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559072168)
     1. [What Are They?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559069192)
     1. [Why Are They Important?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559059240)
     1. [Configuring Admission Webhooks](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559045176)
  1. [Admission Control Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559026088)
  1. [Authorization](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559003592)
     1. [Authorization Modules](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116558998344)
  1. [Authorization Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116558959864)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116558957880)
- [18. Conclusion](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch18.html#conclusion)
