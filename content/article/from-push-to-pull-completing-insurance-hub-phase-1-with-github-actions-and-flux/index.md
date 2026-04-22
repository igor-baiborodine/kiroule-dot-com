---
title: "From Push to Pull: Completing Insurance Hub Phase 1 with GitHub Actions and Flux"
date: 2026-04-20T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Kubernetes", "GitOps", "CICD", "Flux", "GitHub Actions", "Docker" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

The previous [article](/article/lift-completed-now-shift-making-the-insurance-hub-kubernetes-native-enough/) concluded with the system successfully running on Kubernetes, yet it still relied on a manual deployment model. While validating the cluster foundation and service manifests was a significant milestone, it also revealed a lingering operational gap. I had migrated the platform to Kubernetes, but the delivery process remained manual—a compromise that prevented the system from fully embodying a cloud-native workload.

<!--more-->

This distinction is more critical than it might seem at first. Kubernetes offers declarative runtime primitives, service discovery, and repeatable scheduling, but it does not inherently make the release process declarative. At this point, I still needed to build, version, and publish images explicitly. Additionally, manifests required manual updates after every release. Although the cluster was configured to run the services, I felt that the process for deploying those workloads was still too manual to be considered complete.

This article addresses the final component of Phase 1, focusing on the CI/CD and GitOps work that I had postponed until now. I will utilize GitHub Actions for modular release workflows, GitHub Packages and GHCR for artifact publication, and Flux for cluster reconciliation based on the repository state.

{{< toc >}}

### Phase 1 Scoping: Anchoring the Delivery Foundation

Before committing to the Phase 1 implementation, I conducted a comprehensive review of the project's scope and identified several critical gaps that needed to be addressed from the beginning. I realized that observability had to extend beyond the Insurance Hub services to include the underlying infrastructure itself. Additionally, it became evident that Zipkin and JSReports were completely absent from the initial deployment plan. I also determined that the integration of GitOps could no longer be postponed until Phase 6, as originally envisioned. As a result, I decided to incorporate the CI/CD and GitOps work into Phase 1. The reasoning was practical: while the initial effort produced a functional Kubernetes runtime, the platform would lack true operational completeness without a declarative delivery path.

This distinction proved crucial during execution. Although the services were successfully deployed into Kind and K3s clusters—with MinIO replacing local filesystems and PostgreSQL managing state—the operational model still felt incomplete. I found myself relying on orchestrated Make targets and manual image tagging instead of repository reconciliation. Without the integration of GitHub Actions, Flux, and GHCR, Phase 1 would have ended with a functional system that was not yet production-ready—it could run, but it could not reliably release itself.

This gap was particularly evident when considering the upcoming [Phase 4 Strangler Fig](https://github.com/igor-baiborodine/insurance-hub/blob/main/docs/system-overview-and-migration-analysis.md#phase-4-phased-service-migration-to-go-strangler-fig-pattern) migration. I anticipated that managing Java and Go versions through separate Kustomize overlays would quickly become an exercise in manual coordination. Therefore, I concluded that automated workflows for modular releases and Flux-based reconciliation were not just optimizations but essential requirements to ensure the platform remained self-consistent and scalable from the outset.

For the local development Kind cluster, I intentionally maintained the manual push model. This approach—building images locally and loading them into nodes via Make—aligns better with the iterative feedback loop I need during active coding. However, for the production-like QA environment running on K3s, I prioritized achieving full operational completeness through a proper delivery pipeline.

### Architecting the Delivery Path: From Requirements to Reconciliation

Transitioning from a manual “push” model to a GitOps-driven pipeline required a clear definition of what a “complete” delivery platform meant for a monorepo. Having spent most of my career outside the context of large-scale monorepos, I found it challenging to establish the objectives and select the right tools that could handle the complexities of cross-module dependencies. I needed a platform that did not just automate the path to production but also enforced the operational discipline I had established during the manual “shift” phase.

Specifically, the platform needed to satisfy five core requirements:

- **Automated PR Validation**: Every change, whether to a shared Java API module or a future Go service, must receive immediate feedback through linting and automated testing to catch regressions and style drift early.
- **Modular Release Flows**: I needed to establish independent pipelines for APIs, shared libraries, and services to manage the monorepo's dependency graph. This modularity ensures that a release of a shared module triggers updates only for its direct downstream consumers, preventing localized changes from causing an uncontrolled, full-stack re-release cycle.
- **Scalable Versioning**: The versioning strategy needed to be compatible with a monorepo and forward-compatible, ensuring that the eventual Java-to-Go migration would not disrupt our artifact history.
- **Unified Artifact Management**: A centralized location was necessary for publishing both JAR files for shared modules and container images for workloads.
- **Declarative Reconciliation**: Finally, the QA cluster state needed to be continuously reconciled with the repository.

Since the source code is hosted on GitHub, I chose to leverage the GitHub ecosystem extensively. Utilizing GitHub Actions (GHA) for continuous integration, along with GitHub Packages and the GitHub Container Registry (GHCR), was a practical choice that minimized integration friction. By keeping the delivery logic close to the code, I eliminated the need for external authentication and benefited from GHA's native support for monorepo-style path filtering. Additionally, using GHCR as our OCI-compliant registry provided a unified view of our images, which was essential for tracking the coexistence of legacy Java and future Go workloads.

For the GitOps reconciliation layer—specifically for the production-like QA environment—I selected Flux CD. Initially, I considered the trade-offs between Flux and Argo CD. While Argo CD offers a rich user interface and sophisticated image automation, I ultimately chose Flux due to its "Git-centric" approach, which felt more lightweight and integrated seamlessly with Kustomize. Flux’s source controller and Kustomize controller allowed me to treat my manifests as the single source of truth, without the heavy scaffolding or management UI required by Argo. Since I was already managing my environment through structured Kustomize overlays, choosing Flux felt like a natural extension of my existing workflow rather than an additional layer of complexity to manage.

### Section 3

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
