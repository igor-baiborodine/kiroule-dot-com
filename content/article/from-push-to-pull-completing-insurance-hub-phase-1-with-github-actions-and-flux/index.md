---
title: "From Push to Pull: Completing Insurance Hub Phase 1 with GitHub Actions and Flux"
date: 2026-04-20T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Monorepo", "Kubernetes", "GitOps", "CICD", "Flux", "GitHub Actions", "Docker" ]
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

For the GitOps reconciliation layer—specifically for the production-like QA environment—I selected [Flux CD](https://fluxcd.io). Initially, I considered the trade-offs between Flux and [Argo CD](https://argoproj.github.io/cd/). While Argo CD offers a rich user interface and sophisticated image automation, I ultimately chose Flux due to its "Git-centric" approach, which felt more lightweight and integrated seamlessly with Kustomize. Flux’s source controller and Kustomize controller allowed me to treat my manifests as the single source of truth, without the heavy scaffolding or management UI required by Argo. Since I was already managing my environment through structured Kustomize overlays, choosing Flux felt like a natural extension of my existing workflow rather than an additional layer of complexity to manage.

### CI/CD: Orchestrating Modular Release

Before proceeding with the implementation of our CI/CD workflows, I gave careful thought to the underlying automation strategy. The Insurance Hub is a Maven monorepo where modules share a common Git history and depend on each other through internal APIs. This structure introduced three principal challenges: establishing a versioning strategy that works for both Java and future Go services, defining safety guardrails for inter-service dependency updates, and implementing automated version bumping driven by the commit history.

#### Module Versioning: Decoupling Release Cycle

I evaluated two primary versioning strategies to manage our modular monorepo: a **global repository version** and **independent per-service versioning**. Initially, the global "release train" approach appeared attractive due to its simple mental model—the entire platform would move from version `1.3.0` to `1.4.0` as a single unit. This model simplifies coordination for tightly coupled modules and keeps pipelines straightforward by eliminating a complex versioning matrix. However, it quickly became clear that this coarse-grained approach would introduce significant operational noise. Frequent, localized changes to a small service would force a version bump across the entire stack, making it difficult to reason about what had actually changed and triggering unnecessary builds for stable components.

I chose to implement independent semantic versioning for each module. This is a strategic choice that aligns with the Go module ecosystem’s requirements for monorepos. In Go, a single repository containing multiple modules must use prefixed tags—such as `legacy/pricing-service-api/v1.2.3`—for the toolchain to correctly resolve sub-directories as dependencies. By adopting this pattern now for our Java APIs, I am building the exact infrastructure and CI/CD habits required for a seamless Go migration later.

This approach also enforces independent service lifecycles, a core principle of the microservices architecture we are pursuing. Independent versioning ensures that consumers of the `product-service-api`, for example, are not forced to ingest updates they don’t need, effectively decoupling release cycles and reducing the risk of a "distributed monolith." From a technical safety perspective, it eliminates race conditions in GitHub Actions; since each matrix job manages a unique, module-specific tag, multiple APIs can be built, tagged, and released in parallel without ever conflicting over the same Git reference.

To maintain clarity between project eras, I restricted legacy Java API versions to the `1.x.x` range, reserving `2.0.0+` for future Go-based services. I also chose to keep the Maven `pom.xml` versions at a static `1.0.0-SNAPSHOT` for legacy modules. This sets a baseline while placing the burden of truth on Git tags, which serve as the only reliable indicator of what code is actually running in production.

#### Artifact Tagging: Balancing Traceability and Immutability

The versioning approach for container images needed to be both monorepo-friendly and forward-compatible. It required a strategy that balanced human-readable release history with the technical immutability necessary for reliable Kubernetes deployments. During the research phase, I evaluated several tagging strategies to find a balance between readability and absolute technical traceability.

Initially, I considered a simple SemVer-only approach to keep the image registry organized. However, while SemVer is excellent for tracking the evolution of a module, it lacks the cryptographic certainty needed to link a running binary back to a specific Git commit. If a tag were accidentally overwritten—a common risk during manual interventions—the link between source code and cluster state would be broken.

To mitigate this, I implemented a dual-tagging strategy that applies both a SemVer tag and a short Git SHA to every build. The SemVer tag provides the human-friendly timeline needed for release management, while the SHA tag acts as an immutable anchor. After testing this approach against our modular workflows, I chose to adopt it despite the minor trade-off of increased registry bloat. In my view, the benefit of having an absolute mapping between a container in GHCR and a commit in Git outweighs the overhead of managing a more aggressive image retention policy.

#### Guardrails: Managing Dependency Propagation

To manage the intricate web of inter-API and service dependencies, I implemented a strict “single-module-per-release” guardrail. This restriction ensures that every GitHub Action run focuses on a unique domain or service update, preventing the risks associated with partial failures or "mixed" version releases. By enforcing this atomic approach, I can guarantee that each scoped Git tag remains a precise point-in-time reference for a specific component’s lifecycle.

Recognizing that `policy-service-api` serves as a critical foundation for the entire landscape, I chose an active propagation strategy to prevent dependency drift. When an API release is finalized, a specialized “update-dependents” job automatically scans the repository for consumer modules and bumps their `pom.xml` versions. This change is committed back to the main branch, which in turn triggers a chained release of the dependent services, ensuring the system remains synchronized against the latest contracts.

Finally, I applied structural guardrails to separate internal library logic from deployable artifacts. While infrastructure modules like `command-bus` or any other service API module are published strictly as JAR files to GitHub Packages, business services bypass this step to build OCI-compliant Docker images directly for GHCR. I also adopted the **Conventional Commits** standard to drive this logic. Enforcing prefixes like `chore(k8s)` or `feat(svc)` allows the CI to automate versioning and provides a machine-readable audit trail of why a specific deployment changed. This ensures our deployment history remains fully auditable and anchored in the repository as the sole source of truth.

### 4. GitOps in QA: keeping Flux focused on reconciliation

- Describe the bootstrap of Flux for QA
- Explain GitRepository + Kustomization loop
- Clarify that Flux is used only for cluster reconciliation
- CI updates manifests in Git; Flux applies what Git declares
- Explain why image automation was intentionally left out
- Include Mermaid diagram ideas:
    - API release flow
    - service release flow
    - Flux reconciliation loop

### 5. The first release was not just a deployment — it was a dependency exercise

- Explain why the initial release had prerequisites
- Tie release order to the module dependency summary and diagram
- Explain API/service separation and why it shaped the rollout
- Walk through the practical order:
    - shared APIs first
    - dependent services after
    - gateway after dependent APIs stabilize
- Show that the first release validated both automation and architecture

### 6. What I deliberately did not automate yet

- No Flux image automation
- No overly clever multi-loop delivery
- No premature promotion orchestration
- QA-first scope
- Why keeping the system explicit was the right trade-off for Phase 1

### 7. Using AI as an implementation accelerator

- AI for workflow scaffolding, diagram drafting, and validation
- AI for researching GitHub Actions and Flux patterns
- AI helped reduce boilerplate and iteration time
- Architecture and final decisions stayed human-driven

### 8. Phase 1 is now actually complete

- Re-state what Phase 1 now includes:
    - clusters
    - infrastructure
    - Kubernetes-ready services
    - delivery automation
    - Git-driven reconciliation
- Explain how this reduces operational risk for later phases
- Tease the next stage:
    - foundational observability maturation
    - and, ultimately, per-service Go migration

---

### What Mermaid diagrams would add the most value

Should be kept to **three diagrams max**.

#### 1. API release workflow

Show:

- API module change
- GitHub Actions release
- Maven artifact publish
- dependent version bump commit
- downstream services consume new version

This is probably the most distinctive technical story in the article.

### 2. Service/web release workflow

Show:

- module change
- workflow run
- image build
- push to GHCR
- manifest update in Git
- Flux reconcile to QA

### 3. Flux reconciliation loop

Show:

- Git repository as desired state
- Flux source fetch
- Kustomization reconcile
- QA cluster state convergence

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
