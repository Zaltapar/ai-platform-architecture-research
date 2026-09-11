# Wave A.3 — Additional subsystem forensic summary

## 1. Scope and evidence posture

This wave investigated exactly one source-available repository per requested subsystem category, without re-investigating the ten Wave A.1/A.2 product candidates. The repository source is primary evidence; claims are classified as **FACT**, **INFERENCE**, or **UNKNOWN**. Detailed evidence is in:

- [`subsystem-litellm.md`](subsystem-litellm.md)
- [`subsystem-mem0.md`](subsystem-mem0.md)
- [`subsystem-mcp-python-sdk.md`](subsystem-mcp-python-sdk.md)
- [`subsystem-qdrant.md`](subsystem-qdrant.md)
- [`subsystem-temporal.md`](subsystem-temporal.md)

**FACT:** All five selected repositories were shallow-cloned under [`research/repos`](../../repos:1). LiteLLM initially hit Windows long-path checkout failures in test fixtures; Git long-path configuration allowed the source checkout to be repaired. No source files in the selected repositories were modified.

## 2. Selected projects and rejected alternatives

| Category | Selected | Repository | Rejected/alternative | Selection reason |
|---|---|---|---|---|
| Model routing/provider abstraction | LiteLLM | [`repos/litellm`](../../repos/litellm:1) | Portkey OSS | LiteLLM supplied both SDK and gateway source, provider handlers, router/fallback logic, budgets, persistence, tests, and deployment assets. |
| Memory/agent state | Mem0 | [`repos/mem0`](../../repos/mem0:1) | Zep/Graphiti | Mem0 exposed complete add/search/update/history/delete semantics, scoped identities, factories, vector adapters, server auth, and tests in one source-visible checkout. |
| MCP runtime/tool integration | Official MCP Python SDK | [`repos/mcp-python-sdk`](../../repos/mcp-python-sdk:1) | Microsoft MCP gateway/server framework | The SDK gives the narrowest protocol/runtime boundary and avoids evaluating a product gateway as though it were the protocol itself; it has server/client/transports/auth/extensions/tests. |
| RAG/indexing/vector | Qdrant | [`repos/qdrant`](../../repos/qdrant:1) | Haystack, Weaviate, Unstructured | Qdrant most clearly exposes a stateful vector-service boundary: point/payload/query APIs, collections, shards, WAL/segments, snapshots, RBAC, and distributed storage. |
| Workflow/background execution | Temporal | [`repos/temporal`](../../repos/temporal:1) | Hatchet | Temporal provided the deepest durable-execution evidence: workflow history, task queues, namespaces, persistence factories, retries, service roles, test server, and production-scale operational structure. |

**INFERENCE:** The rejected alternatives remain viable future implementation comparisons. They were not cloned because each selected project better isolates the requested architectural question in this wave. **UNKNOWN:** This wave does not establish that each selected project is superior in every benchmark, license, community, or commercial dimension.

## 3. Comparative scores

Scale: 0–5, where higher is better for modularity, API quality, replaceability, production readiness, multi-tenancy, and extensibility. For **operational complexity**, higher means more operational burden.

| Project | Modularity | API quality | Replaceability | Production readiness | Multi-tenancy | Extensibility | Operational complexity |
|---|---:|---:|---:|---:|---:|---:|---:|
| LiteLLM | 4 | 5 | 4 | 4 | 3 | 5 | 3 |
| Mem0 | 4 | 4 | 4 | 3 | 2 | 4 | 2 |
| MCP Python SDK | 5 | 4 | 5 | 4 | 1 | 5 | 2 |
| Qdrant | 4 | 5 | 4 | 5 | 3 | 3 | 4 |
| Temporal | 3 | 5 | 3 | 5 | 4 | 4 | 5 |

### Score evidence and interpretation

- **LiteLLM FACT:** SDK/provider/gateway boundaries, router strategies, callbacks, and custom handlers are explicit in [`ARCHITECTURE.md`](../../repos/litellm/ARCHITECTURE.md:9), [`router.py`](../../repos/litellm/litellm/router.py:681), and [`custom_handler.py`](../../repos/litellm/litellm/proxy/example_config_yaml/custom_handler.py:8). **INFERENCE:** Gateway persistence/auth/budget coupling lowers replaceability from a perfect score.
- **Mem0 FACT:** Provider/vector factories and public memory lifecycle APIs exist in [`factory.py`](../../repos/mem0/mem0/utils/factory.py:35) and [`server/main.py`](../../repos/mem0/server/main.py:367). **INFERENCE:** Caller-derived identity filters and incomplete tenant authority lower multi-tenancy and production-readiness scores.
- **MCP SDK FACT:** `MCPServer`, extension interfaces, `EventStore`, transports, auth providers, and tests are explicit in [`server.py`](../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:157) and [`streamable_http.py`](../../repos/mcp-python-sdk/src/mcp/server/streamable_http.py:116). **INFERENCE:** It scores high on replaceability because it does not own application persistence; its multi-tenancy score is intentionally low because it does not provide it.
- **Qdrant FACT:** Typed REST/gRPC point/query contracts and storage/RBAC layers are present in [`schema.rs`](../../repos/qdrant/lib/api/src/rest/schema.rs:1210) and [`auth.rs`](../../repos/qdrant/lib/storage/src/rbac/auth.rs:7). **INFERENCE:** It is production-ready infrastructure but operationally complex and not an application tenant authority.
- **Temporal FACT:** Durable workflows, persistence factories, namespaces, retries, and service roles are source-visible in [`fx.go`](../../repos/temporal/temporal/fx.go:126), [`retrypolicy.go`](../../repos/temporal/common/backoff/retrypolicy.go:34), and [`README.md`](../../repos/temporal/README.md:21). **INFERENCE:** It is highly production-ready but carries the highest operational complexity and migration cost.

## 4. Recommended ownership boundaries

### Platform-owned control plane

**INFERENCE:** The future platform should own authenticated user/workspace/project identity, membership and authorization, API keys/secrets references, conversation/message domain data, consent/retention/deletion policy, billing/credits, storage quotas, user-visible job state, audit correlation, and canonical document metadata.

### LiteLLM boundary: model access plane

**FACT:** LiteLLM’s provider normalization, router/fallback, budget, callback, and cost surfaces are source-visible in [`router.py`](../../repos/litellm/litellm/router.py:681), [`schema.prisma`](../../repos/litellm/schema.prisma:12), and [`ARCHITECTURE.md`](../../repos/litellm/ARCHITECTURE.md:60). **RECOMMENDATION/INFERENCE:** Own LiteLLM as a model-access service or SDK adapter. Treat platform tenant identity as authoritative and map it to LiteLLM team/project/key context; do not make LiteLLM’s schema the canonical product tenant database.

### Mem0 boundary: long-term memory engine

**FACT:** Mem0 has add/search/get/update/history/delete and scoped user/agent/run identity in [`server/main.py`](../../repos/mem0/server/main.py:367) and [`memory/main.py`](../../repos/mem0/mem0/memory/main.py:134). **RECOMMENDATION/INFERENCE:** Own memory policy, access checks, consent, retention, and project-to-scope mapping in the platform; call Mem0 through a narrow adapter. Derive filters from authenticated context rather than accepting arbitrary client filters.

### MCP SDK boundary: protocol/runtime adapter

**FACT:** MCP server tools/resources/prompts, transports, auth, extension handlers, and event-store interfaces are explicit in [`server.py`](../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:157) and [`streamable_http.py`](../../repos/mcp-python-sdk/src/mcp/server/streamable_http.py:116). **RECOMMENDATION/INFERENCE:** Keep MCP behind a platform gateway/adapter that owns server registration, project permissions, tool approval, credentials, rate limits, audit events, and execution isolation.

### Qdrant boundary: vector/index infrastructure

**FACT:** Qdrant stores points/vectors/payloads, exposes search/query APIs, and enforces collection operation access in [`schema.rs`](../../repos/qdrant/lib/api/src/rest/schema.rs:1215) and [`ops_checks.rs`](../../repos/qdrant/lib/storage/src/rbac/ops_checks.rs:27). **RECOMMENDATION/INFERENCE:** The RAG service should own parsing, chunking, embedding version, document status, citation metadata, and tenant authorization; Qdrant should receive stable IDs and derived filterable payloads through an internal repository interface.

### Temporal boundary: durable execution infrastructure

**FACT:** Temporal separates server services, persistence, matching, history, namespaces, task queues, workers, and retry policies in [`fx.go`](../../repos/temporal/temporal/fx.go:126), [`task_queue_id.go`](../../repos/temporal/common/tqid/task_queue_id.go:49), and [`retrypolicy.go`](../../repos/temporal/common/backoff/retrypolicy.go:34). **RECOMMENDATION/INFERENCE:** Temporal should own durable workflow execution and retry state; the platform should own job semantics, authorization, idempotency keys, business records, quotas, and user-facing status. Use an adapter and versioned workflow contracts; avoid server forks.

## 5. Integration guidance without deep coupling

| Selected subsystem | Safe connection pattern to shortlisted product candidates |
|---|---|
| LiteLLM | Route product model calls through an internal `ModelGateway` interface or OpenAI-compatible endpoint. Pass platform request ID, tenant/project context, model policy, and usage event correlation; keep conversation/memory/RAG state outside. |
| Mem0 | Expose `MemoryProvider.add/search/update/delete/history` to product candidates. Convert authenticated project/user identity to server-side Mem0 filters; return normalized memory records, not raw vector payloads. |
| MCP SDK | Product candidate calls a platform MCP gateway, not arbitrary SDK transports directly. Gateway resolves approved server/tool records, credentials, scopes, and audit context, then delegates protocol work to the SDK. |
| Qdrant | RAG adapter owns collection naming, payload schema, and query translation. Product candidate sees document/search/citation APIs, not Qdrant collection internals. Keep a migration path to another vector backend. |
| Temporal | Platform job service starts versioned workflows and observes status/events. Product candidates do not import Temporal server packages or write directly to Temporal persistence. |

**INFERENCE:** This arrangement allows the A1/A2 product candidates to remain replaceable product shells/orchestrators while subsystem ownership is externalized. A product candidate should integrate through platform interfaces, not bind its domain models to LiteLLM/Mem0/Qdrant/Temporal schemas.

## 6. Cross-cutting risks

1. **Tenant context propagation — FACT/INFERENCE:** LiteLLM and Mem0 accept identity/resource scopes, Qdrant accepts collection/payload access, MCP accepts auth context, and Temporal accepts namespaces; none alone proves end-to-end application tenant isolation. The platform must derive and audit every scope.
2. **Duplicate policy planes — INFERENCE:** LiteLLM budgets, Qdrant RBAC, MCP OAuth scopes, and Temporal namespace limits can conflict with platform credits, permissions, and quotas. Choose platform policy as authoritative and use subsystem limits as defense-in-depth.
3. **Retry side effects — FACT/INFERENCE:** LiteLLM retries provider calls and Temporal retries activities/workflows. Tool calls, billing reservations, memory writes, and document indexing must be idempotent or explicitly non-retryable.
4. **Schema leakage — INFERENCE:** LiteLLM Prisma tables, Mem0 payload fields, Qdrant collection names, and Temporal workflow IDs can become accidental public contracts. Hide them behind platform adapters.
5. **Operational burden — FACT/INFERENCE:** Qdrant and Temporal are stateful distributed infrastructure; LiteLLM gateway adds Redis/PostgreSQL; Mem0 needs vector/LLM dependencies. A modular architecture still carries a substantial operations bill.
6. **Upgrade drift — FACT:** MCP explicitly documents a high-level API rename in [`fastmcp.py`](../../repos/mcp-python-sdk/src/mcp/server/fastmcp.py:1); **INFERENCE:** Narrow adapters and contract tests are required across all five projects, especially for protocol/provider versions.
7. **Data privacy — INFERENCE:** Memory payloads, prompts, provider logs, vector payloads, and workflow histories may contain sensitive customer data. Redaction, retention, encryption, residency, and deletion must be platform-wide, not delegated implicitly.

## 7. What each subsystem does not provide

- **LiteLLM FACT/UNKNOWN:** No product UI, canonical conversation/memory/RAG domain, general file tenancy, workflow engine, or complete SaaS authorization model.
- **Mem0 FACT/UNKNOWN:** No complete organization/workspace authority, document ingestion product, durable workflow queue, billing, or general agent runtime.
- **MCP SDK FACT/UNKNOWN:** No gateway registry, durable job queue, tool marketplace, sandbox, billing, vector/memory service, or application tenant policy engine.
- **Qdrant FACT/UNKNOWN:** No parsing/chunking/embedding pipeline, citations policy, conversation memory, workflow execution, billing, or product tenant lifecycle.
- **Temporal FACT/UNKNOWN:** No model routing, memory/RAG, document semantics, tool approval policy, billing, or product UI.

## 8. Impact on Wave B shortlist

**DECISION:** **This wave does not replace the Wave B product shortlist or promote a subsystem project to a product-foundation candidate.** The A1/A2 shortlist remains a product-level investigation of LibreChat, Dify, Open WebUI, and AnythingLLM, with prior ranking/uncertainties preserved in [`triage-summary-A1.md`](triage-summary-A1.md:12) and [`triage-summary-A2.md`](triage-summary-A2.md:5).

**INFERENCE:** A.3 changes Wave B’s investigation framing materially: Wave B should test whether shortlisted products can sit behind or beside explicit subsystem boundaries rather than owning provider routing, memory, vector infrastructure, MCP policy, and durable execution internally. LiteLLM, Mem0, the MCP SDK, Qdrant, and Temporal should be treated as reference/implementation candidates for replaceable services, not as additional product shortlist entries.

## 9. Unknowns and follow-up limits

- **UNKNOWN:** No load, failover, migration, cost, security penetration, or multi-replica experiments were executed in this wave.
- **UNKNOWN:** Shallow clones do not establish release cadence, historical migration compatibility, or long-term API stability.
- **UNKNOWN:** Exact license/commercial-policy implications beyond repository-visible license files were not adjudicated as a legal review.
- **UNKNOWN:** The precise integration effort with each of the ten product candidates was not re-investigated, per task scope; guidance above is boundary-level and intentionally avoids re-tracing those products.

## 10. Artifact and clone inventory

- Report: [`subsystem-litellm.md`](subsystem-litellm.md)
- Report: [`subsystem-mem0.md`](subsystem-mem0.md)
- Report: [`subsystem-mcp-python-sdk.md`](subsystem-mcp-python-sdk.md)
- Report: [`subsystem-qdrant.md`](subsystem-qdrant.md)
- Report: [`subsystem-temporal.md`](subsystem-temporal.md)
- Summary: [`subsystem-summary-A3.md`](subsystem-summary-A3.md)
- Clones: [`research/repos/litellm`](../../repos/litellm:1), [`research/repos/mem0`](../../repos/mem0:1), [`research/repos/mcp-python-sdk`](../../repos/mcp-python-sdk:1), [`research/repos/qdrant`](../../repos/qdrant:1), [`research/repos/temporal`](../../repos/temporal:1)
