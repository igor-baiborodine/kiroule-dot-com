---
title: "Lift Completed, Now Shift: Making the Insurance Hub Kubernetes-Native (Enough)" 
date: 2026-02-05T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Kubernetes", "MinIO", "S3", "PostgreSQL", "Kustomize", "Service Discovery", "Kind", "K3s" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

The Insurance Hub migration continues, and after completing Phase 1’s “lift” work described in the previous [post](/article/from-docker-compose-to-kubernetes-lifting-the-insurance-hub-into-the-cloud/)—provisioning Kubernetes clusters and standing up all the supporting infrastructure—it’s time to focus on the “shift” that makes the legacy system truly run on that new foundation. In this article, a summary of the targeted changes needed to move the existing Java services into Kubernetes will be provided.

<!--more-->

This “shift” is intentionally pragmatic: services are not being rewritten yet, but the biggest blockers preventing the legacy Java stack from behaving like a well-mannered Kubernetes workload are being removed. This begins with storage and state. Services currently writing to the local filesystem—or hiding blobs in the database—need to speak S3 instead, using a unified, S3-compatible SDK pointed at the MinIO tenants. In parallel, the convenience of in-memory H2 will be replaced with real PostgreSQL instances running in the cluster, ensuring that each service can be validated with the same persistence model it will use beyond a developer laptop.

With those foundations in place, the rest of the work will be focused on making the platform operationally boring (the highest compliment in infrastructure). Container images will be validated and tightened, each microservice will be deployed with a repeatable Kustomize base plus environment overlays, and Consul will be decommissioned in favor of Kubernetes-native service discovery (internal DNS). Finally, the existing `agent-portal-gateway` service will be brought into the cluster as the initial entry point, and its routes will be wired to the new service endpoints. The guiding principle throughout is “minimal, configuration-driven change”: everything should be reproducible via Make targets and verifiable locally by running services from IntelliJ while they interact with MinIO/Postgres/MongoDB/Kafka from the local dev Kind-based cluster, and verifiably exercised through a properly functioning application in both the local dev and QA Kubernetes environments.

{{< toc >}}

### Details 1
### Details 2
### Details 3
