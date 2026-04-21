---
title: "From Push to Pull: Completing Insurance Hub Phase 1 with GitHub Actions and Flux"
date: 2026-04-20T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Kubernetes", "GitOps", "CICD", "Flux", "GitHub Actions", "Docker" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

The previous [article](/article/lift-completed-now-shift-making-the-insurance-hub-kubernetes-native-enough/) concluded with the system successfully running in Kubernetes but still relying on a manual deployment process. While this was a significant milestone, demonstrating that the cluster foundation and service manifests were established, it also highlighted an obvious operational gap. The platform had been migrated to Kubernetes, yet the delivery process remained manual, meaning the system did not fully embody a cloud-native workload.

<!--more-->

This distinction is more important than it may initially appear. Kubernetes offers declarative runtime primitives, service discovery, and repeatable scheduling, but it does not inherently make the release process declarative. At this stage, images still needed to be built, versioned, and published explicitly. Manifests had to be updated after each release. Although the cluster was configured to run the services, the process for deploying those services into the cluster was still too manual to be considered complete.

This article addresses the final component of Phase 1, focusing on the CI/CD and GitOps work that was postponed until now. It will utilize GitHub Actions for modular release workflows, GitHub Packages and GHCR for artifact publication, and Flux for cluster reconciliation based on the repository state.

{{< toc >}}

### Why GitOps Belongs in Phase 1

Before diving into Phase 1, a thorough scope review surfaced several gaps that needed to be addressed upfront. Observability had to extend beyond just Insurance Hub services to cover the underlying infrastructure, Zipkin and JSReports were missing entirely from the deployment plan, and GitOps integration could no longer wait until Phase 6. The decision to move CI/CD and GitOps into Phase 1 was straightforward: the lift and shift work was delivering a Kubernetes runtime, but without a declarative delivery path, the platform would never reach true operational completeness.

That distinction proved critical during execution. The services were successfully deployed into Kind and K3s clusters, MinIO replaced local filesystems, PostgreSQL clusters handled persistence, and Kustomize manifests made deployments repeatable. Yet the operational model remained incomplete. Images still required manual builds and tags, manifests needed explicit version updates after each release, and the cluster state depended on orchestrated Make targets rather than repository reconciliation. Without GitHub Actions, Flux, GitHub Packages, and GHCR, Phase 1 would have ended with a functional system that was not yet 100% production-ready—capable of running, but not of releasing itself consistently.

This operational completeness was targeted specifically at the production-like QA environment running on K3s. For the local-dev Kind cluster, the manual push model was deliberately retained. The fast iteration cycle—building images locally, loading them directly into Kind nodes, and applying Kustomize overlays via Make targets—better suits the developer workflow and immediate feedback needs during active coding and testing.

This gap became even more apparent as the future Strangler Fig migration. With Java and Go versions needing to coexist through separate Kustomize bases and overlays, manual coordination would have quickly become untenable. Automated workflows for modular releases, dependency bumping across the monorepo, and Flux reconciliation from Git were no longer optimizations, but structural requirements to make the platform self-consistent and scalable from Phase 1 forward.

### Section 2

### Section 3

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
