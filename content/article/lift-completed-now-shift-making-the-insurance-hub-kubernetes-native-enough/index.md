---
title: "Lift Completed, Now Shift: Making the Insurance Hub Kubernetes-Native (Enough)"
date: 2026-02-05T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Kubernetes", "MinIO", "S3", "PostgreSQL", "Kustomize", "Service Discovery", "Kind", "K3s" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

The Insurance Hub migration continues, and after completing Phase 1’s “lift” work described in the previous [post](/article/from-docker-compose-to-kubernetes-lifting-the-insurance-hub-into-the-cloud/)—provisioning Kubernetes clusters and standing up all the supporting infrastructure—it’s time to focus on the “shift” that makes the legacy system truly run on that new foundation. In this article, a summary of the targeted changes needed to move the existing Java services into Kubernetes will be provided.

<!--more-->

This “shift” is intentionally pragmatic: services are not being rewritten yet, but the biggest blockers preventing the legacy Java stack from behaving like a well-mannered Kubernetes workload are being removed. This begins with storage and state. Services currently writing to the local filesystem—or hiding blobs in the database—need to speak S3 instead, using a unified, S3-compatible SDK pointed at the MinIO tenants. In parallel, the convenience of in-memory H2 will be replaced with real PostgreSQL instances running in the cluster, ensuring that each service can be validated with the same persistence model it will use beyond a developer laptop.

With those foundations in place, the rest of the work will be focused on making the platform operationally boring (the highest compliment in infrastructure). Container images will be validated and tightened, each microservice will be deployed with a repeatable Kustomize base plus environment overlays, and Consul will be decommissioned in favor of Kubernetes-native service discovery (internal DNS). Finally, the existing `agent-portal-gateway` service will be brought into the cluster as the initial entry point, and its routes will be wired to the new service endpoints. The guiding principle throughout is “minimal, configuration-driven change”: everything should be reproducible via Make targets and verifiable locally by running services from IntelliJ while they interact with MinIO/Postgres/MongoDB/Kafka from the local dev Kind-based cluster, and verifiably exercised through a properly functioning application in both the local dev and QA Kubernetes environments.

{{< toc >}}

### Shift Backlog & Preparations

While the previous "lift" phase was characterized as infra-first and largely app-agnostic, the "shift" marks the transition to an **app-first, infra-consuming** approach. The foundational cluster work is complete; now, the legacy Java microservices must be modified to utilize these new cloud-native resources. This requires a deliberate move away from local-only shortcuts—such as in-memory databases and local file writes—toward durable, cluster-managed services.

To maintain momentum without introducing excessive instability, the shift has been organized into four primary deliverables. I have prioritized storage and state first, followed by networking and service discovery, ensuring that the most invasive changes are validated before the final application cutover.

| Ticket    | Deliverable                             | Description                                                                                                                       |
|:----------|:----------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------|
| **[7](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054538&issue=igor-baiborodine%7Cinsurance-hub%7C12)** | **MinIO/S3 Integration**                | Modify `payment-service` and `documents-service` to use S3-compatible SDKs for file operations against MinIO.                     |
| **[8](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054674&issue=igor-baiborodine%7Cinsurance-hub%7C13)** | **PostgreSQL External Persistence**     | Transition services from in-memory H2 to dedicated PostgreSQL clusters running in the Kubernetes environments.                    |
| **[9](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054790&issue=igor-baiborodine%7Cinsurance-hub%7C14)** | **Gateway in K8s**                      | Deploy the `agent-portal-gateway` as the initial ingress point, establishing the external routing boundary.                       |
| **[10](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054853&issue=igor-baiborodine%7Cinsurance-hub%7C15)**   | **Per-service deploy + Consul removal** | Decommission Consul; migrate inter-service communication to Kubernetes-native DNS and finalize per-service Kustomize deployments. |

The rationale for this risk-ordering is pragmatic: storage and database connectivity represent the highest points of failure. By resolving these first, the backend services can be validated as stable workloads. Once established, inter-service discovery and gateway routing are configured to wire the system together into a functional mesh.

Supporting a productive local development loop while interacting with a remote or Kind-based cluster requires careful planning of the networking surface. Since all Micronaut-based Java services default to port `8080`, running multiple services locally via an IDE would inevitably lead to port exhaustion and conflicts.

To mitigate this, a deterministic port assignment table was established. These mappings are implemented through `application-local.yaml` configuration files, ensuring each service has a non-conflicting home on the developer's localhost.

| Service             | Local Port | Service                 | Local Port |
|:--------------------|:-----------|:------------------------|:-----------|
| `auth-service`      | 8081       | `payment-service`       | 8086       |
| `chat-service`      | 8082       | `pricing-service`       | 8087       |
| `dashboard-service` | 8083       | `product-service`       | 8088       |
| `document-service`  | 8084       | `policy-search-service` | 8089       |
| `policy-service`    | 8085       | `agent-portal-gateway`  | 8090       |

A similar logic was applied to the stateful components running inside the cluster. Because a "dedicated cluster per microservice" approach was chosen for PostgreSQL, unique local ports must be exposed to allow IDE-based services to reach their respective data stores.

| Postgres Service                     | Cluster port mapping  |
|--------------------------------------|-----------------------|
| `svc/local-dev-postgres-auth-rw`     | 5432 → localhost:5442 |
| `svc/local-dev-postgres-document-rw` | 5432 → localhost:5452 |
| `svc/local-dev-postgres-payment-rw`  | 5432 → localhost:5462 |
| `svc/local-dev-postgres-policy-rw`   | 5432 → localhost:5472 |
| `svc/local-dev-postgres-pricing-rw`  | 5432 → localhost:5482 |

Finally, since two MinIO tenants were used (document and payment), local dev MinIO endpoints were also assigned non-overlapping ports so that each service could be tested against its tenant deterministically during the S3 migration work:

| MinIO Tenant Service              | Port |
|-----------------------------------|------|
| `svc/local-dev-minio-document-hl` | 9001 |
| `svc/local-dev-minio-payment-hl`  | 9002 |

This upfront organization ensures that the environment's infrastructure connectivity remains transparent, and the focus stays on the code modifications required for the shift.

### Pragmatic Persistence: Adapting Storage for S3 and MinIO

The transition of stateful workloads began with [Ticket 7](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054538&issue=igor-baiborodine%7Cinsurance-hub%7C12), which centered on moving the storage layer from local filesystems and database blobs to S3-compatible object storage. I targeted the `documents-service` and `payment-service` first, as they represented the most significant dependencies on local persistence. Before applying any code changes, I needed a stable local development loop; I added `application-local.yml` configurations to ensure services could run in IntelliJ while interacting with the MinIO tenants in the local dev's Kind cluster.

During this initial setup, I encountered an immediate bottleneck: even though Consul decommissioning was scheduled for a later deliverable, I had to prune the `micronaut-discovery-client` dependencies and configurations immediately. Application startup was failing as the services attempted to reach a discovery server that no longer existed in the new architecture—a pragmatic pivot was required to unblock the refactoring.

Provisioning the MinIO tenants required more than just creating buckets; I chose to strictly adhere to MinIO best practices to ensure security and scalability. This involved implementing purpose-driven naming conventions, granular bucket sharding, and enforcing the principle of least privilege through dedicated IAM policies. While these configurations can be managed via the MinIO Console UI, doing so manually across local-dev and QA environments is error-prone and inconsistent. To ensure deployments remained reproducible and "boring," I automated the creation of buckets, service users, and Kubernetes secrets via new [Makefile targets](https://github.com/igor-baiborodine/insurance-hub/blob/3bab17d580e9baa5702d67715e60301bb749e33e/k8s/Makefile#L624):

* `minio-svc-bucket-create` — Provisions a purpose-specific bucket for a microservice.
* `minio-svc-user-secret-create` — Generates the Opaque Kubernetes secret for service credentials.
* `minio-svc-user-with-policy-create` — Creates the MinIO user and attaches the necessary S3 IAM policy.

In the Kotlin-based `documents-service`, I deliberately chose an architectural approach to avoid a database migration. To keep the `PolicyDocument` class unchanged, I adapted the `bytes` field to serve a dual purpose: storing raw PDF binary data for legacy records or storing the UTF-8 encoded MinIO object key for new documents. I encapsulated this logic in a new `PolicyDocumentService` that uses a regex-based pattern check to determine whether to return the database bytes directly or fetch the content from S3 via the `MinioClient`.

The Java-based `payment-service` underwent a similar overhaul. I refactored the `InPaymentRegistrationService` to replace legacy `java.io` logic with S3 operations, using the MinIO SDK to verify the existence of the bank statement and stream the CSV content for processing. After a successful import, files are now marked as processed by copying them to a "processed" prefix within the bucket. I also simplified the `BankStatementFile` class, stripping out file-manipulation logic to focus solely on object key construction. Both services were then integrated with a `MinioClientFactory` for client lifecycle management, completing the shift toward a storage-agnostic architecture. For a closer look at the diffs and testing evidence, consult the [associated pull request](https://github.com/igor-baiborodine/insurance-hub/pull/46).