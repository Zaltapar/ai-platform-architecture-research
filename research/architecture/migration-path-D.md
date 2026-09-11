# Wave D — Migration Path to Platform-Owned Interfaces

**Date:** 2026-09-11  
**Status:** Architecture plan; no production implementation.  
**Objective:** Move from selected reuse candidates and subsystem integrations to platform-owned contracts without modifying cloned repositories.

## 1. Migration principles

1. **Platform authority before engine adoption.** Do not let an engine schema become canonical before identity, policy, usage, and deletion contracts exist.
2. **One-way ownership migration.** Every domain has one canonical owner at each phase; projections may be dual-read, but permanent dual-write is prohibited.
3. **Anti-corruption layers (ACLs) at every boundary.** Translate identity, policy, schemas, errors, usage, streaming, and lifecycle semantics.
4. **API before SDK, SDK before source integration.** Prefer Level 1 HTTP/API; use Level 2 SDK only inside an adapter. Avoid Level 3 shared databases and reject Level 4/5 except a deliberate foundation fork.
5. **Reversible cutovers.** Every migration has feature flags, shadow mode, parity metrics, read fallback, export checkpoints, and a rollback owner.
6. **No silent security downgrade.** Missing tenant context, policy uncertainty, credential ambiguity, or deletion uncertainty fails closed.

- **FACT:** The evidence recommends platform plane plus engines, with LibreChat as the strongest forced-pick shell, and specialized subsystems externalized behind interfaces ([`audit-summary-C.md`](../reports/audit/audit-summary-C.md:159)).
- **INFERENCE:** The migration must prioritize control-plane authority and escape hatches over feature completeness.
- **UNKNOWN:** Exact throughput, latency, and operational staffing are not established.

## 2. End-state and transition map

```text
Phase 0: evidence and contracts
  Candidate products remain isolated; define platform IDs and interfaces.

Phase 1: platform shell + external model gateway
  Platform identity/policy/metadata + LiteLLM adapter; direct-provider fallback.

Phase 2: canonical conversations/context
  Platform conversation API; LibreChat/Open WebUI/LobeChat become clients or projections.

Phase 3: memory extraction and retrieval
  Platform MemoryProvider + Mem0 adapter; native engine memory becomes compatibility read path.

Phase 4: document/RAG ownership
  Platform document metadata + RAG service + Qdrant; candidate RAG engines become projections/optional providers.

Phase 5: durable jobs and integrations
  JobService + Temporal; MCP gateway + SDK; external tools move behind platform grants.

Phase 6: engine retirement or specialization
  Remove duplicate state/policy paths; retain engines only where their domain advantage is measurable.
```

## 3. Phase 0 — Contract and safety foundation

### Deliverables

- Platform IDs: `principal_id`, `tenant_id`, `workspace_id`, `project_id`, `conversation_id`, `document_version_id`, `memory_id`, `agent_run_id`, `job_id`, `tool_call_id`.
- Signed/mTLS `RequestContext` with deadline, policy version, consent, data classification, and idempotency key.
- Canonical platform SQL schema for identity, membership, policy, conversations, files/document metadata, usage events, audit, and migration mappings.
- Versioned contracts for `ModelGateway`, `MemoryProvider`, `RagProvider`, `McpGateway`, `JobService`, and `ContextBudgetService`.
- Contract-test harness with fake provider, fake memory, fake vector store, fake MCP server, and fake workflow engine.
- Data-classification, retention, deletion, and export policies.

### Anti-corruption layer

`PlatformContextAdapter` converts edge authentication to internal context. It rejects user-supplied tenant headers unless they are merely a non-authoritative selector validated against membership. Each downstream adapter receives a scoped service token plus context claims.

### Exit gates

- **Security:** no downstream call can execute without validated tenant/workspace/project scope.
- **Data:** platform IDs and provenance are stable and globally unique.
- **Operations:** trace IDs are visible end-to-end in test environments.
- **Rollback:** no production engine data is migrated yet; this phase is inherently reversible.

## 4. Phase 1 — Model plane: LiteLLM behind `ModelGateway`

### Starting state

LibreChat, Dify, Open WebUI, LobeChat, and other engines have provider paths. LiteLLM provides provider normalization, routing, retry/fallback, budgets, callbacks, and usage/cost surfaces ([`subsystem-litellm.md`](../reports/forensic/subsystem-litellm.md:19)).

### Migration steps

1. Deploy LiteLLM as an isolated service with dedicated PostgreSQL/Redis where gateway features require them.
2. Implement `ModelGateway` HTTP adapter. Hide LiteLLM virtual keys, teams, projects, and Prisma IDs.
3. Map platform model policies to LiteLLM model groups and scoped credentials.
4. Add request/tenant/project correlation headers and callback ingestion.
5. Run shadow mode: duplicate only metadata/capability estimation where safe; do not duplicate billable provider calls.
6. Route a canary percentage of non-tool model calls through LiteLLM.
7. Reconcile provider usage/cost against platform ledger before expanding.
8. Keep direct provider adapter as an escape hatch behind the same interface.

### Anti-corruption layer

`LiteLLMAdapter` translates:

- platform model alias → LiteLLM model group;
- platform reservation → gateway key/budget context;
- provider stream/error/usage → normalized platform events;
- LiteLLM spend record → observed usage event, never an authoritative invoice;
- retry/fallback attempt → one platform request with attempt children.

### Cutover and rollback

- **Cutover:** usage parity, stream cancellation, timeout, fallback, and error mapping pass contract tests.
- **Rollback:** route traffic to direct provider adapter; retain LiteLLM attempt events for reconciliation.
- **Escape hatch:** replace LiteLLM with direct adapters, another gateway, or an alternate OpenAI-compatible service without changing conversations or agents.

### Risks

- Retry multiplication and duplicate billing.
- Context-window differences between platform model registry and LiteLLM/provider metadata.
- LiteLLM gateway schema becoming accidental public contract.

**Confidence:** HIGH for boundary; MEDIUM for performance until tested.

## 5. Phase 2 — Platform identity, conversations, and context

### Starting state

LibreChat has Mongo/Mongoose conversation/message/project/file/agent state and a strong but default-off tenant plugin; Open WebUI has SQL user/chat state; LobeChat has workspace-aware routes but a closed business plane ([`reconciliation-C5.md`](../evidence/reconciliation-C5.md:41)).

### Migration steps

1. Make platform identity and membership authoritative at the edge.
2. Put LibreChat behind a `ChatShellAdapter` using API/session translation; set strict tenant isolation as a mandatory deployment gate if LibreChat is used.
3. Introduce platform conversation IDs and mapping table: `platform_conversation_id ↔ engine_conversation_id`.
4. Capture engine events into platform conversation/message records in append-only form.
5. Expose platform conversation API to new frontends; legacy shell reads through compatibility mapping.
6. Move new writes to platform conversation service; engine state becomes a projection/cache where practical.
7. Validate message order, stream resume, attachments, tool events, and public-share policy.
8. Retire direct engine conversation writes after parity and export verification.

### Anti-corruption layer

`ChatShellAdapter` handles:

- identity/session/API-key translation;
- workspace/project selection;
- message and attachment ID mapping;
- SSE/event normalization;
- engine-specific agent/checkpoint references;
- canonical platform audit and usage correlation.

### Context budget migration

Use LibreChat’s token-counted pruning and compaction as a reference and temporary implementation. Wrap it with `ContextBudgetService` so the platform exposes model limit, used tokens, remaining capacity, priority decisions, warnings, condensation proposals, approval, and continuation. Do not silently promote or delete memory during migration.

### Cutover and rollback

- **Cutover:** platform and engine transcript hashes/order, attachment authorization, stream completion, and context counts match within documented tolerances.
- **Rollback:** restore engine as read/write shell while preserving platform event log; stop platform projection writes only after a checkpoint.
- **Escape hatch:** replace LibreChat with a native frontend, Open WebUI, LobeChat, or custom UI without changing platform conversations.

## 6. Phase 3 — Memory: native memory to `MemoryProvider` + Mem0

### Starting state

LibreChat has dedicated persistent memory and compaction; Open WebUI has SQL/vector personal memory; RAGFlow has the strongest taxonomy and extraction lifecycle; Mem0 provides add/search/update/history/delete but caller-derived scope is not a complete tenant authority ([`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md:27)).

### Migration steps

1. Define platform memory classes: profile, project, semantic, episodic, conversation summary, and agent/run state.
2. Add consent, provenance, confidence, source-message, retention, and deletion records in platform SQL.
3. Deploy Mem0 behind `MemoryProvider`; derive scope from authenticated context.
4. Import native engine memories into staging with source and confidence metadata.
5. Run shadow search: compare Mem0 results to native memory without changing model prompts.
6. Enable Mem0 retrieval for a canary project with strict token budget.
7. Move writes to one path: platform policy → Mem0 adapter. Disable native dual writes.
8. Reconcile deletes, exports, updates, and user-approved memory promotion.
9. Retain native memory only as read-only compatibility until migration retention expires.

### Anti-corruption layer

`Mem0Adapter`:

- rejects arbitrary client filters;
- maps project/user/agent/run scopes to server-derived Mem0 identities;
- translates Mem0 payloads into stable `MemoryRecord` objects;
- stores platform provenance and consent separately;
- converts Mem0 failures to explicit degraded-memory statuses;
- enforces idempotency using source event hash and memory operation key.

### Cutover and rollback

- **Cutover:** no cross-scope search; deterministic deletion receipt; duplicate rate below threshold; retrieval quality and token budget accepted by product review.
- **Rollback:** disable Mem0 retrieval and return to native read-only memory; preserve platform memory events for replay.
- **Escape hatch:** replace Mem0 with another provider without changing memory IDs, scope, provenance, or API.

### Special RAGFlow case

RAGFlow’s memory taxonomy is a reference for semantic/episodic/procedural categories, but do not combine RAGFlow memory, Mem0 memory, and native engine memory as co-equal stores. Choose one canonical provider per memory class.

## 7. Phase 4 — Documents/RAG: platform service + Qdrant

### Starting state

LibreChat already exposes an external `rag_api`; Open WebUI has a vector factory; Dify has a vector factory/domain model; RAGFlow owns an integrated advanced RAG pipeline; Qdrant provides the vector storage boundary only ([`subsystem-qdrant.md`](../reports/forensic/subsystem-qdrant.md:19)).

### Migration steps

1. Create platform document/file/version metadata and object-storage references.
2. Implement `RagProvider` with ingestion status, retrieval citations, delete, and reindex.
3. Deploy Qdrant behind a private service network; platform RAG service controls collections/payload filters.
4. Migrate file bytes through object-storage copy or signed transfer; never expose engine internal URLs.
5. Re-embed/re-index asynchronously using Temporal; maintain embedding version and source hash.
6. Adapt LibreChat `rag_api` endpoints to platform RAG contract.
7. Compare retrieval/citation parity in shadow mode.
8. For RAGFlow evaluation, place a gateway in front of every API and block raw image/thumbnail/artifact paths until ownership checks are independently verified.
9. Cut over per workspace/project; retain old index for rollback until delete and citation parity are proven.

### Anti-corruption layer

`RagEngineAdapter` maps engine KB/document/chunk IDs to platform document/version/chunk IDs. It strips engine collection names, normalizes citations, injects server-side filters, and translates deletion into a platform deletion workflow with verification.

### Cutover and rollback

- **Cutover:** all retrievals return only authorized citations; index counts/statuses reconcile; deletion verification passes SQL/object/vector checks.
- **Rollback:** switch `RagProvider` implementation or retrieval read route to old engine; do not reverse-delete platform canonical metadata.
- **Escape hatch:** replace Qdrant with pgvector/Weaviate/Milvus or RAGFlow with no frontend/domain schema rewrite.

### Security gate

**FACT:** RAGFlow has confirmed unguarded image and thumbnail enumeration paths ([`reconciliation-C5.md`](../evidence/reconciliation-C5.md:76)). RAGFlow cannot be a direct production data plane until gateway enforcement or upstream fixes are verified by targeted tests.

## 8. Phase 5 — Durable jobs: Temporal behind `JobService`

### Starting state

Candidates use in-process work, Celery, Redis streams, or per-process semaphores; none provides the platform’s durable workflow authority. Temporal supplies histories, task queues, retries, namespaces, and workers ([`subsystem-temporal.md`](../reports/forensic/subsystem-temporal.md:19)).

### Migration steps

1. Define platform job records and idempotency keys first.
2. Use Temporal namespaces/task queues mapped to environment or bounded security domains, not as the sole tenant authorization mechanism.
3. Implement versioned workflows for document ingestion, memory extraction, deletion verification, tool execution, approvals, schedules, and reconciliation.
4. Move one non-user-visible ingestion path from Celery/Redis/in-process execution.
5. Compare completion, retry, cancellation, and duplicate-side-effect metrics.
6. Move approval-dependent and long-running agent tasks.
7. Keep synchronous short operations outside Temporal where durable orchestration adds no value.
8. Retire candidate workers only after outstanding jobs are drained/exported.

### Anti-corruption layer

`TemporalJobAdapter` translates platform job type/input references to workflow IDs and versioned payloads. Activities receive platform context and use idempotent APIs. Temporal history is operational execution state; platform job table is user-visible business state.

### Cutover and rollback

- **Cutover:** workflow replay/version tests, idempotent activity tests, cancellation/timeout, quota admission, and audit correlation pass.
- **Rollback:** stop new workflow starts, allow existing jobs to drain, route new jobs to legacy worker adapter, and reconcile status.
- **Escape hatch:** implement another workflow engine behind `JobService`; workflow histories are not exposed as product APIs.

## 9. Phase 6 — MCP and external applications

### Starting state

LobeChat has deliberate connector manifests and security patterns; Open WebUI has per-call MCP authorization; MCP Python SDK provides protocol transports/auth/extensions but no tenant policy or durable jobs ([`subsystem-mcp-python-sdk.md`](../reports/forensic/subsystem-mcp-python-sdk.md:27)).

### Migration steps

1. Create platform connector registry, server/tool manifests, credential references, scopes, approval state, and audit schema.
2. Deploy MCP gateway using SDK transports; no product client connects directly to arbitrary MCP endpoints.
3. Import candidate connector metadata as read-only projections.
4. Enforce tool listing and per-call authorization; default deny when grants are absent.
5. Add sandbox/timeouts/rate limits/data classification and human approval for risky tools.
6. Move GitHub, Telegram, Odoo, Office/Google, REST/database, local-file, and custom applications one connector at a time.
7. Use Temporal for long-running or side-effectful integrations.
8. Remove direct candidate credentials and connector calls after audit parity.

### Anti-corruption layer

`McpEngineAdapter` translates engine tool schemas to platform manifests, maps credentials by reference, normalizes tool results/errors, and records call provenance. SDK protocol changes are confined to this adapter.

### Escape hatch

Replace MCP SDK with another conformant implementation or use direct REST/SDK connectors behind the same `ToolProvider` contract.

## 10. Phase 7 — Billing, quotas, and storage ownership

### Migration steps

1. Implement append-only usage ledger, reservation/commit/release/refund, quota policy, and reconciliation jobs.
2. Ingest LiteLLM usage/cost callbacks, engine usage, tool calls, storage bytes, vector units, and workflow compute as observed events.
3. Make platform admission authoritative before model/tool/ingestion/job execution.
4. Keep LiteLLM budgets and engine balances as defensive limits or projections.
5. Migrate file metadata and storage quotas to platform file service; object bytes may be copied lazily.
6. Reconcile engine balances/transactions; do not import them as invoices without provenance.

- **FACT:** Dify quota is cloud-gated/fail-open outside Cloud; RAGFlow’s credit decrement code has zero call sites; LobeChat business billing is stubbed ([`reconciliation-C5.md`](../evidence/reconciliation-C5.md:161)).
- **INFERENCE:** Billing must be built at the platform plane, not retrofitted by selecting one candidate.

## 11. Engine-specific migration policies

### LibreChat

- Use as initial shell only after mandatory strict isolation configuration, credential-store audit, and public-share threat model.
- Preserve its agent/MCP/context features through adapters; do not make its balance or Mongo tenant model canonical.
- Preferred migration path: API integration → platform conversation projection → platform conversation authority.

### Dify

- Use only as an isolated workflow/RAG engine or benchmark.
- Do not enable OSS RBAC assumptions as platform authorization; do not rely on non-cloud quota enforcement.
- Preferred path: platform gateway → Dify service API → mapped app/workflow IDs; no shared DB.

### RAGFlow

- Use only behind RAG gateway, endpoint allowlist, patched storage-read paths, and deletion verification.
- Keep RAGFlow tenant/KB/index IDs as external projection IDs.
- Preferred path: specialized RAG provider; no foundation role.

### Open WebUI

- Use as optional frontend/appliance or tenant-per-instance deployment.
- Do not enable fragile naming-convention multitenancy mode as commercial isolation.
- Native personal memory and vector data must be migrated through platform adapters if platform memory/RAG is authoritative.

### LobeChat

- Remediate example JWKS immediately before any deployment; use external identity in production.
- Treat business-server stubs as a signal to use platform business APIs, not as an implementation shortcut.
- Use MCP manifest/security patterns as reference; do not fork for commercial business-plane completion unless a deliberate high-cost decision is approved.

### Flowise/Langflow

- Keep conditional and isolated until complete source/security audit.
- Use only flow/run API adapters; platform owns identity, credentials, quota, jobs, and durable state.
- Do not allow graph components to choose arbitrary memory/vector/tool scopes.

## 12. Data migration and deletion protocol

Every migrated record carries:

```text
platform_id
source_system
source_id
source_version
tenant/workspace/project scope
provenance
created_at/updated_at
migration_batch_id
checksum
retention/deletion state
```

Deletion is a workflow:

`authorize → mark tombstone → stop new reads/writes → delete platform metadata → delete object bytes → delete vector points → delete Mem0 records → cancel/complete workflows → verify projections → emit receipt`.

If any step is uncertain, report `deletion_pending` and block re-exposure. Do not treat best-effort engine cascades as platform deletion proof.

## 13. Observability and acceptance matrix

| Gate | Required evidence before cutover |
|---|---|
| Tenant safety | Negative tests for wrong tenant/workspace/project at every adapter |
| Authorization | Policy decision and downstream enforcement audit match |
| Usage | Reservation/commit parity and retry behavior reconciled |
| Context | Model-specific limit, remaining budget, compaction/approval behavior verified |
| Memory | Scope, provenance, delete, export, duplicate, and degraded-mode tests |
| RAG | Citation ACL, reindex, deletion, stale-index, and embedding-version tests |
| MCP | Tool listing/call authorization, credential isolation, timeout, audit, default-deny tests |
| Jobs | Replay, retry, cancellation, idempotency, compensation, and status parity |
| Operations | Traces, metrics, alerts, backups, restore, and migration rehearsal |
| Upgrade | Contract tests against pinned and candidate upstream versions |

## 14. Long-term escape hatches

- `ModelGateway`: LiteLLM → direct provider adapters → another gateway.
- `MemoryProvider`: Mem0 → custom service → another memory engine.
- `VectorIndex`: Qdrant → pgvector/Weaviate/Milvus.
- `RagProvider`: custom RAG → RAGFlow → another RAG engine.
- `AgentRuntime`: LibreChat runtime → platform runtime → Dify/Langflow/other isolated engine.
- `Frontend`: platform UI ↔ LibreChat/Open WebUI/LobeChat.
- `JobService`: Temporal → another durable execution engine.
- `McpGateway`: MCP Python SDK → another conformant SDK/direct connectors.
- `ObjectStore`: S3-compatible → cloud/object provider.
- `Identity`: external OIDC/SSO provider → another IdP.
- `Billing`: platform ledger provider → external invoicing/billing service.

## 15. Migration risks and mitigations

1. **Dual-write divergence:** use outbox/events and one canonical write owner; dual-read only temporarily.
2. **ID mapping leaks:** never accept source IDs as authorization inputs; resolve through platform mapping.
3. **Stream incompatibility:** normalize events and preserve resumable sequence numbers.
4. **Retry multiplication:** classify operations and assign one retry owner.
5. **Backfill privacy:** run migration workers with tenant-scoped context and audit every record.
6. **Vendor lock-in through convenience:** prohibit candidate IDs/schemas in public APIs.
7. **Operational overload:** phase stateful services; start with LiteLLM and platform DB before adding Mem0/Qdrant/Temporal at scale.
8. **Security regression:** mandatory negative tests and fail-closed defaults at every cutover.

## 16. Recommended sequence

1. Platform identity/policy/IDs/contracts.
2. ModelGateway + LiteLLM with direct-provider escape hatch.
3. Canonical conversation/context service; LibreChat as shell.
4. MemoryProvider + Mem0 with controlled migration.
5. Platform RAG + Qdrant; adapt LibreChat RAG seam.
6. JobService + Temporal for ingestion/approvals/long-running agents.
7. MCP gateway + SDK and connector migration.
8. Billing/storage quota enforcement and engine balance retirement.
9. Evaluate Dify/RAGFlow/Open WebUI/LobeChat/Langflow/Flowise as replaceable specialized engines.
10. Retire duplicate state and policy only after export, reconciliation, and rollback windows close.

**Final recommendation:** The platform should grow by replacing adapters, not by deepening forks. A LibreChat foundation fork is a possible later product decision, but migration should begin with API boundaries so the decision remains reversible.