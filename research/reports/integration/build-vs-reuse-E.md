# Wave E — Build vs Reuse Decisions

**Date:** 2026-09-11 **Status:** Final research artifact (Wave E)

Each capability is classified as **REUSE**, **INTEGRATE**, **WRAP**, **FORK**, **BUILD**, or **DO NOT BUILD**. Decisions state: Recommendation, Evidence, Alternatives, Why it wins, Modification cost, Integration cost, Long-term maintenance cost, Upgrade risk, Confidence. Claims are **FACT**, **INFERENCE**, or **UNKNOWN**.

**Evidence base:** [`candidate-matrix-E.md`](../comparison/candidate-matrix-E.md), [`capability-comparison-E.md`](../comparison/capability-comparison-E.md), [`modification-blast-radius-E.md`](../comparison/modification-blast-radius-E.md), [`integration-summary-D.md`](../reports/integration/integration-summary-D.md), [`target-blueprint-D.md`](../architecture/target-blueprint-D.md).

**Integration level policy (from Wave D, preserved):** Level 1 HTTP/API by default; Level 2 SDK only behind adapters. Reject shared DB (Level 3), source-code integration (Level 4), and fork (Level 5) except deliberate LibreChat foundation decision.

---

## 1. Capability decision table

| Capability | Decision | Level | Primary evidence | Confidence |
|---|---|---|---|---|
| LLM gateway / model router | **INTEGRATE** LiteLLM | 1 HTTP/API | FACT: LiteLLM provider normalization, routing, fallback, cost surfaces ([`subsystem-litellm.md`](../reports/forensic/subsystem-litellm.md) §3) | HIGH |
| Identity / authentication | **BUILD** | Platform | FACT: no candidate provides safe, complete commercial identity ([`audit-summary-C.md`](../reports/audit/audit-summary-C.md) §6) | HIGH |
| Authorization / tenant isolation | **BUILD** | Platform | FACT: LibreTech's tenant mechanism is strongest but default-off; RAGFlow IDOR-breachable; Dify closed enterprise ([`reconciliation-summary-C5.md`](../reports/audit/reconciliation-summary-C5.md) §3) | HIGH |
| Conversations / messages | **BUILD** | Platform; shell API-wrap candidates | FACT: all candidates store conversations in their own schema; no candidate's conversation schema should be canonical for a commercial platform | HIGH |
| Context budgeting | **BUILD** with LibreChat reference | Platform service | FACT: LibreChat has token pruning, compaction, remaining context ([`capability-comparison-E.md`](../comparison/capability-comparison-E.md) §2) | HIGH for need; MEDIUM for exact UX |
| Memory (long-term) | **INTEGRATE** Mem0 | 1–2 HTTP/SDK behind adapter | FACT: Mem0 add/search/update/history/delete lifecycle; caller-derived scope weak ([`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md) §3) | HIGH |
| RAG orchestration | **BUILD** + Qdrant for vector | Platform RAG service Level 1 → Qdrant | FACT: Qdrant vector/query API; no document/RAG semantics ([`subsystem-qdrant.md`](../reports/forensic/subsystem-qdrant.md) §3); FACT: LibreChat external `rag_api` seam is clean RAG replacement boundary ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §8) | HIGH |
| Vector store | **INTEGRATE** Qdrant | 1 HTTP/gRPC | FACT: Qdrant typed point/query/payload API; shards, WAL, snapshots; replaceable ([`subsystem-qdrant.md`](../reports/forensic/subsystem-qdrant.md) §3) | HIGH |
| File / object storage | **INTEGRATE** S3-compatible + platform File Service | Platform service Level 1 | FACT: all candidates have S3/local adapters; platform must own authorization, quotas, retention, deletion verification | HIGH |
| Agent runtime | **WRAP/REUSE** LibreChat initially | Level 1 → Level 2 adapter | FACT: LibreChat has durable agents, checkpoints, MCP, tools, subagents ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §9) | HIGH for role; MEDIUM for final fork decision |
| Workflow engine (durable) | **INTEGRATE** Temporal | 1/gRPC through adapter | FACT: Temporal durable execution; no candidate has comparable durability ([`subsystem-temporal.md`](../reports/forensic/subsystem-temporal.md) §3) | HIGH |
| Tools / MCP protocol | **REUSE** MCP Python SDK inside gateway | Level 2 behind Level 1 gateway | FACT: SDK provides protocol/session/transport; no tenant policy ([`subsystem-mcp-python-sdk.md`](../reports/forensic/subsystem-mcp-python-sdk.md) §3) | HIGH |
| External integrations (REST/webhooks/connectors) | **BUILD** platform connector registry | Level 1 adapter per integ | INFERENCE: candidates' connector code is product-specific; platform needs unified credential, approval, and audit model | HIGH |
| Frontend / UI | **REUSE/BUILD** LibreChat/Open WebUI/LobeChat as optional shells | Level 1 API | FACT: all three have independent frontend/backend separation; none should be the canonical UX forever | HIGH for role; MEDIUM for choice |
| Billing / credits | **BUILD** | Platform ledger | FACT: Dify quota is cloud-gated fail-open; RAGFlow credits are dead code; LobeChat stubbed; LibreChat balances exist but not platform-billing ([`capability-comparison-E.md`](../comparison/capability-comparison-E.md) §10) | HIGH |
| Quotas (storage/compute) | **BUILD** | Platform policy + enforcement | FACT: no candidate has complete commercial quota enforcement; all require platform addition | HIGH |
| Usage / audit ledger | **BUILD** | Platform | FACT: LiteLLM spend, engine usage, vector units, workflow compute all need centralized observation and reconciliation ([`target-blueprint-D.md`](../architecture/target-blueprint-D.md) §10) | HIGH |
| Observability (centralized) | **BUILD** + integrate candidate telemetry | Platform canonical stream | FACT: all candidates have OpenTelemetry/Langfuse/Sentry; none provides the platform's cross-service audit stream | HIGH |

---

## 2. Detailed decision records

### 2.1 LLM gateway / model router — INTEGRATE LiteLLM

- **Recommendation:** Make `ModelGateway` the only model-call boundary and use LiteLLM behind it at Level 1 HTTP/API. Treat LiteLLM budgets as defensive or observed data, not canonical billing.
- **Evidence (FACT):** LiteLLM exposes SDK/provider/gateway boundaries, router strategies (weighted/latency/usage-based), callbacks, custom handlers, cost tracking, and persistence ([`subsystem-litellm.md`](../reports/forensic/subsystem-litellm.md) §3). Architecture (`ARCHITECTURE.md:9`), router (`router.py:681`), custom handler (`custom_handler.py:8`).
- **Alternatives considered:** Direct provider SDKs; Dify provider manager; LibreChat provider clients; another custom gateway.
- **Why it wins:** Provider abstraction and routing are independently deployable and naturally API-shaped. No candidate needs a fork to utilize LiteLLM.
- **Modification cost:** LOW at HTTP; HIGH if its Prisma schema is adopted as canonical.
- **Integration cost:** MEDIUM for streaming, usage, auth, and policy mapping.
- **Long-term maintenance cost:** MEDIUM; model/provider behavior and cost tables evolve.
- **Upgrade risk:** LOW–MEDIUM over HTTP; HIGH with source/database coupling.
- **Confidence:** HIGH.

### 2.2 Identity / authentication — BUILD

- **Recommendation:** Platform owns user identity, SSO/OIDC, service principals, sessions, API keys, OAuth connection references. Terminate auth at the platform edge; propagate signed `RequestContext` to all engines.
- **Evidence (FACT):** No audited candidate provides safe complete commercial identity. LibreChat tenant mechanism is default-off; Dify RBAC is closed enterprise; LobeChat JWKS key is forgeable; RAGFlow has IDOR; Open WebUI has no tenant primitive ([`audit-summary-C.md`](../reports/audit/audit-summary-C.md) §6, [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) items 1–9).
- **Alternatives considered:** Delegate to LibreChat identity; delegate to Dify identity; per-candidate identity per engine.
- **Why it wins:** Only one identity authority; every engine receives scoped derived tokens. No candidate's OSS identity model is safe for commercial multi-user.
- **Modification cost:** HIGH (the platform is being built).
- **Integration cost:** MEDIUM (every engine adapter maps identity).
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** LOW (platform identity is owned).
- **Confidence:** HIGH.

### 2.3 Authorization / tenant isolation — BUILD

- **Recommendation:** Platform owns tenant/workspace/project model, membership, roles, policy bindings, resource ACLs. Engine authorization is defense-in-depth only.
- **Evidence (FACT):** LibreChat's tenant plugin is the strongest mechanism (DB-level query scoping with coverage tests) but strict mode is default-off ([`audit-librechat.md`](../reports/audit/audit-librechat.md) red flag LC-1). RAGFlow's tenant boundary is IDOR-breachable (RF-1). Dify's RBAC is closed enterprise. LobeChat's workspace routes are checked but universal coverage UNKNOWN. Open WebUI has no tenant primitive ([`capability-comparison-E.md`](../comparison/capability-comparison-E.md) §6).
- **Alternatives considered:** Make LibreChat/Dify/RAGFlow authorization authoritative; database-only RLS; per-tenant deployment.
- **Why it wins:** Avoids duplicating policy across engines; each engine's local checks become defense-in-depth rather than authority.
- **Modification cost:** MEDIUM for platform policy; HIGH to retrofit all engines.
- **Integration cost:** MEDIUM for signed context and policy checks.
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** LOW at platform API; HIGH if engine authorization is patched.
- **Confidence:** HIGH.

### 2.4 Conversations / messages — BUILD

- **Recommendation:** Platform owns canonical conversation, message, turn, attachment, citation, and agent-run metadata. Engine projections are temporary compatibility paths.
- **Evidence (FACT):** All candidates store conversations in their own schemas (Mongo, SQLAlchemy, Prisma, etc.). No candidate's conversation schema should become the platform's canonical authority — doing so creates irreversible schema coupling. Candidates can remain as chat/agent shells during migration (Phase 2 in [`migration-path-D.md`](../architecture/migration-path-D.md) §5).
- **Alternatives considered:** Adopt LibreChat conversation schema as canonical; adopt Dify app/conversation schema.
- **Why it wins:** Preserves frontend/engine replacement without conversation migration.
- **Modification cost:** MEDIUM; platform conversation API + migration from engine projections.
- **Integration cost:** MEDIUM; adapter for each engine.
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** LOW at platform API boundary; MEDIUM for engine adapter contract stability.
- **Confidence:** HIGH.

### 2.5 Context budgeting — BUILD (with LibreChat reference)

- **Recommendation:** Build platform `ContextBudgetService`. Use LibreChat's token-counted pruning, remaining-context calculation, and compaction as a reference and temporary implementation. No engine may silently discard or promote user content outside platform context policy.
- **Evidence (FACT):** LibreChat tracks token counts, remaining context, pruning, historical-file authorization, and compaction ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §7). **INFERENCE:** This is the strongest candidate foundation for context telemetry, but it does not establish a unified cross-engine approval/promotion protocol.
- **Alternatives considered:** Direct model-window tracking; no context service.
- **Why it wins:** Prevents engines from silently discarding user content; provides user-visible token budgets and condensation proposals.
- **Modification cost:** MEDIUM for platform implementation; LOW for LibreChat adapter.
- **Integration cost:** MEDIUM.
- **Long-term maintenance cost:** MEDIUM; model capacity registry evolves.
- **Upgrade risk:** LOW at API boundary.
- **Confidence:** HIGH for need; MEDIUM for exact UX/latency.

### 2.6 Memory — INTEGRATE Mem0

- **Recommendation:** Use Mem0 behind `MemoryProvider` interface. Platform owns consent, scope, authorization, retention, provenance, promotion, and deletion. Derive scope from authenticated context; never accept arbitrary client filters.
- **Evidence (FACT):** Mem0 exposes add/search/update/history/delete and scoped `user_id`, `agent_id`, `run_id` lifecycle ([`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md) §3). Mem0's server accepts caller-supplied identity filters; evidence does not establish universal tenant authority. **INFERENCE:** Derive filters server-side rather than trusting client-supplied scope.
- **Alternatives considered:** LibreChat native memory; Open WebUI native memory; RAGFlow memory taxonomy; custom memory service.
- **Why it wins:** Strong lifecycle engine with replaceable provider/vector boundary, while platform can correct caller-filter tenancy weakness. Avoids dual-write between native memory and Mem0 (rejected combination in [`combinations-D.md`](../reports/integration/combinations-D.md) §3).
- **Modification cost:** LOW–MEDIUM adapter; HIGH for replacing native engine semantics.
- **Integration cost:** MEDIUM for scope mapping and migration.
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** MEDIUM at API/schema boundary.
- **Confidence:** HIGH.

### 2.7 RAG orchestration — BUILD + Qdrant

- **Recommendation:** Platform/RAG service owns documents, parsing, chunking, embeddings, citations, authorization, deletion, and reindex. Qdrant owns vector persistence behind HTTP/gRPC. LibreChat's external `rag_api` seam is the compatibility path. RAGFlow only behind security gateway.
- **Evidence (FACT):** Qdrant provides typed vector/query/RBAC/storage APIs but not document/RAG product semantics ([`subsystem-qdrant.md`](../reports/forensic/subsystem-qdrant.md) §3). LibreChat external `rag_api` is the strongest replacement seam among candidates ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §8). RAGFlow has confirmed IDOR at image/thumbnail endpoints (C.5 item 3).
- **Alternatives considered:** RAGFlow as full RAG owner; Dify datasets; Open WebUI vector factory; LobeChat pgvector (locked to 1024-dim).
- **Why it wins:** Preserves vector backend replacement; avoids RAGFlow's security/ownership coupling; LibreChat's API boundary reduces migration cost.
- **Modification cost:** MEDIUM/HIGH (platform RAG service is built); Qdrant adapter is LOW.
- **Integration cost:** MEDIUM.
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** LOW at API boundary.
- **Confidence:** HIGH.

### 2.8 Agent runtime — WRAP/REUSE LibreChat initially

- **Recommendation:** Use LibreChat agent runtime through a `ChatShellAdapter` for chat/agent UX. Platform controls tool authorization, approval, usage, and durable state through `McpGateway` and `JobService`.
- **Evidence (FACT):** LibreChat has durable agents, queued/resumable turns, subagents, tools, MCP, and checkpoint state ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §9). LobeChat has strong tool engineering/capability manifests but is secondary evidence. Platform `McpGateway` owns tool authorization (LobeChat's share gate is a reference pattern to copy).
- **Alternatives considered:** Build custom agent runtime; use LobeChat agent runtime; use Dify agent runtime.
- **Why it wins:** Lowest blast radius to get a working agent surface; platform policy wraps all critical operations.
- **Modification cost:** MEDIUM for shell adapter; HIGH for foundation fork.
- **Integration cost:** MEDIUM due to stream, checkpoint, identity, message mapping.
- **Long-term maintenance cost:** MEDIUM/HIGH if tenant/context/agent internals are forked; MEDIUM if API-wrapped.
- **Upgrade risk:** MEDIUM at API boundary; HIGH for forked schema/runtime.
- **Confidence:** HIGH for role; MEDIUM for final fork decision.

### 2.9 Workflow engine — INTEGRATE Temporal

- **Recommendation:** Put Temporal behind `JobService` for ingestion, approvals, long-running agents, schedules, deletion verification, and reconciliation. Platform owns business job state, quotas, and user-visible status.
- **Evidence (FACT):** Temporal provides durable histories, task queues, retries, namespaces, workers, persistence boundaries — but not platform business semantics ([`subsystem-temporal.md`](../reports/forensic/subsystem-temporal.md) §3). No candidate has comparable durable execution.
- **Alternatives considered:** Candidate Celery/Redis jobs; QStash; custom queue; Langflow/Dify workflow runtime.
- **Why it wins:** Durable execution is independently replaceable and stronger than per-process/background patterns found in candidates.
- **Modification cost:** LOW–MEDIUM for new workflows; HIGH for replacing workflow semantics later.
- **Integration cost:** MEDIUM/HIGH operationally.
- **Long-term maintenance cost:** HIGH operationally, MEDIUM at adapter boundary.
- **Upgrade risk:** MEDIUM; workflow versioning is required.
- **Confidence:** HIGH.

### 2.10 MCP / tools — REUSE SDK inside gateway

- **Recommendation:** Use official MCP Python SDK for protocol/session/transport implementation. Platform gateway owns registry, tenant/project grants, credentials, approvals, audit, rate limits, and sandboxing. LobeCloud's synced-manifest/default-deny share pattern is a reference to copy.
- **Evidence (FACT):** SDK provides server/client/transports/auth/extensions/EventStore but no tenant policy, marketplace, durable job, or billing authority ([`subsystem-mcp-python-sdk.md`](../reports/forensic/subsystem-mcp-python-sdk.md) §3). Open WebUI's per-call MCP authorization is fail-closed ([C.5 item 9](../evidence/reconciliation-C5.md)). LobeChat's synced-manifest model is the reference for secure MCP UX.
- **Alternatives considered:** Candidate-native MCP; direct SDK use in every product; custom protocol implementation.
- **Why it wins:** Lowest protocol coupling while retaining a single security/policy boundary.
- **Modification cost:** LOW–MEDIUM gateway; HIGH for custom protocol fork.
- **Integration cost:** MEDIUM.
- **Long-term maintenance cost:** MEDIUM due to protocol/API evolution.
- **Upgrade risk:** MEDIUM; isolate major API renames in adapter.
- **Confidence:** HIGH.

### 2.11 Billing/credits — BUILD

- **Recommendation:** Build append-only usage ledger, reservation/commit/release/refund, quota policy, and reconciliation jobs. No candidate's billing schema is safe to adopt as canonical.
- **Evidence (FACT):** Dify quota is cloud-gated fail-open ([C.5 item 5](../evidence/reconciliation-C5.md)). RAGFlow credits are dead code (zero call sites) ([C.5 item 4](../evidence/reconciliation-C5.md)). LobeChat billing stubs are typed no-ops (40+ modules) ([C.5 item 2](../evidence/reconciliation-C5.md)). LibreChat balances exist but are not a platform billing system.
- **Alternatives considered:** Adopt Dify quota service with custom billing backend; adopt LibreChat balances as canonical; use LiteLLM spend keys as canonical.
- **Why it wins:** Only the platform can be the authoritative billing and quota authority across model calls, tool execution, storage, vector, and workflow units.
- **Modification cost:** HIGH (the platform ledger is built).
- **Integration cost:** MEDIUM (ingest LiteLLM spend events, engine usage callbacks).
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** LOW (platform-owned).
- **Confidence:** HIGH.

---

## 3. Rejected build decisions

| Capability | Rejected route | Why rejected | Evidence |
|---|---|---|---|
| RAG as pure BUILD without Qdrant | Build custom vector store from scratch | Qdrant provides vector/query/storage/RBAC at production scale; building custom is unnecessary infrastructure work | [`subsystem-qdrant.md`](../reports/forensic/subsystem-qdrant.md) §3 |
| Memory as pure BUILD without Mem0 | Build custom memory engine | Mem0 provides complete lifecycle semantics; building custom memory extraction/search/update/history adds no differentiator | [`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md) §3 |
| Model routing as pure BUILD without LiteLLM | Build custom provider normalization and routing | LiteLLM has 100+ provider handlers, router strategies, fallback, cost tracking; a custom gateway would duplicate years of work | [`subsystem-litellm.md`](../reports/forensic/subsystem-litellm.md) §3 |
| Workflow engine as IN-PROCESS without Temporal | Use engine-native Celery/Redis/in-process jobs | No candidate's job system provides durable execution, replay, and retry semantics at Temporal's level; per-process semaphores (RAGFlow) or in-process state (Open WebUI, Langflow) are not production-scale | [`subsystem-temporal.md`](../reports/forensic/subsystem-temporal.md) §3 |
| Fork Dify as foundation | Adopt Dify as full platform including RBAC/billing | Dify RBAC is closed enterprise dependency; quota is fail-open outside Cloud. Forking requires implementing enterprise RBAC API from scratch — a closed-source reverse-engineering risk | C.5 item 5 |
| Fork LobeChat as foundation | Adopt LobeChat OSS as full platform | 40+ business modules are typed stubs; JWKS key is live/forgeable. Forking means privately rebuilding the entire business plane | C.5 items 1–2 |

---

## 4. Artifact index

All build-vs-reuse decisions reference the following source reports:

- [`audit-summary-C.md`](../reports/audit/audit-summary-C.md) — Wave C independent audit scores and red flags
- [`reconciliation-summary-C5.md`](../reports/audit/reconciliation-summary-C5.md) — C.5 reconciliation with composite impact
- [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) — item-by-item evidence for all 9 C.5 items
- [`capability-comparison-E.md`](../comparison/capability-comparison-E.md) — memory, context, RAG, agents, MCP, tenancy, API, storage, jobs, usage, deployment
- [`modification-blast-radius-E.md`](../comparison/modification-blast-radius-E.md) — 13 modification tests across all 10 candidates
- [`target-blueprint-D.md`](../architecture/target-blueprint-D.md) — platform-owned control plane blueprint with interface contracts
- [`migration-path-D.md`](../architecture/migration-path-D.md) — phase-by-phase migration from candidate code to platform interfaces
- [`integration-summary-D.md`](../reports/integration/integration-summary-D.md) — executive integration recommendation
- [`combinations-D.md`](../reports/integration/combinations-D.md) — combination-by-combination integration analysis