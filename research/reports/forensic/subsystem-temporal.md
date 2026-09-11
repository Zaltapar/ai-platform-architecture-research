# Temporal — workflow and background execution subsystem

## Selection rationale

**FACT:** Temporal was selected because it is a mature durable-execution server rather than a lightweight queue: it persists workflow histories, dispatches tasks to workers, supports namespaces/task queues, retries activities/workflows, exposes gRPC service boundaries, and has test/development server support. The repository identifies itself as the Temporal server in [`README.md`](../../../repos/temporal/README.md:62). **INFERENCE:** It is the strongest candidate for evaluating whether commercial AI-platform background work should be delegated to an independent durable execution engine.

## Architecture and stack

**FACT:** Temporal server is Go, with API/protobufs, frontend/history/matching/worker services, persistence clients/factories, schema tools, and test infrastructure under [`service`](../../../repos/temporal/service:1), [`api`](../../../repos/temporal/api:1), [`common`](../../../repos/temporal/common:1), and [`temporaltest`](../../../repos/temporal/temporaltest:1). **FACT:** The server initializes persistence factories, cluster metadata, search-attribute mapping, telemetry, authorization, and service groups in [`fx.go`](../../../repos/temporal/temporal/fx.go:126). **FACT:** The README states that workflows, activities, and workers are implemented using supported SDKs outside this server repository in [`README.md`](../../../repos/temporal/README.md:62).

## Repository structure

**FACT:** [`service/frontend`](../../../repos/temporal/service/frontend:1) exposes client-facing workflow APIs; [`service/history`](../../../repos/temporal/service/history:1) manages workflow histories/state; [`service/matching`](../../../repos/temporal/service/matching:1) routes tasks to workers; [`service/worker`](../../../repos/temporal/service/worker:1) runs internal maintenance workflows; [`common/persistence`](../../../repos/temporal/common/persistence:1) defines persistence abstractions; [`schema`](../../../repos/temporal/schema:1) contains database schema tooling; [`temporaltest`](../../../repos/temporal/temporaltest:1) provides embedded/test server support; and [`tests`](../../../repos/temporal/tests:1) contains functional API and retry tests.

## Core data model

**FACT:** Temporal’s core domain is namespace, workflow execution/history, activity/task, task queue, worker deployment/version, visibility/search attributes, and replication metadata. Task queue partitions and persistence TTL semantics are modeled in [`task_queue_id.go`](../../../repos/temporal/common/tqid/task_queue_id.go:49). Worker deployment interfaces operate with namespace, deployment, version, build ID, task queue, and identity in [`client.go`](../../../repos/temporal/service/worker/workerdeployment/client.go:45). **FACT:** Persistence initialization uses a configured number of history shards and data stores in [`fx.go`](../../../repos/temporal/temporal/fx.go:652). **INFERENCE:** This is an execution-state database with event/history semantics, not an application business database.

## API/interface boundary

**FACT:** Temporal’s public server boundary is gRPC/protobuf-based service APIs; internal client interfaces expose namespace/workflow/task-queue/deployment operations, as shown by the worker deployment `Client` interface in [`client.go`](../../../repos/temporal/service/worker/workerdeployment/client.go:45). **FACT:** Application code uses Temporal SDKs to define workflows, activities, and workers, while this repository supplies the server according to [`README.md`](../../../repos/temporal/README.md:62). **INFERENCE:** A platform should call Temporal through a platform-owned workflow adapter and use versioned workflow/activity contracts, not import server internals.

## Deployment and persistence

**FACT:** Temporal supports configurable SQL/NoSQL persistence factories and visibility stores; server code resolves configured data stores and verifies persistence compatibility in [`fx.go`](../../../repos/temporal/temporal/fx.go:188) and [`server_impl.go`](../../../repos/temporal/temporal/server_impl.go:948). **FACT:** An embedded test server uses SQLite persistence in [`lite_server.go`](../../../repos/temporal/temporaltest/internal/lite_server.go:112), while production schema tooling includes SQL/Cassandra paths under [`schema`](../../../repos/temporal/schema:1). **INFERENCE:** Production operation requires multiple service roles, durable persistence, visibility/search configuration, namespace management, worker deployment, metrics/tracing, and upgrade/migration procedures; the SQLite test mode is not a production topology.

## Security and multi-tenancy assumptions

**FACT:** Temporal’s namespace is a first-class execution boundary used throughout persistence and service APIs; worker deployment operations receive a namespace entry in [`client.go`](../../../repos/temporal/service/worker/workerdeployment/client.go:45). **FACT:** Server initialization wires an audience getter/authorization and namespace-aware dynamic limits in [`fx.go`](../../../repos/temporal/temporal/fx.go:391) and [`service/worker/service.go`](../../../repos/temporal/service/worker/service.go:88). **INFERENCE:** Namespaces provide a meaningful execution isolation/control plane. **UNKNOWN:** Namespace isolation is not automatically equivalent to the target platform’s tenant authorization, data residency, billing, or project membership; those require platform policy and credential mapping.

## Extension/plugin model

**FACT:** Application extension is through SDK-defined workflows, activities, workers, task queues, payload/data converters, interceptors, and worker deployments; server-side persistence factories and service resolvers are injectable options in [`server_option.go`](../../../repos/temporal/temporal/server_option.go:125). **FACT:** The server contains internal worker components and maintenance workflows, but arbitrary platform business logic is not added as a server plugin. **INFERENCE:** Temporal’s intended extension model is external worker code, which is favorable for keeping platform business logic outside a fork.

## Failure, retry, and background behavior

**FACT:** Retry policy is a first-class interface with exponential, constant, conditional, and error-dependent implementations in [`retrypolicy.go`](../../../repos/temporal/common/backoff/retrypolicy.go:34). **FACT:** Activities/workflows attach retry policies, timeouts, heartbeats, child workflows, and task queues; examples are visible in [`activity_api_pause_test.go`](../../../repos/temporal/tests/activity_api_pause_test.go:138). **FACT:** Client/service retry policies are bounded or conditional in [`common/util.go`](../../../repos/temporal/common/util.go:161). **INFERENCE:** Temporal is unusually strong for durable AI ingestion, long-running agents, human approval waits, scheduled jobs, and compensating workflows, provided activities are idempotent and side effects are carefully modeled.

## Observability and testing

**FACT:** Temporal wires telemetry, metrics, tracing, logging, and service interceptors through server construction in [`fx.go`](../../../repos/temporal/temporal/fx.go:391). **FACT:** Tests cover retry policy behavior, activity option updates, security/batch APIs, workflow lifecycle, persistence fault injection, and embedded server behavior, e.g. [`retrypolicy_test.go`](../../../repos/temporal/common/backoff/retrypolicy_test.go:64) and [`acquire_shard_test.go`](../../../repos/temporal/tests/acquire_shard_test.go:89). **INFERENCE:** The execution engine has strong correctness testing, but application-specific workflow determinism/idempotency remains the integrator’s responsibility.

## Upgradeability and coupling

**FACT:** Temporal exposes SDK/gRPC contracts and persistence factory interfaces, but the server itself is a distributed stateful system with history shards, matching, frontend, persistence, visibility, cluster metadata, and schema compatibility checks in [`fx.go`](../../../repos/temporal/temporal/fx.go:188). **INFERENCE:** HTTP/API-level use through SDKs is replaceable at the platform boundary; operating or modifying the server is high-complexity shared infrastructure. Forking server internals would create a substantial long-term maintenance burden.

## Integration level and fit

- **Likely integration level:** **1 — HTTP/API/gRPC through SDKs**; **3 — shared infrastructure/database** operationally; **4 — source integration** only for server internals/custom persistence behavior.
- **Fit:** Strong independent workflow engine for durable ingestion, tool execution, approvals, retries, scheduled maintenance, and agent jobs. Platform owns workflow IDs, tenant/project authorization, business payloads, idempotency keys, usage accounting, and user-visible job state.
- **Not provided:** **FACT/UNKNOWN:** Temporal does not provide AI model routing, memory/RAG, document parsing, tenant billing, product UI, tool approval policy, or platform-specific quota enforcement. It persists execution state, not the canonical conversation/document domain.

## Modification difficulty experiments

| Experiment | Score | Reason |
|---|---:|---|
| Add a background job | 1–2 | Define workflow/activity/worker through SDK and register a task queue. |
| Durable document ingestion | 2 | Multiple activities, retry/idempotency, payload/version contracts, and external storage. |
| Human approval step | 2 | Workflow signals/updates and platform UI/API integration. |
| Per-tenant quotas/credits | 3 | Platform-side policy plus workflow admission/cancellation; namespace limits alone are insufficient. |
| Replace persistence backend | 3–4 | Factory/schema/visibility/history-shard compatibility and operational migration. |
| Replace Temporal with another engine | 4–5 | Workflow semantics, history/replay, retry, task queues, SDK contracts, and operational state differ. |

## Bottom line

**INFERENCE:** Temporal is a powerful replaceable execution service but should remain outside the product domain database and outside model/tool policy ownership. Use it behind a narrow job/workflow adapter; do not fork the server unless the platform is prepared to own a distributed systems product.
