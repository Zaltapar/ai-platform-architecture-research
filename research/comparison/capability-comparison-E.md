# Wave E — Capability Comparison

**Date:** 2026-09-11 **Status:** Final research artifact (Wave E)

Each claim is labelled **FACT** (source-verified, file:line), **INFERENCE** (reasoned from evidence), or **UNKNOWN** (evidence unavailable).

---

## 1. Memory taxonomy comparison

| Memory dimension | Dify | AnythingLLM | LibreChat | Open WebUI | RAGFlow | Khoj | LobeChat | Mem0 (subsystem reference) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| **Persistent user memory** | ⚠ INFERENCE: conversation/token-buffer memory exists; generic personal/project semantic memory not established ([`deep-dify.md`](../reports/forensic/deep-dify.md) §7) | FACT: `memories` table, global+workspace scope, injection module ([`triage-anythingllm.md`](../reports/forensic/triage-anythingllm.md) §8) | FACT: dedicated memory model + agent callback integration + compaction ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §7) | FACT: SQL memory rows + per-user vector collections + CRUD/reindex/reset ([`deep-open-webui.md`](../reports/forensic/deep-open-webui.md) §7) | FACT: raw/semantic/episodic/procedural; extraction with tenant LLM; forgetting policy ([`deep-ragflow.md`](../reports/forensic/deep-ragflow.md) §7) | FACT: `UserMemory` with list/update/delete APIs, user+agent scoping ([`triage-khoj.md`](../reports/forensic/triage-khoj.md) §8) | INFERENCE: personal memory/tool seams exist; workspace/project memory not uniform ([`deep-lobechat.md`](../reports/forensic/deep-lobechat.md) §7) | FACT: add/search/update/history/delete; `user_id`,`agent_id`,`run_id` scope; no tenant authority ([`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md) §3) |
| **Project/workspace memory** | INFERENCE: tenant-scoped conversation context; no project-level memory class established (§7) | FACT: workspace-scoped memories + injection + reranking ([`triage-anythingllm.md`](../reports/forensic/triage-anythingllm.md) §8) | FACT: project-scoped memory via tenant+conversation context ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §7) | INFERENCE: no universal project partition; per-user collections are user-scoped ([`deep-open-webui.md`](../reports/forensic/deep-open-webui.md) §7) | FACT: tenant-scoped memory extraction, indexes, forgetting per tenant ([`deep-ragflow.md`](../reports/forensic/deep-ragflow.md) §7) | UNKNOWN: no workspace/project identity established ([`triage-khoj.md`](../reports/forensic/triage-khoj.md) §16) | INFERENCE: workspace context in routes; project memory not demonstrated | INFERENCE: Mem0 supports scoped identity; platform must derive scope server-side |
| **Semantic/episodic/procedural** | UNKNOWN: no taxonomy found | UNKNOWN: memories have scope but no explicit taxonomy | INFERENCE: compaction produces summaries; no explicit semantic/episodic class | UNKNOWN: no explicit taxonomy | **FACT: BEST** — modeled and embedded per tenant ([`deep-ragflow.md`](../reports/forensic/deep-ragflow.md) §7, `memory_message_service.py`) | UNKNOWN: no taxonomy found | UNKNOWN: no taxonomy established | INFERENCE: Mem0 can store any category; platform must define the ontology |
| **Forgetting/eviction** | UNKNOWN | UNKNOWN | INFERENCE: compaction is user-approved; no automatic forgetting | UNKNOWN | **FACT:** FIFO eviction + size limits + index deletion ([`deep-ragflow.md`](../reports/forensic/deep-ragflow.md) §7, memory C.5 item 4) | UNKNOWN | UNKNOWN | INFERENCE: Mem0 has `forget`; platform must set policy |
| **Memory usefulness as output** | 2 | 3 | 3 | 3 | **5** | 4 | 3 | 4 |

**Full sources:** per-candidate deep reports §7 / triage §8. Mem0: [`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md). **Recommendation:** Use Mem0 as memory engine for the platform `MemoryProvider`; treat RAGFlow's taxonomy as a reference for ontological design; do not allow dual-write between native memory and Mem0.

---

## 2. Context management

| Capability | LibreChat (best reference) | Dify | Open WebUI | RAGFlow | LobeChat |
|---|---:|---:|---:|---:|---:|
| Token-counted message pruning | **FACT**: yes — `BaseClient.js` tracks tokens, prunes ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §7) | INFERENCE: token-buffer memory exists ([`deep-dify.md`](../reports/forensic/deep-dify.md) §7) | UNKNOWN: not established | UNKNOWN: not established | UNKNOWN: not established in OSS |
| Remaining context budget | **FACT**: remaining tokens exposed ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §7) | UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN |
| User-approved condensation | **INFERENCE**: compaction exists and is gated ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §7) | UNKNOWN | UNKNOWN | INFERENCE: FIFO eviction without user approval | UNKNOWN |
| Authorization for historical context | **FACT**: authorized historical files resolved per message ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §4) | INFERENCE: app-level ACL | FACT: AccessGrants (CWE-863 doc'd) | INFERENCE: tenant scope check | INFERENCE: workspace scope |
| Cross-engine policy | INFERENCE: none — LibreChat-local only | INFERENCE: none | INFERENCE: none | INFERENCE: none | INFERENCE: none |

**Recommendation (from [`target-blueprint-D.md`](../architecture/target-blueprint-D.md) §6):** Build a platform `ContextBudgetService`. Use LibreChat's existing token pruning and compaction as a reference and temporary implementation. No engine may silently discard or promote user content outside platform context policy.

---

## 3. RAG

| RAG dimension | LibreChat | Open WebUI | Dify | RAGFlow | LobeChat |
|---|---:|---:|---:|---:|---:|
| Replaceability of RAG backend | **STRONGEST**: external HTTP `rag_api` seam ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §8) | STRONG: vector factory (10 backends) + async facade ([`deep-open-webui.md`](../reports/forensic/deep-open-webui.md) §8) | MODERATE: vector factory + provider packages; dataset/RBAC contracts remain (§8) | **MOST INTEGRATED**: parser, model, citations, graph — core domain, not replaceable wholesale (§8) | MODERATE: document context mapping exists; parser/index UNKNOWN (§8) |
| Vector backend options | External `rag_api` (UNKNOWN) | **10 backends** inc. Qdrant, Milvus, Chroma | Multiple DBs via `vector_factory.py` | ES/Infinity/OceanBase docstore only (no Qdrant port) | Locked to pgvector @ 1024-dim ([`audit-lobechat.md`](../reports/audit/audit-lobechat.md) §2) |
| Multitenancy isolation | External (seam is clean) | Naming-convention mapping is fragile (red flag OW-1) | Dataset-level ACLs | Per-tenant index naming fixes (IDOR excepted) | Workspace-scoped document service |
| Cross-tenant exposure | None verified | Fragile milvus/qdrant mode (self-documented "HUGE DATA CORRUPTION") | None verified | **CONFIRMED IDOR** at `/documents/images/` and `/thumbnails` (C.5 item 3) | None verified |

**Recommendation (from [`integration-summary-D.md`](../reports/integration/integration-summary-D.md) §5):** Build platform RAG service over Qdrant. Use LibreChat's external `rag_api` seam as compatibility adapter. RAGFlow only through security gateway. Open WebUI vector factory is reference architecture for multi-backend support.

---

## 4. Agents and workflows

| Agent dimension | LibreChat | Dify | RAGFlow | LobeChat | Langflow/Flowise |
|---|---:|---:|---:|---:|---:|
| Durable agent state | FACT: persisted, queued/resumable, checkpoints ([`deep-librechat.md`](../reports/forensic/deep-librechat.md) §9) | FACT: workflow/agent persistence (§9) | INFERENCE: canvas/agent components + tools (§9) | FACT: tool manifests, model runtime (§9) | FACT: graph-flow runtime (the primary extension model) |
| Subagents/multi-agent | FACT: explicit subagent support | INFERENCE: workflow nodes | INFERENCE: agent tools | INFERENCE: tool chains | FACT: graph composition |
| MCP integration | FACT: OAuth, registry, cache (§9) | FACT: native MCP (§9) | FACT: tenant-aware MCP host (§9) | **BEST**: synced manifests, default-deny share, per-call auth (§9, C.5 item 9) | FACT: MCP endpoints/nodes |
| Workflow engine | INFERENCE: none — agent-only | **FACT**: durable workflow/node/agent/tool schemas | INFERENCE: canvas flows | INFERENCE: none — agent-only | **FACT**: graph execution is the core abstraction |

**Recommendation:** Start with LibreChat agent runtime for chat/agent surface; wrap tool calls/approval/usage in platform interfaces. Use Temporal for durable/business workflows. Use Dify/Langflow/Flowise as isolated orchestration engines only when their specific graph models are needed.

---

## 5. MCP / integrations

| MCP dimension | LobeChat (reference) | LibreChat | Open WebUI | RAGFlow | Dify |
|---|---:|---:|---:|---:|---:|
| Tool listing authorization | **BEST**: synced-manifest, optional admin approval ([`deep-lobechat.md`](../reports/forensic/deep-lobechat.md) §9) | INFERENCE: agent-level | **FAIL-CLOSED**: default admin-only when no grants ([C.5 item 9](../evidence/reconciliation-C5.md) item 9) | INFERENCE: MCP tools fetched but no verified call-time gating | INFERENCE: plugin runtime |
| Per-call authorization | FACT: per-call auth in agent runtime | INFERENCE: per-message agent path | **FACT**: `has_connection_access` fail-closed (C.5 item 9) | UNKNOWN | UNKNOWN |
| OAuth/MCP discovery | FACT: OAuth connector, RFC 9728 PRM | INFERENCE: MCP registry | INFERENCE: MCP client | INFERENCE: MCP server config | INFERENCE: MCP config |
| Share-gate default-deny | **FACT**: `spendGate.ts` default deny ([C.5 item 2](../evidence/reconciliation-C5.md) item 2) | INFERENCE: share middleware | INFERENCE: resource ACLs | UNKNOWN | UNKNOWN |

**Recommendation (from [`target-blueprint-D.md`](../architecture/target-blueprint-D.md) §8):** Platform MCP gateway owns registry, grants, credentials, approvals, audit, rate limits. Copy LobeChat's default-deny share gate and Open WebUI's fail-closed per-call auth patterns as design references — not source dependencies. Use MCP Python SDK for protocol transport.

---

## 6. Multi-tenancy

| Tenancy dimension | LibreChat | RAGFlow | Dify | LobeChat | Open WebUI |
|---|---:|---:|---:|---:|---:|
| Tenant primitive | **STRONGEST mechanism**: AsyncLocalStorage + Mongoose query injection + test coverage; **default-off** ([`audit-librechat.md`](../reports/audit/audit-librechat.md) §2) | Strongest domain-native: tenant ID crosses providers, KBs, indexes, MCP; **IDOR-breachable** (§2) | Strong workspace/RBAC: tenant membership + distributed predicates; **closed enterprise RBAC** (§2) | Workspace membership checked in shared helper (`workspace.ts`); **universal coverage UNKNOWN** (§2) | **No tenant unit exists**; roles + groups + AccessGrants only (§2) |
| DB-level isolation | Yes (Mongoose plugin, test-covered) | Per-tenant index naming + per-site `accessible()` checks | Distributed predicates | Application-level | Application-level (AccessGrants) |
| Universal enforcement | **Fail-open default** (TENANT_ISOLATION_STRICT=false) | **IDOR gap** at image/thumbnail endpoints | **RBAC dual regime** (closed enterprise) | Workspace routes checked; coverage unknown | No universal boundary |
| Global credential store risk | **SkillSyncCredential** has no `tenantId` field ([C.5 item 6](../evidence/reconciliation-C5.md) item 6) | N/A | N/A | N/A | N/A |
| **Score** | **4** (mechanism); **0** if strictly scored for default safety | 3 | 4 (mechanism); lower if scored for open-source availability) | 4 | 0 (no tenant unit) / 4 (ACL primitive quality) |

**Recommendation (from [`integration-summary-D.md`](../reports/integration/integration-summary-D.md) §1):** Platform owns identity, tenancy, and authorization. No candidate is safe as universal tenant authority. LibreChat's plugin pattern is the best reference; Dify's RBAC is closed; RAGFlow's is IDOR-breachable; LobeChat's is workspace-only; Open WebUI has no tenant primitive.

---

## 7. APIs

| API dimension | LibreChat | Dify | Open WebUI | RAGFlow | LobeChat |
|---|---:|---:|---:|---:|---:|
| REST/route families | Explicit: auth, agents, files, mcp, balance, admin, API-key, OpenAI-compat | Console/web/service/inner/files/workflow/trigger/MCP | OpenAI-compatible: chat, model, embedding, files, memories, tools | User/admin/provider/document/bot/search/MCP | Next.js route handlers + Hono/tRPC + SDK |
| Streaming | SSE, native | SSE, native | SSE | SSE | SSE (through model runtime) |
| OpenAI-compatible | Yes (agents, chat) | Yes (service API) | Yes (primary API shape) | No | Yes |
| API-key auth | FACT: `AgentApiKey` model, tenant-scoped ([C.5 item 6](../evidence/reconciliation-C5.md) item 6) | FACT: service API keys | FACT: API keys | FACT: JWT/API/beta auth | FACT: session/OIDC/API |
| **Score** | 4 | 3 | 3 | 3 | 4 |

---

## 8. Storage / files

| File dimension | LibreChat | LobeChat | Open WebUI | RAGFlow | Dify |
|---|---:|---:|---:|---:|---:|
| Provider abstraction | Local/S3 (storage adapters) | Local/S3 (file service) | Local/S3 (file router) | Local/MinIO/S3/OSS (`STORAGE_IMPL`) | Local/S3/Azure/GCS (storage adapters) |
| Tenant isolation | FACT: tenant-scoped file metadata + Mongoose plugin | FACT: workspace-scoped document routes | FACT: user-owned + AccessGrants (CWE-863 doc'd) | FACT: KB-scoped; **IDOR** at image/thumbnail points | FACT: dataset-scoped |
| Thumbnail/image exposure | None verified | Public preview is intentional design exception | None verified | **CONFIRMED CROSS-TENANT IDOR** ([C.5 item 3](../evidence/reconciliation-C5.md) item 3) | None verified |
| **Score** | 4 | **5** | 3 | 2 | 3 |

---

## 9. Background jobs

| Job dimension | Dify | LibreChat | RAGFlow | LobeChat | Open WebUI |
|---|---:|---:|---:|---:|---:|
| Queue system | **FACT**: Celery/Redis ([`deep-dify.md`](../reports/forensic/deep-dify.md) §11) | FACT: Redis Streams / in-memory modes (§11) | FACT: Redis Streams + per-process semaphores (§11) | FACT: Redis/QStash (§11) | FACT: Redis/session/socket (§11) |
| Durable execution | **Celery** (mature, vendor-shaped) | Stream/checkpoint modes, not durable | Per-process limiter — weakest at scale | QStash | In-process |
| Scale safety | FACT: Celery workers scale | INFERENCE: checkpoint modes need shared state | FACT: per-process semaphores multiply; no per-tenant fairness (C.5 item 4) | INFERENCE: QStash-backed | INFERENCE: in-process tasks duplicate under scale |

**Recommendation:** Use Temporal as durable execution engine behind `JobService`. Retain candidate job systems for migration compatibility. No candidate's job system is suitable as the platform's durable execution authority.

---

## 10. Usage / accounting

| Usage dimension | Dify | LibreChat | RAGFlow | LobeChat | Open WebUI |
|---|---:|---:|---:|---:|---:|
| Credit/billing schema | **FAIL-OPEN**: Cloud edition only; non-Cloud → allow-all ([`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 5) | **FACT**: balances/transactions; designed but not platform-billing (§11) | **DEAD CODE**: `credit=512` default, `decrease()` has zero call sites ([C.5 item 4](../evidence/reconciliation-C5.md) item 4) | **STUBBED**: 40+ modules are no-ops; usage recording is real OSS; enforcement is closed ([C.5 item 2](../evidence/reconciliation-C5.md) item 2) | **NONE**: session usage reporting only (§11) |
| Quota enforcement | Cloud-gated fail-open | Token/balance based | Dead schema | Closed-source stubs | None |
| Platform-adoptable? | Only with custom billing backend | Only with usage-ledger adapter | **No** (dead schema is misleading) | No (must rebuild 40+ modules) | No |

**Recommendation (from [`integration-summary-D.md`](../reports/integration/integration-summary-D.md) §10):** Build platform usage ledger, quota policy, and billing completely. Do not adopt any candidate's billing schema as canonical. Use LiteLLM spend keys as defensive observation, not product invoices.

---

## 11. Deployment / observability / testing

| Ops dimension | Dify | AnythingLLM | LibreChat | Open WebUI | RAGFlow | Khoj | LobeChat |
|---|---:|---:|---:|---:|---:|---:|---:|
| Default deployment | Docker compose + Celery/Redis | Single container + SQLite | Docker compose + Mongo/Redis/Meili | Docker compose + SQLite/Ollama | Docker compose + ES/MinIO/Redis | Docker/Django/FastAPI/pgvector | Docker compose + PG/Redis/S3/QStash |
| Horizontal scaling | Celery workers scale | **Not default** (SQLite) | Shared DB + Redis modes | Stateless replicas (in-process tasks) | Per-process semaphores; **weakest at scale** | Shared DB needed | Shared external deps |
| Observability | Langfuse | HTTP logging + Jest | Module-level | Analytics + tests | Langfuse + pytest | SQLite/PostHog telemetry | OpenTelemetry/Langfuse |
| Test coverage | Mixed (2/5) | Jest coverage (3/5) | Module tests + e2e (3/5) | Provider/router tests (3/5) | pytest (2/5) | Agent/privacy tests (3/5) | **Thin** (2/5) |

**Source evidence:** per-candidate deep reports §14–15 / triage §14–15.

---

## 12. Source evidence index

| Claim | Source report | File:line anchor |
|---|---|---|
| LibreChat tenant plugin mechanism | [`deep-librechat.md`](../reports/forensic/deep-librechat.md) §3 | [`tenantIsolation.ts`](../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts) |
| LibreChat strict mode default-off | [`audit-librechat.md`](../reports/audit/audit-librechat.md) §2 | [`policy.ts:48-63`](../../repos/librechat/packages/data-schemas/src/tenant/policy.ts) |
| LibreChat compaction | [`deep-librechat.md`](../reports/forensic/deep-librechat.md) §7 | [`compaction.ts`](../../repos/librechat/packages/api/src/agents/compaction.ts) |
| Dify RBAC dual regime | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 5 | [`account.py:192-239`](../../repos/dify/api/models/account.py) |
| Dify quota fail-open | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 5 | [`quota_service.py:121-123`](../../repos/dify/api/services/quota_service.py) |
| RAGFlow IDOR | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 3 | [`document_api.py:1831-1866`](../../repos/ragflow/api/apps/restful_apis/document_api.py) |
| RAGFlow dead credit | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 4 | [`db_models.py:1163`](../../repos/ragflow/api/db/db_models.py) |
| RAGFlow memory taxonomy | [`deep-ragflow.md`](../reports/forensic/deep-ragflow.md) §7 | [`memory_message_service.py`](../../repos/ragflow/api/db/joint_services/memory_message_service.py) |
| LobeChat JWKS live key | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 1 | [`deploy/.env.example:69`](../../repos/lobechat/docker-compose/deploy/.env.example) |
| LobeChat business stubs | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 2 | [`workspace.ts:46-82`](../../repos/lobechat/packages/business-server/src/lambda-routers/workspace.ts) |
| LobeChat MCP synced manifests | [`deep-lobechat.md`](../reports/forensic/deep-lobechat.md) §9 | [`buildClientConnectorManifests.ts`](../../repos/lobechat/src/helpers/toolEngineering/buildClientConnectorManifests.ts) |
| Open WebUI CWE-863 defense | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 8 | [`files.py:45-63`](../../repos/open-webui/backend/open_webui/utils/access_control/files.py) |
| Open WebUI MCP fail-closed | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 9 | [`middleware.py:2335`](../../repos/open-webui/backend/open_webui/utils/middleware.py) |
| Open WebUI fragile vector multitenancy | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 7 | [`milvus_multitenancy.py:84-106`](../../repos/open-webui/backend/open_webui/retrieval/vector/dbs/milvus_multitenancy.py) |
| LibreChat SkillSyncCredential global store | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 6 | [`schema/skillSyncCredential.ts`](../../repos/librechat/packages/data-schemas/src/schema/skillSyncCredential.ts) |
| LibreChat runAsSystem share path | [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 6 | [`optionalShareFileAuth.js:23,71`](../../repos/librechat/api/server/middleware/optionalShareFileAuth.js) |
| LiteLLM gateway/router seams | [`subsystem-litellm.md`](../reports/forensic/subsystem-litellm.md) §3 | [`router.py`](../../repos/litellm/litellm/router.py) |
| Mem0 lifecycle API | [`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md) §3 | [`main.py`](../../repos/mem0/mem0/memory/main.py) |
| MCP SDK protocol runtime | [`subsystem-mcp-python-sdk.md`](../reports/forensic/subsystem-mcp-python-sdk.md) §3 | [`server.py`](../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py) |
| Qdrant vector API | [`subsystem-qdrant.md`](../reports/forensic/subsystem-qdrant.md) §3 | [`schema.rs`](../../repos/qdrant/lib/api/src/rest/schema.rs) |
| Temporal durable execution | [`subsystem-temporal.md`](../reports/forensic/subsystem-temporal.md) §3 | [`fx.go`](../../repos/temporal/temporal/fx.go) |