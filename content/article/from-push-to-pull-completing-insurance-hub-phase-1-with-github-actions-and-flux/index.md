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

### Section 1

### Section 2

### Section 3

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
