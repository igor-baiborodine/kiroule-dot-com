---
title: "Lift Completed, Now Shift: Making the Insurance Hub Kubernetes-Native (Enough)"
date: 2026-02-05T08:00:00-04:00

categories: [ "Java", "Go" , "Write-up" ]
tags: [ "Java-to-Go", "Kubernetes", "MinIO", "S3", "PostgreSQL", "Kustomize", "Service Discovery", "Kind", "K3s" ]
toc: false
series: [ "Insurance Hub: The Way to Go" ]

author: "Igor Baiborodine" 
---

The Insurance Hub migration continues.
Following [Phase 1's "lift"](/article/from-docker-compose-to-kubernetes-lifting-the-insurance-hub-into-the-cloud/)
—provisioning clusters and infrastructure—we are now focusing on the "shift" required to make the
legacy system run on this new foundation. This article summarizes the targeted changes needed to
move our existing Java services into Kubernetes.

<!--more-->

This “shift” is intentionally pragmatic: services are not being rewritten yet, but the biggest
blockers preventing the legacy Java stack from behaving like a well-mannered Kubernetes workload are
being removed. This begins with storage and state. Services currently writing to the local
filesystem—or hiding blobs in the database—need to speak S3 instead, using a unified, S3-compatible
SDK pointed at the MinIO tenants. In parallel, the convenience of in-memory H2 will be replaced with
real PostgreSQL instances running in the cluster, ensuring that each service can be validated with
the same persistence model it will use beyond a developer laptop.

With those foundations in place, the rest of the work will be focused on making the platform
operationally boring (the highest compliment in infrastructure). Container images will be validated
and tightened, each microservice will be deployed with a repeatable Kustomize base plus environment
overlays, and Consul will be decommissioned in favor of Kubernetes-native service discovery (
internal DNS). Finally, the existing `agent-portal-gateway` service will be brought into the cluster
as the initial entry point, and its routes will be wired to the new service endpoints. The guiding
principle throughout is “minimal, configuration-driven change”: everything should be reproducible
via Make targets and verifiable locally by running services from IntelliJ while they interact with
MinIO/Postgres/MongoDB/Kafka from the `local-dev` Kind-based cluster, and verifiably exercised
through a properly functioning application in both the `local-dev` and `qa` Kubernetes environments.

{{< toc >}}

### Shift Scoping: Prioritizing Deliverables and Networking Surface

While the previous "lift" phase was characterized as infra-first and largely app-agnostic, the 
"shift" marks the transition to an **app-first, infra-consuming** approach. The foundational cluster
work is complete; now, the legacy Java microservices must be modified to utilize these new
cloud-native resources. This requires a deliberate move away from local-only shortcuts—such as
in-memory databases and local file writes—toward durable, cluster-managed services.

To maintain momentum without introducing excessive instability, the shift has been organized into
four primary deliverables. I have prioritized storage and state first, followed by networking and
service discovery, ensuring that the most invasive changes are validated before the final
application cutover.

| Ticket                                                                                                                                 | Deliverable                             | Description                                                                                                                       |
|:---------------------------------------------------------------------------------------------------------------------------------------|:----------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------|
| **[7](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054538&issue=igor-baiborodine%7Cinsurance-hub%7C12)**  | **MinIO/S3 Integration**                | Modify `payment-service` and `documents-service` to use S3-compatible SDKs for file operations against MinIO.                     |
| **[8](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054674&issue=igor-baiborodine%7Cinsurance-hub%7C13)**  | **PostgreSQL External Persistence**     | Transition services from in-memory H2 to dedicated PostgreSQL clusters running in the Kubernetes environments.                    |
| **[9](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054790&issue=igor-baiborodine%7Cinsurance-hub%7C14)**  | **Gateway in K8s**                      | Deploy the `agent-portal-gateway` as the initial ingress point, establishing the external routing boundary.                       |
| **[10](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054853&issue=igor-baiborodine%7Cinsurance-hub%7C15)** | **Per-service deploy + Consul removal** | Decommission Consul; migrate inter-service communication to Kubernetes-native DNS and finalize per-service Kustomize deployments. |

The rationale for this risk-ordering is pragmatic: storage and database connectivity represent the
highest points of failure. By resolving these first, the backend services can be validated as stable
workloads. Once established, inter-service discovery and gateway routing are configured to wire the
system together into a functional mesh.

Supporting a productive local development loop while interacting with a remote or Kind-based cluster
requires careful planning of the networking surface. Since all Micronaut-based Java services default
to port `8080`, running multiple services locally via an IDE would inevitably lead to port
exhaustion and conflicts.

To mitigate this, a deterministic port assignment table was established. These mappings are
implemented through `application-local.yml` configuration files, ensuring each service has a
non-conflicting home on the developer's localhost.

| Service             | Local Port | Service                 | Local Port |
|:--------------------|:-----------|:------------------------|:-----------|
| `auth-service`      | 8081       | `payment-service`       | 8086       |
| `chat-service`      | 8082       | `pricing-service`       | 8087       |
| `dashboard-service` | 8083       | `product-service`       | 8088       |
| `document-service`  | 8084       | `policy-search-service` | 8089       |
| `policy-service`    | 8085       | `agent-portal-gateway`  | 8090       |

A similar logic was applied to the stateful components running inside the cluster. Because a 
"dedicated cluster per microservice" approach was chosen for PostgreSQL, unique local ports must be
exposed to allow IDE-based services to reach their respective data stores.

| Postgres Service                     | Cluster Port Mapping  |
|--------------------------------------|-----------------------|
| `svc/local-dev-postgres-auth-rw`     | 5432 → localhost:5442 |
| `svc/local-dev-postgres-document-rw` | 5432 → localhost:5452 |
| `svc/local-dev-postgres-payment-rw`  | 5432 → localhost:5462 |
| `svc/local-dev-postgres-policy-rw`   | 5432 → localhost:5472 |
| `svc/local-dev-postgres-pricing-rw`  | 5432 → localhost:5482 |

Finally, since two MinIO tenants were used (document and payment), local development MinIO endpoints
were also assigned non-overlapping ports so that each service could be tested against its tenant
deterministically during the S3 migration work.

| MinIO Tenant Service              | Cluster Port Mapping  |
|-----------------------------------|-----------------------|
| `svc/local-dev-minio-document-hl` | 9000 → localhost:9001 |
| `svc/local-dev-minio-payment-hl`  | 9000 → localhost:9002 |

This upfront organization ensures that the environment's infrastructure connectivity remains
transparent, and the focus stays on the code modifications required for the shift.

### Pragmatic Persistence: Adapting Storage for S3 and MinIO

The transition of stateful workloads began
with [Ticket 7](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054538&issue=igor-baiborodine%7Cinsurance-hub%7C12),
which centered on moving the storage layer from local filesystems and database blobs to
S3-compatible object storage. I targeted the `documents-service` and `payment-service` first, as
they represented the most significant dependencies on local persistence. Before applying any code
changes, I needed a stable local development loop; I added `application-local.yml` configurations to
ensure services could run in IntelliJ while interacting with the MinIO tenants in the `local-dev`
Kind cluster.

While Consul decommissioning was slated for a later phase, initial testing revealed an immediate
bottleneck. The `micronaut-discovery-client` was triggering startup failures by searching for a
non-existent server. I made the pragmatic choice to prune these dependencies immediately to unblock
the S3 refactoring.

Provisioning the MinIO tenants required more than just creating buckets; I chose to strictly adhere
to MinIO best practices to ensure security and scalability. This involved implementing
purpose-driven naming conventions, granular bucket sharding, and enforcing the principle of least
privilege through dedicated IAM policies. While these configurations can be managed via the MinIO
Console UI, doing so manually across `local-dev` and `qa` environments is error-prone and
inconsistent. To ensure deployments remained reproducible and "boring," I automated the creation of
buckets, service users, and Kubernetes secrets via
new [Makefile targets](https://github.com/igor-baiborodine/insurance-hub/blob/3bab17d580e9baa5702d67715e60301bb749e33e/k8s/Makefile#L624):

* `minio-svc-bucket-create` - Provisions a purpose-specific bucket for a microservice.
* `minio-svc-user-secret-create` - Generates the Opaque Kubernetes secret for service credentials.
* `minio-svc-user-with-policy-create` - Creates the MinIO user and attaches the necessary S3 IAM policy.

In the Kotlin-based `documents-service`, I deliberately chose an architectural approach to avoid a
database migration. To keep the [PolicyDocument](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/documents-service/src/main/kotlin/pl/altkom/asc/lab/micronaut/poc/documents/domain/PolicyDocument.kt)
class unchanged, I adapted the `bytes` field to serve a dual purpose: storing raw PDF binary data
for legacy records or storing the UTF-8 encoded MinIO object key for new documents. I encapsulated
this logic in a new [PolicyDocumentService](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/documents-service/src/main/kotlin/pl/altkom/asc/lab/micronaut/poc/documents/domain/PolicyDocumentService.kt) 
that uses a regex-based pattern check to determine whether to return the database bytes directly or 
fetch the content from S3 via the `MinioClient`.

The Java-based `payment-service` underwent a similar overhaul. I refactored the `InPaymentRegistrationService` to replace legacy `java.io` logic with S3 operations, using the MinIO SDK to verify the existence of the bank statement and stream the CSV content for processing. After a successful import, files are now marked as processed by copying them to a "processed" prefix within the bucket. I also simplified the `BankStatementFile` class, stripping out file-manipulation logic to focus solely on object key construction. Both services were then integrated with a `MinioClientFactory` for client lifecycle management, completing the shift toward a storage-agnostic architecture. For a closer look at the diffs and testing evidence, consult the [associated pull request](https://github.com/igor-baiborodine/insurance-hub/pull/46).

### Externalizing Relational Layer: From In-Memory H2 to Managed PostgreSQL

The transition from in-memory persistence to cluster-managed data stores, as outlined
in [Ticket 8](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054674&issue=igor-baiborodine%7Cinsurance-hub%7C13),
required a systematic overhaul of how the legacy Java services bootstrap their database connections.
I implemented a consistent architectural pattern across the `auth`, `documents`, `payment`,
`policy`, and `pricing` services by introducing a [PostgresDatasourceConfigurationListener](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/auth-service/src/main/java/pl/altkom/asc/lab/micronaut/poc/auth/config/PostgresDatasourceConfigurationListener.java) 
and a corresponding [PostgresDatasourceProperties](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/auth-service/src/main/java/pl/altkom/asc/lab/micronaut/poc/auth/config/PostgresDatasourceProperties.java)
class. These components leverage Micronaut’s `BeanCreatedEventListener` to intercept the
`DatasourceConfiguration` at runtime—dynamically building the JDBC URL while supporting configurable
SSL modes and environment-specific credentials.

The configuration layer was split to support both cloud-native portability and a friction-free local
developer experience. In the main [application.yml](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/auth-service/src/main/resources/application.yml)
files, I replaced hardcoded H2 connection strings with environment variable placeholders such as
`${PG_HOST}` and `${PG_PORT}`. To allow Micronaut to still bootstrap the `DataSource` and JPA beans,
I kept a minimal default configuration that the listener subsequently overrides. Meanwhile, the new 
[application-local.yml](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/auth-service/src/main/resources/application-local.yml)
files define the specific port sharding mentioned earlier—mapping each service to its respective
local port (e.g., 8081 for `auth`, 8087 for `pricing`) and its dedicated PostgreSQL forward.

In the `auth-service`, I made a technical trade-off by refactoring the [InsuranceAgent](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/auth-service/src/main/java/pl/altkom/asc/lab/micronaut/poc/auth/InsuranceAgent.java) 
component from a Java record to a standard `@MappedEntity` class using Lombok’s `@Data` and
`@NoArgsConstructor`. While records provide a clean syntax, the move to a persistent PostgreSQL
schema necessitated a more traditional entity structure to ensure full compatibility with Micronaut
Data JDBC and JPA mapping. I also implemented an explicit `availableProductCodes` helper to handle
serialization of semicolon-separated strings stored in the database, and updated the
[InsuranceAgentsRepository](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/auth-service/src/main/java/pl/altkom/asc/lab/micronaut/poc/auth/InsuranceAgentsRepository.java) 
to remove its hardcoded H2 dialect reference.

The Kotlin-based `documents-service` saw similar adjustments to accommodate the relational storage
of binary objects. I updated the [PolicyDocument](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/documents-service/src/main/kotlin/pl/altkom/asc/lab/micronaut/poc/documents/domain/PolicyDocument.kt)
entity to include the `@Lob` annotation and a `bytea` column definition, ensuring that document
content is handled correctly by the PostgreSQL driver. Across all services, I updated the `pom.xml`
files to include the `postgresql` driver and `micronaut-management` dependencies. Crucially, I also
removed the `micronaut-discovery-client` dependencies; even though service discovery was a later
deliverable, the transition to external persistence forced an early decommission of the Consul-based
discovery logic to prevent startup failures. For a deep dive into the code changes, see
the [associated pull request](https://github.com/igor-baiborodine/insurance-hub/pull/47).

### Deployment Readiness: Establishing Service Conventions

While the preceding "lift" work focused on standing up the cluster and its stateful dependencies,
the focus now shifts toward the deployment pipeline for the services themselves. When initially
planning the transition for the 11 legacy services, I considered creating individual tickets for
each to track granular progress. However, from a project management perspective, this threatened to
pollute the board with low-signal updates. I chose instead to consolidate the effort into two
primary deliverables: a dedicated ticket for the
`agent-portal-gateway` [Ticket 9](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054790&issue=igor-baiborodine%7Cinsurance-hub%7C14)—given
its role as the critical entry point and architectural anchor—and a grouping ticket for all
remaining services [Ticket 10](https://github.com/users/igor-baiborodine/projects/8?pane=issue&itemId=124054853&issue=igor-baiborodine%7Cinsurance-hub%7C15).
This allowed me to establish patterns on the gateway first, then apply those validated recipes to
the rest of the stack via individual pull requests.

For the Insurance Hub, a standardized implementation sequence was established to ensure consistency.
Each service follows a five-step lifecycle: validating or updating the Docker image, building
locally, loading the image into the cluster, implementing Kustomize manifests, and finally
triggering the deployment. I adopted a semantic naming convention for these images:
`insurance-hub-<service-name>-<component-type>-legacy:<tag>`. By sorting by service name first 
(e.g., `insurance-hub-policy-api-legacy` rather than `insurance-hub-api-policy-legacy`), the registry
remains organized and searchable. Initially, until a full Flux-based GitOps workflow is implemented,
all image tags default to `latest` to simplify local iteration.

The transition to Kubernetes-native manifests required a rigorous adherence to best practices to
avoid the "configuration drift" common in manual deployments. I established several mandatory
patterns for Kustomize base and overlay manifests:

* **Metadata and Selectors:** I standardized on the `app.kubernetes.io` label set. Labels like
  `app.kubernetes.io/name`, `component`, and `part-of` are used consistently across Deployments and
  Services. I ensured that Deployment selectors exactly match the pod template labels to prevent
  orphaned pods—a common issue when manually editing manifests.
* **Networking and Probes:** A "Service on 80, App on 8080" pattern was implemented, where the
  Service `port: 80` maps to a `targetPort: 8080`. Every container now explicitly declares its
  `containerPort: 8080` and utilizes the Micronaut `/health` endpoint for liveness and readiness
  probes, with a measured `initialDelaySeconds` of 30–50s to account for JVM startup overhead.
* **Environment Configuration:** Sensitive data is strictly externalized via Kubernetes Secrets. I
  used Micronaut’s `MICRONAUT_ENVIRONMENTS` environment variable to trigger profile-specific
  behavior (e.g., `local-dev`), while environment-specific ConfigMaps are merged to override
  defaults.

Looking ahead to the Go migration, I faced a structural challenge: how to run Java and Go versions
of the same service side-by-side without creating a mess of manifests. I chose a directory structure
that separates these implementations at both the Kustomize base and overlay levels. For the `auth`
service, the legacy Java manifests reside in `k8s/apps/svc/auth/base/legacy/`, while the new Go
implementation occupies the parent `k8s/apps/svc/auth/base/`.

This layout enables a pragmatic "Strangler Fig" approach; I can deploy either version independently
or both simultaneously by targeting the specific overlay. When the legacy gateway is eventually
replaced by Envoy Proxy, traffic routing between these versions will be managed via Ingress rules
rather than invasive manifest changes, ensuring the underlying infrastructure remains stable and
operationally transparent.

### Automated Orchestration: Standardizing Service Lifecycle

Deployments remain reliable only when they are automated end-to-end. To automate the five-step
lifecycle mentioned above and to enable repeatable builds for Java services, a set of 
[Makefile Docker targets](https://github.com/igor-baiborodine/insurance-hub/blob/3bab17d580e9baa5702d67715e60301bb749e33e/Makefile#L65)
was created. These handle both individual service builds and bulk operations across the entire
legacy stack:

* `docker-java-svc-build` – Builds a Docker image for a specific Java service.
* `docker-java-svc-all-build` – Builds Docker images for all Java services.
* `docker-frontend-build` – Builds the Vue frontend image.

To streamline service deployment across Kind (`local-dev`) and LXD/K3s (`qa`) environments, a
dedicated set of Makefile targets was created for loading and unloading service images and
orchestrating Kustomize deployments:

* `svc-image-local-dev-load` – Loads a locally built image into the `local-dev` Kind cluster.
* `svc-image-local-dev-unload` – Removes an image from Kind cluster nodes.
* `svc-image-qa-load` – Loads an image into QA LXD cluster nodes.
* `svc-deploy` – Deploys a service via Kustomize overlay.
* `svc-delete` – Purges a service deployment.

Implementing a multi-namespace architecture for the Insurance Hub—where infrastructure components
like PostgreSQL and MinIO reside in `qa-data` while services occupy `qa-svc`—introduced a specific
challenge: secret accessibility. During the infrastructure deployment phase, service-specific
credentials, such as MinIO access keys or Postgres user secrets, are generated within the
infrastructure’s own namespace to keep them local to the resource they protect. However, Kubernetes
strictly enforces namespace boundaries, preventing a Deployment in `qa-svc` from directly mounting a
Secret stored in `qa-data`. I had to decide whether to implement complex RBAC for cross-namespace
ServiceAccount access or adopt a more pragmatic, portable solution.

To bypass namespace boundaries without complex RBAC, I implemented "copy-on-create" and 
"copy-after-create" patterns in the Make targets. For example, the `minio-svc-user-secret-create`
target pipes the generated YAML through `sed` to re-apply it from `qa-data` to `qa-svc`. It is a
portable, low-overhead solution for secret sharing.

Before starting on implementing the deployment of the `agent-portal-gateway`, I established the
migration sequence. The migration should start with core auth and edge, then move outward to
lower‑risk or less-central services, wired via Kustomize overlays for `local-dev` and `qa`.

**Gateway & Auth**

| Seq. | Service                | Description                                                                      |
|:-----|:-----------------------|:---------------------------------------------------------------------------------|
| 1    | `agent-portal-gateway` | Single entry point to backend services; enables early routing smoke tests.       |
| 2    | `auth-service`         | Required for most flows; external clients depend on it.                          |
| 3    | `web-vue` (frontend)   | Validates the full browser → gateway → auth path once core edge/auth are stable. |

**Supporting Services**

| Seq. | Service                 | Description                                                                          |
|:-----|:------------------------|:-------------------------------------------------------------------------------------|
| 4    | `document-service`      | Relatively isolated; not critical to core policy/payment flows.                      |
| 5    | `product-service`       | Provides reference data; used by others but off the main transaction path initially. |
| 6    | `policy-search-service` | Read-only over Elasticsearch, no critical writes.                                    |
| 7    | `dashboard-service`     | Primarily read-heavy; safe once upstreams (policy/product/search) are in place.      |
| 8    | `chat-service`          | Can be validated independently (WebSocket + API).                                    |

**Core Transactional**

| Seq. | Service           | Description                                                                                 |
|:-----|:------------------|:--------------------------------------------------------------------------------------------|
| 9    | `policy-service`  | Central domain logic; depends on product, document, and search.                             |
| 10   | `payment-service` | Financially critical; should follow a stable policy path and upstreams.                     |
| 11   | `pricing-service` | Critical but narrower; depends on product/policy and can be proven via internal APIs first. |

With the decommissioning of Consul, service discovery has transitioned to a Kubernetes-native model,
leveraging the cluster's internal DNS (CoreDNS). In this architecture, manual registration is
replaced by deterministic FQDNs following the standard `<service>.<namespace>.svc.cluster.local`
pattern. This shift allows services to resolve their dependencies without an external agent; for
instance, the `payment-service` in the `local-dev` environment now reaches its database at
`local-dev-postgres-payment-rw.local-dev-all.svc.cluster.local` and its Kafka bootstrap server at
`local-dev-kafka-kafka-bootstrap.local-dev-all.svc.cluster.local`.

I chose to implement these mappings through Kustomize `configMapGenerator` literals, which inject
the correct DNS endpoints directly into the services' environment variables. By using these
cluster-internal records, I’ve eliminated the overhead of managing a separate discovery sidecar
while gaining the reliability of Kubernetes' built-in service abstractions. This approach is not
only more robust but also operationally cleaner, as a simple `kubectl get svc -A` combined with a
bit of `awk` formatting provides a complete, real-time map of the cluster's internal routing
surface. For example, `auth-service` related internal DNS:

```shell
kubectl get svc -A --no-headers \
  | awk '{printf "%s.%s.svc.cluster.local\n", $2, $1}' \
  | grep auth
local-dev-auth-api-legacy.local-dev-all.svc.cluster.local
local-dev-postgres-auth-r.local-dev-all.svc.cluster.local
local-dev-postgres-auth-ro.local-dev-all.svc.cluster.local
local-dev-postgres-auth-rw.local-dev-all.svc.cluster.local
```

The technical "recipe" for a service deployment is best exemplified by the `document-service`
configuration in the QA environment. Within the k8s/overlays/qa/svc/document/legacy
folder, the [kustomization.yaml](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/k8s/overlays/qa/svc/document/legacy/kustomization.yaml)
acts as the primary orchestrator. I configured it to target the `qa-svc` namespace and apply a `qa-`
name prefix and `-legacy` suffix, ensuring that these resources are distinct from the future Go
implementation. The accompanying [deployment-patch.yaml](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/k8s/overlays/qa/svc/document/legacy/deployment-patch.yaml)
provides the environment-specific "glue"—it defines the resource requests and limits tailored for
the K3s cluster and mounts the necessary PostgreSQL and MinIO secrets. These secrets are accessed
locally within the `qa-svc` namespace, leveraging the "copy-on-create" Makefile automation to
maintain strict namespace isolation for the underlying data stores.

External dependency wiring is handled by a `configMapGenerator` in the overlay, which injects the
internal DNS FQDNs directly into the service's environment. For the `document-service`, this
involves mapping variables like `MINIO_TENANT_ENDPOINT` and `PG_HOST` to their respective
cluster-internal records, such as `qa-postgres-document-rw.qa-data.svc.cluster.local`. I also used
this layer to manage environment-specific toggles, such as enabling Zipkin tracing and defining the
`JSREPORT_HOST` endpoint. By centralizing these connection strings and resource adjustments in the
overlay, the base manifests remain generic and portable, allowing the deployment logic to scale
across environments without invasive changes to the core application templates.

### Edge and Logic: Bridging the Kubernetes Gap

While the transition to S3 and PostgreSQL addressed the stateful requirements of the "shift," the
final hurdles involved the services' networking and runtime behavior. Moving from the flat network
of Docker Compose to the segmented, DNS-heavy environment of Kind and K3s surfaced several edge
cases where the legacy configuration didn't translate.

#### Standardizing NGINX Path Matching and Resolution

In the `web-vue` module, the UI itself remained healthy, but the navigation flow broke down under
Kubernetes networking. Redirects triggered during authentication—and certain API calls routed
through the gateway—failed to land the browser back on the expected routes. The root cause was NGINX
behaving subtly differently in a containerized environment compared to the simpler, earlier setup,
particularly around URI matching and upstream resolution when using variables in `proxy_pass`. In
the original configuration, proxy locations were defined broadly as `/api` and `/login`. While this
looked sufficient, it left enough ambiguity for edge cases—like trailing slash handling and route
precedence—to bounce the browser to the wrong paths or trigger unexpected fallbacks to the SPA index
route.

I tightened the configuration to make routing deterministic. First, I made the proxy locations
prefix-explicit and assigned them higher priority via `location ^~ /api/` and `location ^~ /login`.
This prevents accidental interactions with the generic SPA handler and removes the path ambiguity
that often causes upstreams to respond with unexpected redirects. Second, I introduced a
Kubernetes-aware DNS strategy using `resolver kube-dns... valid=5s` combined with
`set $api_upstream`. This forces NGINX to reliably resolve service DNS names when they are sourced
from variables—a common "gotcha" where variable-based `proxy_pass` fails to re-resolve without an
explicit resolver, manifesting as flaky routing.

Finally, I addressed a related UX issue by adding a map for `$connection_upgrade` and a dedicated
`^~ /ws/chat/` location. Including the necessary WebSocket headers fixes the handshake failures that
typically occur when real-time features move behind an NGINX ingress. Collectively, these changes
transformed the UI into a stable edge gateway with correct protocol handling and path matching,
which is exactly what a consistent login flow requires once running behind Kubernetes. For more 
details, see [nginx-app.conf](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/web-vue/nginx-app.conf).

#### Mapping Micronaut Clients to Cluster DNS

After moving the legacy Micronaut 2 `agent-portal-gateway` into the cluster, routing failures
initially appeared to be a network-level issue. The `product-service` seemed unreachable from the
gateway, even though direct in-cluster connectivity tests succeeded. This pattern eventually
surfaced across all downstreams—document, policy, and policy-search. The clue was that while
Kubernetes DNS worked, the gateway couldn't resolve logical service identifiers into actual URLs.

The root cause lay in how Micronaut’s declarative HTTP clients—configured with
`@Client(id = "<service-id>")`—behave. In Docker Compose, service discovery often "just happens" via
hostnames, but in Micronaut 2, the `id` is a logical name that requires either a discovery client or
an explicit URL mapping.

Initially, I tested the Micronaut Kubernetes discovery client. However, it required extensive RBAC
permissions to watch cluster resources, introducing more complexity than it solved. I switched to
explicit service URL mappings in [application-local-dev.yml](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/agent-portal-gateway/src/main/resources/application-local-dev.yml)
and [application-qa.yml](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/agent-portal-gateway/src/main/resources/application-qa.yml)
by defining `micronaut.http.services` entries along with activating a dedicated Micronaut
environment profile via `MICRONAUT_ENVIRONMENTS` in the gateway deployment for the 
[local-dev](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/k8s/overlays/local-dev/svc/agent-portal-gateway/legacy/deployment-patch.yaml) 
and [qa](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/k8s/overlays/qa/svc/agent-portal-gateway/legacy/deployment-patch.yaml). 
It is a "dumb-but-reliable" approach that made routing deterministic without security overhead.

#### Tuning JVM Footprint and DNS Fallbacks

While validating services in the `local-dev` Kind cluster, I observed an unsettling pattern: healthy
pods would restart every 10–15 minutes. The deployments had modest memory limits—typically 512
MiB—but metrics showed the Java processes steadily climbing toward 1 GiB before being terminated.
Kubernetes was performing routine OOM enforcement, but the "mystery" was why the memory usage was so
aggressive.

I had fallen into a classic "lift-and-shift" trap: I reused the legacy Dockerfiles unchanged,
running the JVM without container-aware guardrails. A plain `java -jar` command is dangerous in
Kubernetes—it allows the JVM to size its heap based on host memory rather than pod limits. I
modernized the runtime by switching to the `eclipse-temurin:17-jre-alpine` base image and setting
container-aware JVM options, specifically `-XX:+UseContainerSupport` and `-XX:MaxRAMPercentage`which
immediatly stabilized memory consumption. As a bonus, the move to Temurin 17 significantly reduced
idle footprint—dropping typical steady-state usage from roughly 300–350 MiB down to 150–200 MiB,
which made the services far more compatible with “developer laptop” cluster constraints.

However, the Alpine-based Temurin 17 image wasn’t a universal win. For the `agent-portal-gateway`
and `policy-service`, I noticed IPv6 fallback timeouts of 30–45 seconds on outbound calls. Alpine’s
`musl libc` resolver tends to prefer IPv6 and only falls back to IPv4 after a timeout if the AAAA
path isn't satisfied. In these specific cases, I chose to retain the older
`adoptopenjdk/openjdk14:jre-14.0.2_12-alpine` base image. Its resolver behavior avoided the fallback
delay and restored instant connectivity. The result was a service-by-service compromise: Temurin 17
for memory stability where possible, and the legacy base image where musl DNS behavior proved to be
a latency trap.

### Visibility and Automation: Closing the Operational Gap

To facilitate proper application functionality and debugging, I made several targeted improvements
to visibility and deployment logic. Initially, the legacy services were "black boxes" regarding
their internal events and external interactions; however, after testing the system in Kind, it
became clear that moving to a distributed environment required far more explicit telemetry.

To improve operational transparency, I added a [LoggingFilter](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/agent-portal-gateway/src/main/java/pl/altkom/asc/lab/micronaut/poc/gateway/infrastructure/LoggingFilter.java)
to the `agent-portal-gateway`. Implementing this as a Micronaut `HttpServerFilter` allows for
non-invasive interception of all traffic passing through the gateway. By logging the HTTP method and
URI for every incoming request—and the resulting status code for every outgoing response—it provides
a clear audit trail of how the UI interacts with the backend. This was particularly useful for
debugging the NGINX redirect issues mentioned earlier; having a high-level view of every
request/response cycle within the cluster helped verify that the gateway was indeed receiving and
correctly forwarding traffic to the intended downstreams. To avoid polluting logs, the filter
explicitly ignores `/health` probe requests.

I extended this strategy to the asynchronous layer by addressing the lack of visibility into Kafka
message publishing. In the legacy `policy-service`, domain events were being emitted without any
trace in the logs, making it nearly impossible to verify if a policy update had actually been
dispatched. I chose a non-intrusive approach by implementing an 
[EventPublisherLoggingInterceptor](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/policy-service/src/main/java/pl/altkom/asc/lab/micronaut/poc/policy/infrastructure/adapters/kafka/EventPublisherLoggingInterceptor.java)
using Micronaut AOP. By creating this cross-cutting interceptor for methods annotated with
`@LogEventPublisher`, I was able to centrally capture the specific policy identifier and event type
for every outbound message. This ensures that every event—including any failures—is logged, allowing
me to confirm that messages are reaching Kafka before troubleshooting downstream consumers.

Beyond telemetry, I addressed the manual, out-of-band step of adding PDF templates to the jsreport
instance. Managing these templates manually is a fragile requirement when handling multiple
ephemeral environments. To ensure the deployment was fully automated, I added a
[JsReportTemplateProvisioner](https://github.com/igor-baiborodine/insurance-hub/blob/v0.22.0/legacy/documents-service/src/main/kotlin/pl/altkom/asc/lab/micronaut/poc/documents/infrastructure/adapters/jsreport/JsReportTemplateProvisioner.kt)
to the `document-service`. This Singleton component hooks into the Micronaut `ServerStartupEvent`
and interacts with the jsreport API to validate the presence of the "POLICY" template. If missing,
the provisioner automatically loads the definition from a local resource and creates it. This makes
the `document-service` self-bootstrapping and environment-agnostic, ensuring it functions
immediately upon deployment without manual UI configuration.

To tie the "shift" together, I transitioned from manual commands to a fully orchestrated deployment
workflow. Managing 11 legacy services and a dozen stateful components is inherently fragile without
a deterministic sequence. I consolidated the logic into high-level Makefile targets that handle
specific dependencies—ensuring operators are reconciled before data stores are provisioned, and the
observability is live before services begin emitting telemetry.

In the QA environment, I prioritized using LXD-based snapshot targets at critical milestones. By
capturing the state of the cluster nodes after standing up the monitoring and infrastructure layers,
I’ve facilitated a nearly "instant" rollback capability. This effectively makes the cluster
disposable; if a service deployment or configuration experiment causes a deadlock, I can revert to a
verified baseline in seconds rather than rebuilding the stack from zero. The automated deployment
sequence now follows these primary [Makefile targets](https://github.com/igor-baiborodine/insurance-hub/blob/3bab17d580e9baa5702d67715e60301bb749e33e/k8s/Makefile#L988):

- `java-all-build` - Orchestrates the sequential build of all legacy Java services and their Docker
  images.
- `cluster-qa-monitoring-deploy` - Provisions the production-like observability stack (
  Elasticsearch, Prometheus, Grafana, and Zipkin).
- `qa-nodes-snapshot` - Captures a named recovery point of the cluster VMs for rapid recreation.
- `cluster-infra-deploy` - Deploys the foundational infrastructure, including Kafka, PostgreSQL
  clusters, and MinIO tenants.
- `cluster-svc-deploy` - Executes the final application rollout, automating image loading and
  Kustomize-based deployment.

### AI Implementation: Optimizing Service Recipes and Agent-Assisted Refactoring

The "shift" work reinforced a lesson from the previous phase: AI is an effective accelerator for
research and implementation, provided it is governed by a disciplined routine. During this stage,
the AI tools were instrumental in establishing service conventions and finalizing the technical 
"recipe" for my K8s workloads. Once a pattern was validated for a single service—covering Kustomize
bases, environment-specific overlays, and resource constraints—producing the manifests for the
remaining ten became a routine of feeding the AI assistant the specific service context and the
established template.

My current workflow relies on a multi-tool strategy to balance cost and capability. For general
implementation and development within the IDE, I primarily use the JetBrains AI Assistant in chat
mode, configured with the Gemini 3 Flash model. This provides a pragmatic balance of speed and
response quality. While JetBrains recently increased the monthly Pro credits from 10.00 to 12.46,
this remains a finite resource that requires careful management. When these credits are exhausted,
or when I need to perform deep-dive technical research into Kubernetes idioms or SDK behaviors, I
switch to Perplexity Pro. While Perplexity is superior for synthesizing documentation, I find it
more cumbersome for implementation tasks, as source code files must be manually attached to the
prompt.

I also explored the more autonomous "agent" modes available within the IDE. While JetBrains offers
Junie and Claude-based agents, their high credit consumption initially made me hesitant to use them
for routine tasks. However, I took advantage of a recent promotion for the OpenAI Codex-based agent
to implement several cross-cutting service improvements. I used the agent to scaffold and refine the
`LoggingFilter` for the gateway, the AOP-based `EventPublisherLoggingInterceptor` for Kafka
visibility, and the `JsReportTemplateProvisioner` logic.

The AI was particularly effective at resolving the "stubborn" issues that surfaced during cluster
validation—such as the NGINX path matching inconsistencies and the Micronaut 2 discovery client RBAC
restrictions mentioned earlier. By providing the AI with the specific error logs and my existing
manifests, I was able to quickly iterate on the `application-<env>.yml` mappings and the `resolver`
configurations. As with the "lift" phase, I continue to maintain a transparent log of these
interactions, ensuring that while the AI assists with the heavy lifting of boilerplate and
debugging, the architectural rationale remains firmly under my control.

### Shift Work: Closing Thoughts

The completion of the "shift" marks a significant milestone in the Insurance Hub modernization: the
legacy Java stack is no longer a guest in Kubernetes but a well-mannered citizen. By stripping away
local-only shortcuts—moving from H2 to external PostgreSQL and from local filesystems to
S3-compatible MinIO—I’ve removed the primary blockers to scalability and state persistence. This
phase was intentionally pragmatic; the goal wasn't to achieve architectural perfection, but to
ensure the existing services remained stable, observable, and reproducible within their new
environment.

The technical "recipe" established during this phase—standardized Kustomize overlays, deterministic
port assignments, and explicit Micronaut client mappings—has transformed the deployment process into
something operationally boring. I chose to prioritize consistency over cleverness, favoring explicit
URL mappings and "copy-on-create" secret automation via Make targets over more complex discovery
agents or cross-namespace RBAC configurations. This approach ensured that the transition from Docker
Compose to Kind and K3s was driven by observed behavior and logs rather than assumptions about
framework-level "magic."

Validating this shift required a rigorous local development loop. Being able to run services
directly from IntelliJ while they interact with cluster-managed infrastructure provided the
immediate feedback necessary to resolve the NGINX path-matching flakiness and the JVM memory
footprint issues. With the addition of the `LoggingFilter` and the
`EventPublisherLoggingInterceptor`, the system has moved from a "black box" to a transparent mesh
where requests and Kafka events are traceable across the distributed environment.

While the current system is stable and functional, it still relies on a manual "push" model for
deployments. Maintaining 11 services and their supporting infrastructure through individual Make
commands is a liability I intend to eliminate as the project scales. In the next post, I will
introduce **Flux-based GitOps**. By moving to a declarative, pull-based model, the cluster state
will finally be anchored in Git—enabling automated deployments to the QA environment whenever
changes are merged into the repository and establishing a robust, version-controlled release
workflow directly from GitHub.

Continue reading the series ["Insurance Hub: The Way to Go"](/series/insurance-hub-the-way-to-go/):
{{< series "Insurance Hub: The Way to Go" >}}
