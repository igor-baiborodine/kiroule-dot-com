---
title: "From Push to Pull: Completing Insurance Hub Phase 1 with GitHub Actions and Flux"
date: 2026-04-20T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Kubernetes", "GitOps", "CICD", "Flux", "GitHub Actions", "Docker" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

The previous article ended with the system already running in Kubernetes but still depending on a manual push model for deployments. That was a useful milestone, because it proved the cluster foundation and the service manifests were in place. But it also left an obvious operational gap. The platform had been lifted and shifted into Kubernetes, yet the delivery process itself was still being driven by hand, which meant the system was not quite behaving like a fully cloud-native workload.

<!--more-->

That distinction matters more than it may seem at first glance. Kubernetes can give you declarative runtime primitives, service discovery, and repeatable scheduling, but it does not automatically make the release process declarative as well. At this point, images still needed to be built, versioned, and published explicitly. Manifests still needed to be updated after each release. The cluster could be made to run the services, but the path that delivered those services into the cluster was still too manual to be called complete.

So this article closes that remaining piece of Phase 1. It covers the CI/CD and GitOps work that was deferred until now, using GitHub Actions for the modular release workflows, GitHub Packages and GHCR for artifact publication, and Flux for cluster reconciliation from the repository state.

{{< toc >}}

### Section 1

### Section 2

### Section 3

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
