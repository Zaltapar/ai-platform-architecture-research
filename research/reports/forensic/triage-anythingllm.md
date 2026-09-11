# AnythingLLM — Source-Level Forensic Triage

## 1. Verdict snapshot

- **FACT:** AnythingLLM is an Express/Node.js application with a Vite frontend, Prisma persistence, a collector process, workspace-centric RAG, agent skills/flows, and an MCP client bridge ([`server/index.js`](../../repos/anythingllm/server/index.js:8), [`server/prisma/schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:1)).
- **FACT:** It has explicit multi-user workspaces, persistent global/workspace memory, many provider adapters, vector backends, agents, scheduled jobs, and MCP client support.
- **INFERENCE:** Best fit for a workspace-oriented product or a comparatively approachable fork where a Node.js team wants direct control over chat/RAG/agent code.
- **INFERENCE:** Worst fit as a highly isolated tenant backend without additional authorization hardening; isolation is enforced through route middleware and query/model conventions rather than a universal tenant boundary.

## 2. Architecture & stack (evidence: file paths)

- **FACT:** Backend is Express with JSON/text/urlencoded body limits and WebSocket support ([`server/index.js`](../../repos/anythingllm/server/index.js:8), [`server/index.js`](../../repos/anythingllm/server/index.js:50)).
- **FACT:** Frontend is a separate Vite application under [`frontend`](../../repos/anythingllm/frontend:1).
- **FACT:** Persistence uses Prisma with SQLite by default and commented PostgreSQL configuration ([`server/prisma/schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:5)).
- **FACT:** Document ingestion is separated into [`collector`](../../repos/anythingllm/collector:1), while the main server contains vector/provider/agent utilities.
- **INFERENCE:** Runtime is a modular Node monolith plus a document collector, not a large independently scalable service mesh.

## 3. Repo structure map

- [`server`](../../repos/anythingllm/server:1): Express API, Prisma schema/migrations, models, endpoint handlers, provider adapters, agents, MCP, storage.
- [`frontend`](../../repos/anythingllm/frontend:1): Vite/React UI.
- [`collector`](../../repos/anythingllm/collector:1): parsing and document-processing service/process.
- [`browser-extension`](../../repos/anythingllm/browser-extension:1), [`embed`](../../repos/anythingllm/embed:1), [`open-computer`](../../repos/anythingllm/open-computer:1): client/integration surfaces.
- [`docker`](../../repos/anythingllm/docker:1), [`cloud-deployments`](../../repos/anythingllm/cloud-deployments:1): deployment packaging.

## 4. Data model (table list with schema file locations)

- **FACT:** Prisma schema is [`server/prisma/schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:1); migrations are under [`server/prisma/migrations`](../../repos/anythingllm/server/prisma/migrations:1).
- Identity/access: `users`, `recovery_codes`, `password_reset_tokens`, `temporary_auth_tokens`, `browser_extension_api_keys`, `api_keys` ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:18)).
- Workspace/conversation: `workspaces`, `workspace_users`, `workspace_threads`, `workspace_chats`, `workspace_agent_invocations` ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:121), [`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:154), [`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:182)).
- Files/RAG: `workspace_documents`, `workspace_parsed_files`, `document_vectors`, `document_sync_queues`, `document_sync_executions` ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:27), [`schema.prisma`](../../repos/anythingllm/server/prisma.schema:113)).
- Memory: `memories` has optional `userId`, optional `workspaceId`, `scope`, content, timestamps, and indexes ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:431)).
- Agents/automation/integrations: `scheduled_jobs`, `scheduled_job_runs`, `external_communication_connectors`, model routers/rules, embed configs/chats ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:394)).

## 5. AuthN / AuthZ / multi-tenancy (with isolation evidence)

- **FACT:** Single-user mode uses an `AUTH_TOKEN` embedded/validated through a JWT payload; multi-user mode decodes a JWT, loads the user, rejects invalid/suspended users, and stores the user in `response.locals` ([`validatedRequest`](../../repos/anythingllm/server/utils/middleware/validatedRequest.js:8)).
- **FACT:** Password recovery, temporary tokens, browser-extension keys, API keys, and optional simple SSO are represented in schema/middleware ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:92), [`simpleSSOEnabled`](../../repos/anythingllm/server/utils/middleware/simpleSSOEnabled.js:11)).
- **FACT:** Workspace membership is a join table and chat/document/thread rows carry workspace identifiers; route middleware validates workspace/thread access ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:214), [`chat.js`](../../repos/anythingllm/server/endpoints/chat.js:23)).
- **INFERENCE:** It is real user/workspace isolation for ordinary routes, but not a strict tenant abstraction: system settings, connectors, some API/session paths, and vector namespace conventions are application-wide. There is no universal DB row-level tenant policy.
- **Modification classification:** strengthening tenant isolation is **3-4**, because route middleware, model queries, vector namespaces, files, embeds, and admin endpoints all participate.

## 6. Provider abstraction

- **FACT:** Provider selection is a large explicit switch in [`server/utils/helpers/index.js`](../../repos/anythingllm/server/utils/helpers/index.js:139), with one adapter directory per provider under [`server/utils/AiProviders`](../../repos/anythingllm/server/utils/AiProviders:1).
- **FACT:** A `generic-openai` provider exists and reads its model/base configuration from environment/provider settings ([`server/utils/helpers/index.js`](../../repos/anythingllm/server/utils/helpers/index.js:192)).
- **FACT:** Vector provider selection is similarly switch-based and includes Pinecone, Chroma, LanceDB, Weaviate, Qdrant, Milvus, Zilliz, Astra, and PGVector ([`server/utils/helpers/index.js`](../../repos/anythingllm/server/utils/helpers/index.js:86)).
- **INFERENCE:** Custom OpenAI-compatible endpoints are configuration-level when the generic adapter matches the API; adding a new provider requires a new adapter plus switch/config/UI changes, not merely a drop-in plugin registration.

## 7. Conversation execution path (file:line chain)

- **FACT:** Chat enters `POST /workspace/:slug/stream-chat`, passes `validatedRequest`, role, and workspace validation, obtains the session user/workspace, enforces a daily message limit, then calls `streamChatWithWorkspace` ([`server/endpoints/chat.js`](../../repos/anythingllm/server/endpoints/chat.js:23)).
- **FACT:** `streamChatWithWorkspace` handles slash commands, attempts agent routing, resolves the LLM connector/model router, checks vector namespace state, and streams SSE output ([`server/utils/chats/stream.js`](../../repos/anythingllm/server/utils/chats/stream.js:20)).
- **FACT:** Workspace chat persistence is represented by `workspace_chats`, with thread and API-session partition fields ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:182)).
- **FACT:** Memory injection fetches global user memories and workspace memories, optionally reranks workspace memories, updates `lastUsedAt`, and appends Markdown to the system prompt ([`server/utils/memories/index.js`](../../repos/anythingllm/server/utils/memories/index.js:26)).
- **INFERENCE:** The path is direct and readable, but context assembly, agent dispatch, provider resolution, vector retrieval, and persistence are coupled in utility/handler code rather than a formal orchestration interface.

## 8. Memory

- **FACT:** AnythingLLM has persistent memories with user/global and workspace scopes in `memories` ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:431)).
- **FACT:** Injection selects global memories plus up to five workspace memories, reranking larger sets using a native embedding reranker ([`server/utils/memories/index.js`](../../repos/anythingllm/server/utils/memories/index.js:31)).
- **FACT:** Conversation history is stored in workspace chats and threads; memory extraction/agent memory tools exist under agent plugins ([`server/utils/agents/aibitat/plugins/memory.js`](../../repos/anythingllm/server/utils/agents/aibitat/plugins/memory.js:1)).
- **INFERENCE:** User and project memory are materially more explicit than in several candidates, but memory is prompt-injection-oriented rather than an independently replaceable memory service.

## 9. RAG

- **FACT:** Workspace documents and parsed files are persisted with workspace/user/thread references ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:27), [`schema.prisma`](../../repos/anythingllm/server/prisma.schema:377)).
- **FACT:** Document synchronization has queue/execution models ([`schema.prisma`](../../repos/anythingllm/server/prisma.schema:294)).
- **FACT:** The collector parses/processes files, while vector implementations are selected by `getVectorDbClass` and expose a common operational shape ([`server/utils/helpers/index.js`](../../repos/anythingllm/server/utils/helpers/index.js:86)).
- **FACT:** Chat checks a workspace vector namespace before query-mode retrieval ([`server/utils/chats/stream.js`](../../repos/anythingllm/server/utils/chats/stream.js:94)).
- **INFERENCE:** Vector backend replacement is relatively approachable; replacing chunking/parser/embedding semantics still touches collector, workspace document state, and chat retrieval.

## 10. Agents / workflows / tools / MCP

- **FACT:** Agent runtime is under [`server/utils/agents`](../../repos/anythingllm/server/utils/agents:1), with Aibitat providers/plugins and agent skills such as memory, web browsing, filesystem, request-user-input, and scheduled jobs ([`server/utils/agents/aibitat/plugins`](../../repos/anythingllm/server/utils/agents/aibitat/plugins:1)).
- **FACT:** Agent flows have executor modules for API calls, LLM instructions, and web scraping ([`server/utils/agentFlows/executors`](../../repos/anythingllm/server/utils/agentFlows/executors:1)).
- **FACT:** MCP is native client-side integration: the hypervisor loads server definitions, supports stdio/SSE/streamable HTTP transports, and converts tools to agent-callable plugins ([`server/utils/MCP/hypervisor/index.js`](../../repos/anythingllm/server/utils/MCP/hypervisor/index.js:4), [`server/utils/MCP/hypervisor/index.js`](../../repos/anythingllm/server/utils/MCP/hypervisor/index.js:21)).
- **FACT:** MCP administration is restricted to admin routes ([`server/endpoints/mcpServers.js`](../../repos/anythingllm/server/endpoints/mcpServers.js:12)).
- **INFERENCE:** Agents/tools are extensible through plugin files and skills, but the central agent runtime remains an AnythingLLM-specific abstraction.

## 11. Files & storage

- **FACT:** Default Docker deployment mounts [`server/storage`](../../repos/anythingllm/docker/docker-compose.yml:18), collector hotdir, and collector outputs ([`docker-compose.yml`](../../repos/anythingllm/docker/docker-compose.yml:18)).
- **FACT:** Backend body-parser limits are configured to `3GB` ([`server/index.js`](../../repos/anythingllm/server/index.js:52)).
- **FACT:** File metadata is workspace-scoped in Prisma, with parsed files additionally carrying user/thread fields ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:27), [`schema.prisma`](../../repos/anythingllm/server/prisma.schema:377)).
- **INFERENCE:** Local filesystem is the default operational storage; object-storage replacement and strict per-tenant storage quotas are not first-class in the inspected core path.

## 12. API surface

- **FACT:** Express mounts system, workspace, chat, document, embed, developer, agent, MCP, mobile, scheduled-job, memory, and browser-extension endpoints under `/api` ([`server/index.js`](../../repos/anythingllm/server/index.js:81)).
- **FACT:** Chat exposes SSE streaming workspace and thread routes ([`server/endpoints/chat.js`](../../repos/anythingllm/server/endpoints/chat.js:23)).
- **FACT:** Developer/API routes and API-key schema exist, along with embed endpoints and temporary auth tokens ([`server/index.js`](../../repos/anythingllm/server/index.js:97), [`schema.prisma`](../../repos/anythingllm/server/prisma.schema:18)).
- **FACT:** Rate/usage-like daily message limits are enforced through user fields and `User.canSendChat` ([`server/endpoints/chat.js`](../../repos/anythingllm/server/endpoints/chat.js:50)).

## 13. Background jobs & usage accounting

- **FACT:** The schema includes document sync queues/executions and scheduled jobs/runs ([`schema.prisma`](../../repos/anythingllm/server/prisma.schema:294), [`schema.prisma`](../../repos/anythingllm/server/prisma.schema:403)).
- **FACT:** A background worker utility exists ([`server/utils/BackgroundWorkers`](../../repos/anythingllm/server/utils/BackgroundWorkers:1)).
- **FACT:** Daily per-user chat limits are enforced; telemetry/model pricing code exists ([`server/endpoints/chat.js`](../../repos/anythingllm/server/endpoints/chat.js:50), [`server/utils/helpers/modelPricing`](../../repos/anythingllm/server/utils/helpers/modelPricing:1)).
- **UNKNOWN:** A complete commercial credit ledger, token-reservation system, or billing entitlement model was not established; classify as **NONE established** beyond daily limits/pricing telemetry.

## 14. Deployment & scaling

- **FACT:** Default compose is one `anything-llm` container with mounted local storage and a bridge network ([`docker/docker-compose.yml`](../../repos/anythingllm/docker/docker-compose.yml:1)).
- **FACT:** Collector directories are mounted into the same deployment ([`docker/docker-compose.yml`](../../repos/anythingllm/docker/docker-compose.yml:18)).
- **INFERENCE:** Horizontal scaling is not the default story; shared SQLite/local storage, singleton MCP hypervisor state, and in-process agent/provider state complicate multi-replica deployment.
- **FACT:** PostgreSQL is present as an alternative Prisma datasource in schema comments, but deployment/scaling implications were not fully validated ([`schema.prisma`](../../repos/anythingllm/server/prisma.schema:5)).

## 15. Observability & testing

- **FACT:** HTTP logging is optional and development-oriented ([`server/index.js`](../../repos/anythingllm/server/index.js:54)); telemetry/event logs are persisted ([`schema.prisma`](../../repos/anythingllm/server/prisma.schema:270)).
- **FACT:** Jest configuration and backend tests exist ([`server/__tests__`](../../repos/anythingllm/server/__tests__:1), [`jest.config.cjs`](../../repos/anythingllm/jest.config.cjs:1)).
- **FACT:** CI contains lint/test workflows ([`.github/workflows`](../../repos/anythingllm/.github/workflows:1)).
- **INFERENCE:** Core provider/vector/agent tests exist, but distributed tracing/metrics maturity is lower and full coverage was not measured.

## 16. Coupling & extensibility assessment

- Provider coupling: **3/5** — adapter directories are clear, but central switches and environment names must be modified ([`server/utils/helpers/index.js`](../../repos/anythingllm/server/utils/helpers/index.js:139)).
- RAG coupling: **2/5** — vector interface/provider selection is reasonably replaceable; parser/collector and workspace schema remain coupled.
- Memory coupling: **3/5** — dedicated model and injection module exist, but injection is directly called from chat/agent flows.
- Auth/tenancy coupling: **4/5** — `validatedRequest`, role middleware, workspace validation, user fields, and every resource route participate.
- Agent/MCP coupling: **3/5** — plugin and MCP conversion surfaces are explicit, but hypervisor and Aibitat contracts are central.
- **FACT:** Extension mechanisms include provider adapters, vector adapters, agent plugins/skills, flows, embed APIs, external connectors, and MCP configuration ([`server/utils/agents/aibitat/plugins`](../../repos/anythingllm/server/utils/agents/aibitat/plugins:1), [`server/utils/MCP/hypervisor/index.js`](../../repos/anythingllm/server/utils/MCP/hypervisor/index.js:21)).

## 17. Modification tests table (0-5)

| Test | Score | Evidence-based reason |
|---|---:|---|
| Custom OpenAI-compatible API | 0-1 | Existing `generic-openai` adapter and configurable provider model/base path ([`server/utils/helpers/index.js`](../../repos/anythingllm/server/utils/helpers/index.js:192)). |
| Add another provider | 2 | Add adapter directory, switch cases, env/config/UI metadata; no universal provider registry. |
| Persistent user memory | 1 | `memories` table and memory injection already exist ([`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:431)). |
| Project-specific memory | 1 | Workspace-scoped memories and retrieval already exist ([`server/utils/memories/index.js`](../../repos/anythingllm/server/utils/memories/index.js:31)). |
| Context monitoring | 2 | Workspace history/token estimates exist, but user-facing token telemetry needs chat/provider/UI changes. |
| User-approved condensation | 2-3 | Summarization/agent plugins exist, but approval state and durable context policy are not a first-class subsystem. |
| Usage credits | 3 | Daily limits/pricing telemetry exist, but no established credit ledger/reservation/billing model. |
| Storage quotas | 3 | Workspace/user file metadata exists, but quota accounting/enforcement is not established in core paths. |
| Custom external integration | 1-2 | External connectors, REST endpoints, webhooks/Telegram/Google/Outlook-style integrations exist. |
| Add MCP server/tool | 1 | Native MCP hypervisor supports stdio/SSE/HTTP and agent conversion ([`server/utils/MCP/hypervisor/index.js`](../../repos/anythingllm/server/utils/MCP/hypervisor/index.js:4)). |
| Independent backend/API in front | 2 | Developer endpoints/API keys/embed APIs exist; session/workspace middleware must be preserved. |
| Replace frontend | 2 | Frontend is a separate Vite app, but it consumes AnythingLLM-specific endpoint/SSE contracts. |
| Connect another candidate as RAG/memory | 2-3 | Vector abstraction is usable, memory module is direct and prompt-oriented; external engine adapter needed. |

## 18. Evidence log: key file:line citations

- [`Express bootstrap`](../../repos/anythingllm/server/index.js:8)
- [`Endpoint mounting`](../../repos/anythingllm/server/index.js:81)
- [`Prisma datasource`](../../repos/anythingllm/server/prisma/schema.prisma:13)
- [`Users`](../../repos/anythingllm/server/prisma/schema.prisma:61)
- [`Workspaces`](../../repos/anythingllm/server/prisma/schema.prisma:121)
- [`Workspace chats`](../../repos/anythingllm/server/prisma/schema.prisma:182)
- [`Memories`](../../repos/anythingllm/server/prisma/schema.prisma:431)
- [`JWT/multi-user middleware`](../../repos/anythingllm/server/utils/middleware/validatedRequest.js:8)
- [`Chat route`](../../repos/anythingllm/server/endpoints/chat.js:23)
- [`Chat stream orchestration`](../../repos/anythingllm/server/utils/chats/stream.js:20)
- [`Memory injection`](../../repos/anythingllm/server/utils/memories/index.js:26)
- [`Provider switch`](../../repos/anythingllm/server/utils/helpers/index.js:139)
- [`Vector switch`](../../repos/anythingllm/server/utils/helpers/index.js:86)
- [`MCP hypervisor`](../../repos/anythingllm/server/utils/MCP/hypervisor/index.js:21)
- [`Compose deployment`](../../repos/anythingllm/docker/docker-compose.yml:7)

## 19. Unknowns / could not establish

- **UNKNOWN:** Exact production cloud-only architecture and whether managed deployments add stronger tenant isolation than the repository default.
- **UNKNOWN:** Full token accounting/credit/billing implementation beyond daily message limits, pricing cache, telemetry, and model-router fields.
- **UNKNOWN:** Complete file upload-to-collector asynchronous execution path for every supported file type.
- **UNKNOWN:** Whether all route/model combinations consistently enforce both workspace and user ownership under every API/embed mode.
- **UNKNOWN:** Release cadence and migration quality beyond the shallow clone and existing Prisma migration directory.
