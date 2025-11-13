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
for Phase 1 was created in the "Insurance Hub—Go Migration" project, followed by more granular
tickets that were aligned with the scope. To streamline ticket creation, the Junie AI coding agent
was employed to assist in drafting detailed descriptions based on a standardized template. For
managing the workflow, all new tickets initially enter the "Backlog" column and must pass through a
refinement process before being moved to "Ready." This process includes validating ticket descriptions
and acceptance criteria, conducting necessary research—such as evaluating whether to use ArgoCD or
Flux for GitOps—and updating the "Dev Notes" section. This structured approach ensures clear
communication, proper preparation, and prioritization, setting a strong foundation for executing the
lift phase of the migration.

### Provisioning Local Dev (Kind) & QA (K3s) Clusters

    * Initial commitment to use Microk8s for a production-like environment, but switched to K3s due
      to issues while creating a cluster.
    * Automate local dev and QA clusters bootstrapping by implementing make targets.
    * Cluster addons - installed by default.
    * Provide more specific implementation details and gotchas.

### Kubernetes Deployment Strategy & Best Practices
    * Elaborate on the necessary change from Bitnami Helm charts to operator-based deployments.
    * Elaborate on proper namespace isolation and resource management.

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
