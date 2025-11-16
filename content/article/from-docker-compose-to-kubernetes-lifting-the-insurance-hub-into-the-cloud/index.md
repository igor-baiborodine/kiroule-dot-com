---
title: "From Docker Compose to Kubernetes: Lifting the Insurance Hub into the Cloud" 
date: 2025-11-11T17:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "TODO" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ] 

author: "Igor Baiborodine" 
---

In the inaugural [post](/article/from-java-to-go-kicking-off-the-insurance-hub-transformation/) of
the ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/) series, I outlined the
ambitious strategy for modernizing a Java-based insurance system into a cloud-native Go application.
With the roadmap established, the past two months have focused on transitioning from architectural
diagrams to hands-on engineering. This article chronicles the execution
of [Phase 1](https://github.com/igor-baiborodine/insurance-hub/blob/main/docs/system-overview-and-migration-analysis.md#phase-1-foundational-infrastructure--environment-migration-lift-and-shift),
which involves a foundational "lift and shift."

<!--more-->

This phase moves the entire legacy application from
the familiar confines of Docker Compose into a robust, production-like Kubernetes environment. By
establishing parallel clusters—using Kind for local development and K3s for production-like QA—this
critical first step, representing the metaphorical "lift," validates the new platform and lays the
stable groundwork necessary for the upcoming deeper service-by-service "shift" migration.

The scope of this article covers the foundational infrastructure "lift" phase of migrating the
Insurance Hub to Kubernetes, specifically:

- Provisioning local development and QA Kubernetes clusters using Kind and Rancher’s K3s, creating
  production-like environments for testing and validation.
- Deploying core cluster observability tools, including Prometheus and Grafana for metrics and
  monitoring, plus Zipkin for distributed tracing of Insurance Hub services.
- Deploying core infrastructure components essential to the Insurance Hub, such as PostgreSQL,
  MongoDB, Elasticsearch, Apache Kafka, and introducing MinIO as the new S3-compatible object
  storage replacement for the existing filesystem.
- Deploying auxiliary services like JSReport to support the Insurance Hub's PDF generation
  capabilities.

This second installment focuses on establishing the cloud-native platform foundation by replicating
the existing environment within Kubernetes. Targeted Java service modifications and gateway
deployment are part of the next "shift" phase and will be covered later. This lift ensures minimal
disruption while validating stable operation on the Kubernetes infrastructure.

{{< toc >}}

### Phase 1 Planning and Scoping

Before starting Phase 1, a thorough review and adjustment of its scope were essential to ensure a
smooth and effective migration process. This included researching best practices for both local
development and production-like Kubernetes environments. It became clear that observability should
encompass not only the Insurance Hub services but also the underlying infrastructure components,
providing comprehensive monitoring and troubleshooting capabilities. During this review, the
importance of deploying Zipkin for distributed tracing and JSReport for PDF generation was
recognized and added to the scope. Furthermore, implementing GitOps early in the migration was
deemed necessary to maintain consistency and automation, prompting its migration from Phase 6 to
Phase 1.

Once the scope was finalized, the first
epic [ticket](https://github.com/users/igor-baiborodine/projects/8/views/1?pane=issue&itemId=124052445&issue=igor-baiborodine%7Cinsurance-hub%7C5)
for Phase 1 was created in the "Insurance Hub - Go Migration" project, followed by more granular
tickets that were aligned with the scope. To streamline ticket creation, the Junie AI coding agent
was employed to assist in drafting detailed descriptions based on a standardized template. For
managing the workflow, all new tickets initially enter the "Backlog" column and must pass through a
refinement process before being moved to "Ready." This process includes validating ticket descriptions
and acceptance criteria, conducting necessary research—such as evaluating whether to use ArgoCD or
Flux for GitOps—and updating the "Dev Notes" section. This structured approach ensures clear
communication, proper preparation, and prioritization, setting a strong foundation for executing the
lift phase of the migration.

### Provisioning Clusters

Since the Kubernetes cluster forms the foundation and starting point of the entire migration, I
carefully evaluated available tools and options to find a solution that balances resource efficiency
with the capability to handle complex networking, storage, and production-like workloads. The goal
was to host a distributed system consisting of six Java/Go microservices alongside stateful components
and comprehensive observability tools.

Given my extensive experience using [Kind](https://kind.sigs.k8s.io/)-based Kubernetes clusters for
local development at my current job, it was an obvious choice to leverage Kind again for the
Insurance Hub’s local development environment.

For the production-like QA environment, I considered two main options: using a well-known cloud
provider's managed Kubernetes service or setting up a local, fully compliant (or close to fully
compliant) Kubernetes environment. I opted for the latter based on several factors: eliminating
cloud infrastructure costs by utilizing existing hardware that has ample CPU and RAM, gaining
complete administrative control for deep customization, and acquiring hands-on experience managing a
production-like cluster, which deepens Kubernetes expertise.

Initially, I chose [MicroK8s](https://microk8s.io/) for the production-like simulation because it
offers a complete, conformant Kubernetes environment with many built-in addons for common
functionalities such as DNS, ingress, storage, metrics, and monitoring. However, deploying and
configuring MicroK8s proved to be resource-intensive and time-consuming.

After additional research and testing, I switched to Rancher's [K3s](https://k3s.io/). While K3s is
not 100% fully conformant with Kubernetes, it is very close and ideally suited for simulating a
production environment locally without excessive resource consumption or a complex setup. K3s strikes
an excellent balance between providing a full-fledged Kubernetes experience and maintaining
development machine efficiency.

Since [Make](https://www.gnu.org/software/make/) is widely used in Go projects, I chose to leverage
it extensively for automating and reliably reproducing every step involved in creating clusters and
deploying both infrastructure and service components. To maintain consistency and readability while
relying on Makefiles, I added a dedicated style
guide [section](https://github.com/igor-baiborodine/insurance-hub/blob/main/CONTRIBUTING.md#makefile)
to the `CONTRIBUTING.md` file, outlining the best practices for using Make within the project. 

In addition, the [Prerequisites](https://github.com/igor-baiborodine/insurance-hub/blob/main/CONTRIBUTING.md#prerequisites)
section was added to clearly document all required dependencies. This ensures everything is provided
from the developer’s perspective for smooth development and proper operation of the Insurance Hub
within a Kubernetes cluster. Before proceeding with creating clusters, all necessary dependencies
should be installed and configured correctly. To facilitate this process, I created a helper Make
target named [prereq-k8s-all](https://github.com/igor-baiborodine/insurance-hub/blob/947f3e492e50e7efbcfa15762e6d54613be4ff85/k8s/Makefile#L72)
that automates it.

#### Local Dev

Setting up a Kind cluster is straightforward and fast, making it ideal for iterative development of
the Insurance Hub’s six Java/Go microservices and accompanying infrastructure. To make cluster
operations repeatable and error-free, I created
several [Make targets](https://github.com/igor-baiborodine/insurance-hub/blob/947f3e492e50e7efbcfa15762e6d54613be4ff85/k8s/bootstrap/Makefile#L24):

- `local-dev-create` - create and start a single-node Kind cluster
- `local-dev-delete` - stop and remove the cluster and its persistent storage
- `local-dev-suspend` - stop the cluster without deleting it
- `local-dev-resume` - start the suspended cluster again

All steps required to create and manage the Kind cluster are documented in
the ["Local Dev"](https://github.com/igor-baiborodine/insurance-hub/blob/main/k8s/base-cluster-how-tos.md#local-dev)
section of the "Base Cluster How-To's" guide.

#### Production-like QA

To make the QA environment closely reflect a production setup, I used LXD-based virtual machines as
cluster nodes instead of Docker containers. Canonical's [LXD](https://canonical.com/lxd) provides
stronger isolation and better compatibility with production-like environments while remaining
lightweight enough for local development.

My development machine is equipped with 24 logical CPUs (16 physical cores) and 64 GB of RAM. Since
the Kubernetes control plane components—such as the API server, scheduler, controller manager, and
etcd—are resource-intensive, I allocated more CPU and memory to the master node. The resources were
distributed as follows, dedicating half of the available machine’s resources to the future cluster:

- Master LXD VM: 6 CPU, 16 GiB RAM
- Worker LXD VM 1: 3 CPU, 8 GiB RAM
- Worker LXD VM 2: 3 CPU, 8 GiB RAM

Similar to the local development setup, I implemented a suite
of [Make targets](https://github.com/igor-baiborodine/insurance-hub/blob/947f3e492e50e7efbcfa15762e6d54613be4ff85/k8s/bootstrap/Makefile#L65)
to ensure consistent and reliable management of QA cluster operations:

- `qa-nodes-create` – create three LXD VMs (one master, two workers)
- `qa-nodes-suspend` – pause all QA VMs
- `qa-nodes-resume` – resume suspended VMs
- `qa-nodes-snapshot` – create snapshots for backup or rollback
- `qa-nodes-snapshots-list` – list existing snapshots
- `qa-nodes-restore` – restore VMs from a snapshot
- `qa-nodes-delete` – delete all QA VMs and snapshots

These targets wrap underlying `lxc` commands and scripts to automate the lifecycle of QA cluster
nodes. Because provisioning and deploying QA environments can be time-consuming, the
`qa-nodes-snapshot` target is particularly valuable for quick backups, rollbacks, and recovery.

Before deploying the K3s cluster on these VMs, the host system’s firewall and network routing must
be properly configured. Kubernetes networking depends on the host’s ability to forward packets
between interfaces for pod-to-pod and external communication. Many Linux distributions default to
dropping forwarded packets, which blocks this traffic. Setting the `iptables` `FORWARD` chain policy
to `ACCEPT` resolves this issue. Detailed instructions are provided in
the [“Prerequisites”](https://github.com/igor-baiborodine/insurance-hub/blob/main/k8s/base-cluster-how-tos.md#prerequisite)
section of the “Base Cluster How-To’s” guide.

Following the creation of VMs, the K3s-based cluster can be deployed with the `qa-cluster-create` and
`qa-cluster-pull-kubeconfig` [Make targets](https://github.com/igor-baiborodine/insurance-hub/blob/947f3e492e50e7efbcfa15762e6d54613be4ff85/k8s/bootstrap/Makefile#L50).
The cluster includes essential addons such as DNS, ingress, and storage by default. The second
target updates the `kubeconfig` file with the master’s IP address, adjusts context names for
clarity, and merges the configuration into the host’s default `kubeconfig`, enabling direct local
access to the QA cluster.

All steps required to create and manage the QA cluster are described in
the [“QA”](https://github.com/igor-baiborodine/insurance-hub/blob/main/k8s/base-cluster-how-tos.md#qa)
section of the “Base Cluster How-To’s” guide. The cluster has been tested to ensure it’s fully
functional and ready to host workloads that require persistent storage and ingress routing from the
outset. Detailed testing notes can be found in the ticket titled [“Phase 1: [1A] Provision local dev and QA Kubernetes clusters”](https://github.com/users/igor-baiborodine/projects/8/views/1?pane=issue&itemId=124053589&issue=igor-baiborodine%7Cinsurance-hub%7C6).

### Kubernetes Deployment Strategy & Best Practices

The recommended deployment approach utilizes Kubernetes Operators in conjunction with Kustomize overlays for all infrastructure components, including PostgreSQL, MongoDB, Kafka, Elasticsearch, and MinIO. Operators manage the lifecycle and configuration of these stateful services, while Kustomize enables modular overlays and environment-specific customization. In contrast, Java and Go microservices are deployed using Kustomize alone, which allows for flexible resource patching and declarative management without the added complexity of operator logic. This model is directly reflected in the project’s directory structure for local development and QA environments, as shown in the images below.

While Helm charts from Bitnami were initially considered for stateful workloads, the discontinuation of free chart support by Bitnami led to a transition toward Operators for service management. Operators offer distinct advantages over traditional Helm approaches:
They automate day-2 operations beyond initial deployment—such as backups, automated failover, upgrades, and scaling—using Custom Resource Definitions (CRDs) that are fully declarative.
Operators integrate tightly with Kubernetes events and status management, allowing real-time reconciliation and sophisticated self-healing, which is critical for stateful systems.
Kustomize complements Operators by enabling environment-specific overlays, resource customization, and modular manifest management without custom templating logic, making infrastructure code easier to reason about, review, and modify.
All stateful infrastructure directories (for example, infra/minio, infra/postgres) contain both Operator-specific manifest files and Kustomize overlays to handle different environments and patches.

Namespace isolation is enforced in QA to simulate production-grade multi-tenancy and security boundaries. By separating resources into functional namespaces for services (qa-svc), data (qa-data), authentication (qa-auth), networking (qa-networking), and monitoring (qa-monitoring), it becomes possible to limit blast radius, simplify access permissions, and streamline troubleshooting. This level of separation is not used for local development, where resources reside in a flat local-dev-all namespace for speed and simplicity.

<-table goes here->

In both environments, Operator manifests and Kustomize overlays reside side by side in well-organized directories. For example, base/cluster.yaml and base/kustomization.yaml are located together for each stateful service, allowing both declarative deployment and environment-specific customization. Patches for resource adjustments, monitoring endpoints, or multi-tenant configurations are managed through Kustomize overlays, ensuring repeatable and reliable deployments across local and QA clusters, as shown below.

<-images go here->

This hybrid deployment strategy achieves several key objectives: automating infrastructure management, simplifying resource customization, enhancing production-like isolation, and facilitating seamless upgrades and scaling as environments evolve.

### Deploying Core Observability Tools

### Deploying Infrastructure Components

### Deploying Auxiliary Services

### Leveraging AI Tools During the "Lift" Phase
    * Practical examples of how AI tools assisted in the infrastructure setup.
    * Reflections on the benefits and limitations of using JetBrains AI tools.
    * Cost lessons and recommendations

### Final Polishing & Validation
    * Refactor k8s structure, resource manifests, and make targets.
    * "Lift" validation checklist.
    * Clear next-article teaser

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
