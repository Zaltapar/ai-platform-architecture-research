# LibreChat — Wave B Deep Forensic Investigation

## 1. Scope/evidence quality

**Scope.** Review limited to [`research/repos/librechat`](../../../repos/librechat:1), Node/Express API, React client, TypeScript packages, schemas, tests, compose/Helm assets, and [`triage-librechat.md`](triage-librechat.md:1).

**FACT:** LibreChat is an Express/Node product with React frontend, Mongo/Mongoose persistence, package-level API/data-schema layers, Redis-backed stream/job options, Meilisearch, object storage, and an external `rag_api` service ([`api/db/index.js`](../../../repos/librechat/api/db/index.js:1), [`docker-compose.yml`](../../../repos/librechat/docker-compose.yml:56)).

**Evidence quality:** High for auth/tenant middleware, chat context/token flow, files/RAG boundary, agents/MCP, usage models, and tests. Medium for the external RAG service’s own parser/vector implementation, which is outside the main Node execution path.

## 2. Architecture/data ownership

**FACT:** Mongoose model factories live under [`packages/data-schemas/src/models`](../../../repos/librechat/packages/data-schemas/src/models:1), with users/sessions, conversations/messages/projects, files, agents, memory, permissions, balances, transactions, MCP, and queued-turn state. Mongo bootstrap calls `createModels(mongoose)` in [`api/db/index.js`](../../../repos/librechat/api/db/index.js:1).

**FACT:** Express routes/controllers own auth, ACL, orchestration, provider clients, file services, agent runtime, MCP, and persistence. React owns presentation/client state. External `rag_api` owns much of ingestion/index/search. Redis/shared stores own stream/job/checkpoint state when configured.

**INFERENCE:** This is a modular product monolith with explicit external service boundaries, not a small chat SDK.

## 3. LOGIN trace

**FACT:** Login enters the Express route chain at [`api/server/routes/auth.js`](../../../repos/librechat/api/server/routes/auth.js:42). It applies header logging, login rate limiting, ban checks, email validation, local or LDAP authentication, balance configuration, and finally `loginController`.

**FACT:** JWT-protected logout, refresh, registration, password reset, 2FA, and social/OIDC/admin routes are registered in the same auth surface ([`auth.js`](../../../repos/librechat/api/server/routes/auth.js:40)).

**FACT:** Tenant context is established with AsyncLocalStorage in [`tenantContext.ts`](../../../repos/librechat/packages/data-schemas/src/config/tenantContext.ts:19). Mongoose query middleware injects tenant filters and blocks cross-tenant mutations in [`tenantIsolation.ts`](../../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96).

**FACT:** Coverage tests require tenant-bearing models to apply the plugin or appear on a reviewed manual-scoping allowlist in [`tenantIsolation.coverage.spec.ts`](../../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.coverage.spec.ts:39).

**Boundary/state:** Login is synchronous route/controller/database work; request tenant/user context then propagates through model hooks and ACL/resource middleware. The isolation plugin is a strong state-ownership boundary, but authorization beyond tenant filtering remains resource-specific.

## 4. CHAT trace

**FACT:** Authenticated conversation/message routes live under [`api/server/routes`](../../../repos/librechat/api/server/routes:1). Standard provider generation enters [`BaseClient.sendMessage()`](../../../repos/librechat/api/app/clients/BaseClient.js:732).

**FACT:** `sendMessage` calls start/persistence preparation, constructs model-bound stored messages, resolves authorized historical files, calls `buildMessages`, records token counts, persists the user message, and then continues into provider completion ([`BaseClient.js`](../../../repos/librechat/api/app/clients/BaseClient.js:739), [`BaseClient.js`](../../../repos/librechat/api/app/clients/BaseClient.js:817), [`BaseClient.js`](../../../repos/librechat/api/app/clients/BaseClient.js:831), [`BaseClient.js`](../../../repos/librechat/api/app/clients/BaseClient.js:850)).

**FACT:** Context pruning is token-budgeted: `remainingContextTokens`, per-message `tokenCount`, reversed context, and `messagesToRefine` are returned by the pruning method at [`BaseClient.js`](../../../repos/librechat/api/app/clients/BaseClient.js:700).

**FACT:** RAG context is requested through `createContextHandlers` and the external service in [`createContextHandlers.js`](../../../repos/librechat/api/app/clients/prompts/createContextHandlers.js:12). Memory has a dedicated model/handlers/authorization package at [`memory.ts`](../../../repos/librechat/packages/data-schemas/src/models/memory.ts:1) and [`handlers.ts`](../../../repos/librechat/packages/api/src/memory/handlers.ts:68).

**FACT:** Agent execution adds queued/resumable runs, tools, checkpoints, MCP, subagents, and usage handling in [`AgentClient.chatCompletion()`](../../../repos/librechat/api/server/controllers/agents/client.js:4359). Agent OpenAI-compatible requests enter [`agents/openai.js`](../../../repos/librechat/api/server/routes/agents/openai.js:119).

**INFERENCE:** Standard path is route/auth/ACL → `BaseClient.sendMessage` → context/file/memory/RAG preparation → provider client streaming/tool callbacks → message/conversation persistence → SSE/client state. Agent path adds asynchronous job/checkpoint/event state and can resume after process boundaries.

**Async/state:** Stream services and generation jobs are Redis-backed or in-memory depending deployment ([`createStreamServices.ts`](../../../repos/librechat/packages/api/src/stream/createStreamServices.ts:71)); agent persistence and checkpoint cleanup are explicitly asynchronous.

## 5. FILE trace

**FACT:** File routes and services are under [`api/server/routes/files`](../../../repos/librechat/api/server/routes/files:1), [`api/server/services/Files`](../../../repos/librechat/api/server/services/Files:1), and [`packages/api/src/files`](../../../repos/librechat/packages/api/src/files:1). First-class file models link files to users, conversations, messages, agents and skills ([`file.ts`](../../../repos/librechat/packages/data-schemas/src/models/file.ts:1)).

**FACT:** Text extraction calls `RAG_API_URL/health` and `/text`, sending a short-lived JWT and multipart file; native parsing is a fallback in [`text.ts`](../../../repos/librechat/packages/api/src/files/text.ts:83). Vector upload/deletion calls `/embed` and `/documents` in [`VectorDB/crud.js`](../../../repos/librechat/api/server/services/Files/VectorDB/crud.js:20) and [`VectorDB/crud.js`](../../../repos/librechat/api/server/services/Files/VectorDB/crud.js:67).

**FACT:** Local/S3/CloudFront storage implementations exist under [`packages/api/src/storage`](../../../repos/librechat/packages/api/src/storage:1). Agent file upload locks and ownership/quota checks use Redis and authorization in [`agents/files.ts`](../../../repos/librechat/packages/api/src/agents/files.ts:134).

**INFERENCE:** Flow is upload → file model/storage → parser or RAG `/text` → external chunk/embed/index service → retrieval query/context handler. Exact chunking/vector internals are RAG-service-owned and therefore UNKNOWN from this repository.

## 6. API trace

**FACT:** REST route families cover auth, conversations, messages, files, projects, agents/assistants, MCP, memory, schedules, balances, API keys, sharing and admin ([`api/server/routes`](../../../repos/librechat/api/server/routes:1)).

**FACT:** API-key middleware is package-level at [`apiKeys/middleware.ts`](../../../repos/librechat/packages/api/src/apiKeys/middleware.ts:41); agent-compatible `POST /v1/chat/completions` uses `checkAgentPermission` ([`agents/openai.js`](../../../repos/librechat/api/server/routes/agents/openai.js:119)).

**INFERENCE:** API request → JWT/session/API-key auth → tenant/resource ACL → client or agent orchestration → provider/tool/MCP execution → Mongo/Redis persistence → JSON/SSE response. Public machine API and browser routes are distinct, but both depend on domain-specific permissions and identity propagation.

## 7. Memory/context

**FACT:** Persistent memory has a dedicated model, handlers, authorization/partition middleware, and agent callback integration ([`memory.ts`](../../../repos/librechat/packages/data-schemas/src/models/memory.ts:1), [`handlers.ts`](../../../repos/librechat/packages/api/src/memory/handlers.ts:68), [`authorization.ts`](../../../repos/librechat/packages/api/src/memory/authorization.ts:91)).

**FACT:** Context management is explicit in `BaseClient`: token counts, remaining budget, pruning, stored-message projection, historical-file authorization, and compaction support. Agent compaction code exists in [`compaction.ts`](../../../repos/librechat/packages/api/src/agents/compaction.ts:79).

**INFERENCE:** User memory is materially implemented and more replaceable than Dify’s generic memory layer, but automatic extraction/prioritization policies are distributed across memory/agent/client code.

**UNKNOWN:** Complete project-memory semantics and user-approved condensation UX/state were not proven as one unified workflow; approval integration would still cross agent persistence and frontend state.

## 8. RAG

**FACT:** LibreChat treats RAG as an HTTP service boundary configured by `RAG_API_URL`; text extraction, embedding and deletion use explicit endpoints ([`text.ts`](../../../repos/librechat/packages/api/src/files/text.ts:83), [`rag.js`](../../../repos/librechat/packages/api/src/files/rag.js:25)). Compose defines `rag_api` separately ([`docker-compose.yml`](../../../repos/librechat/docker-compose.yml:85).

**INFERENCE:** RAG replacement is comparatively easy at the Node boundary: preserve `/health`, `/text`, `/embed`, `/query`, `/documents`, JWT/entity scoping and response shapes. Vector engine ownership and parser quality cannot be concluded from the LibreChat repository.

## 9. Agents/workflows/tools/MCP

**FACT:** Agent models support actions/tools, skills/files, subagents, queued turns, schedules, code environments and resumable execution ([`packages/api/src/agents`](../../../repos/librechat/packages/api/src/agents:1)).

**FACT:** MCP has connection, OAuth, tool publication/cache, registry, authorization and request modules under [`packages/api/src/mcp`](../../../repos/librechat/packages/api/src/mcp:1); routes are exposed by [`api/server/routes/mcp.js`](../../../repos/librechat/api/server/routes/mcp.js:1).

**FACT:** Agent completion captures tool/event lifecycle and usage in [`client.js`](../../../repos/librechat/api/server/controllers/agents/client.js:4359) and [`client.js`](../../../repos/librechat/api/server/controllers/agents/client.js:5151).

**INFERENCE:** Agent/MCP runtime is feature-rich but deeply coupled to files, permissions, queues, checkpoints, balance spending, and message persistence.

## 10. Auth/multi-tenancy

**FACT:** Tenant isolation is not merely a user field: AsyncLocalStorage tenant context, query middleware, write stamping, fail-closed strict mode, aggregate filtering, and mutation guards are implemented in [`tenantIsolation.ts`](../../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96).

**FACT:** Tests exercise tenant filtering, strict missing-context behavior, writes, aggregates, and cross-tenant mutation protection in [`tenantIsolation.spec.ts`](../../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.spec.ts:77).

**INFERENCE:** LibreChat has the strongest explicit general-purpose tenant query boundary among these six, while user/resource ACLs remain a second independent authorization layer. Manual-scoped allowlisted models remain audit targets.

## 11. Storage/jobs/usage

**FACT:** Redis/in-memory stream services, queued turns, background completion wakeups and resumable jobs are implemented ([`createStreamServices.ts`](../../../repos/librechat/packages/api/src/stream/createStreamServices.ts:71), [`backgroundCompletionWakeup.ts`](../../../repos/librechat/packages/api/src/agents/backgroundCompletionWakeup.ts:252)).

**FACT:** Balance and transaction models exist under [`models`](../../../repos/librechat/packages/data-schemas/src/models:1), and agent client usage/token-spending paths exist at [`client.js`](../../../repos/librechat/api/server/controllers/agents/client.js:5151).

**FACT:** OpenTelemetry/Langfuse assets exist under [`otel`](../../../repos/librechat/otel:1) and deployment compose files.

**UNKNOWN:** Exact provider-cost reconciliation across every path and production HA guarantees for in-memory mode. Storage quota policy exists in pieces but aggregate tenant quota enforcement needs product work.

## 12. Interface/coupling analysis

- **Provider:** endpoint/client packages are explicit, including custom/OpenAI endpoints, but streaming/tool/usage contracts make provider changes moderate.
- **RAG:** strong HTTP replacement seam; the external vector/index implementation is outside the Node repository.
- **Memory:** dedicated model/handlers/authorization and dependency injection improve replacement; agent callbacks still couple semantics.
- **Auth/tenant:** query plugin is explicit and robust; replacing the entire authorization model is invasive because context and ACLs are pervasive.
- **Agent/MCP:** high coupling to queues, checkpoints, files, permissions, persistence and usage.
- **Frontend:** separate React client and REST/SSE surface make replacement feasible but stream/event contracts are extensive.

## 13. Modification tests

| Test | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 1 | Custom endpoint/OpenAI client boundary exists. |
| Another provider | 2 | Add endpoint/config/client plus streaming/tool/usage behavior. |
| Persistent user memory | 1 | Model, handlers, auth and agent callback already exist. |
| Project-specific memory | 2 | Projects and memory exist separately; link/context partitioning is new. |
| Context monitoring | 2 | Token counters exist; remaining capacity needs API/UI exposure. |
| User-approved condensation | 2 | Compaction exists; approval state/UX is additional. |
| Usage credits | 1-2 | Balance/transaction and usage hooks exist; entitlement policy may expand. |
| Storage quotas | 2 | File ownership/locks/retention provide base; aggregate quotas need policy. |
| Custom external integration | 1-2 | Endpoints, actions/tools, OAuth, schedules and DI handlers exist. |
| MCP server/tool | 1 | Native MCP client/OAuth/registry/tool routes exist. |
| Independent backend/API in front | 2 | API keys and OpenAI agent endpoint exist; identity propagation required. |
| Frontend replacement | 1-2 | REST/SSE separation is strong; agent event contracts remain. |
| Connect another candidate | 1-2 | RAG HTTP seam and memory package are practical integration points. |

## 14. Upgrade/forkability

**INFERENCE:** LibreChat is relatively forkable for provider, RAG-service and frontend integrations if public contracts are preserved. A fork that changes tenant context, agent checkpoint semantics, message schema or balance spending will be expensive to rebase.

**FACT:** The repository has substantial tests around tenant isolation, agents, MCP, files and routes, which reduces regression risk but also signals a large contract surface.

## 15. Risks/unknowns

- **UNKNOWN:** External RAG parser/chunker/vector implementation and its full isolation model.
- **UNKNOWN:** Complete balance-to-provider-cost reconciliation.
- **UNKNOWN:** HA semantics when stream/checkpoint stores are in-memory.
- **RISK/INFERENCE:** Agent resumability and durable event ordering create operational complexity during upgrades.

## 16. Evidence index

Key evidence: [`auth.js`](../../../repos/librechat/api/server/routes/auth.js:42), [`tenantContext.ts`](../../../repos/librechat/packages/data-schemas/src/config/tenantContext.ts:19), [`tenantIsolation.ts`](../../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96), [`BaseClient.js`](../../../repos/librechat/api/app/clients/BaseClient.js:732), [`text.ts`](../../../repos/librechat/packages/api/src/files/text.ts:83), [`VectorDB/crud.js`](../../../repos/librechat/api/server/services/Files/VectorDB/crud.js:67), [`memory.ts`](../../../repos/librechat/packages/data-schemas/src/models/memory.ts:1), [`handlers.ts`](../../../repos/librechat/packages/api/src/memory/handlers.ts:68), [`client.js`](../../../repos/librechat/api/server/controllers/agents/client.js:4359), [`openai.js`](../../../repos/librechat/api/server/routes/agents/openai.js:119), and [`createStreamServices.ts`](../../../repos/librechat/packages/api/src/stream/createStreamServices.ts:71).
