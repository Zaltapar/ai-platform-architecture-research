# Wave B Deep Forensic Summary — Dify, LibreChat, Open WebUI, RAGFlow, LobeChat, Langflow

## 1. Scope/evidence quality

**Scope.** This Wave B investigation covers exactly six repositories: [`Dify`](../../repos/dify:1), [`LibreChat`](../../repos/librechat:1), [`Open WebUI`](../../repos/open-webui:1), [`RAGFlow`](../../repos/ragflow:1), [`LobeChat`](../../repos/lobechat:1), and [`Langflow`](../../repos/langflow:1). Orientation reports were used only to locate anchors; conclusions were verified against cloned source where materialized.

**FACT:** Six deep reports were produced: [`deep-dify.md`](deep-dify.md:1), [`deep-librechat.md`](deep-librechat.md:1), [`deep-open-webui.md`](deep-open-webui.md:1), [`deep-ragflow.md`](deep-ragflow.md:1), [`deep-lobechat.md`](deep-lobechat.md:1), and [`deep-langflow.md`](deep-langflow.md:1).

**Evidence limitations:** Langflow has a partial Windows checkout caused by filename-length limits; its login/token and full schema paths remain UNKNOWN. LibreChat’s vector/parser implementation is an external `rag_api` boundary, not fully inspectable in the Node repository. LobeChat’s complete parser/chunker/index path and billing/quota ledger were not fully established. These are blockers for targeted Wave C audits, not inferred absences.

## 2. Architecture/data ownership

- **FACT — Dify:** Tenant/app/dataset/workflow/agent/provider/tool/message state is API-database-owned; workers/Redis and separate runtime services own asynchronous execution. Provider and vector factories are explicit, but the product domain is deeply integrated ([`provider_manager.py`](../../repos/dify/api/core/provider_manager.py:17), [`vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29)).
- **FACT — LibreChat:** Mongo/Mongoose schemas own user/session/conversation/message/project/file/agent/memory/permission/balance state; Redis/job/checkpoint, Meilisearch and `rag_api` are externalized ([`api/db/index.js`](../../repos/librechat/api/db/index.js:1), [`docker-compose.yml`](../../repos/librechat/docker-compose.yml:56)).
- **FACT — Open WebUI:** SQLAlchemy owns user/chat/file/knowledge/memory/model/tool state; vector and storage clients are configurable. The main chat orchestration remains concentrated in [`main.py`](../../repos/open-webui/backend/open_webui/main.py:1085).
- **FACT — RAGFlow:** Tenant/domain state crosses relational models, storage/index names, provider bundles, memory and API services ([`db_models.py`](../../repos/ragflow/api/db/db_models.py:1141)).
- **FACT — LobeChat:** Drizzle/database, workspace/auth, context engine, model runtime and multiple client surfaces are separate packages/routes but cross-coupled by user/workspace context ([`workspace.ts`](../../repos/lobechat/src/app/(backend)/webapi/_utils/workspace.ts:8)).
- **FACT — Langflow:** Flow graphs/components are the primary orchestration artifact; backend, `lfx`, bundles, SDK, database, job and event layers are separate, with incomplete checkout evidence ([`build.py`](../../repos/langflow/src/backend/base/langflow/api/build.py:1)).

## 3. LOGIN trace findings

- **Dify FACT:** Login calls `WebAppAuthService.authenticate` then `login`; tenant/RBAC identity is applied by shared wrappers and `TenantAccountJoin` ([`login.py`](../../repos/dify/api/controllers/web/login.py:106), [`wraps.py`](../../repos/dify/api/controllers/common/wraps.py:20)).
- **LibreChat FACT:** Login middleware composes rate limiting, ban/email/LDAP/local auth and controller; tenant context then injects filters into Mongoose queries and guards mutations ([`auth.js`](../../repos/librechat/api/server/routes/auth.js:42), [`tenantIsolation.ts`](../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96)).
- **Open WebUI FACT:** Bearer/cookie/API-key/OAuth token sources resolve to verified user; most resource selection is user/ACL scoped ([`asgi_middleware.py`](../../repos/open-webui/backend/open_webui/utils/asgi_middleware.py:128), [`auth.py`](../../repos/open-webui/backend/open_webui/utils/auth.py:522)).
- **RAGFlow FACT:** Session, beta, JWT and API-token authentication flow through `login_required`; tenant/user membership is first-class ([`apps/__init__.py`](../../repos/ragflow/api/apps/__init__.py:235), [`db_models.py`](../../repos/ragflow/api/db/db_models.py:1170)).
- **LobeChat FACT:** Better Auth web sessions and OIDC CLI auth are resolved by `checkAuth`; workspace header is accepted only after existence and active membership checks ([`auth/index.ts`](../../repos/lobechat/src/app/(backend)/middleware/auth/index.ts:61), [`workspace.ts`](../../repos/lobechat/src/app/(backend)/webapi/_utils/workspace.ts:8)).
- **Langflow UNKNOWN:** Complete login/token chain could not be verified due to partial checkout. Available code proves user/flow scoping, not complete auth architecture ([`flow.py`](../../repos/langflow/src/backend/base/langflow/helpers/flow.py:60)).

## 4. CHAT trace findings

- **Dify FACT:** `ChatApi.post` validates app/conversation and calls `AppGenerateService.generate`; generator/runner owns context, memory, task/stream and mode-specific execution ([`completion.py`](../../repos/dify/api/controllers/web/completion.py:208), [`app_generator.py`](../../repos/dify/api/core/app/apps/chat/app_generator.py:37)).
- **LibreChat FACT:** `BaseClient.sendMessage` prepares/persists user message, resolves authorized historical files, token-prunes/builds messages, invokes provider clients and persists response; agent path adds queues/checkpoints/MCP ([`BaseClient.js`](../../repos/librechat/api/app/clients/BaseClient.js:732), [`client.js`](../../repos/librechat/api/server/controllers/agents/client.js:4359)).
- **Open WebUI FACT:** `chat_completion` resolves model/access/params/files/tools/variables and usage flags in a single async route, then downstream middleware/provider/vector/tool functions perform execution ([`main.py`](../../repos/open-webui/backend/open_webui/main.py:1085)).
- **RAGFlow FACT:** `async_chat_solo` resolves tenant models, attachments, prompt and stream/non-stream bundle calls; RAG dialog paths add embeddings/rerank/graph retrieval ([`dialog_service.py`](../../repos/ragflow/api/db/services/dialog_service.py:293)).
- **LobeChat FACT:** Authenticated chat route validates workspace, initializes DB-backed model runtime and calls `modelRuntime.chat`; tool/context/persistence sequence beneath runtime is distributed ([`route.ts`](../../repos/lobechat/src/app/(backend)/webapi/chat/[provider]/route.ts:18)).
- **Langflow FACT/INFERENCE:** Build API executes a graph with memory/RAG/model/tool/MCP components, events and checkpoints; there is no single central chat pipeline ([`build.py`](../../repos/langflow/src/backend/base/langflow/api/build.py:1)).

## 5. FILE trace findings

- **Dify FACT:** Upload route delegates to application file service; SQL file metadata, storage adapters, dataset documents/segments/chunks and vector factories span ingestion ([`files.py`](../../repos/dify/api/controllers/web/files.py:41), [`dataset.py`](../../repos/dify/api/models/dataset.py:166)).
- **LibreChat FACT:** File services own storage/model/permissions while text extraction and embedding cross an authenticated `RAG_API_URL` contract ([`text.ts`](../../repos/librechat/packages/api/src/files/text.ts:83), [`VectorDB/crud.js`](../../repos/librechat/api/server/services/Files/VectorDB/crud.js:67)). Vector/parser internals are external UNKNOWN.
- **Open WebUI FACT:** Upload stores bytes/owner tags, inserts metadata, then indexing splits documents, calls embedding function and inserts vectors through the vector client ([`files.py`](../../repos/open-webui/backend/open_webui/routers/files.py:271), [`retrieval.py`](../../repos/open-webui/backend/open_webui/routers/retrieval.py:1637)).
- **RAGFlow FACT:** Upload creates tenant/KB folders, stores bytes/thumbnails, creates document state, and asynchronous executor parses/chunks/embeds/inserts tenant-indexed chunks ([`file_service.py`](../../repos/ragflow/api/db/services/file_service.py:578), [`task_executor.py`](../../repos/ragflow/rag/svr/task_executor.py:1322)).
- **LobeChat FACT/UNKNOWN:** Workspace-aware `DocumentService` and context-document mapping are proven; complete parser/chunker/embed/index sequence remains UNKNOWN ([`document/events/route.ts`](../../repos/lobechat/src/app/(backend)/webapi/document/events/route.ts:28), [`agentDocumentContextMapping.ts`](../../repos/lobechat/src/helpers/agentDocumentContextMapping.ts:1)).
- **Langflow FACT/INFERENCE:** File/RAG flow is graph-dependent: file loader → splitter → retriever/vector node, not one centralized pipeline ([`vector_store_rag.py`](../../repos/langflow/src/backend/base/langflow/initial_setup/starter_projects/vector_store_rag.py:1)).

## 6. API trace findings

- **Dify FACT:** Console/web/service/inner/files/workflow/trigger/MCP route families are explicit; service API/end-user identities are distinct ([`api/controllers`](../../repos/dify/api/controllers:1)).
- **LibreChat FACT:** REST, API-key and OpenAI-compatible agent routes are explicit; public machine access still enters permission/tenant-aware agent paths ([`openai.js`](../../repos/librechat/api/server/routes/agents/openai.js:119)).
- **Open WebUI FACT:** OpenAI-compatible chat/model/embedding/Anthropic-style APIs share FastAPI auth and backend orchestration ([`main.py`](../../repos/open-webui/backend/open_webui/main.py:874)).
- **RAGFlow FACT:** User, admin, provider, document, bot, search and MCP APIs coexist with tenant injection/auth decorators ([`document_api.py`](../../repos/ragflow/api/apps/restful_apis/document_api.py:123)).
- **LobeChat FACT:** Next.js route handlers plus Hono/tRPC/SDK surfaces expose auth/chat/models/documents/agents/connectors; auth injects DB and telemetry context ([`auth/index.ts`](../../repos/lobechat/src/app/(backend)/middleware/auth/index.ts:61)).
- **Langflow FACT:** FastAPI build/run/stream/cancel/event and SDK contracts are explicit; full auth route remains UNKNOWN ([`build_utils.py`](../../repos/langflow/src/backend/tests/unit/build_utils.py:17)).

## 7. Memory/context

- **Strongest implemented memory taxonomy — FACT:** RAGFlow models raw/semantic/episodic/procedural memory, extracts with tenant LLM, embeds/indexes, applies forgetting/size policy and hybrid searches ([`memory_message_service.py`](../../repos/ragflow/api/db/joint_services/memory_message_service.py:170)).
- **Strong persistent personal memory — FACT:** Open WebUI has SQL memory rows plus per-user vector collections and CRUD/reindex/reset routes ([`memories.py`](../../repos/open-webui/backend/open_webui/routers/memories.py:156)).
- **Strong context/token control — FACT:** LibreChat has token-counted message pruning, remaining-context budgets and compaction support ([`BaseClient.js`](../../repos/librechat/api/app/clients/BaseClient.js:700), [`compaction.ts`](../../repos/librechat/packages/api/src/agents/compaction.ts:79)).
- **Dify FACT/UNKNOWN:** Conversation/token-buffer memory is real, but generic persistent personal/project memory and approval-based condensation were not established ([`token_buffer_memory.py`](../../repos/dify/api/core/memory/token_buffer_memory.py:31)).
- **LobeChat UNKNOWN:** Personal/tool memory exists, but global taxonomy, exact context assembly, token monitoring and condensation approval were not established.
- **Langflow FACT/INFERENCE:** User/flow/session message memory is explicit; semantics depend on selected graph components, and agentic in-memory buffers are restart-volatile ([`memory.py`](../../repos/langflow/src/backend/base/langflow/memory.py:24)).

## 8. RAG ownership and replaceability

1. **LibreChat — strongest replacement seam INFERENCE:** Node code calls an external HTTP RAG service. Preserve endpoint/auth/entity contracts; vector engine is service-owned ([`text.ts`](../../repos/librechat/packages/api/src/files/text.ts:83)).
2. **Open WebUI — strong internal seam FACT/INFERENCE:** vector factory plus async façade and embedding function centralize backend selection ([`factory.py`](../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:90)).
3. **Dify — strong backend seam but domain-coupled INFERENCE:** vector factory/provider packages are explicit; dataset/RBAC/app contracts remain ([`vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29)).
4. **Langflow — component seam FACT:** RAG is graph-composable, but saved-flow serialization and component output contracts matter ([`vector_store_rag.py`](../../repos/langflow/src/backend/base/langflow/initial_setup/starter_projects/vector_store_rag.py:1)).
5. **LobeChat — moderate/UNKNOWN:** document/context mapping is explicit, but full index implementation was not established.
6. **RAGFlow — most integrated INFERENCE:** parser, model, metadata, citations, graph search, memory and tenant index naming are core domain.

## 9. Agents/workflows/tools/MCP

- **Dify FACT:** Durable workflow/node/agent/tool schemas, plugins and native MCP are product-grade but deeply integrated ([`workflow.py`](../../repos/dify/api/models/workflow.py:209), [`tools.py`](../../repos/dify/api/models/tools.py:78)).
- **LibreChat FACT:** Durable agents, queued/resumable turns, subagents, tools, MCP OAuth/registry/cache and checkpoint state are extensive ([`client.js`](../../repos/librechat/api/server/controllers/agents/client.js:4359), [`packages/api/src/mcp`](../../repos/librechat/packages/api/src/mcp:1)).
- **Open WebUI FACT:** Functions, built-ins, automations and MCP are strong extension surfaces, but runtime is more tool/function-oriented than a standalone durable workflow engine ([`functions.py`](../../repos/open-webui/backend/open_webui/routers/functions.py:46), [`mcp/client.py`](../../repos/open-webui/backend/open_webui/utils/mcp/client.py:59)).
- **RAGFlow FACT:** Canvas/agent components, retrieval tools and tenant MCP host are RAG-centric ([`agent_with_tools.py`](../../repos/ragflow/agent/component/agent_with_tools.py:88), [`mcp/server.py`](../../repos/ragflow/mcp/server/server.py:831)).
- **LobeChat FACT:** Tool manifests, MCP OAuth, server-side token protection and web/desktop availability are unusually deliberate ([`buildClientConnectorManifests.ts`](../../repos/lobechat/src/helpers/toolEngineering/buildClientConnectorManifests.ts:1), [`toolAvailability.ts`](../../repos/lobechat/src/helpers/toolAvailability.ts:1)).
- **Langflow FACT:** Graph components are the primary workflow/agent/tool extension mechanism; MCP depends on serving-plane security/event topology ([`main.py`](../../repos/langflow/src/backend/base/langflow/main.py:1)).

## 10. Auth/multi-tenancy ranking

1. **LibreChat — strongest generalized enforcement INFERENCE:** AsyncLocalStorage tenant context, Mongoose query injection, write/mutation guards and coverage tests ([`tenantIsolation.ts`](../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96)).
2. **RAGFlow — strongest domain-native tenant model FACT:** Tenant/UserTenant membership and tenant IDs cross providers, KBs, documents, indexes and MCP ([`db_models.py`](../../repos/ragflow/api/db/db_models.py:1141)).
3. **Dify — strong workspace/RBAC FACT/INFERENCE:** Tenant membership and distributed RBAC/app/dataset/provider predicates ([`checks.py`](../../repos/dify/api/controllers/common/rbac/checks.py:33)).
4. **LobeChat — strong workspace route boundary FACT:** Membership checked in shared helper, but complete universal coverage UNKNOWN ([`workspace.ts`](../../repos/lobechat/src/app/(backend)/webapi/_utils/workspace.ts:8)).
5. **Open WebUI — strong user/resource access FACT, weaker tenant inference:** Direct user ownership/groups/access grants; no universal tenant partition established ([`chats.py`](../../repos/open-webui/backend/open_webui/models/chats.py:129)).
6. **Langflow — user/resource scope FACT, organization UNKNOWN:** Flow/file/memory user predicates exist; full workspace/tenant coverage blocked by checkout.

## 11. Storage/jobs/usage

- **Dify FACT:** Celery/Redis workers, object storage adapters, quota reservation/commit/release and credit pools exist ([`celery_entrypoint.py`](../../repos/dify/api/celery_entrypoint.py:1), [`quota_service.py`](../../repos/dify/api/services/quota_service.py:92)).
- **LibreChat FACT:** Redis/in-memory stream/job/checkpoint modes, balances/transactions and agent token-spending paths exist ([`createStreamServices.ts`](../../repos/librechat/packages/api/src/stream/createStreamServices.ts:71)).
- **Open WebUI FACT/UNKNOWN:** Redis/session/socket, automations and provider usage analytics exist; commercial credits and storage quotas were not established ([`analytics.py`](../../repos/open-webui/backend/open_webui/routers/analytics.py:235)).
- **RAGFlow FACT/UNKNOWN:** Async document tasks, storage abstraction and per-user file limits exist; full billing ledger is UNKNOWN ([`task_executor.py`](../../repos/ragflow/rag/svr/task_executor.py:1322)).
- **LobeChat FACT/UNKNOWN:** Redis/S3/QStash/observability dependencies exist; complete billing/quota/token ledger is UNKNOWN ([`package.json`](../../repos/lobechat/package.json:189)).
- **Langflow FACT/UNKNOWN:** Worker/build jobs and preflight checks exist; in-memory queues/cache are explicitly unsafe for multi-replica event delivery, and credits are UNKNOWN ([`preflight.py`](../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1)).

## 12. Interface/coupling cross-cutting analysis

- **Most replaceable provider seam:** Langflow bundles/components, followed by Dify provider/plugin manager, LobeChat model runtime, Open WebUI model/config, LibreChat endpoint packages, and RAGFlow `LLMBundle`/tenant model services. This ranking is an INFERENCE about blast radius, not a benchmark.
- **Most replaceable RAG boundary:** LibreChat’s HTTP service, then Open WebUI/Dify vector factories, then Langflow graph components. RAGFlow is the least replaceable wholesale because RAG is its domain center.
- **Most explicit memory:** RAGFlow taxonomy/extraction; LibreChat persistent memory plus compaction; Open WebUI user memory. Dify/LobeChat/Langflow require more custom product semantics for project memory.
- **Most replaceable frontend/API:** Open WebUI, LibreChat and LobeChat have clean independent frontend/backend surfaces; Dify also separates web/API but has larger product contracts. Langflow SDK/API is strong but authoring/runtime contracts are specialized. RAGFlow APIs are product-domain specific.
- **Highest fork risk:** Dify domain model, LibreChat agent/tenant/checkpoint model, RAGFlow RAG/tenant/index model, LobeChat auth/workspace/context monorepo and Langflow graph/runtime compatibility migration.

## 13. Modification tests — comparative table

Scores are 0 configuration, 1 small extension, 2 moderate development, 3 substantial modification, 4 invasive modification, 5 architectural rewrite.

| Test | Dify | LibreChat | Open WebUI | RAGFlow | LobeChat | Langflow |
|---|---:|---:|---:|---:|---:|---:|
| Custom OpenAI-compatible provider | 1 | 1 | 0 | 2 | 1 | 1 |
| Add another provider | 2 | 2 | 2 | 2 | 2 | 1 |
| Persistent user memory | 3 | 1 | 1 | 1 | 2 | 2 |
| Project-specific memory | 2 | 2 | 3 | 2 | 2 | 3 |
| Context monitoring | 2 | 2 | 2 | 3 | 2 | 2 |
| User-approved condensation | 3 | 2 | 2-3 | 3 | 3 | 3 |
| Usage credits | 1 | 1-2 | 3 | 4 | 3 | 4 |
| Storage quotas | 2 | 2 | 3 | 2 | 3 | 3 |
| Custom external integration | 1-2 | 1-2 | 1-2 | 2 | 1 | 1 |
| MCP server/tool | 1 | 1 | 1 | 1 | 1 | 1 |
| Independent backend/API in front | 2 | 2 | 1 | 2 | 2 | 2 |
| Frontend replacement | 2 | 1-2 | 1 | 2 | 2 | 2 |
| Connect another candidate | 3 | 1-2 | 1-2 | 3 | 2 | 3 |

**Evidence basis:** Detailed per-repository reasoning and citations are in sections 13 of [`deep-dify.md`](deep-dify.md:1), [`deep-librechat.md`](deep-librechat.md:1), [`deep-open-webui.md`](deep-open-webui.md:1), [`deep-ragflow.md`](deep-ragflow.md:1), [`deep-lobechat.md`](deep-lobechat.md:1), and [`deep-langflow.md`](deep-langflow.md:1).

## 14. Upgrade/forkability cross-cutting findings

- **FACT:** All six have explicit extension seams, but all six are product/runtime systems rather than tiny libraries.
- **INFERENCE:** Upstream-compatible additions are most feasible for providers, tools, MCP connectors, vector adapters and external HTTP integrations. Replacing auth, tenant/workspace identity, database schema, agent state, event semantics or context policy causes migration/fork divergence.
- **FACT:** Langflow’s `langflow`→`lfx` compatibility aliases are direct migration evidence ([`__init__.py`](../../repos/langflow/src/backend/base/langflow/__init__.py:1)).
- **INFERENCE:** Dify/RAGFlow/LobeChat are especially sensitive to schema/domain forks; LibreChat is sensitive to tenant plugin/agent checkpoint contracts; Open WebUI is sensitive to adding a new tenant layer; Langflow is sensitive to graph serialization/runtime compatibility.

## 15. Risks/unknowns and blockers

1. **Langflow full-checkout blocker — FACT:** Cannot independently audit complete login/migrations/tenant/job paths from the Windows materialization.
2. **LibreChat external RAG blocker — FACT:** Need separate RAG service audit to establish parser/chunk/embed/index implementation and isolation.
3. **LobeChat context/RAG blocker — FACT/UNKNOWN:** Need deeper audit of database document ingestion, context-engine assembly, token counting and persistence.
4. **Cross-cutting billing/quota blocker — UNKNOWN:** Complete commercial credits, storage quotas and provider-cost reconciliation are not established uniformly across candidates.
5. **Security risk — INFERENCE:** Distributed authorization predicates, public-share exceptions, tenant-derived index names and asynchronous job identity propagation are the highest likely audit-risk surfaces.

## 16. Recommended candidates for Wave C independent audit

This is a shortlist for independent verification, not a final product decision.

### Proceed — Tier 1

1. **LibreChat:** Audit tenant isolation plugin/manual-scope exceptions, agent checkpoint/resume correctness, API-key/public agent auth, balance-to-provider usage reconciliation, and the external RAG service contract. Rationale: strongest explicit tenant enforcement, real memory/compaction, durable agents/MCP, and replaceable RAG boundary.
2. **Dify:** Audit distributed RBAC/tenant predicates, provider/plugin invocation and quota reservation paths, workflow/agent persistence, service API isolation, and long-term memory absence/extension cost. Rationale: broadest product/backend surface and strongest native credits/workflow/MCP/RAG combination.
3. **RAGFlow:** Audit tenant/index isolation, document task identity propagation, provider credential access, memory vector deletion/forgetting, MCP host mode and billing/usage gaps. Rationale: strongest RAG and memory specialization with first-class tenant domain, but highest RAG coupling.

### Proceed — Tier 2 / targeted audit

4. **Open WebUI:** Audit addition of universal workspace tenancy, group/access enforcement uniformity, file/vector ownership, MCP tool authorization, and credit/quota insertion points. Rationale: strongest replaceable frontend/vector/user-memory foundation, but lacks established tenant/credits/quota primitives.
5. **LobeChat:** Audit full context-engine/document ingestion, workspace coverage, public file preview, agent stream ownership, provider runtime persistence and billing/quota hooks. Rationale: strong modern workspace/provider/MCP seams, but several required paths remain unresolved.

### Conditional / unblock first

6. **Langflow:** Proceed only after obtaining a full Linux checkout or extracting blocked Git paths. Then audit login/API auth, organization isolation, graph checkpoint side effects, queue/cache topology, token/credit accounting and component sandboxing. Rationale: excellent provider/component/workflow extensibility, but current evidence quality is insufficient for an independent architecture decision.

## 17. Evidence index

Repository reports: [`deep-dify.md`](deep-dify.md:1), [`deep-librechat.md`](deep-librechat.md:1), [`deep-open-webui.md`](deep-open-webui.md:1), [`deep-ragflow.md`](deep-ragflow.md:1), [`deep-lobechat.md`](deep-lobechat.md:1), [`deep-langflow.md`](deep-langflow.md:1). Orientation reports: [`triage-dify.md`](triage-dify.md:1), [`triage-librechat.md`](triage-librechat.md:1), [`triage-open-webui.md`](triage-open-webui.md:1), [`triage-ragflow.md`](triage-ragflow.md:1), [`triage-lobechat.md`](triage-lobechat.md:1), [`triage-langflow.md`](triage-langflow.md:1).
