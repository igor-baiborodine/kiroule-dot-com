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

Before starting Phase 1, a thorough review of the project's scope revealed several gaps that needed to be addressed from the outset. It was essential to extend observability beyond just the Insurance Hub services to encompass the underlying infrastructure. Additionally, Zipkin and JSReports were completely absent from the deployment plan, and the integration of GitOps could no longer be postponed until Phase 6. Therefore, the decision to include CI/CD and GitOps in Phase 1 was clear; the initial work was delivering a Kubernetes runtime, but without a declarative delivery path, the platform could not achieve true operational completeness.

This distinction became critical during execution. The services were successfully deployed into Kind and K3s clusters, MinIO replaced local filesystems, PostgreSQL clusters managed data persistence, and Kustomize manifests ensured repeatable deployments. However, the operational model was still incomplete. Images still required manual builds and tagging, manifests needed explicit version updates after each release, and the cluster state relied on orchestrated Make targets instead of repository reconciliation. Without GitHub Actions, Flux, GitHub Packages, and GHCR, Phase 1 would have concluded with a functional system that was not yet fully production-ready—it could run but not reliably release itself.

This gap became even more apparent in light of the upcoming Strangler Fig migration. With the need for Java and Go versions to coexist through separate Kustomize bases and overlays, manual coordination would quickly become unmanageable. Therefore, automated workflows for modular releases, dependency updates across the monorepo, and Flux reconciliation from Git had shifted from being optimizations to essential requirements, ensuring that the platform was self-consistent and scalable from Phase 1 onward.

The goal of achieving operational completeness was specifically focused on the production-like QA environment that was running on K3s. For the local development Kind cluster, the manual push model was intentionally maintained. This approach, which involved building images locally, loading them directly into Kind nodes, and applying Kustomize overlays via Make targets, aligned better with the developer workflow and the immediate feedback needed during active coding and testing.

### Section 2

### Section 3

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
