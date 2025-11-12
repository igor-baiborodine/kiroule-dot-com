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

This phase moves the entire legacy application from the familiar confines of Docker Compose into a
robust, production-like Kubernetes environment. By establishing parallel clusters—using Kind for
local development and K3s for quality assurance—this critical first step, representing the
metaphorical "lift," validates the new platform and lays the stable groundwork necessary for the
upcoming deeper service-by-service "shift" migration.

{{< toc >}}

### Article Scope

The scope of this article covers the foundational infrastructure "lift" phase of migrating the
Insurance Hub to Kubernetes, specifically:

* Provisioning local development and QA Kubernetes clusters using Kind and Rancher’s K3s, creating
  production-like environments for testing and validation.

* Deploying core cluster observability tools, including Prometheus and Grafana for metrics and
  monitoring, plus Zipkin for distributed tracing of Insurance Hub services.

* Deploying core infrastructure components essential to the Insurance Hub, such as PostgreSQL,
  MongoDB, Elasticsearch, Apache Kafka, and introducing MinIO as the new S3-compatible object
  storage replacement for the existing filesystem.

* Deploying auxiliary services like JSReport to support the Insurance Hub's PDF generation
  capabilities.

This second installment focuses on establishing the cloud-native platform foundation by replicating
the existing environment within Kubernetes. Targeted Java service modifications and gateway
deployment are part of the next "shift" phase and will be covered later. This lift ensures minimal
disruption while validating stable operation on the Kubernetes infrastructure.

### Header 2

Header 2 content

### Header 3

Header 3 content

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
