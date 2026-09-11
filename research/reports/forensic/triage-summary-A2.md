# A2 forensic triage summary

## 1. Verdict snapshot

**Wave B ranking based on modularity, replaceability, multi-tenancy, extensibility, and suitability as a platform subsystem—not feature count:**

1. **RAGFlow** — strongest demonstrated tenant model and RAG subsystem; more invasive domain coupling.
2. **LobeChat** — strongest broad product extension surface and workspace-aware backend; large monorepo/context coupling.
3. **Langflow** — strongest graph/provider bundle modularity; deployment state and compatibility migration create operational/fork risk.
4. **Flowise** — approachable Node graph/component platform with practical MCP/provider seams; tenancy/storage semantics are distributed.
5. **Khoj** — strong user-owned memory/personal knowledge product; project tenancy and generic orchestration are less established.
6. **Letta** — **not rankable on implementation evidence**: captured commit contains only 12 non-implementation files.

- **FACT:** RAGFlow has explicit `Tenant`, `UserTenant`, tenant-filtered services, tenant-derived indexes, and tenant-aware provider access. Evidence: [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1), [`tenant_model_service.py`](../../../repos/ragflow/api/db/joint_services/tenant_model_service.py:1).
- **FACT:** LobeChat has workspace membership checks, authenticated model/chat/document routes, and DB-backed model runtime initialization. Evidence: [`workspace.ts`](../../../repos/lobechat/src/app/(backend)/webapi/_utils/workspace.ts:8), [`route.ts`](../../../repos/lobechat/src/app/(backend)/webapi/chat/[provider]/route.ts:18).
- **FACT:** Langflow and Flowise expose strong component/graph/provider extension seams. Evidence: [`pyproject.toml`](../../../repos/langflow/pyproject.toml:130), [`package.json`](../../../repos/flowise/package.json:6).
- **FACT:** Khoj implements persistent user memory but inspected evidence does not establish first-class workspace/project tenancy. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1).
- **FACT:** Letta source is unavailable in the captured commit.

## 2. Architecture & stack

- **RAGFlow:** Python/Quart/Peewee tenant-aware RAG product with search/object storage/MCP.
- **LobeChat:** TypeScript monorepo using Next.js, Drizzle/PostgreSQL, Redis, S3, Better Auth, model runtime, tools, MCP, and desktop/web packages.
- **Langflow:** Python workspace with FastAPI backend, `lfx`, SDK, provider bundles, graph execution, MCP, and Redis options.
- **Flowise:** Node/TypeScript pnpm monorepo with Express server, graph components, LangChain/provider integrations, TypeORM migrations, Redis workers, and MCP.
- **Khoj:** Django/FastAPI hybrid with pgvector, user memory, agents, automation, tools, MCP, and S3.
- **Letta:** UNKNOWN because source/manifest/server implementation is absent.

## 3. Repo structure

- **FACT:** RAGFlow separates API DB services, admin services, MCP server, and runtime startup.
- **FACT:** LobeChat separates backend routes, database/services, model runtime, tool engineering, business packages, and multiple clients.
- **FACT:** Langflow separates backend, `lfx`, SDK, and bundles; the Windows checkout is partial because of filename length.
- **FACT:** Flowise separates server/controllers/services/database from component/node packages.
- **FACT:** Khoj separates Django models/adapters, FastAPI routers, processors/tools, and telemetry.
- **FACT:** Letta’s captured Git tree has no application directories.

## 4. Data model

- **RAGFlow FACT:** Tenant and user-tenant membership are first-class, with conversations, dialogs, knowledge bases, documents, files, memories, API tokens, provider configuration, and MCP entities.
- **LobeChat FACT:** Workspace/member, user, document/file, conversation/agent, and provider/runtime data are represented in Drizzle-backed application schemas; agent documents map into context-engine documents.
- **Langflow FACT/UNKNOWN:** Flows, users/permissions, files, build/memory state, and checkpoints are evident in backend services; complete migrations were not fully available.
- **Flowise FACT:** Chatflows, chat messages/history, credentials, tools, workspace/organization relationships, and storage usage are represented through entities/migrations/services.
- **Khoj FACT:** User, conversation, file, entry, agent, user-memory, subscription, and model configuration entities exist.
- **Letta UNKNOWN.**

## 5. AuthN/AuthZ/multi-tenancy

- **FACT:** Product accounts are not automatically tenant isolation.
- **RAGFlow:** Best direct evidence of real tenant isolation: tenant predicates in conversation/file/dialog services, tenant-derived indexes, and joined-tenant provider checks.
- **LobeChat:** Real workspace membership checks protect many routes; personal-only memory and deliberately public file-preview semantics show route-specific boundaries.
- **Flowise:** Workspace/organization/permission checks protect management operations, while public prediction uses chatflow ID/origin/rate-limit controls; these are different trust planes.
- **Langflow:** User/resource authorization and per-user file sandboxes are strong, but a complete organization/tenant model was not established.
- **Khoj:** User/resource ownership and privacy are tested, but first-class workspace/organization isolation is not established.
- **Letta UNKNOWN.**

## 6. Provider abstraction

- **Langflow and Flowise FACT:** Provider/component seams are explicit and comparatively modular.
- **LobeChat FACT:** DB-backed model runtime initialization is an explicit abstraction and provider packages are numerous.
- **RAGFlow FACT:** `LLMBundle` and tenant model services provide a real provider boundary coupled to tenant configuration.
- **Khoj FACT:** Configurable OpenAI-compatible base URL and provider/model entities exist.
- **Letta UNKNOWN.**

## 7. Conversation execution path

- **Flowise CHAT FACT:** Prediction loads chatflow, sets/accepts `chatId`, checks origin, invokes `buildChatflow`, and streams via SSE/Redis in queue mode. Evidence: [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:14).
- **Langflow CHAT FACT:** Build API authorizes/builds/executes graphs, handles streaming/cancellation/jobs/files/checkpoints, and carries current user identity. Evidence: [`build.py`](../../../repos/langflow/src/backend/base/langflow/api/build.py:1).
- **RAGFlow CHAT FACT:** Completion verifies tenant dialog access, resolves tenant models, retrieves/cites context, and invokes the LLM bundle. Evidence: [`conversation_service.py`](../../../repos/ragflow/api/db/services/conversation_service.py:1), [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:1).
- **Khoj CHAT FACT:** Chat API provides POST/WebSocket/history/session/share/fork/feedback operations; helpers perform search/rate limiting. Evidence: [`api_chat.py`](../../../repos/khoj/src/khoj/routers/api_chat.py:1), [`helpers.py`](../../../repos/khoj/src/khoj/routers/helpers.py:1).
- **LobeChat CHAT FACT:** Auth → workspace membership → DB model runtime → `modelRuntime.chat`; tools/context are downstream runtime concerns. Evidence: [`route.ts`](../../../repos/lobechat/src/app/(backend)/webapi/chat/[provider]/route.ts:18).
- **Letta UNKNOWN.**

## 8. Memory

- **RAGFlow FACT:** Raw, semantic, episodic, and procedural memory types plus tenant-specific extraction, embeddings, vector indexes, size, and forgetting policies.
- **Khoj FACT:** Persistent `UserMemory` with list/update/delete APIs and user/agent-scoped adapters.
- **Flowise FACT:** Redis, MongoDB, and Zep session/window/perpetual memory nodes.
- **Langflow FACT:** User/flow-scoped memory and volatile agentic conversation buffers.
- **LobeChat FACT/INFERENCE:** Personal memory/tool seams exist; workspace/project memory is not uniformly demonstrated.
- **Letta UNKNOWN.**

## 9. RAG

- **RAGFlow:** Most mature and integrated RAG pipeline, including retrieval, citations, SQL/web/knowledge-graph retrieval, document limits, tenant indexes, and async ingestion.
- **LobeChat:** Workspace-aware document service and context-engine mapping; complete parser/index internals were not traced.
- **Langflow:** File/splitter/retriever/vector components are graph-composable.
- **Flowise:** Vector-store/retriever nodes and chat/file metadata filters are configurable but flow-dependent.
- **Khoj:** User content/search/pgvector and knowledge-base paths exist, but complete ingestion chain was not traced.
- **Letta UNKNOWN.**

## 10. Agents/workflows/tools/MCP

- **FACT:** Langflow and Flowise provide the most obvious graph/workflow/component extension models.
- **FACT:** RAGFlow provides RAG-centric agents/sandboxes and a tenant-aware MCP server.
- **FACT:** LobeChat has permission-aware tool manifests, server-side MCP connectors, OAuth discovery, and desktop/web capability controls.
- **FACT:** Khoj has agents, online-search tools, sandboxed/E2B code execution, automation, and MCP dependencies.
- **Letta UNKNOWN.**

## 11. Files/storage

- **RAGFlow:** Tenant-filtered file/document services, object storage/search dependencies, async ingestion.
- **LobeChat:** S3 preview/upload/document service with explicit public-share exception and workspace-aware document routes.
- **Langflow:** Per-user sandbox roots; production preflight requires external storage for replicas.
- **Flowise:** Local/cloud storage and chatflow/chat metadata validation; storage usage updates on deletion.
- **Khoj:** File objects, authenticated content APIs, and S3 media storage.
- **Letta UNKNOWN.**

## 12. API

- **RAGFlow:** Quart API, admin login/API, token auth, and MCP host API.
- **LobeChat:** Next.js route handlers plus Hono/tRPC ecosystem, authenticated streaming/provider/document/agent APIs.
- **Langflow:** FastAPI build/stream/cancel APIs, SDK, and MCP routers.
- **Flowise:** Express prediction/internal/webhook/file/chatflow/credential/MCP APIs.
- **Khoj:** FastAPI/Django routers for chat, agents, content, memory, models, automation, and integrations.
- **Letta UNKNOWN.**

## 13. Background jobs/usage

- **RAGFlow FACT:** Async document tasks, background chat channels, per-user document count limit; complete billing ledger UNKNOWN.
- **LobeChat FACT:** Redis/QStash dependencies and streamed/server operations; credits/storage quota ledger UNKNOWN.
- **Langflow FACT:** Build jobs, worker retry/time limits, memory-base capture; in-memory queues/caches are unsafe across replicas.
- **Flowise FACT:** Worker/queue mode, Redis coordination, storage usage updates; complete credits/token ledger UNKNOWN.
- **Khoj FACT:** APScheduler automation, SQLite/PostHog telemetry, subscriptions; complete quota/credit ledger UNKNOWN.
- **Letta UNKNOWN.**

## 14. Deployment/scaling

- **RAGFlow INFERENCE:** Requires shared relational DB, object storage, search/index services, task state, and consistent tenant index naming.
- **LobeChat INFERENCE:** External DB/Redis/S3/QStash support a scalable product topology, subject to route/runtime state.
- **Langflow FACT:** Production preflight explicitly warns about local storage, in-memory cache, and in-memory queues across pods.
- **Flowise INFERENCE:** Queue/Redis mode supports scale for serializable state; MCP toolkit process-local state remains a seam.
- **Khoj INFERENCE:** Shared DB/vector/object storage and scheduler coordination are required for horizontal scale.
- **Letta UNKNOWN.**

## 15. Observability/testing

- **RAGFlow:** Langfuse dependency and pytest configuration.
- **LobeChat:** OpenTelemetry/Langfuse/W3C trace propagation.
- **Langflow:** Sentry, pytest, production preflight.
- **Flowise:** OpenTelemetry dependency and tests.
- **Khoj:** SQLite/PostHog telemetry and privacy/agent tests.
- **INFERENCE:** All five source-available projects have observability seams, but tenant-boundary and multi-replica coverage needs independent validation.

## 16. Coupling/extensibility

- **Most modular provider/tool seams:** Langflow, Flowise, LobeChat.
- **Strongest tenant/RAG coupling:** RAGFlow, which is a strength for its target domain and a replacement cost for generic reuse.
- **Strongest personal-memory seam:** Khoj.
- **Highest uncertainty:** Letta due to missing source.
- **Cross-cutting FACT:** Auth, storage, context, and billing are more coupled than provider adapters in every source-available candidate.

## 17. 13 modification tests

| Repository | Provider | Another provider | User memory | Project memory | Context monitor | Approved condensation | Credits | Storage quota | External integration | MCP | Backend/API front | Frontend replacement | Connect candidate |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| RAGFlow | 2 | 2 | 1 | 2 | 3 | 3 | 4 | 2 | 2 | 1 | 2 | 2 | 3 |
| LobeChat | 1 | 2 | 2 | 2 | 2 | 3 | 3 | 3 | 1 | 1 | 2 | 2 | 2 |
| Langflow | 1 | 1 | 2 | 3 | 2 | 3 | 4 | 3 | 1 | 1 | 2 | 2 | 3 |
| Flowise | 1 | 2 | 3 | 3 | 3 | 3 | 4 | 3 | 2 | 1 | 2 | 2 | 3 |
| Khoj | 1 | 2 | 1 | 4 | 3 | 3 | 3 | 2 | 2 | 2 | 2 | 2 | 3 |
| Letta | U | U | U | U | U | U | U | U | U | U | U | U | U |

`U` = UNKNOWN because source implementation was unavailable; it is not a claim that the real modification is necessarily invasive.

## 18. Evidence log

- **RAGFlow FACT:** [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1), [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:1), [`tenant_model_service.py`](../../../repos/ragflow/api/db/joint_services/tenant_model_service.py:1), [`memory_message_service.py`](../../../repos/ragflow/api/db/joint_services/memory_message_service.py:1).
- **LobeChat FACT:** [`index.ts`](../../../repos/lobechat/src/app/(backend)/middleware/auth/index.ts:61), [`workspace.ts`](../../../repos/lobechat/src/app/(backend)/webapi/_utils/workspace.ts:8), [`route.ts`](../../../repos/lobechat/src/app/(backend)/webapi/chat/[provider]/route.ts:18), [`index.ts`](../../../repos/lobechat/src/helpers/toolEngineering/index.ts:1).
- **Langflow FACT:** [`build.py`](../../../repos/langflow/src/backend/base/langflow/api/build.py:1), [`flow.py`](../../../repos/langflow/src/backend/base/langflow/helpers/flow.py:1), [`preflight.py`](../../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1), [`openai_chat_model.py`](../../../repos/langflow/src/bundles/openai/src/lfx_openai/components/openai/openai_chat_model.py:1).
- **Flowise FACT:** [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:14), [`chatflows/index.ts`](../../../repos/flowise/packages/server/src/controllers/chatflows/index.ts:50), [`VectorStoreUtils.ts`](../../../repos/flowise/packages/components/nodes/vectorstores/VectorStoreUtils.ts:1).
- **Khoj FACT:** [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1), [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1), [`api_chat.py`](../../../repos/khoj/src/khoj/routers/api_chat.py:1), [`api_memories.py`](../../../repos/khoj/src/khoj/routers/api_memories.py:1).
- **Letta FACT:** local Git tree contains only 12 tracked non-implementation files; all substantive architecture remains UNKNOWN.

## 19. Unknowns

- **FACT:** Letta cannot be evaluated from the captured source state.
- **UNKNOWN:** Complete schema inventories, billing/credits, storage quotas, token accounting, condensation, and deployment scale limits remain incomplete for multiple candidates.
- **UNKNOWN:** Full tenant predicate coverage requires route-by-route and backend-by-backend audit, even where strong tenant evidence exists.
- **INFERENCE:** The shortlist ranking should be treated as triage priority, not final adoption guidance.
- **FACT:** Tenant isolation is an implementation property, not a consequence of having user accounts. RAGFlow demonstrates the clearest end-to-end tenant propagation into relational authorization, provider resolution, file services, memory, and search-index naming. LobeChat and Flowise have meaningful workspace boundaries but route-specific/public trust planes. Khoj has strong user-owned memory/content isolation without established project tenancy. Langflow has resource/user authorization and production hardening, but its stateful serving model needs externalization for replicas. Letta remains unevaluable until a source-complete checkout is obtained.
