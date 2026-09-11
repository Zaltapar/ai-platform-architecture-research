# RAGFlow — Wave B Deep Forensic Investigation

## 1. Scope/evidence quality

**Scope.** Review limited to [`research/repos/ragflow`](../../../repos/ragflow:1), Quart/API/admin services, Peewee models, RAG/document/memory/agent/MCP code, tests/deployment assets, and [`triage-ragflow.md`](triage-ragflow.md:1).

**FACT:** RAGFlow is a vertically integrated Python/Quart RAG product with Peewee database services, tenant/model configuration, document ingestion, search/index backends, memory services, agent/canvas runtime, admin APIs and MCP ([`pyproject.toml`](../../../repos/ragflow/pyproject.toml:1), [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1)).

**Evidence quality:** High for tenant schema/auth, provider/model resolution, dialog/RAG and file ingestion, memory extraction/indexing, MCP, token accounting and storage/index ownership. Medium for full login UI flow, every background queue implementation and complete billing ledger.

## 2. Architecture/data ownership

**FACT:** Core relational state is in [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1), including `User`, `Tenant`, `UserTenant`, `TenantLLM`, `Knowledgebase`, `Document`, `File`, `Dialog`, `Conversation`, `APIToken`, `Memory`, `MCPServer`, and agent/canvas records.

**FACT:** Tenant model configuration is stored in `Tenant`, `TenantLLM`, and newer tenant model/provider/instance tables. Documents/files belong to knowledge bases and tenant-aware storage/index paths. Search backends are selected through doc-store services under [`common/doc_store`](../../../repos/ragflow/common/doc_store:1) and [`rag/utils`](../../../repos/ragflow/rag/utils:1).

**INFERENCE:** RAGFlow’s tenant/domain identity is a cross-cutting ownership key across SQL predicates, object storage, index names, provider credentials, memory collections and API authorization. This enables product isolation but raises replacement cost.

## 3. LOGIN trace

**FACT:** API authentication is resolved by `_load_user()` and `current_user` in [`apps/__init__.py`](../../../repos/ragflow/api/apps/__init__.py:220), supporting session cookies, beta tokens, JWT and API tokens. [`login_required`](../../../repos/ragflow/api/apps/__init__.py:235) calls `_load_user`, rejects unauthenticated requests and then invokes the async handler.

**FACT:** Admin authentication is separate through admin routes/auth modules at [`admin/server/routes.py`](../../../repos/ragflow/admin/server/routes.py:1) and [`admin/server/auth.py`](../../../repos/ragflow/admin/server/auth.py:1).

**FACT:** Tenant/workspace state is represented by `Tenant` and `UserTenant` membership in [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1141). Service methods resolve tenant access and tenant-specific model configuration before resource operations.

**Boundary/state:** Auth is request-scoped through Quart/Flask-Login-compatible state. Tenant identity becomes the primary service argument and database/index predicate. Admin and user APIs are separate trust planes.

## 4. CHAT trace

**FACT:** User-facing chat APIs are under [`chat_api.py`](../../../repos/ragflow/api/apps/restful_apis/chat_api.py:1), bot/OpenAI-style assistant APIs under [`bot_api.py`](../../../repos/ragflow/api/apps/restful_apis/bot_api.py:73), and dialog orchestration in [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:293).

**FACT:** `async_chat_solo` resolves the dialog’s tenant-specific chat model, constructs `LLMBundle`, extracts file attachments, builds system/user messages, and invokes `async_chat_streamly_delta` or `async_chat` at [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:293).

**FACT:** For RAG-enabled dialog, `get_models` resolves knowledge bases, embedding, rerank, chat and optional TTS bundles at [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:355). Retrieval/knowledge graph paths use tenant IDs, KB IDs, embedding and chat bundles at [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:785).

**FACT:** Agent/canvas execution uses `Canvas`, tenant-aware `LLMBundle`, retrieval tools and MCP tool bindings; `agent_with_tools.py` creates tenant model bundles and MCP sessions at [`agent_with_tools.py`](../../../repos/ragflow/agent/component/agent_with_tools.py:88).

**INFERENCE:** Chat chain is auth → tenant/dialog authorization → model/embedding/rerank resolution → message/file context → dense/keyword/rerank/graph/web retrieval → prompt/citation assembly → `LLMBundle` provider invocation → tool/agent/MCP execution → stream and conversation state. Exact persistence timing differs by endpoint and must be audited per API.

**Async/state:** Quart handlers are async; provider calls may stream asynchronously. Retrieval and parsing frequently cross `thread_pool_exec`/background task boundaries. Conversation/dialog state is database-owned; canvas/task state is persisted or queued separately.

## 5. FILE trace

**FACT:** Upload services accept tenant and parent-folder context in [`file_api_service.py`](../../../repos/ragflow/api/apps/services/file_api_service.py:34) and REST upload route [`document_api.py`](../../../repos/ragflow/api/apps/restful_apis/document_api.py:444).

**FACT:** `FileService.upload_document` creates tenant/user root and KB folder state, sanitizes parent paths, checks KB/document collisions and supported file type, stores bytes through `settings.STORAGE_IMPL`, stores thumbnails, and creates document metadata at [`file_service.py`](../../../repos/ragflow/api/db/services/file_service.py:578).

**FACT:** Document processing is asynchronous/task-oriented. Parser selection/configuration is stored in document rows; `task_executor.py` resolves tenant chat/embedding models and inserts chunks/embeddings at [`task_executor.py`](../../../repos/ragflow/rag/svr/task_executor.py:1322) and [`task_executor.py`](../../../repos/ragflow/rag/svr/task_executor.py:1488).

**FACT:** Document metadata/index ownership is tenant-specific; metadata service derives index names and doc-store implementations provide insert/search operations in [`doc_metadata_service.py`](../../../repos/ragflow/api/db/services/doc_metadata_service.py:1) and [`doc_store_base.py`](../../../repos/ragflow/common/doc_store/doc_store_base.py:223).

**INFERENCE:** File flow is upload → tenant/KB storage + SQL document row → parser/OCR/layout extraction → chunking/advanced RAG compilation → tenant embedding `LLMBundle` → ES/OpenSearch/Infinity/other doc-store index → retrieval/citation. This is an integrated pipeline, not a thin adapter.

## 6. API trace

**FACT:** Quart REST APIs, admin APIs, provider APIs, document/chunk/search APIs, bot/OpenAI-compatible APIs and MCP APIs coexist. Provider API model invocation is explicit in [`provider_api_service.py`](../../../repos/ragflow/api/apps/services/provider_api_service.py:1449), where tenant model configuration creates `LLMBundle`.

**FACT:** Decorator/auth context and tenant injection protect routes; document APIs use `add_tenant_id_to_kwargs` at [`document_api.py`](../../../repos/ragflow/api/apps/restful_apis/document_api.py:123). MCP REST routes use `login_required` and compare server tenant to `current_user.id` in [`mcp_api.py`](../../../repos/ragflow/api/apps/restful_apis/mcp_api.py:42).

**INFERENCE:** API request → session/JWT/beta/API-token authentication → tenant/dialog/KB authorization → service orchestration → provider/retrieval/tool execution → stream/JSON and DB/index state. Admin API is a separate administrative plane.

## 7. Memory/context

**FACT:** Memory rows and types are explicitly modeled in [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1810), and constants define raw, semantic, episodic and procedural types at [`constants.py`](../../../repos/ragflow/common/constants.py:239).

**FACT:** `memory_message_service` uses tenant-specific chat `LLMBundle` to extract memory from user/agent turns at [`memory_message_service.py`](../../../repos/ragflow/api/db/joint_services/memory_message_service.py:170), tenant-specific embeddings to create vector index/messages at [`memory_message_service.py`](../../../repos/ragflow/api/db/joint_services/memory_message_service.py:210), applies memory-size/forgetting policy, and queries hybrid text/dense memory at [`memory_message_service.py`](../../../repos/ragflow/api/db/joint_services/memory_message_service.py:261).

**FACT:** RAGFlow tracks token usage in `LLMBundle` and a per-run sink in [`common/token_utils.py`](../../../repos/ragflow/common/token_utils.py:64); canvas resets/aggregates usage in [`canvas.py`](../../../repos/ragflow/agent/canvas.py:344).

**INFERENCE:** RAGFlow has the clearest implemented memory taxonomy and extraction/index lifecycle among the six. User-approved condensation is not established; existing forgetting/size policy is automatic rather than approval-driven.

## 8. RAG

**FACT:** Dialog retrieval combines knowledge-base chunks, embeddings, reranking, keyword extraction, citations, SQL/web/knowledge-graph paths in [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:785) and [`chunk_api.py`](../../../repos/ragflow/api/apps/restful_apis/chunk_api.py:399).

**FACT:** File parsing supports OCR/vision/model-backed paths through tenant `LLMBundle` calls in [`flow/parser/parser.py`](../../../repos/ragflow/rag/flow/parser/parser.py:411) and advanced document compilation. Chunk insertion and embedding are handled by asynchronous task executor code at [`task_executor.py`](../../../repos/ragflow/rag/svr/task_executor.py:1322).

**INFERENCE:** RAG is the core product subsystem. Storage/search adapters are named, but retrieval, metadata, citations, graph search, parser configuration, model resolution and tenant index naming are interdependent. Replacing RAG wholesale is invasive.

## 9. Agents/workflows/tools/MCP

**FACT:** Agents/canvases are persisted and executed through `agent` components/canvas services; tool retrieval uses [`agent/tools/retrieval.py`](../../../repos/ragflow/agent/tools/retrieval.py:26). MCP server records are tenant-owned in [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1589).

**FACT:** MCP host supports self-host single-tenant and host multi-tenant modes in [`mcp/server/server.py`](../../../repos/ragflow/mcp/server/server.py:831). MCP tools call RAGFlow chat/dataset/retrieval APIs and expose API/bearer access.

**INFERENCE:** Agent/tool runtime is RAG-centric and product-integrated. MCP is a credible extension boundary, but sessions, agent canvas lifecycle, tenant provider state and retrieval tools remain coupled.

## 10. Auth/multi-tenancy

**FACT:** `Tenant`, `UserTenant`, tenant-owned provider configuration, tenant-scoped document/file services, tenant-specific model/index naming and MCP tenant checks are implemented ([`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1141), [`file_service.py`](../../../repos/ragflow/api/db/services/file_service.py:578), [`mcp_api.py`](../../../repos/ragflow/api/apps/restful_apis/mcp_api.py:112)).

**FACT:** Tenant provider access is centralized in [`tenant_model_service.py`](../../../repos/ragflow/api/db/joint_services/tenant_model_service.py:21). Dialog authorization and model resolution carry `tenant_id` explicitly.

**INFERENCE:** This is genuine tenant/workspace isolation stronger than user-only ownership. Residual risk is distributed predicate propagation through every adapter, task and integration.

## 11. Storage/jobs/usage

**FACT:** Storage is abstracted through `settings.STORAGE_IMPL`; MinIO is declared in dependencies/config. File limits include `MAX_FILE_NUM_PER_USER` enforcement in document services.

**FACT:** Document ingestion and chunk insertion use asynchronous task execution in [`task_executor.py`](../../../repos/ragflow/rag/svr/task_executor.py:1322). Server startup initializes background chat channels and services in [`ragflow_server.py`](../../../repos/ragflow/api/ragflow_server.py:1).

**FACT:** Token usage is available through provider bundle usage and context-variable sinks; Langfuse/OpenTelemetry-related dependencies and services exist in [`pyproject.toml`](../../../repos/ragflow/pyproject.toml:1).

**UNKNOWN:** Complete credit/billing ledger, reservation/commit semantics and production worker scaling guarantees. Existing per-user file/document limits are not proof of aggregate storage quota policy.

## 12. Interface/coupling analysis

- **Provider:** `LLMBundle` and tenant model services are explicit, but provider credentials/model types/tenant schema are coupled. Replaceability: moderate for additions, low for contract replacement.
- **Memory:** dedicated model/service/index paths are real, but tenant/provider/index policy is integrated. Replaceability: moderate.
- **RAG:** service/doc-store boundaries exist, yet core domain is RAG. Replaceability: low for wholesale replacement.
- **Agent/MCP:** explicit components and sessions, but tied to canvas, tenant models, retrieval and lifecycle. Replaceability: moderate/low.
- **Storage:** `STORAGE_IMPL` is a useful adapter boundary; document/index metadata and tenant paths remain coupled.
- **Frontend/API:** separated enough for a new client, but APIs expose product-specific tenant/KB/dialog contracts.

## 13. Modification tests

| Test | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 2 | Add provider/model config and `LLMBundle` factory integration. |
| Another provider | 2 | Provider registry/model service is explicit but tenant config must be supported. |
| Persistent user memory | 1 | Memory entity/extraction/vector indexing/policy already exist. |
| Project-specific memory | 2 | Tenant/KB scope exists; project mapping and access semantics are new. |
| Context monitoring | 3 | Instrument retrieval, memory, bundle calls, streaming and background tasks. |
| User-approved condensation | 3 | Existing forgetting is automatic; approval state/API/UI must be added. |
| Usage credits | 4 | Billing ledger absent from evidence; cross-cuts tenant/provider/jobs. |
| Storage quotas | 2 | Existing file/document limits and storage services provide a base. |
| Custom external integration | 2 | API/MCP/tools/admin/service seams exist. |
| MCP server/tool | 1 | Native MCP server/client/session and tenant checks exist. |
| Independent backend/API in front | 2 | API/token/tenant boundaries are explicit but domain mapping is substantial. |
| Frontend replacement | 2 | API/admin separation helps; KB/dialog/canvas contracts remain. |
| Connect another candidate | 3 | API/MCP/provider seams work; direct data-model reuse is tightly coupled. |

## 14. Upgrade/forkability

**INFERENCE:** RAGFlow is forkable as a specialized RAG product and reusable through APIs/MCP/provider/storage seams. A fork replacing tenant model, index/search engine, parser pipeline, memory semantics or canvas runtime will carry high migration and rebase cost.

**FACT:** Migration tooling and model evolution are active; tenant model tables are being evolved through migrations. This increases upgrade diligence for custom provider/index changes.

## 15. Risks/unknowns

- **UNKNOWN:** Complete billing/credit ledger and token-to-cost reconciliation.
- **UNKNOWN:** Full login UI and every route’s authorization predicate.
- **UNKNOWN:** Production worker/queue scale guarantees and all adapter test coverage.
- **UNKNOWN:** User-facing approval semantics for condensation/forgetting.
- **RISK/INFERENCE:** Tenant IDs in object/index names and task payloads create high cross-system isolation risk if omitted.

## 16. Evidence index

Key anchors: [`apps/__init__.py`](../../../repos/ragflow/api/apps/__init__.py:235), [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1141), [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:293), [`file_service.py`](../../../repos/ragflow/api/db/services/file_service.py:578), [`task_executor.py`](../../../repos/ragflow/rag/svr/task_executor.py:1322), [`memory_message_service.py`](../../../repos/ragflow/api/db/joint_services/memory_message_service.py:170), [`llm_service.py`](../../../repos/ragflow/api/db/services/llm_service.py:79), [`tenant_model_service.py`](../../../repos/ragflow/api/db/joint_services/tenant_model_service.py:21), [`agent_with_tools.py`](../../../repos/ragflow/agent/component/agent_with_tools.py:88), [`mcp/server.py`](../../../repos/ragflow/mcp/server/server.py:831), [`mcp_api.py`](../../../repos/ragflow/api/apps/restful_apis/mcp_api.py:42), and [`token_utils.py`](../../../repos/ragflow/common/token_utils.py:64).
