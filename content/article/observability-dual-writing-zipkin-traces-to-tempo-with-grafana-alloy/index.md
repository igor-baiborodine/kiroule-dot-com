---
title: "Observability: Dual-Writing Zipkin Traces to Tempo with Grafana Alloy"
date: 2026-05-15T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

After Phase 1, the Insurance Hub had a Kubernetes runtime and a GitOps delivery loop that could be relied on in the QA K3s environment. That closed the operational gap around deployment, but it also made another gap more visible. The platform could be reconciled from Git, yet the tooling for understanding cross-service behavior still lived in a legacy corner.

Zipkin tracing existed and it worked, however it was still anchored to a single backend and a single era of the system. Phase 4 will not be a clean cutover. It is designed to run in a mixed state, where some services remain legacy Java while others are migrated to Go, and requests routinely cross that boundary. In that hybrid period, distributed traces must land in one place, otherwise troubleshooting becomes a game of stitching partial views together.

Phase 2 addresses that constraint directly. The objective is to make Grafana Tempo the durable tracing backend early, while keeping the promise of no changes to the existing Java services. To do that, I introduced Grafana Alloy as a permanent, cluster-wide telemetry gateway. It receives Zipkin-format traces from the legacy services and exports them to both Zipkin and Tempo during the transition. Zipkin remains available as the baseline and safety net, while Tempo becomes the single store that can span both Java and Go services as the migration progresses.

This phase also deploys Grafana Loki, but deliberately stops short of enabling log ingestion. Loki is treated as part of the target stack and a prerequisite for Phase 4, not as a completed logging rollout. The focus here is narrow and intentional: stand up Tempo, put Alloy in front of it, validate dual-writing end-to-end, and harden the QA K3s provisioning steps so the entire observability installation is repeatable.

### Phase 2 Scope: Foundational Observability

Phase 2 was designed as a strategic bridge between the initial Kubernetes migration and the upcoming service rewrites. The primary objective was to modernize the telemetry backend while the Java services remained untouched. By establishing a durable tracing and logging foundation early, I ensured that the hybrid environment—where legacy Java and new Go services will eventually coexist—would have a unified "single pane of glass" for troubleshooting.

The development work was managed through a dedicated Epic in GitHub Projects, decomposed into four child tickets. I focused on standing up the core Grafana stack and implementing a centralized collection layer that decoupled the applications from their telemetry backends.

| Ticket  | Deliverable                            | Description                                                                                                                    |
|:--------|:---------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------|
| **[1]** | **Grafana Loki Installation**          | Deploy Loki in the QA cluster with MinIO-backed storage to serve as the future target for centralized log aggregation.         |
| **[2]** | **Grafana Tempo Installation**         | Set up Tempo as the primary tracing store, providing a scalable, S3-compatible alternative to the legacy Zipkin backend.       |
| **[3]** | **Grafana Alloy Deployment**           | Implement Alloy as a permanent telemetry gateway to ingest legacy traces and route them into the modern stack.                 |
| **[4]** | **QA Cluster Provisioning Refinement** | Update Make targets and infrastructure automation to handle the increased resource footprints of the observability components. |

In this phase, the scope was strictly infrastructure-centric. I treated the installation of Loki and Tempo as a prerequisite for the platform's long-term health, ensuring that the storage strategy remained consistent with the MinIO-based approach established in Phase 1. The introduction of Grafana Alloy allowed me to validate the end-to-end trace flow from existing Java services into Tempo, effectively hardening the observability pipeline before a single line of Go code was even written. This proactive setup minimizes the operational risk of Phase 4, as the monitoring environment is already mature and ready to receive OTLP signals.

### Grafana Loki

* Provide implementation details.
* List new Makefile targets for managing Grafana Loki and a runbook for validation.

### Grafana Tempo

* Provide implementation details.
* List new Makefile targets for managing Grafana Tempo and a runbook for validation.

### Grafana Alloy

* Provide implementation details.
* List new Makefile targets for managing Grafana Alloy and a runbook for validation.

### Provisioning of K3s Cluster in QA

* Explain in detail why the hardening was needed.
* Provide implementation details: main change - wait until CRDs are ready.

### AI Usage

* Change in AI usage: a drift from using it as a chat companion to a specification-first, agent-based development approach.
* Provide an excerpt from previous articles. State that this change in AI usage is not finalized and is work in progress, as I’m trying to polish the procedure and find the best sequence by applying best practices and trying different things.

### Conclusion

* What’s next: Phase 3.

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
