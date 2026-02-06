---
title: "Lift Completed, Now Shift: Making the Insurance Hub Kubernetes-Native (Enough)" 
date: 2026-02-05T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Kubernetes", "MinIO", "S3", "PostgreSQL", "Kustomize", "Service Discovery", "Kind", "K3s" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

The Insurance Hub migration continues, and after completing Phase 1’s “lift” work described in the previous [post](/article/from-docker-compose-to-kubernetes-lifting-the-insurance-hub-into-the-cloud/)—provisioning Kubernetes clusters and standing up all the supporting infrastructure—it’s time to focus on the “shift” that makes the legacy system truly run on that new foundation. In this article, I’ll summarize the targeted changes needed to move the existing Java services into Kubernetes.

<!--more-->

This “shift” is intentionally pragmatic: we’re not rewriting services yet, but we are removing the biggest blockers that prevent the legacy Java stack from behaving like a well-mannered Kubernetes workload. That starts with storage and state. Services that currently write to the local filesystem—or hide blobs in the database—need to speak S3 instead, using a unified, S3-compatible SDK pointed at our MinIO tenants. In parallel, we’ll replace the convenience of in-memory H2 with real PostgreSQL instances running in the cluster, so each service can be validated with the same persistence model it will use beyond a developer laptop.

With those foundations in place, the rest of the work is about making the platform operationally boring (the highest compliment in infrastructure). We’ll validate and tighten container images, deploy each microservice with a repeatable Kustomize base plus environment overlays, and decommission Consul in favor of Kubernetes-native service discovery (internal DNS). Finally, we’ll bring the existing `agent-portal-gateway` into the cluster as the initial entry point and wire its routes to the new service endpoints. The guiding principle throughout is “minimal, configuration-driven change”: everything should be reproducible via Make targets and verifiable locally by running services from IntelliJ while they interact with MinIO/Postgres in a local Kind cluster.

These incremental steps—driven by Make targets for reproducibility—transform the lifted infrastructure into a functional, service-ready platform while preserving operational continuity.

{{< toc >}}

### Details 1
### Details 2
### Details 3
