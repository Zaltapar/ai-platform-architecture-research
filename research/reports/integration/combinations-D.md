# Wave D — Combination-by-Combination Integration Analysis

**Date:** 2026-09-11  
**Evidence basis:** Wave C/C.5 audit and reconciliation reports, Wave B forensic reports, and subsystem reports for LiteLLM, Mem0, MCP Python SDK, Qdrant, and Temporal.  
**Scope:** Product-engine composition only. This document does not authorize production implementation and does not modify cloned repositories.

## 1. Decision framework

Each claim is labelled **FACT**, **INFERENCE**, or **UNKNOWN**. Integration levels are:

- **Level 1:** HTTP/API or gRPC boundary
- **Level 2:** SDK/library boundary
- **Level 3:** shared infrastructure/database
- **Level 4:** source-code integration
- **Level 5:** fork

The platform control plane is authoritative for identity, tenant/workspace/project authorization, policy, billing, quotas, audit, canonical metadata, and deletion. Candidate products are engines, shells, sidecars, or references—not authorities by default.

## 2. Preferred composition at a glance

| Component | Classification | Decision | Preferred level | Ownership |
|---|---|---|---:|---|
| LibreChat | FOUNDATION candidate / chat-agent shell | Conditional foundation or reference shell; only with strict tenant mode and platform policy in front | 1–2 | Platform owns identity, policy, billing, canonical conversations; LibreChat may own presentation/runtime-local state during transition |
| LiteLLM | MODEL ACCESS ENGINE / sidecar | Adopt behind `ModelGateway`; do not adopt its tenant or spend schema as canonical | 1, optionally 2 | LiteLLM routes/providers/retries/cost events; platform owns entitlements and ledger |
| Mem0 | MEMORY ENGINE / sidecar | Adopt behind `MemoryProvider`; derive scope server-side | 1 or 2 | Platform owns memory policy/consent/retention; Mem0 owns extraction/search lifecycle |
| Qdrant | VECTOR STORAGE LAYER / sidecar | Adopt behind `VectorIndex` used by a platform RAG service | 1 | RAG service owns document/chunk/citation semantics and tenant filters |
| MCP Python SDK | INTEGRATION LAYER / protocol adapter | Use inside a platform MCP gateway, not directly from product clients | 2 behind a Level 1 gateway | Platform owns registry, grants, credentials, approvals, audit, rate limits, isolation |
| Temporal | WORKFLOW ENGINE / sidecar | Add for durable ingestion, approvals, schedules, and long-running agent jobs | 1/gRPC through SDK | Platform owns job semantics, idempotency, quotas, user-visible status |
| RAGFlow | SPECIALIZED RAG ENGINE / reference implementation | Conditional remote engine only behind a hard security gateway and patched deployment; not foundation | 1 | Platform owns document authorization and canonical metadata; RAGFlow owns candidate indexing/runtime |
| Dify | ORCHESTRATION/AGENT/RAG ENGINE / reference implementation | Conditional isolated engine for workflow/RAG experiments; reject as platform foundation | 1 | Platform maps identity/policy; Dify cannot own commercial RBAC/billing |
| Open WebUI | CHAT FRONTEND / single-tenant appliance | Use as optional frontend/appliance, not tenant foundation | 1 | Platform owns tenancy; disable its fragile multitenancy mode |
| LobeChat | CHAT FRONTEND / MCP reference | Use as frontend/reference for MCP patterns only unless business plane is rebuilt | 1 | Platform owns business plane; remediate JWKS before any deployment |
| Flowise | ORCHESTRATION ENGINE candidate | Conditional, evidence-limited; API-isolate if selected | 1 | Platform owns identity, jobs, policy |
| Langflow | ORCHESTRATION ENGINE / reference implementation | Conditional only after full checkout and auth/tenant audit | 1 | Platform owns identity, jobs, policy |

## 3. LibreChat + Mem0

### Verdict

**Recommendation:** Integrate Mem0 as an external memory sidecar behind a platform `MemoryProvider`; keep LibreChat’s native memory only as a migration compatibility path. **Classification:** LibreChat = foundation candidate/chat-agent shell; Mem0 = memory engine. **Integration:** Level 1 HTTP/API preferred; Level 2 SDK only inside a dedicated memory adapter.

### Evidence and rationale

- **FACT:** LibreChat has a dedicated memory model, handlers, authorization, and agent callback integration; it also has token-counted pruning, remaining-context calculations, and compaction ([`deep-librechat.md`](../forensic/deep-librechat.md:65)).
- **FACT:** Mem0 exposes add/search/get/update/history/delete and scoped `user_id`, `agent_id`, and `run_id` lifecycle semantics ([`subsystem-mem0.md`](../forensic/subsystem-mem0.md:19)).
- **FACT:** Mem0’s server accepts caller-supplied identity filters; the evidence does not establish a universal platform tenant authority ([`subsystem-mem0.md`](../forensic/subsystem-mem0.md:27)).
- **INFERENCE:** Directly enabling both memory systems creates duplicate extraction, duplicate retrieval, competing forgetting policies, and ambiguous source-of-truth semantics.
- **UNKNOWN:** Exact LibreChat-to-Mem0 event timing, duplicate-write behavior, and end-to-end latency under agent streaming were not benchmarked.

### Compatibility and data flow

`platform request context → LibreChat adapter → platform MemoryProvider → Mem0 add/search → normalized memories → context budget service → model gateway`.

The adapter maps `(tenant_id, workspace_id, project_id, user_id, conversation_id, agent_run_id)` to server-derived Mem0 scope. Client-provided filters are ignored or intersected with platform policy. Native LibreChat memory is read-only during migration, then disabled for new writes.

### Security, state, and ownership

- **Authentication:** Platform service identity to Mem0; end-user authentication terminates at platform/API. Mem0 JWT/API-key auth is defense-in-depth, not the product identity authority.
- **Authorization:** Platform policy decides read/write/delete; Mem0 receives derived immutable scope.
- **State:** Platform owns memory consent, retention, promotion, deletion ledger, and memory-to-project links. Mem0 owns extraction, vector search, update/history mechanics.
- **Memory ownership:** User/profile memory and semantic/episodic candidates may be Mem0-backed. Conversation transcript and project membership remain platform-owned. Agent state remains agent/workflow-runtime-owned.
- **RAG ownership:** Separate; do not let Mem0 become the document knowledge store.
- **Tool/MCP ownership:** Separate; memory tools are exposed by platform policy, not arbitrary Mem0 filters.

### Operational assessment

- **Latency:** One or more synchronous extraction/search calls add latency to a chat turn. Search should be bounded and optionally parallel with RAG; extraction/write should be asynchronous or post-response where semantics permit.
- **Failure modes:** Mem0 unavailable, extraction succeeds but vector write fails, duplicate retries, stale memory after deletion, or cross-scope filters accidentally broaden. Use idempotency keys and fail closed for reads; degrade to no long-term memory rather than cross-scope data.
- **Duplication:** Native LibreChat memory plus Mem0 is unacceptable as a permanent dual-write architecture.
- **Deployment:** Mem0 requires durable vector infrastructure and LLM/embedder dependencies; this is less complex than a full product engine but is an additional stateful service.
- **Debugging:** Correlate platform request ID, memory operation ID, extraction model, embedding version, and source message ID.
- **Upgradeability:** Good at HTTP boundary; higher risk if LibreChat agents depend on native memory payload shape.
- **Migration difficulty:** Medium. Backfill requires deduplication, provenance, scope mapping, and user deletion reconciliation.

### Major recommendation record

- **Recommendation:** Use Mem0 behind `MemoryProvider`, with platform-derived scope and one canonical write path.
- **Direct evidence:** Mem0 lifecycle/factory/API boundary; LibreChat memory model and compaction; Mem0 lacks full tenant authority.
- **Alternatives considered:** LibreChat native memory only; Mem0 embedded in LibreChat; RAGFlow memory; custom memory service.
- **Why it wins:** It gives a replaceable long-term memory engine without making product chat state depend on Mem0 schema; LibreChat’s native context controls remain useful.
- **Modification cost:** LOW–MEDIUM for adapter; HIGH if rewriting native agent memory semantics.
- **Integration cost:** MEDIUM due to scope mapping and dual-write migration.
- **Long-term maintenance cost:** MEDIUM; extraction prompts, vector schema, and deletion contracts must be tested.
- **Upgrade risk:** MEDIUM at API/schema boundary; HIGH if SDK internals are imported widely.
- **Confidence:** HIGH.

## 4. LibreChat + LiteLLM

### Verdict

**Recommendation:** Strongly recommended as the model-plane composition. LibreChat calls a platform `ModelGateway`; gateway calls LiteLLM over OpenAI-compatible HTTP or a narrow SDK adapter. **Integration:** Level 1 preferred; Level 2 acceptable inside gateway adapter. **Classification:** LibreChat = foundation/chat-agent shell; LiteLLM = model-access engine.

### Evidence and rationale

- **FACT:** LibreChat has explicit provider/client boundaries and supports custom/OpenAI-compatible endpoints ([`deep-librechat.md`](../forensic/deep-librechat.md:109)).
- **FACT:** LiteLLM exposes provider normalization, router/fallback, budgets, callbacks, cost metadata, and gateway persistence ([`subsystem-litellm.md`](../forensic/subsystem-litellm.md:19)).
- **FACT:** LiteLLM’s organization/team/project/budget schema is not a general SaaS tenant database ([`subsystem-litellm.md`](../forensic/subsystem-litellm.md:27)).
- **INFERENCE:** This is the lowest-coupling serious combination because model access is already a natural remote boundary and does not require shared domain tables.

### Data flow and ownership

`platform auth/policy → LibreChat or platform chat API → ModelGateway → LiteLLM router → provider → normalized stream/usage → platform ledger and response stream`.

Platform sends model policy, request ID, tenant/project correlation, context limit, allowed tools, and budget reservation token. LiteLLM may enforce provider key budgets and fallback limits, but platform entitlement remains authoritative.

- **Authentication:** Platform-to-gateway service auth; gateway-to-LiteLLM service credential or scoped virtual key.
- **Authorization:** Platform model allowlist and tenant entitlement first; LiteLLM key/model/budget policy second.
- **State:** Platform owns conversations/messages and canonical usage ledger. LiteLLM owns provider routing state, model deployment config, retry state, and optional spend records.
- **Memory/RAG/tools:** All remain outside LiteLLM.

### Operational assessment

- **Latency:** Adds a network hop and possible retry/fallback delay. Benefits include normalized providers and lower application complexity. Streaming must preserve backpressure and cancellation.
- **Failure modes:** Provider timeout, context-window rejection, retry-induced duplicate side effects, asynchronous spend-log lag, usage mismatch, or gateway outage. Model calls are generally retryable; tool calls and billing commits are not.
- **Duplication:** LiteLLM budgets plus platform credits can conflict. Use LiteLLM limits as defense-in-depth, not product entitlement truth.
- **Deployment:** LiteLLM gateway may require PostgreSQL/Redis when persistence/rate/spend features are enabled; operationally moderate.
- **Debugging:** Propagate request/tenant/project/model-attempt IDs through headers and callbacks; retain normalized provider error classes.
- **Upgradeability:** High at OpenAI-compatible HTTP boundary; lower if platform adopts LiteLLM Prisma schema.
- **Migration difficulty:** Low–medium. Provider calls can be switched behind gateway; usage reconciliation needs backfill and dual observation.

### Major recommendation record

- **Recommendation:** Make `ModelGateway` the only platform model-call boundary and use LiteLLM as its initial engine.
- **Direct evidence:** LiteLLM’s explicit gateway/router/provider seams and LibreChat’s custom provider boundary.
- **Alternatives considered:** Direct provider SDKs; Dify provider manager; LibreChat provider clients only; LiteLLM SDK embedded everywhere.
- **Why it wins:** Provider replacement, routing, retries, fallback, and cost normalization are isolated while preserving platform ownership.
- **Modification cost:** LOW for HTTP adapter; HIGH if LiteLLM schema becomes canonical.
- **Integration cost:** MEDIUM for streaming, usage, auth, and policy mapping.
- **Long-term maintenance cost:** MEDIUM; model/provider behavior and cost tables evolve.
- **Upgrade risk:** LOW–MEDIUM over HTTP; HIGH for source-level coupling.
- **Confidence:** HIGH.

## 5. LibreChat + Qdrant / RAGFlow

### 5.1 LibreChat + Qdrant through a platform RAG service

**Recommendation:** Recommended. Use LibreChat’s existing external RAG seam as a compatibility adapter to a platform-owned RAG service, which uses Qdrant. **Integration:** Level 1 HTTP/API to RAG service, Level 1 HTTP/gRPC from RAG service to Qdrant. **Classification:** LibreChat = foundation shell; Qdrant = storage layer; custom RAG service = RAG backend.

- **FACT:** LibreChat already calls an external `rag_api` for text, embedding, document operations, and retrieval ([`deep-librechat.md`](../forensic/deep-librechat.md:75)).
- **FACT:** Qdrant provides typed point/query/filter APIs but does not provide parsing, chunking, embedding, citations, or tenant lifecycle ([`subsystem-qdrant.md`](../forensic/subsystem-qdrant.md:19)).
- **INFERENCE:** This is a clean anti-corruption path: preserve LibreChat’s endpoint contract while preventing Qdrant collections from becoming application-domain contracts.
- **Data flow:** upload/object metadata → RAG workflow → parser/chunker/embedder → Qdrant points with tenant/project/version payload → filtered retrieval → citation-normalized context.
- **Auth/authorization:** Platform signs service calls; RAG service derives tenant/project filters. Users never select Qdrant collections or filters.
- **Latency:** Retrieval adds one service hop; ingestion is asynchronous. Query latency can be bounded with top-k/rerank budgets.
- **Failure:** Qdrant unavailable, embedding version mismatch, partial ingestion, stale index, or filter omission. Use document status, versioned indexes, idempotent point IDs, and fail-closed filters.
- **Duplication:** LibreChat file metadata and platform canonical document metadata must not both own lifecycle permanently. During migration, platform metadata wins.
- **Maintenance/upgrade:** Low–medium at HTTP boundary; Qdrant API is replaceable with another vector backend.

**Recommendation record:** Direct evidence = LibreChat `rag_api` seam and Qdrant API boundary. Alternatives = Open WebUI vector factory, Dify vector factory, RAGFlow. Why it wins = strongest replaceability with explicit Qdrant infrastructure boundary. Modification cost = LOW–MEDIUM. Integration cost = MEDIUM. Long-term maintenance = MEDIUM. Upgrade risk = LOW at API boundary. Confidence = HIGH.

### 5.2 LibreChat + RAGFlow

**Recommendation:** Conditional only as a specialized remote RAG engine, behind a platform RAG gateway that re-checks every document, thumbnail, artifact, and citation request. Do not share databases. **Integration:** Level 1 HTTP/API. **Classification:** RAGFlow = specialized RAG engine/reference implementation; LibreChat = shell.

- **FACT:** RAGFlow has deep parser, OCR/layout, hybrid/graph retrieval, citations, tenant-aware document/index paths, and memory taxonomy ([`deep-ragflow.md`](../forensic/deep-ragflow.md:63)).
- **FACT:** RAGFlow has confirmed cross-tenant image IDOR and thumbnail enumeration paths ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:76)).
- **FACT:** RAGFlow deletion cascade is tenant-scoped but best-effort across storage/index steps ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:137)).
- **INFERENCE:** RAGFlow is valuable where document intelligence is the differentiator, but its security defect prevents direct exposure.
- **Latency:** Rich parsing and reranking can be slower; remote chat/RAG chaining adds hops and duplicate model calls.
- **Failure:** Gateway/engine mismatch, partial delete, per-process limiter multiplication, stale tenant identity, IDOR recurrence in new endpoints.
- **Duplication:** LibreChat RAG metadata, RAGFlow KB/document state, and platform document state would overlap. Canonical platform metadata must be external; RAGFlow is a projection.
- **Debugging:** Need correlation IDs spanning upload, RAGFlow task, index, retrieval, citation, and platform response.
- **Migration:** Medium–high due to KB/document/citation shape mapping; escape hatch is the platform `RagProvider` interface.

**Recommendation record:** Direct evidence = RAGFlow RAG depth plus confirmed IDOR and per-process scaling risk. Alternatives = Qdrant-backed custom RAG; Dify RAG. Why it wins = only when advanced parsing/GraphRAG value outweighs isolation and operational risk. Modification cost = LOW for gateway patch, HIGH for a safe fork. Integration cost = HIGH. Long-term maintenance = HIGH until upstream fixes are verified. Upgrade risk = HIGH. Confidence = HIGH.

## 6. Dify combinations

### 6.1 Dify + LiteLLM

**Recommendation:** Conditional engine integration for model routing experiments, not foundation. Use Dify’s provider compatibility or OpenAI-compatible endpoint to call LiteLLM. **Integration:** Level 1 HTTP/API. **Classification:** Dify = orchestration/RAG/agent engine; LiteLLM = model sidecar.

- **FACT:** Dify has provider/plugin factories, workflows, agents, tools, native MCP, RAG, and Celery/Redis execution ([`deep-dify.md`](../forensic/deep-dify.md:29)).
- **FACT:** LiteLLM normalizes providers and retries/fallbacks ([`subsystem-litellm.md`](../forensic/subsystem-litellm.md:35)).
- **FACT:** Dify’s RBAC relies on a closed enterprise service when enabled and legacy predicates return permissively; quota is cloud-gated/fail-open ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:161)).
- **Data flow:** platform policy → Dify app/model request → LiteLLM → provider → Dify stream/usage → platform reconciliation.
- **Compatibility:** High for OpenAI-compatible calls; lower for Dify-specific token/cost semantics, tool calls, embeddings, and retries.
- **Latency/failure:** Two orchestration layers can retry independently and double provider attempts. Disable overlapping retries where possible; mark tool calls non-retryable.
- **State ownership:** Dify may retain app/workflow/conversation/RAG state; platform must not treat Dify tenant/RBAC/billing as authoritative.
- **Duplication:** Dify provider credentials/budgets and LiteLLM keys/budgets duplicate. Select platform-owned credentials and make the engine use delegated credentials.
- **Upgradeability:** Medium over API; high if Dify internals are patched.

**Recommendation record:** Direct evidence = Dify provider boundary plus closed commercial plane and LiteLLM gateway. Alternatives = LibreChat+LiteLLM; direct LiteLLM from platform. Why it wins = useful when Dify workflow/RAG semantics are needed. Modification cost = LOW at HTTP, HIGH for policy integration. Integration cost = HIGH due to identity/usage mapping. Maintenance = HIGH. Upgrade risk = HIGH. Confidence = HIGH.

### 6.2 Dify + Mem0

**Recommendation:** Additive integration only; do not attempt to replace Dify’s conversation/token memory in place initially. **Integration:** Level 1 HTTP/API or Level 2 adapter inside a dedicated Dify connector. **Classification:** Dify = orchestration engine; Mem0 = memory sidecar.

- **FACT:** Dify has conversation/message/token-buffer memory but no established generic personal/project semantic memory lifecycle ([`deep-dify.md`](../forensic/deep-dify.md:61)).
- **FACT:** Mem0 provides the missing add/search/update/delete lifecycle but not tenant/workspace authority ([`subsystem-mem0.md`](../forensic/subsystem-mem0.md:47)).
- **INFERENCE:** Add Mem0 as an explicit tool/context provider, with platform scope, rather than changing Dify generator internals.
- **Risks:** Dify context assembly and Mem0 retrieval can consume competing token budgets; memory extraction may execute inside Celery retries; deletion must traverse both systems.
- **Verdict:** Viable for a bounded experiment, but Dify’s closed RBAC/billing makes the combination unsuitable as the core commercial plane.

**Recommendation record:** Direct evidence = Dify memory gap and Mem0 lifecycle. Alternatives = Dify-native memory; RAGFlow memory. Why it wins = additive and lower blast radius. Modification = MEDIUM. Integration = MEDIUM–HIGH. Maintenance = HIGH if Dify internals are touched. Upgrade risk = MEDIUM–HIGH. Confidence = MEDIUM.

### 6.3 Dify + RAGFlow

**Recommendation:** Reject as a permanent product stack; permit isolated benchmark or specialized migration experiment only. **Integration:** Level 1 HTTP/API if tested. **Classification:** Two competing RAG/orchestration engines.

- **FACT:** Dify has first-class datasets/vector factories/workflows; RAGFlow has integrated parser, retrieval, graph, citations, memory, and tenant/index domain ([`deep-dify.md`](../forensic/deep-dify.md:69); [`deep-ragflow.md`](../forensic/deep-ragflow.md:73)).
- **INFERENCE:** The overlap is too large: both want to own documents, embeddings, retrieval, provider calls, agents, and tenant-scoped context.
- **Failure/latency:** Dify→RAGFlow→provider creates chained streaming, duplicate retrieval, duplicate embeddings, and complex cancellation.
- **State:** No clean single owner for dataset/document/chunk/citation or model credentials.
- **Security:** RAGFlow IDOR and Dify distributed/closed RBAC make defense-in-depth mandatory.
- **Migration:** High because both encode domain schemas and retrieval semantics.

**Rejected combination record:** Evidence = duplicated RAG/orchestration plus RAGFlow IDOR. Alternatives = Dify + Qdrant; LibreChat + RAGFlow. Why rejected = no bounded responsibility map without building a third orchestration plane. Modification cost HIGH; integration cost HIGH; maintenance HIGH; upgrade risk HIGH; confidence HIGH.

## 7. RAGFlow + LibreChat

This is the most viable RAGFlow composition, but only through the platform RAG gateway described in §5.2. LibreChat’s external RAG seam is materially better than embedding RAGFlow internals. Do not use shared databases or direct RAGFlow object URLs. Treat RAGFlow’s native memory as a specialized reference or disable it when Mem0/platform memory is authoritative to avoid three competing memory planes.

**Overall confidence:** HIGH that HTTP isolation is the correct level; MEDIUM that RAGFlow’s current APIs can satisfy all citation/streaming/file semantics without adapter work.

## 8. Open WebUI combinations

### 8.1 Open WebUI + external backend

**Recommendation:** Viable as a frontend/appliance behind the platform API, not as a tenant foundation. **Integration:** Level 1 HTTP/API. **Classification:** Open WebUI = chat frontend/appliance.

- **FACT:** Open WebUI has broad OpenAI-compatible APIs and a separate Svelte frontend/backend ([`deep-open-webui.md`](../forensic/deep-open-webui.md:53)).
- **FACT:** It has user/resource ownership and strong AccessGrants patterns, but no universal tenant primitive ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:262)).
- **INFERENCE:** External backend integration works best when Open WebUI is treated as a user-facing client and platform owns conversation/project/file APIs. If Open WebUI persists canonical chats independently, synchronization becomes a product liability.
- **Risks:** API auth forwarding, duplicate chats/files, tool/MCP policy mismatch, frontend assumptions about user identity.
- **Decision:** Accept for single-tenant or tenant-per-instance deployments; reject as a shared multi-tenant foundation.

### 8.2 Open WebUI + Mem0

**Recommendation:** Possible additive memory integration, but do not dual-own personal memory. **Integration:** Level 1 HTTP/API or Level 2 function/tool adapter.

- **FACT:** Open WebUI already has SQL/vector user memory and per-user collections ([`deep-open-webui.md`](../forensic/deep-open-webui.md:61)).
- **INFERENCE:** Mem0 adds extraction/history semantics but duplicates Open WebUI memory tables, vector collections, built-in memory tools, and routes.
- **Decision:** Use Mem0 only when Open WebUI is a thin frontend over platform memory; otherwise keep Open WebUI-native memory and avoid the sidecar.
- **Confidence:** MEDIUM because exact external-backend interception points and event timing were not audited.

### 8.3 Open WebUI + Qdrant

**Recommendation:** Viable through its vector factory only in non-platform or carefully wrapped deployments; do not enable Open WebUI’s naming-convention multitenancy mode for commercial tenancy. **Integration:** Level 2 internal adapter or Level 1 external vector service.

- **FACT:** Open WebUI has a clean vector backend factory including Qdrant ([`deep-open-webui.md`](../forensic/deep-open-webui.md:71)).
- **FACT:** Its optional Qdrant multitenancy mapping is fragile and can route to incorrect collections; it is disabled by default ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:240)).
- **Decision:** Platform RAG service should own Qdrant payload filters and collection strategy. Open WebUI’s internal Qdrant path is acceptable only as a non-authoritative appliance path.
- **Confidence:** HIGH.

## 9. LobeChat + external backend

**Recommendation:** Use LobeChat as an optional frontend/MCP reference only, not as the commercial business-plane foundation without implementing its 40+ business modules. **Integration:** Level 1 HTTP/API. **Classification:** frontend/reference implementation.

- **FACT:** LobeChat has real workspace-aware route checks, strong provider/runtime and MCP manifests ([`deep-lobechat.md`](../forensic/deep-lobechat.md:19)).
- **FACT:** OSS business-server modules for workspace, billing, usage, spend, quotas, RBAC, and storage are no-op/cloud-only stubs ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:41)).
- **FACT:** The example JWKS contains a forgeable private key and auto-enables OIDC/internal JWT signing ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:10)).
- **INFERENCE:** A platform can front LobeChat with external identity and APIs, but workspace/session/stream contracts still create substantial coupling.
- **Decision:** Accept only after secret remediation and with platform business APIs authoritative; reject as a low-cost foundation.
- **Confidence:** HIGH.

## 10. Flowise and Langflow as orchestration engines

### Flowise

- **FACT:** Wave A triage identifies Flowise as a visual flow/agent orchestration candidate, but the completed Wave C audit scope did not independently verify its tenant, auth, durable execution, or commercial policy boundaries.
- **UNKNOWN:** The evidence package does not establish sufficient Flowise source-level facts for a commercial multi-tenant decision.
- **Recommendation:** Conditional reference/orchestration engine only; integrate through Level 1 API and isolate flows, credentials, execution, and tenant mapping behind platform policy. Do not fork or share databases before a dedicated audit.
- **Confidence:** LOW–MEDIUM.

### Langflow

- **FACT:** Langflow exposes graph execution, component/provider bundles, flow-run APIs, memory/RAG/tool/MCP components, events, cancellation, and checkpoints ([`deep-langflow.md`](../forensic/deep-langflow.md:33)).
- **FACT:** Its Windows checkout was partial; login, migrations, complete tenant scope, and serving-plane behavior remain unknown ([`deep-langflow.md`](../forensic/deep-langflow.md:1)).
- **INFERENCE:** Langflow is best positioned as an orchestration/component engine, not a commercial tenant foundation.
- **Recommendation:** Conditional Level 1 integration after full Linux checkout and security audit. Use platform-owned workflow/job/tenant controls and Temporal for durable business execution where Langflow’s own event/queue topology is insufficient.
- **Risks:** graph serialization coupling, component identity propagation, opaque checkpoint side effects, in-memory queue/cache replica hazards, and duplicated RAG/memory/tool ownership.
- **Confidence:** MEDIUM for role classification; LOW for production integration decision.

## 11. Rejected combinations summary

| Combination | Decision | Primary evidence-based reason |
|---|---|---|
| Dify + RAGFlow as co-equal platforms | REJECT | Overlapping RAG/agent/provider/domain ownership; RAGFlow IDOR; high chained latency and migration cost |
| LibreChat + Mem0 with dual permanent memory writes | REJECT | Duplicate memory extraction/search/deletion and ambiguous ownership |
| Open WebUI shared multitenancy mode + Qdrant | REJECT | Naming-convention mapping can corrupt routing; no universal tenant boundary |
| LobeChat OSS as full commercial platform | REJECT | 40+ business stubs and confirmed live default JWKS risk |
| Dify foundation with OSS RBAC/billing | REJECT | Closed enterprise RBAC dependency and non-cloud fail-open quota |
| RAGFlow foundation without gateway/patch | REJECT | Confirmed image IDOR and thumbnail enumeration |
| Shared database between product engines | REJECT | Schema/policy coupling, migration deadlocks, duplicate domain ownership |
| Source-code merge of all engines | REJECT | Level 4/5 coupling destroys replaceability and multiplies upgrade risk |

## 12. Final composition recommendation

1. **Platform control plane (BUILD):** identity, tenants/workspaces/projects, authorization/policy, audit, billing/quotas, canonical conversations, files/document metadata, deletion, usage ledger, API gateway.
2. **LibreChat (REUSE/WRAP):** initial chat/agent shell or reference for tenant query boundaries, context pruning, compaction, and agent/MCP UX. Strict mode must be mandatory and platform authorization must remain authoritative.
3. **LiteLLM (INTEGRATE):** model gateway/engine behind `ModelGateway`.
4. **Mem0 (INTEGRATE):** long-term memory engine behind `MemoryProvider`.
5. **Platform RAG service + Qdrant (BUILD + REUSE):** document lifecycle and retrieval contract owned by platform/RAG service; Qdrant is replaceable vector infrastructure.
6. **MCP Python SDK (REUSE/WRAP):** protocol runtime inside platform MCP gateway, using fail-closed tool manifests and approval gates inspired by LobeChat/Open WebUI.
7. **Temporal (INTEGRATE):** durable workflow/background execution behind `JobService`.
8. **RAGFlow/Dify/Langflow/Flowise (REFERENCE or conditional engines):** isolated capability experiments, never shared authorities.

This composition minimizes permanent dependency on any one product while preserving escape hatches at HTTP/API boundaries.