# Wave D — Executive Integration Recommendation

**Date:** 2026-09-11  
**Scope:** Integration and product architecture analysis based on completed Wave C/C.5 evidence and subsystem reports.  
**Status:** Final Wave D artifact; this is not the final 29-deliverable package. No production code was implemented and no cloned repository was modified.

## 1. Executive recommendation

**Recommendation:** Do not select a single open-source product as the commercial platform. Build a platform-owned control plane and compose replaceable product engines and sidecars behind stable internal interfaces.

### Preferred composition

```text
Platform control plane (BUILD / FOUNDATION)
  ├─ identity, tenant/workspace/project authorization, policy, audit
  ├─ canonical conversations, files/document metadata, usage ledger, billing, quotas
  ├─ context-budget service and API gateway
  ├─ ModelGateway ── LiteLLM (MODEL ACCESS ENGINE)
  ├─ MemoryProvider ── Mem0 (MEMORY ENGINE)
  ├─ RagProvider ── platform RAG service ── Qdrant (RAG + VECTOR SIDEcars)
  ├─ McpGateway ── MCP Python SDK (INTEGRATION LAYER)
  ├─ JobService ── Temporal (WORKFLOW ENGINE)
  └─ optional chat/agent shell ── LibreChat (FOUNDATION CANDIDATE / REUSE)

Conditional specialized engines behind API gateways:
  Dify, RAGFlow, Open WebUI, LobeChat, Langflow, Flowise
```

**One-line decision:** LibreChat is the strongest forced-pick product shell because of its explicit tenant query boundary, context controls, agent/MCP surface, and external RAG seam—but the commercial truth must belong to the platform plane. LiteLLM, Mem0, Qdrant, MCP SDK, and Temporal are replaceable engines/sidecars, not tenant or billing authorities.

## 2. Evidence basis and confidence

- **FACT:** Wave C.5 resolved all nine reconciliation items; LobeChat’s example JWKS is live/forgeable, its OSS business plane has 40+ no-op/cloud-only modules, RAGFlow has recurring IDOR exposure, Dify has a dual RBAC regime and cloud-gated quota, LibreChat’s tenant plugin is real but strict mode is not default, and Open WebUI’s MCP authorization is fail-closed while its vector multitenancy mapping is fragile ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:9)).
- **FACT:** Subsystem evidence identifies LiteLLM as model plane, Mem0 as memory engine, MCP Python SDK as protocol adapter, Qdrant as vector infrastructure, and Temporal as durable execution infrastructure ([`subsystem-summary-A3.md`](../forensic/subsystem-summary-A3.md:47)).
- **FACT:** No audited candidate is adoptable as-is as a commercial multi-user platform ([`audit-summary-C.md`](../audit/audit-summary-C.md:159)).
- **INFERENCE:** A composable platform plane is the lowest-regret architecture.
- **UNKNOWN:** Benchmark data, exact license/legal review, historical API stability, and production migration duration remain unverified.
- **Overall confidence:** HIGH for composition and boundaries; MEDIUM for sizing/latency until experiments are run.

## 3. Classification map

| Candidate/component | Classification | Role in target architecture | Decision |
|---|---|---|---|
| Platform control plane | FOUNDATION / BACKEND SERVICE | Canonical commercial authority | BUILD |
| LibreChat | FOUNDATION candidate / CHAT FRONTEND / AGENT ENGINE | Initial shell or reference for tenant/context/agent UX | REUSE/WRAP; conditional fork later |
| LiteLLM | MODEL ACCESS ENGINE | Provider normalization, routing, fallback, cost observation | INTEGRATE behind API |
| Mem0 | MEMORY ENGINE | Long-term memory lifecycle | INTEGRATE behind API/SDK adapter |
| Platform RAG service | RAG ENGINE | Canonical ingestion/retrieval/citation lifecycle | BUILD |
| Qdrant | STORAGE LAYER | Vector/index persistence | INTEGRATE behind RAG adapter |
| MCP Python SDK | INTEGRATION LAYER | Protocol/session implementation | REUSE inside gateway |
| Temporal | ORCHESTRATION ENGINE | Durable workflows/jobs/approvals | INTEGRATE behind JobService |
| RAGFlow | SPECIALIZED RAG ENGINE / REFERENCE | Advanced parser/GraphRAG experiment or provider | Conditional API integration; not foundation |
| Dify | ORCHESTRATION/AGENT/RAG ENGINE / REFERENCE | Isolated workflow/RAG engine | Conditional API integration; reject foundation |
| Open WebUI | CHAT FRONTEND / APPLIANCE | Optional client or tenant-per-instance appliance | Conditional API integration |
| LobeChat | CHAT FRONTEND / REFERENCE IMPLEMENTATION | MCP/connector UX reference or optional client | Conditional API integration; reject turnkey foundation |
| Langflow | ORCHESTRATION ENGINE / REFERENCE | Graph/component engine after full audit | Conditional only |
| Flowise | ORCHESTRATION ENGINE / REFERENCE | Visual workflow experiment | Evidence-limited conditional only |

## 4. Integration-level policy

| Boundary | Level | Recommendation |
|---|---:|---|
| Platform → LiteLLM | 1 HTTP/API | Preferred; hide LiteLLM schema and keys |
| Platform → Mem0 | 1 HTTP/API; 2 SDK inside adapter | Preferred; derive identity scope server-side |
| RAG service → Qdrant | 1 HTTP/gRPC | Preferred; hide collections/payload schema |
| MCP gateway → SDK | 2 SDK/library | Confine protocol changes to gateway |
| Platform → Temporal | 1 gRPC/API through SDK | Preferred; use versioned workflows |
| Platform → LibreChat | 1 HTTP/API | Preferred shell boundary; no shared DB |
| Platform → Dify/RAGFlow/Open WebUI/LobeChat/Langflow/Flowise | 1 HTTP/API | Isolate as engines/projections |
| Candidate → platform DB | 3 shared DB | Reject as permanent architecture |
| Source-code merge | 4 | Reject except narrowly reviewed adapter contributions |
| Fork | 5 | Only deliberate LibreChat foundation decision; no default fork |

**Why lower levels win:** They preserve independent deployment, schema ownership, security testing, replacement, and upstream upgrades. **FACT:** Subsystem reports explicitly identify Level 1/2 as the safe boundary and source-level coupling as upgrade risk ([`subsystem-summary-A3.md`](../forensic/subsystem-summary-A3.md:73)).

## 5. Major recommendation records

### Recommendation A — Build platform control plane

- **Recommendation:** Platform owns identity, tenants/workspaces/projects, authorization, policy, audit, billing, quotas, canonical conversations, file/document metadata, consent/retention/deletion, and usage ledger.
- **Direct evidence:** All five audited products keep key commercial capabilities closed, stubbed, absent, or fail-open; no candidate provides complete platform authority ([`audit-summary-C.md`](../audit/audit-summary-C.md:105)).
- **Alternatives considered:** Dify as foundation; LobeChat as foundation; LibreChat as full fork; per-customer Open WebUI appliance.
- **Why it wins:** Avoids adopting a closed/unsafe commercial plane and makes every major subsystem replaceable.
- **Modification cost:** HIGH initially; this is the product being built.
- **Integration cost:** MEDIUM because all engines need context/policy adapters.
- **Long-term maintenance cost:** MEDIUM/HIGH but owned and testable rather than hidden in forks.
- **Upgrade risk:** LOW for external engines; platform APIs become the stable contract.
- **Confidence:** HIGH.

### Recommendation B — Use LibreChat as initial shell/forced foundation candidate

- **Recommendation:** Use LibreChat through API boundaries for chat/agent UX and as a reference for context pruning, compaction, tenant query middleware, and MCP/agent behavior. Make strict isolation mandatory; do not make its Mongo/balance/tenant schema canonical.
- **Direct evidence:** LibreChat has the strongest general-purpose tenant query mechanism, dedicated memory/context controls, external `rag_api`, agent/MCP surfaces, and API routes ([`deep-librechat.md`](../forensic/deep-librechat.md:91)). Strict mode is default-off and some global credential/runAsSystem paths remain audit targets ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:192)).
- **Alternatives considered:** LobeChat for typed modern architecture/MCP; Dify for integrated workflow/RAG; Open WebUI for simpler frontend/vector factory; custom frontend from day one.
- **Why it wins:** Best open tenant primitive and cleanest RAG replacement seam among audited products, with mature agent/context features.
- **Modification cost:** MEDIUM for shell adapter; HIGH for commercial foundation fork.
- **Integration cost:** MEDIUM due to stream, agent checkpoint, identity, and message mapping.
- **Long-term maintenance cost:** MEDIUM/HIGH if tenant/context/agent internals are forked; MEDIUM if API-wrapped.
- **Upgrade risk:** MEDIUM at API boundary; HIGH for forked schema/runtime semantics.
- **Confidence:** HIGH for role; MEDIUM for final fork decision.

### Recommendation C — Use LiteLLM for model access

- **Recommendation:** Make `ModelGateway` the only model-call boundary and use LiteLLM behind it. Treat LiteLLM budgets/spend as defensive or observed data, not canonical billing.
- **Direct evidence:** LiteLLM exposes SDK/gateway/provider/router/fallback/cost boundaries and persistence ([`subsystem-litellm.md`](../forensic/subsystem-litellm.md:19)).
- **Alternatives considered:** Direct provider SDKs; Dify provider manager; LibreChat provider clients; another gateway.
- **Why it wins:** Provider abstraction and routing are independently deployable and naturally API-shaped.
- **Modification cost:** LOW at HTTP; HIGH if its Prisma schema is adopted.
- **Integration cost:** MEDIUM for streaming, usage, reservations, and policy mapping.
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** LOW–MEDIUM over HTTP; HIGH with source/database coupling.
- **Confidence:** HIGH.

### Recommendation D — Use Mem0 for long-term memory

- **Recommendation:** Use Mem0 behind `MemoryProvider`; platform owns consent, scope, authorization, retention, promotion, provenance, and deletion.
- **Direct evidence:** Mem0 has add/search/update/history/delete, identity dimensions, factories, and server API; it does not establish complete tenant/workspace authority ([`subsystem-mem0.md`](../forensic/subsystem-mem0.md:15)).
- **Alternatives considered:** LibreChat native memory; Open WebUI native memory; RAGFlow memory; custom service.
- **Why it wins:** Strong lifecycle engine with a replaceable provider/vector boundary, while platform can correct caller-filter tenancy weakness.
- **Modification cost:** LOW–MEDIUM adapter; HIGH for replacing native engine semantics.
- **Integration cost:** MEDIUM for scope mapping and migration.
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** MEDIUM.
- **Confidence:** HIGH.

### Recommendation E — Build RAG service over Qdrant

- **Recommendation:** Platform/RAG service owns documents, parsing, chunking, embeddings, citations, authorization, deletion, and reindex; Qdrant owns vector persistence behind HTTP/gRPC.
- **Direct evidence:** Qdrant exposes typed vector/query/RBAC/storage APIs but not document/RAG product semantics ([`subsystem-qdrant.md`](../forensic/subsystem-qdrant.md:15)). LibreChat’s external RAG seam is strong ([`deep-librechat.md`](../forensic/deep-librechat.md:75)).
- **Alternatives considered:** RAGFlow as full RAG owner; Dify datasets; Open WebUI vector factory; LobeChat pgvector.
- **Why it wins:** Preserves vector backend replacement and avoids RAGFlow’s security/ownership coupling.
- **Modification cost:** MEDIUM/HIGH because the platform RAG service is built; Qdrant adapter is LOW.
- **Integration cost:** MEDIUM.
- **Long-term maintenance cost:** MEDIUM.
- **Upgrade risk:** LOW at API boundary.
- **Confidence:** HIGH.

### Recommendation F — Use MCP SDK only behind a platform MCP gateway

- **Recommendation:** Use the official MCP Python SDK for protocol/session/transport implementation; platform owns registry, tenant/project grants, credentials, approvals, audit, rate limits, and sandboxing.
- **Direct evidence:** SDK provides protocol/auth/transport/extensions/EventStore but no tenant policy, marketplace, durable job, or billing authority ([`subsystem-mcp-python-sdk.md`](../forensic/subsystem-mcp-python-sdk.md:19)). Open WebUI per-call MCP authorization is fail-closed and LobeChat has strong synced-manifest/default-deny patterns ([`reconciliation-C5.md`](../../evidence/reconciliation-C5.md:278)).
- **Alternatives considered:** Candidate-native MCP; direct SDK use in every product; custom protocol implementation.
- **Why it wins:** Lowest protocol coupling while retaining a single security/policy boundary.
- **Modification cost:** LOW–MEDIUM gateway; HIGH for custom protocol fork.
- **Integration cost:** MEDIUM.
- **Long-term maintenance cost:** MEDIUM due to protocol/API evolution.
- **Upgrade risk:** MEDIUM; isolate major API renames in adapter.
- **Confidence:** HIGH.

### Recommendation G — Use Temporal for durable workflows

- **Recommendation:** Put Temporal behind `JobService` for ingestion, approvals, long-running agents, schedules, deletion verification, and reconciliation. Platform owns business job state and quotas.
- **Direct evidence:** Temporal provides durable histories, task queues, retries, namespaces, workers, and persistence boundaries, but not platform business semantics ([`subsystem-temporal.md`](../forensic/subsystem-temporal.md:15)).
- **Alternatives considered:** Candidate Celery/Redis/in-process jobs; QStash; custom queue; Langflow/Dify workflow runtime.
- **Why it wins:** Durable execution and retries are independently replaceable and stronger than per-process/background patterns found in candidates.
- **Modification cost:** LOW–MEDIUM for new workflows; HIGH for replacing workflow semantics later.
- **Integration cost:** MEDIUM/HIGH operationally.
- **Long-term maintenance cost:** HIGH operationally, MEDIUM at adapter boundary.
- **Upgrade risk:** MEDIUM; workflow versioning is required.
- **Confidence:** HIGH.

### Recommendation H — Keep Dify/RAGFlow/Open WebUI/LobeChat/Langflow/Flowise non-authoritative

- **Recommendation:** Use them as isolated engines, references, optional frontends, or specialized capability providers—not as platform authority.
- **Direct evidence:** Dify closed RBAC/cloud billing, RAGFlow IDOR and per-process scaling, Open WebUI no universal tenancy/fragile vector mode, LobeChat business stubs/live JWKS, and Langflow incomplete checkout are all documented ([`reconciliation-summary-C5.md`](../reports/audit/reconciliation-summary-C5.md:46)).
- **Alternatives considered:** Choose the highest raw score as foundation; merge all best features; fork Dify/LobeChat/RAGFlow.
- **Why it wins:** Avoids duplicating policy/state and preserves capability-specific experimentation.
- **Modification cost:** LOW for API adapter; HIGH for safe foundation fork.
- **Integration cost:** MEDIUM–HIGH depending domain overlap.
- **Long-term maintenance cost:** HIGH if forked; MEDIUM if isolated.
- **Upgrade risk:** HIGH for source/schema integration.
- **Confidence:** HIGH.

## 6. Combination decisions

| Combination | Verdict | Level | Role |
|---|---|---:|---|
| LibreChat + Mem0 | ACCEPT with one canonical memory path | 1/2 | Chat shell + memory engine |
| LibreChat + LiteLLM | STRONGLY ACCEPT | 1 | Chat shell + model engine |
| LibreChat + Qdrant | ACCEPT through platform RAG | 1 | Shell + RAG/vector sidecar |
| LibreChat + RAGFlow | CONDITIONAL | 1 | Shell + specialized RAG engine behind gateway |
| Dify + LiteLLM | CONDITIONAL engine experiment | 1 | Workflow/RAG engine + model sidecar |
| Dify + Mem0 | ADDITIVE only | 1/2 | Workflow engine + memory sidecar |
| Dify + RAGFlow | REJECT as co-equal stack | 1 only for benchmark | Duplicate RAG/orchestration ownership |
| RAGFlow + LibreChat | CONDITIONAL and gateway-required | 1 | Chat shell + specialized RAG |
| Open WebUI + external backend | ACCEPT as frontend/appliance | 1 | Optional client; not shared tenancy foundation |
| Open WebUI + Mem0 | CONDITIONAL; avoid dual memory | 1 | Thin frontend + memory sidecar |
| Open WebUI + Qdrant | ACCEPT only via platform RAG; reject fragile mode | 1/2 | Frontend + vector adapter |
| LobeChat + external backend | CONDITIONAL after JWKS remediation | 1 | Frontend/reference |
| Flowise + platform | EVIDENCE-LIMITED conditional | 1 | Orchestration experiment |
| Langflow + platform | CONDITIONAL after full audit | 1 | Graph orchestration engine |

## 7. Ownership map

| Domain | Canonical platform owner | Engine/sidecar role |
|---|---|---|
| Identity/tenant/workspace/project | Platform | Engine receives scoped context |
| Authorization/policy | Platform | Local checks defense-in-depth |
| Conversations/messages | Platform | Shell/engine projection during migration |
| Short-term context | ContextBudgetService + agent run | LibreChat logic as reference/temporary adapter |
| User/project/episodic/semantic memory | Platform policy + selected MemoryProvider | Mem0 initial engine |
| Document metadata/files | Platform | Engines receive projections |
| RAG/citations | Platform RAG service | Qdrant storage; RAGFlow optional provider |
| Model routing/providers | ModelGateway | LiteLLM initial engine |
| Tools/MCP registry and authorization | Platform MCP gateway | MCP SDK protocol runtime |
| Durable jobs/workflows | Platform JobService | Temporal execution engine |
| Billing/quotas/usage | Platform ledger | LiteLLM/engines emit observations/defense limits |
| Object bytes | Platform file service + object store | Candidate storage migration source only |
| Audit/observability | Platform canonical stream | Engines emit correlated events |

## 8. Context-management recommendation

**Recommendation:** Build a platform context-budget service and use LibreChat’s existing token pruning/compaction as a reference and temporary implementation.

- **FACT:** LibreChat tracks token counts, remaining context, pruning, historical-file authorization, and compaction ([`deep-librechat.md`](../forensic/deep-librechat.md:31)).
- **INFERENCE:** This is the strongest candidate foundation for the required context telemetry, but it does not establish a unified cross-engine approval/promotion protocol.
- **Required platform features:** model-specific capacity registry; current/remaining usage; prioritization; warnings; user-approved condensation; summary/provenance; memory promotion; continuation after compression.
- **Decision:** No engine may silently discard or promote user content outside the platform context policy.
- **Confidence:** HIGH for need; MEDIUM for exact UX/latency.

## 9. Security and commercial risks

1. **Tenant propagation:** no candidate proves uniform tenant propagation through vector, LLM, MCP, and job edges; platform context is mandatory.
2. **RAGFlow data exposure:** confirmed image/thumbnail IDOR; gateway and negative tests required.
3. **LobeChat default secret:** public private JWKS key can forge OIDC/internal JWTs; no deployment before remediation.
4. **Dify commercial dependency:** OSS RBAC and quota paths are unsafe/incomplete for self-hosted commercial use.
5. **Open WebUI tenancy:** no universal tenant unit; fragile naming-convention multitenancy must not be used.
6. **Duplicate policy:** LiteLLM budgets, engine ACLs, Qdrant RBAC, MCP scopes, and Temporal namespaces cannot override platform authorization.
7. **Retry side effects:** model retries may be safe; tool calls, billing, memory writes, and deletions require idempotency.
8. **Deletion drift:** SQL/object/vector/memory/workflow projections need verifiable deletion receipts.
9. **Operational burden:** the preferred architecture is modular but includes several stateful systems; phase adoption and automate recovery.

## 10. Build vs reuse vs integrate

| Capability | Decision | Rationale |
|---|---|---|
| Identity/tenant/workspace/RBAC | BUILD | No candidate provides safe complete commercial authority |
| Policy/audit/quotas/billing | BUILD | Candidate commercial planes are closed, absent, dead, or fail-open |
| Conversation metadata/API | BUILD | Prevent frontend/engine lock-in |
| Chat frontend | REUSE/BUILD | LibreChat/Open WebUI/LobeChat can accelerate; retain replacement option |
| Agent runtime | WRAP/REUSE | LibreChat useful; platform controls tools/state/usage |
| Model provider normalization | INTEGRATE | LiteLLM is strong and API-shaped |
| Long-term memory | INTEGRATE | Mem0 lifecycle is reusable; platform owns scope/policy |
| RAG orchestration | BUILD + INTEGRATE | Platform owns semantics; Qdrant supplies vector infrastructure |
| Advanced parser/GraphRAG | INTEGRATE conditionally | RAGFlow only behind security gateway |
| MCP protocol | REUSE/WRAP | Official SDK; gateway must be platform-owned |
| Durable workflows | INTEGRATE | Temporal is designed as external execution infrastructure |
| Object storage | INTEGRATE | Use S3-compatible abstraction behind platform file service |
| Provider/billing ledger | BUILD | Do not adopt LiteLLM/engine schema as canonical |
| Candidate shared DB | DO NOT USE | Creates irreversible coupling |
| Deep forks | DO NOT USE by default | Reserve only for explicit LibreChat foundation decision |

## 11. Recommended implementation/migration sequence

1. Define platform IDs, `RequestContext`, policy, audit, usage, deletion, and stable interfaces.
2. Add `ModelGateway` with LiteLLM and direct-provider escape hatch.
3. Put LibreChat behind chat-shell adapter; enforce strict isolation and audit global credential/share paths.
4. Move canonical conversation/message metadata to platform; keep engine projections temporarily.
5. Add Mem0 through `MemoryProvider`; migrate native memory with provenance and no permanent dual writes.
6. Build platform RAG service over Qdrant; adapt LibreChat `rag_api`; evaluate RAGFlow only behind gateway.
7. Add Temporal through `JobService` for ingestion, approvals, long-running agents, and deletion verification.
8. Add MCP gateway using SDK; migrate connectors and enforce default-deny/per-call authorization.
9. Implement quotas, billing, storage aggregation, and engine balance retirement.
10. Retain Dify/RAGFlow/Open WebUI/LobeChat/Langflow/Flowise as isolated optional engines until measurable value justifies operation.

## 12. Final decision

The preferred commercial architecture is:

> **Platform-owned control plane + LibreChat-compatible chat/agent shell + LiteLLM model gateway + Mem0 memory sidecar + platform RAG service over Qdrant + MCP gateway using the official Python SDK + Temporal durable execution.**

Dify is a workflow/RAG reference or isolated engine, not the foundation. RAGFlow is a specialized RAG provider only behind a hard gateway and security verification. Open WebUI is an appliance/frontend option, not a shared tenant foundation. LobeChat is a frontend/MCP reference with a substantial closed business-plane gap and mandatory JWKS remediation. Langflow and Flowise remain conditional until evidence quality supports a production decision.

This composition satisfies the target platform’s need for personal AI, workspaces/projects, conversations, persistent memory, project memory, RAG, files, agents, tools, MCP, external applications, APIs, and automation while preserving replacement of the model provider, router, memory engine, vector database, RAG engine, agent runtime, workflow engine, frontend, authentication, object storage, billing provider, MCP runtime, and external integrations.