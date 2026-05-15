---
title: "Observability: Dual-Writing Zipkin Traces to Tempo with Grafana Alloy"
date: 2026-05-15T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

Display intro paragraph

<!--more-->

Intro paragraphs

### Phase 2 scope

* Pre-implementation analysis
* Finalize the suggested implementation: collector vs. Grafana Alloy (selected). Provide reasoning for using Alloy.
* List tasks and corresponding tickets in GitHub Project.
* Without giving a lot of details, explain why the scope grew by 1 ticket: Harden provisioning of the K3S cluster in QA.

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
