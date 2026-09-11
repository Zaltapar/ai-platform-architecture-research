# Flowise forensic triage

## 1. Verdict snapshot

- **FACT:** Flowise is a TypeScript/Node monorepo built around chatflow/agentflow graphs, LangChain integrations, Express controllers, TypeORM migrations, workspace-scoped persistence, and optional Redis worker mode. Evidence: [`package.json`](../../../repos/flowise/package.json:1), [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:14).
- **INFERENCE:** It is more naturally reusable as a Node orchestration/product backend than as a narrowly isolated library. The graph and component packages are reusable, but authorization, storage, credentials, and execution are distributed across server services and node implementations.
- **FACT:** Workspace and organization identifiers are used in controller/service authorization, but public prediction endpoints can be invoked through chatflow IDs and origin/rate-limit controls; this is not equivalent to every execution being authenticated as a tenant member.
- **INFERENCE:** Good candidate for direct extension when retaining its Node/LangChain model; replacing persistence, tenancy, or execution semantics has moderate-to-high blast radius.

## 2. Architecture & stack

- **FACT:** The root package declares pnpm workspaces under `packages/*`, Node/pnpm engine constraints, LangChain/OpenAI/provider packages, OpenTelemetry, Redis, and MCP SDK dependencies. Evidence: [`package.json`](../../../repos/flowise/package.json:6).
- **FACT:** The server exposes Express-style controllers; component packages contain provider, vector-store, memory, tool, and record-manager nodes.
- **INFERENCE:** Runtime composition is graph-driven: a stored chatflow is loaded, converted into executable node instances, and invoked by prediction/webhook/internal-prediction services.
- **UNKNOWN:** Exact production topology and commercial enterprise-only modules cannot be fully determined from the community checkout alone.

## 3. Repo structure

- **FACT:** `packages/server` contains controllers, services, entities, migrations, queues, and storage integration; `packages/components` contains graph nodes; the frontend and UI packages are separate workspace areas.
- **FACT:** The repository includes distinct prediction, webhook, file, credential, custom-MCP, and MCP-endpoint controller surfaces. Evidence: [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:14), [`custom-mcp-servers/index.ts`](../../../repos/flowise/packages/server/src/controllers/custom-mcp-servers/index.ts:1).
- **INFERENCE:** Server/controllers are a product shell around a comparatively modular component graph runtime.

## 4. Data model

- **FACT:** SQLite migrations create `chat_flow`, `chat_message`, `credential`, and `tool` tables. Evidence: [`1693835579790-Init.ts`](../../../repos/flowise/packages/server/src/database/migrations/sqlite/1693835579790-Init.ts:1).
- **FACT:** Chat-history migration adds `chatId`, `chatType`, `memoryType`, and `sessionId`, establishing persisted conversation partitioning. Evidence: [`1694657778173-AddChatHistory.ts`](../../../repos/flowise/packages/server/src/database/migrations/sqlite/1694657778173-AddChatHistory.ts:1).
- **FACT:** Credentials, chatflows, files, and workspace relationships are persisted through server entities/services and migrations.
- **INFERENCE:** The schema is application-centric rather than a clean tenant data-plane abstraction; tenant identity is propagated by services and selected columns/relations.
- **UNKNOWN:** A complete cross-database schema comparison was not performed for every supported database dialect.

## 5. AuthN/AuthZ/multi-tenancy

- **FACT:** Authenticated controller paths derive `workspaceId` from `req.user?.activeWorkspaceId`; chatflow deletion checks organization, workspace, and permissions such as `chatflows:delete`. Evidence: [`chatflows/index.ts`](../../../repos/flowise/packages/server/src/controllers/chatflows/index.ts:50).
- **FACT:** Credential CRUD is workspace-scoped, and custom MCP records are assigned to the active workspace with protected identifiers. Evidence: [`credentials/index.ts`](../../../repos/flowise/packages/server/src/controllers/credentials/index.ts:1), [`custom-mcp-servers/index.ts`](../../../repos/flowise/packages/server/src/controllers/custom-mcp-servers/index.ts:1).
- **FACT:** External prediction loads a chatflow by ID and applies allowed-origin checks; internal prediction explicitly supplies the active workspace. Evidence: [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:28), [`internal-predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/internal-predictions/index.ts:1).
- **INFERENCE:** Flowise has real workspace-aware authorization for management operations, but public chatbot execution is a deliberately different trust boundary. Product accounts therefore do not imply uniform tenant isolation across all entry points.
- **UNKNOWN:** Full row-level isolation guarantees for every node, vector backend, and storage provider require provider-by-provider review.

## 6. Provider abstraction

- **FACT:** Providers are represented as graph nodes and component packages, including OpenAI-compatible/model, embedding, vector-store, memory, and tool implementations.
- **INFERENCE:** Adding a provider is usually an adapter/node integration rather than a core runtime rewrite, provided it conforms to the LangChain/component contract.
- **FACT:** Credentials are separately persisted and workspace-scoped, while node configuration references credential data.
- **UNKNOWN:** There is no single universal provider interface covering all model, embedding, reranker, and tool categories; abstractions differ by node family.

## 7. Conversation execution path

- **CHAT FACT:** `POST` prediction validates the chatflow ID, loads the chatflow, checks allowed origins, generates/accepts a `chatId`, and calls `predictionsServices.buildChatflow`; streaming uses SSE and Redis subscription in queue mode. Evidence: [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:14).
- **CHAT FACT:** Chat history uses `chatId`, `sessionId`, `chatType`, and memory metadata; node memory can be Redis, MongoDB, or Zep-backed. Evidence: [`1694657778173-AddChatHistory.ts`](../../../repos/flowise/packages/server/src/database/migrations/sqlite/1694657778173-AddChatHistory.ts:1), [`RedisBackedChatMemory.ts`](../../../repos/flowise/packages/components/nodes/memory/RedisBackedChatMemory/RedisBackedChatMemory.ts:1).
- **INFERENCE:** Context construction is delegated to the selected graph nodes; there is no evidence of one global context manager controlling every flow.
- **UNKNOWN:** Exact response persistence ordering for every synchronous/streaming failure path was not exhaustively traced.

## 8. Memory

- **FACT:** Redis-backed memory stores user/AI messages in Redis lists keyed by session and supports configurable windows. Evidence: [`RedisBackedChatMemory.ts`](../../../repos/flowise/packages/components/nodes/memory/RedisBackedChatMemory/RedisBackedChatMemory.ts:1).
- **FACT:** MongoDB memory stores messages under a session-keyed document; Zep nodes support external perpetual or window memory. Evidence: [`MongoDBMemory.ts`](../../../repos/flowise/packages/components/nodes/memory/MongoDBMemory/MongoDBMemory.ts:1), [`ZepMemoryCloud.ts`](../../../repos/flowise/packages/components/nodes/memory/ZepMemoryCloud/ZepMemoryCloud.ts:1).
- **INFERENCE:** Short-term/session memory is strong and replaceable at node level. Persistent user, project, semantic, episodic, and procedural memory are not unified platform primitives; they depend on selected integrations/flow design.
- **UNKNOWN:** Built-in user-approved condensation and global memory prioritization were not established.

## 9. RAG

- **FACT:** Vector-store utilities associate uploaded files with `chatId` metadata and filter retrieval by chat ID. Evidence: [`VectorStoreUtils.ts`](../../../repos/flowise/packages/components/nodes/vectorstores/VectorStoreUtils.ts:1), [`Upstash.ts`](../../../repos/flowise/packages/components/nodes/vectorstores/Upstash/Upstash.ts:1).
- **FACT:** Record managers track writes and namespace defaults can use `chatflowid`. Evidence: [`SQLiteRecordManager.ts`](../../../repos/flowise/packages/components/nodes/recordmanager/SQLiteRecordManager/SQLiteRecordManager.ts:1).
- **INFERENCE:** RAG is graph-composable and backend-replaceable at vector/retriever node boundaries, but isolation metadata must be configured correctly by each flow.
- **UNKNOWN:** A single built-in ingestion pipeline spanning all file types and stores does not appear to be the dominant architecture.

## 10. Agents/workflows/tools/MCP

- **FACT:** Agentflow/multiagent/assistant types are permissioned as chatflow categories in controller logic. Evidence: [`chatflows/index.ts`](../../../repos/flowise/packages/server/src/controllers/chatflows/index.ts:69).
- **FACT:** Custom MCP servers have CRUD, authorization, and discovered-tool retrieval; the MCP endpoint accepts bearer authorization and JSON-RPC. Evidence: [`custom-mcp-servers/index.ts`](../../../repos/flowise/packages/server/src/controllers/custom-mcp-servers/index.ts:1), [`mcp-endpoint/index.ts`](../../../repos/flowise/packages/server/src/controllers/mcp-endpoint/index.ts:1).
- **FACT:** Queue-mode cache serializes LLM/embedding caches into Redis, while MCP toolkit instances remain process-local because they cannot be serialized. Evidence: [`CachePool.ts`](../../../repos/flowise/packages/server/src/CachePool.ts:1).
- **INFERENCE:** Workflows and tools are first-class graph nodes; MCP is integrated but process-local toolkit state creates a scaling seam.

## 11. Files/storage

- **FILE FACT:** Upload/file retrieval validates chatflow/chat identifiers, UUIDs, and resolves organization/workspace context before streaming storage files. Evidence: [`get-upload-file/index.ts`](../../../repos/flowise/packages/server/src/controllers/get-upload-file/index.ts:1).
- **FACT:** Storage supports local and cloud providers including S3/GCS/Azure according to repository contribution/configuration material, and deletion updates usage. Evidence: [`files/index.ts`](../../../repos/flowise/packages/server/src/controllers/files/index.ts:1).
- **INFERENCE:** File isolation is split between controller validation, storage paths, and vector metadata; replacing storage is moderate rather than purely configuration if custom semantics are required.

## 12. API

- **FACT:** API surfaces include external/internal prediction, webhook/SSE, file, chatflow, credential, MCP, and webhook-listener routes.
- **FACT:** Prediction supports synchronous JSON and streaming responses. Evidence: [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:57).
- **INFERENCE:** The API is a product API, not a narrowly versioned domain SDK; wrapping it is easier than replacing its internal execution contracts.

## 13. Background jobs/usage

- **FACT:** Flowise has `start-worker` and queue-mode behavior, with Redis used for streaming/cache coordination. Evidence: [`package.json`](../../../repos/flowise/package.json:13), [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:77).
- **FACT:** File deletion updates storage usage.
- **UNKNOWN:** A complete built-in billing/credits ledger and token-usage accounting model was not established from inspected source.
- **INFERENCE:** Credits, quotas, and metering would cross prediction services, model callbacks, persistence, and workspace policy.

## 14. Deployment/scaling

- **FACT:** Queue mode and Redis are explicit deployment options; local/process memory remains for some caches/toolkits.
- **INFERENCE:** Horizontal scaling is viable for queue-compatible paths but requires shared Redis/storage and careful treatment of non-serializable MCP state.
- **UNKNOWN:** Full Kubernetes/deployment topology was not audited.

## 15. Observability/testing

- **FACT:** OpenTelemetry is a dependency and the repository has tests/scripts.
- **UNKNOWN:** Coverage and trace completeness across every provider/node are not established.
- **INFERENCE:** Observability is extensible at server/runtime boundaries but provider-specific failures may remain heterogeneous.

## 16. Coupling/extensibility

- **FACT:** Node packages expose practical extension points for models, embeddings, vector stores, memory, tools, and record managers.
- **INFERENCE:** Provider and RAG additions are relatively modular; database/auth/tenant replacement is coupled to controllers, services, entities, migrations, and UI assumptions.
- **INFERENCE:** The graph JSON/configuration model lowers workflow extension cost but increases compatibility obligations for node names, credential fields, and serialized parameters.

## 17. 13 modification tests

| Modification | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 1 | Add/configure a provider node/base URL; SSRF/secrets still require review. |
| Another provider | 2 | Usually a component/node and credential integration; may require UI metadata. |
| Persistent user memory | 3 | Session memory exists, but identity-to-memory policy is not one unified primitive. |
| Project memory | 3 | Requires workspace/project key propagation through graph, storage, and retrieval metadata. |
| Context monitoring | 3 | Must instrument heterogeneous model/node execution and token callbacks. |
| User-approved condensation | 3 | Requires a shared context policy and UI/API state beyond node-local windows. |
| Credits | 4 | Cross-cuts prediction, streaming, provider usage, workspace policy, and persistence. |
| Storage quotas | 3 | File usage exists, but quota enforcement must cover all storage/vector paths. |
| Custom external integration | 2 | Webhooks/tools/components are practical extension seams. |
| MCP tool/server | 1 | Built-in custom MCP CRUD/endpoint support exists. |
| Independent backend/API in front | 2 | External prediction API is usable, but auth/tenant semantics must be mapped. |
| Frontend replacement | 2 | API/product separation helps, but serialized graph/admin contracts must be reproduced. |
| Connecting to another candidate | 3 | Best through API/webhook/provider/vector seams; direct internal reuse is coupled. |

## 18. Evidence log

- **FACT:** Root workspace/dependency architecture: [`package.json`](../../../repos/flowise/package.json:1).
- **FACT:** Prediction/chat execution and SSE: [`predictions/index.ts`](../../../repos/flowise/packages/server/src/controllers/predictions/index.ts:14).
- **FACT:** Workspace/permission checks: [`chatflows/index.ts`](../../../repos/flowise/packages/server/src/controllers/chatflows/index.ts:50).
- **FACT:** MCP management and bearer endpoint: [`custom-mcp-servers/index.ts`](../../../repos/flowise/packages/server/src/controllers/custom-mcp-servers/index.ts:1), [`mcp-endpoint/index.ts`](../../../repos/flowise/packages/server/src/controllers/mcp-endpoint/index.ts:1).
- **FACT:** Conversation schema: [`1694657778173-AddChatHistory.ts`](../../../repos/flowise/packages/server/src/database/migrations/sqlite/1694657778173-AddChatHistory.ts:1).
- **FACT:** Memory/RAG implementation anchors: [`RedisBackedChatMemory.ts`](../../../repos/flowise/packages/components/nodes/memory/RedisBackedChatMemory/RedisBackedChatMemory.ts:1), [`VectorStoreUtils.ts`](../../../repos/flowise/packages/components/nodes/vectorstores/VectorStoreUtils.ts:1).

## 19. Unknowns

- **UNKNOWN:** Complete enterprise-only authorization and billing behavior.
- **UNKNOWN:** Full provider-by-provider data isolation guarantees.
- **UNKNOWN:** Global condensation, semantic/episodic memory, and token-budget policy.
- **UNKNOWN:** Exhaustive deployment topology and benchmarked horizontal scaling limits.
