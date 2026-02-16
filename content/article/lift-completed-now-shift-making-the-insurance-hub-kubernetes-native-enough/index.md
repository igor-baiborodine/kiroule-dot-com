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

### Shift Scoping: Prioritizing Deliverables and Networking Surface

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

### Externalizing Relational Layer: From In-Memory H2 to Managed PostgreSQL

The transition from in-memory persistence to cluster-managed data stores, as outlined in [Ticket 8](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054674&issue=igor-baiborodine%7Cinsurance-hub%7C13), required a systematic overhaul of how the legacy Java services bootstrap their database connections. I implemented a consistent architectural pattern across the `auth`, `documents`, `payment`, `policy`, and `pricing` services by introducing a `PostgresDatasourceConfigurationListener` and a corresponding `PostgresDatasourceProperties` class. These components leverage Micronaut’s `BeanCreatedEventListener` to intercept the `DatasourceConfiguration` at runtime—dynamically building the JDBC URL while supporting configurable SSL modes and environment-specific credentials.

The configuration layer was split to support both cloud-native portability and a friction-free local developer experience. In the main `application.yml` files, I replaced hardcoded H2 connection strings with environment variable placeholders such as `${PG_HOST}` and `${PG_PORT}`. To allow Micronaut to still bootstrap the `DataSource` and JPA beans, I kept a minimal default configuration that the listener subsequently overrides. Meanwhile, the new `application-local.yml` files define the specific port sharding mentioned earlier—mapping each service to its respective local port (e.g., 8081 for `auth`, 8087 for `pricing`) and its dedicated PostgreSQL forward.

In the `auth-service`, I made a technical trade-off by refactoring the `InsuranceAgent` component from a Java record to a standard `@MappedEntity` class using Lombok’s `@Data` and `@NoArgsConstructor`. While records provide a clean syntax, the move to a persistent PostgreSQL schema necessitated a more traditional entity structure to ensure full compatibility with Micronaut Data JDBC and JPA mapping. I also implemented an explicit `availableProductCodes` helper to handle serialization of semicolon-separated strings stored in the database, and updated the `InsuranceAgentsRepository` to remove its hardcoded H2 dialect reference.

The Kotlin-based `documents-service` saw similar adjustments to accommodate the relational storage of binary objects. I updated the `PolicyDocument` entity to include the `@Lob` annotation and a `bytea` column definition, ensuring that document content is handled correctly by the PostgreSQL driver. Across all services, I updated the `pom.xml` files to include the `postgresql` driver and `micronaut-management` dependencies. Crucially, I also commented out the `micronaut-discovery-client` dependencies; even though service discovery was a later deliverable, the transition to external persistence forced an early decommission of the Consul-based discovery logic to prevent startup failures. For a deep dive into the code changes, see the [associated pull request](https://github.com/igor-baiborodine/insurance-hub/pull/47).

### Deployment Readiness: Establishing Service Conventions

While the preceding "lift" work focused on standing up the cluster and its stateful dependencies, the focus now shifts toward the deployment pipeline for the services themselves. When initially planning the transition for the 11 legacy services, I considered creating individual tickets for each to track granular progress. However, from a project management perspective, this threatened to pollute the board with low-signal updates. I chose instead to consolidate the effort into two primary deliverables: a dedicated ticket for the `agent-portal-gateway` [Ticket 9](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054790&issue=igor-baiborodine%7Cinsurance-hub%7C14)—given its role as the critical entry point and architectural anchor—and a grouping ticket for all remaining services [Ticket 10](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054853&issue=igor-baiborodine%7Cinsurance-hub%7C15). This allowed me to establish patterns on the gateway first, then apply those validated recipes to the rest of the stack via individual pull requests.

For the Insurance Hub, a standardized implementation sequence was established to ensure consistency. Each service follows a five-step lifecycle: validating or updating the Docker image, building locally, loading the image into the cluster, implementing Kustomize manifests, and finally triggering the deployment. I adopted a semantic naming convention for these images: `insurance-hub-<service-name>-<component-type>-legacy:<tag>`. By sorting by service name first (e.g., `insurance-hub-policy-api-legacy` rather than `insurance-hub-api-policy-legacy`), the registry remains organized and searchable. Initially, until a full Flux-based GitOps workflow is implemented, all image tags default to `latest` to simplify local iteration.

The transition to Kubernetes-native manifests required a rigorous adherence to best practices to avoid the "configuration drift" common in manual deployments. I established several mandatory patterns for Kustomize base and overlay manifests:

*   **Metadata and Selectors:** I standardized on the `app.kubernetes.io` label set. Labels like `app.kubernetes.io/name`, `component`, and `part-of` are used consistently across Deployments and Services. I ensured that Deployment selectors exactly match the pod template labels to prevent orphaned pods—a common issue when manually editing manifests.
*   **Networking and Probes:** A "Service on 80, App on 8080" pattern was implemented, where the Service `port: 80` maps to a `targetPort: 8080`. Every container now explicitly declares its `containerPort: 8080` and utilizes the Micronaut `/health` endpoint for liveness and readiness probes, with a measured `initialDelaySeconds` of 30–50s to account for JVM startup overhead.
*   **Environment Configuration:** Sensitive data is strictly externalized via Kubernetes Secrets. I used Micronaut’s `MICRONAUT_ENVIRONMENTS` environment variable to trigger profile-specific behavior (e.g., `local-dev`), while environment-specific ConfigMaps are merged to override defaults.

Looking ahead to the Go migration, I faced a structural challenge: how to run Java and Go versions of the same service side-by-side without creating a mess of manifests. I chose a directory structure that separates these implementations at both the Kustomize base and overlay levels. For the `auth` service, the legacy Java manifests reside in `k8s/apps/svc/auth/base/legacy/`, while the new Go implementation occupies the parent `k8s/apps/svc/auth/base/`.

This layout enables a pragmatic "Strangler Fig" approach; I can deploy either version independently or both simultaneously by targeting the specific overlay. When the legacy gateway is eventually replaced by Envoy Proxy, traffic routing between these versions will be managed via Ingress rules rather than invasive manifest changes, ensuring the underlying infrastructure remains stable and operationally transparent.

### Automated Orchestration: Standardizing the Service Lifecycle

Deployments only stay reliable when they are automated end-to-end. To automate the five-step lifecycle mentioned above and to enable repeatable builds for Java services, a set of Makefile Docker targets was created. These handle both individual service builds and bulk operations across the entire legacy stack:

* `docker-java-svc-build` – Builds a Docker image for a specific Java service.
* `docker-java-svc-all-build` – Builds Docker images for all Java services.
* `docker-frontend-build` – Builds the Vue frontend image.

To streamline service deployment across Kind (local-dev) and LXD/K3s (QA) environments, a dedicated set of Makefile targets was created for loading and unloading service images and orchestrating Kustomize deployments:

* `svc-image-local-dev-load` – Loads a locally built image into the local-dev Kind cluster.
* `svc-image-local-dev-unload` – Removes an image from Kind cluster nodes.
* `svc-image-qa-load` – Loads an image into QA LXD cluster nodes.
* `svc-deploy` – Deploys a service via Kustomize overlay.
* `svc-delete` – Purges a service deployment.

Implementing a multi-namespace architecture for the Insurance Hub—where infrastructure components like PostgreSQL and MinIO reside in `qa-data` while services occupy `qa-svc`—introduced a specific challenge: secret accessibility. During the infrastructure deployment phase, service-specific credentials, such as MinIO access keys or Postgres user secrets, are generated within the infrastructure’s own namespace to keep them local to the resource they protect. However, Kubernetes strictly enforces namespace boundaries, preventing a Deployment in `qa-svc` from directly mounting a Secret stored in `qa-data`. I had to decide whether to implement complex RBAC for cross-namespace ServiceAccount access or adopt a more pragmatic, portable solution.

I chose to handle this by implementing "copy-on-create" or "copy-after-create" patterns within the project’s Make targets. As seen in the `minio-svc-user-secret-create` target, the process first creates the master secret in the infrastructure namespace and then immediately pipes the YAML through `sed` to re-apply it to the service namespace. I opted for this approach because it maintains a simpler RBAC model and avoids the configuration complexity of cross-namespace secret references introduced in newer Kubernetes versions. This ensures that the secret’s lifecycle remains within the control of the service deployment while matching our existing Makefile automation patterns, providing a consistent and reproducible setup across all environments from `local-dev` to `qa`.

Before starting on implementing the deployment of the `agent-portal-gateway`, I established the migration sequence. The migration should start with core auth and edge, then move outward to lower‑risk or less-central services, wired via Kustomize overlays for `local-dev` and `qa`.

| Sequence | Category                | Service                 | Description                                                                                 |
|:---------|:------------------------|:------------------------|:--------------------------------------------------------------------------------------------|
| 1        | **Gateway & Auth**      | `agent-portal-gateway`  | Single entry point to backend services; enables early routing smoke tests.                  |
| 2        | **Gateway & Auth**      | `auth-service`          | Required for most flows; external clients depend on it.                                     |
| 3        | **Gateway & Auth**      | `web-vue` (frontend)    | Validates the full browser → gateway → auth path once core edge/auth are stable.            |
| 4        | **Supporting Services** | `document-service`      | Relatively isolated; not critical to core policy/payment flows.                             |
| 5        | **Supporting Services** | `product-service`       | Provides reference data; used by others but off the main transaction path initially.        |
| 6        | **Supporting Services** | `policy-search-service` | Read-only over Elasticsearch, no critical writes.                                           |
| 7        | **Supporting Services** | `dashboard-service`     | Primarily read-heavy; safe once upstreams (policy/product/search) are in place.             |
| 8        | **Supporting Services** | `chat-service`          | Can be validated independently (WebSocket + API).                                           |
| 9        | **Core Transactional**  | `policy-service`        | Central domain logic; depends on product, document, and search.                             |
| 10       | **Core Transactional**  | `payment-service`       | Financially critical; should follow a stable policy path and upstreams.                     |
| 11       | **Core Transactional**  | `pricing-service`       | Critical but narrower; depends on product/policy and can be proven via internal APIs first. |

With the decommissioning of Consul, service discovery has transitioned to a Kubernetes-native model, leveraging the cluster's internal DNS (CoreDNS). In this architecture, manual registration is replaced by deterministic FQDNs following the standard `<service>.<namespace>.svc.cluster.local` pattern. This shift allows services to resolve their dependencies without an external agent; for instance, the `payment-service` in the `local-dev` environment now reaches its database at `local-dev-postgres-payment-rw.local-dev-all.svc.cluster.local` and its Kafka bootstrap server at `local-dev-kafka-kafka-bootstrap.local-dev-all.svc.cluster.local`.

I chose to implement these mappings through Kustomize `configMapGenerator` literals, which inject the correct DNS endpoints directly into the services' environment variables. By using these cluster-internal records, I’ve eliminated the overhead of managing a separate discovery sidecar while gaining the reliability of Kubernetes' built-in service abstractions. This approach is not only more robust but also operationally cleaner, as a simple `kubectl get svc -A` combined with a bit of `awk` formatting provides a complete, real-time map of the cluster's internal routing surface. For example, auth service related internal DNS:

```shell
kubectl get svc -A --no-headers | awk '{printf "%s.%s.svc.cluster.local\n", $2, $1}'
local-dev-auth-api-legacy.local-dev-all.svc.cluster.local
local-dev-postgres-auth-r.local-dev-all.svc.cluster.local
local-dev-postgres-auth-ro.local-dev-all.svc.cluster.local
local-dev-postgres-auth-rw.local-dev-all.svc.cluster.local
```

The technical "recipe" for a service deployment is best exemplified by the `document-service` configuration in the QA environment. Within the `k8s/overlays/qa/svc/document/legacy` folder, the `kustomization.yaml` acts as the primary orchestrator. I configured it to target the `qa-svc` namespace and apply a `qa-` name prefix and `-legacy` suffix, ensuring that these resources are distinct from the future Go implementation. The accompanying `deployment-patch.yaml` provides the environment-specific "glue"—it defines the resource requests and limits tailored for the K3s cluster and mounts the necessary PostgreSQL and MinIO secrets. These secrets are accessed locally within the `qa-svc` namespace, leveraging the "copy-on-create" Makefile automation to maintain strict namespace isolation for the underlying data stores.

The wiring of external dependencies is handled through a `configMapGenerator` in the overlay, which injects the internal DNS FQDNs directly into the service's environment. For the `document-service`, this involves mapping variables like `MINIO_TENANT_ENDPOINT` and `PG_HOST` to their respective cluster-internal records, such as `qa-postgres-document-rw.qa-data.svc.cluster.local`. I also used this layer to manage environment-specific toggles, such as enabling Zipkin tracing and defining the `JSREPORT_HOST` endpoint. By centralizing these connection strings and resource adjustments in the overlay, the base manifests remain generic and portable, allowing the deployment logic to scale across environments without invasive changes to the core application templates.