---
title: "Observability: Dual-Writing Zipkin Traces to Tempo with Grafana Alloy"
date: 2026-05-15T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

After [Phase 1](https://github.com/igor-baiborodine/insurance-hub/blob/main/docs/system-overview-and-migration-analysis.md#phase-1-foundational-infrastructure--environment-migration-lift-and-shift), 
the Insurance Hub had established a Kubernetes runtime and a GitOps delivery loop that functioned reliably in the QA K3s environment. This successfully closed the operational gap around deployments, but it also highlighted another issue. Although the platform could be reconciled from Git, the tools for understanding cross-service behavior were still confined to a legacy setup.

Zipkin tracing was in place and operational; however, it was limited to a single backend and an outdated version of the system. 
[Phase 4](https://github.com/igor-baiborodine/insurance-hub/blob/main/docs/system-overview-and-migration-analysis.md#phase-4-phased-service-migration-to-go-strangler-fig-pattern) 
won't involve a clean cutover; instead, it is designed to operate in a mixed state where some services are still in legacy Java while others have migrated to Go. Consequently, requests will frequently cross this boundary. During this hybrid period, it is crucial for distributed traces to converge in one location; otherwise, troubleshooting can become a challenging task of piecing together fragmented views.

[Phase 2](https://github.com/igor-baiborodine/insurance-hub/blob/main/docs/system-overview-and-migration-analysis.md#phase-2-foundational-observability) 
directly addresses this constraint. The goal is to establish Grafana Tempo as the primary tracing backend early on, all while ensuring that there are no changes to the existing Java services. To achieve this, I introduced Grafana Alloy as a permanent telemetry gateway across the cluster. Alloy receives Zipkin-format traces from the legacy services and exports them to both Zipkin and Tempo during this transition. Zipkin serves as a baseline and safety net, while Tempo will be the singular repository capable of spanning both Java and Go services as the migration unfolds.

In this phase, Grafana Loki is also deployed, but we intentionally refrain from enabling log ingestion. Loki is viewed as part of the target stack and a prerequisite for Phase 4, rather than a complete logging rollout. The focus here is clear and intentional: set up Tempo, place Alloy in front of it, validate end-to-end dual-writing, and strengthen the QA K3s provisioning steps to ensure that the entire observability installation is repeatable.

{{< toc >}}

### Phase 2 Scoping: Establishing Observability Foundation

Phase 2 was designed as a strategic bridge between the initial Kubernetes migration and the forthcoming service rewrites. The primary objective was to modernize the telemetry backend while keeping the Java services unchanged. By establishing a robust tracing and logging foundation early on, I ensured that the hybrid environment—where legacy Java services and new Go services will eventually coexist—would have a unified "single pane of glass" for troubleshooting.

The development work was managed through a dedicated [Epic](https://github.com/igor-baiborodine/insurance-hub/issues/79#issue-4376197009) 
in GitHub Projects, which was divided into four child tickets. I focused on setting up the core Grafana stack and implementing a centralized collection layer that decoupled the applications from their telemetry backends.

| Ticket                                                                                          | Deliverable                            | Description                                                                                                                    |
|:------------------------------------------------------------------------------------------------|:---------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------|
| **[[1]issue-80](https://github.com/igor-baiborodine/insurance-hub/issues/80#issue-4376712516)** | **Grafana Loki Installation**          | Deploy Loki in the QA cluster with MinIO-backed storage to serve as the future target for centralized log aggregation.         |
| **[[2]issue-81](https://github.com/igor-baiborodine/insurance-hub/issues/81#issue-4376717776)** | **Grafana Tempo Installation**         | Set up Tempo as the primary tracing store, providing a scalable, S3-compatible alternative to the legacy Zipkin backend.       |
| **[[3]issue-82](https://github.com/igor-baiborodine/insurance-hub/issues/82#issue-4379324528)** | **Grafana Alloy Deployment**           | Implement Alloy as a permanent telemetry gateway to ingest legacy traces and route them into the modern stack.                 |
| **[[4]issue-86](https://github.com/igor-baiborodine/insurance-hub/issues/86#issue-4425327776)** | **QA Cluster Provisioning Refinement** | Update Make targets and infrastructure automation to handle the increased resource footprints of the observability components. |

In this phase, the focus was primarily on infrastructure. I regarded the installation of Loki and Tempo as essential for the long-term health of the platform, ensuring that our storage strategy remained aligned with the MinIO-based approach established in Phase 1. The introduction of Grafana Alloy allowed me to validate the end-to-end trace flow from existing Java services into Tempo, effectively strengthening the observability pipeline even before any Go code was written. This proactive setup minimizes operational risks in Phase 4, as the monitoring environment is already mature and ready to receive OTLP signals.

### Gateway Pivot: Decoupling Tracing with Grafana Alloy

Initially, my plan for Phase 2 was to use a standard OpenTelemetry Collector as a temporary bridge, with the intent to decommission it once our new Go services were sending traces directly to Tempo. However, after researching OpenTelemetry best practices and the architectural requirements for a multi-language cluster, I switched to **Grafana Alloy** as a permanent telemetry gateway.

This was a strategic pivot rather than a tooling preference. The core problem wasn't just Zipkin; it was that our telemetry pipeline was tightly coupled to specific backends. By implementing Alloy as a permanent infrastructure layer, I’ve established what I call a "permanent contract" for the cluster. Alloy provides stable, cluster-wide endpoints for both legacy Zipkin traffic and modern OTLP signals, allowing us to evolve our backend storage (from Zipkin to Tempo) without ever needing to touch service-level instrumentation again. Furthermore, since our entire modern stack is Grafana-centered, Alloy serves as the native glue that ensures seamless correlation between logs, metrics, and traces within the same ecosystem.

#### Trace Flow Transition: The Dual-Write Strategy

The architecture of our trace flow underwent a significant shift during this phase. In the legacy state, services pushed directly to the Zipkin backend. In the new Phase 2 state, Alloy sits in the center, acting as a traffic controller.

<details>
  <summary><b>Flow Transition: From Legacy to Target State</b></summary>

![Trace Flow Transition](trace-flow-transition-diagram.png)

</details> 
&nbsp;

I chose to implement a dual-write strategy where Alloy forwards traces to both Zipkin and Tempo simultaneously. This was worth the additional complexity for three reasons:

1.  **Zero Blast Radius**: Zipkin remained the primary safety net for the team. If Tempo or the MinIO backing store struggled under load, our existing debugging workflow remained untouched.
2.  **Side-by-Side Validation**: We could compare the same traces in the legacy Zipkin UI and the new Grafana-first Tempo dashboards to ensure no data was being dropped or malformed during the OTLP translation.
3.  **The Clean Flip**: This setup provides a clear decommissioning path. Once we are confident in Tempo's retention and performance, we simply remove the Zipkin exporter from the Alloy configuration, resulting in no service restarts and no code changes.

This transition effectively decouples our telemetry "producers" from the "consumers," turning observability from a hard-coded dependency into a managed infrastructure service.

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
