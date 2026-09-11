# RAGFlow forensic triage

## 1. Verdict snapshot

- **FACT:** RAGFlow is a Python/Quart product with Peewee-style database models, tenant-aware services, document ingestion/retrieval, model/provider configuration, background execution, admin APIs, and MCP. Evidence: [`pyproject.toml`](../../../repos/ragflow/pyproject.toml:1), [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1).
- **FACT:** `Tenant`, `UserTenant`, tenant-scoped resources, tenant-specific model configuration, and tenant-derived index names are implemented in source.
- **INFERENCE:** RAGFlow has the strongest demonstrated tenant isolation and RAG specialization in this triage set.
- **INFERENCE:** It is a strong platform RAG subsystem or tenant-aware product foundation, but its domain model and retrieval/indexing internals are more invasive to replace than a generic provider/component layer.

## 2. Architecture & stack

- **FACT:** The manifest uses Quart, Flask-Login, Peewee, MySQL/PostgreSQL drivers, Elasticsearch/OpenSearch, Infinity, MinIO, MCP, Langfuse, and multiple model providers. Evidence: [`pyproject.toml`](../../../repos/ragflow/pyproject.toml:1).
- **FACT:** Runtime startup initializes database/configuration, MCP cleanup, global plugin services, and background chat channels. Evidence: [`ragflow_server.py`](../../../repos/ragflow/api/ragflow_server.py:1).
- **INFERENCE:** The system is a vertically integrated RAG product with API/admin/UI/worker concerns rather than a small library.

## 3. Repo structure

- **FACT:** `api/db` contains models, services, joint services, and provider/tenant logic; `admin/server` contains administrative authentication/configuration; `mcp/server` contains MCP hosting; API route modules expose product operations.
- **FACT:** Document, file, dialog, conversation, memory, tenant-model, and metadata services are separate but share tenant/domain entities.
- **INFERENCE:** Service separation is real at module level, while tenant identity and storage/index conventions are cross-cutting dependencies.

## 4. Data model

- **FACT:** Models include `User`, `Tenant`, `UserTenant`, `TenantLLM`, `Knowledgebase`, `Document`, `File`, `Conversation`, `APIToken`, `UserCanvas`, `MCPServer`, and `Memory`. Evidence: [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1).
- **FACT:** `Memory.memory_type` bit flags distinguish raw, semantic, episodic, and procedural memory. Evidence: [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1).
- **FACT:** Document metadata indexes are named with tenant IDs, e.g. `ragflow_doc_meta_<tenant_id>`. Evidence: [`doc_metadata_service.py`](../../../repos/ragflow/api/db/services/doc_metadata_service.py:1).
- **INFERENCE:** The data model treats tenant as a first-class domain boundary rather than inferring isolation from user ownership alone.

## 5. AuthN/AuthZ/multi-tenancy

- **FACT:** API authentication supports session cookies, beta tokens, JWT, and API tokens, with `login_required` route protection. Evidence: [`apps/__init__.py`](../../../repos/ragflow/api/apps/__init__.py:1).
- **FACT:** Admin login is a dedicated endpoint and admin routes use Flask-Login plus admin authorization checks. Evidence: [`routes.py`](../../../repos/ragflow/admin/server/routes.py:1), [`auth.py`](../../../repos/ragflow/admin/server/auth.py:1).
- **FACT:** Conversation completion verifies dialog ownership using `tenant_id`; dialog listing and model resolution are tenant-scoped. Evidence: [`conversation_service.py`](../../../repos/ragflow/api/db/services/conversation_service.py:1), [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:1).
- **FACT:** File operations filter by tenant ID and knowledge-base/root folders are tenant-specific. Evidence: [`file_service.py`](../../../repos/ragflow/api/db/services/file_service.py:1).
- **FACT:** Tenant model service rejects provider access unless the caller is owner or joined tenant. Evidence: [`tenant_model_service.py`](../../../repos/ragflow/api/db/joint_services/tenant_model_service.py:1).
- **INFERENCE:** This is genuine tenant/workspace isolation evidence, materially stronger than merely having user accounts. Residual risk remains in every route/backend integration that must propagate tenant predicates correctly.

## 6. Provider abstraction

- **FACT:** Dialog service resolves tenant-specific chat, embedding, rerank, and TTS models and invokes `LLMBundle`. Evidence: [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:1).
- **FACT:** Tenant model service centralizes provider/model access checks. Evidence: [`tenant_model_service.py`](../../../repos/ragflow/api/db/joint_services/tenant_model_service.py:1).
- **INFERENCE:** Provider abstraction is real and tenant-aware, but model configuration is coupled to RAGFlow’s tenant/domain schema and bundle semantics.
- **INFERENCE:** Adding a provider is easier than replacing the model-resolution contract or tenant provider store.

## 7. Conversation execution path

- **LOGIN FACT:** API route protection accepts session, JWT, beta-token, and API-token forms; admin login separately establishes admin auth. Evidence: [`apps/__init__.py`](../../../repos/ragflow/api/apps/__init__.py:1), [`auth.py`](../../../repos/ragflow/admin/server/auth.py:1).
- **CHAT FACT:** Conversation completion validates tenant-scoped dialog access, then dialog service resolves models, performs retrieval/citations and optional SQL/web/knowledge-graph retrieval, and invokes the LLM bundle. Evidence: [`conversation_service.py`](../../../repos/ragflow/api/db/services/conversation_service.py:1), [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:1).
- **CHAT INFERENCE:** Response generation is a tenant-aware chain: authenticated request → tenant/dialog authorization → model bundle → retrieval/citation/tool-like augmentations → streaming or persisted conversation response.
- **FILE FACT:** File and document services enforce tenant filters, derive tenant-specific indexes, and enqueue ingestion work. Evidence: [`file_service.py`](../../../repos/ragflow/api/db/services/file_service.py:1), [`document_service.py`](../../../repos/ragflow/api/db/services/document_service.py:1).
- **API FACT:** MCP host mode uses API key/bearer access and delegates resource checks to RAGFlow APIs. Evidence: [`server.py`](../../../repos/ragflow/mcp/server/server.py:1).

## 8. Memory

- **FACT:** Memory model explicitly represents raw, semantic, episodic, and procedural types. Evidence: [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1).
- **FACT:** Memory message service extracts memory with tenant-specific LLM/embedding configuration and applies size/forgetting policies, creating vector indexes per memory. Evidence: [`memory_message_service.py`](../../../repos/ragflow/api/db/joint_services/memory_message_service.py:1).
- **INFERENCE:** RAGFlow has the clearest implemented semantic/episodic/procedural memory taxonomy in this set.
- **UNKNOWN:** Whether user-approved condensation is exposed as a complete user-facing workflow rather than internal forgetting/size management.

## 9. RAG

- **FACT:** Dialog service performs retrieval, citations, SQL-style retrieval, web search, and knowledge-graph retrieval. Evidence: [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:1).
- **FACT:** File/document services handle tenant-scoped documents, limits, asynchronous tasks, and tenant-specific indexes. Evidence: [`document_service.py`](../../../repos/ragflow/api/db/services/document_service.py:1), [`doc_metadata_service.py`](../../../repos/ragflow/api/db/services/doc_metadata_service.py:1).
- **INFERENCE:** RAG is a core integrated subsystem, not merely a pluggable graph node. Vector/search backend replacement is possible through service/storage abstractions but likely high effort because retrieval, metadata, citations, graph search, and tenant index naming are interdependent.

## 10. Agents/workflows/tools/MCP

- **FACT:** Admin services manage agents and sandboxes; user canvases and MCP servers are represented in the data model. Evidence: [`services.py`](../../../repos/ragflow/admin/server/services.py:1), [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1).
- **FACT:** MCP server supports self-host and host multi-tenant modes and exposes dataset/chat/retrieval tools with API/bearer access. Evidence: [`server.py`](../../../repos/ragflow/mcp/server/server.py:1).
- **INFERENCE:** RAGFlow’s agent/tool layer is product-integrated and RAG-centric; it is less neutral than a workflow engine such as Langflow/Flowise.

## 11. Files/storage

- **FACT:** File service enforces tenant filters and tenant-specific roots; storage is abstracted through configured object/document stores. Evidence: [`file_service.py`](../../../repos/ragflow/api/db/services/file_service.py:1).
- **FACT:** MinIO is a declared dependency and document ingestion is asynchronous. Evidence: [`pyproject.toml`](../../../repos/ragflow/pyproject.toml:1), [`document_service.py`](../../../repos/ragflow/api/db/services/document_service.py:1).
- **INFERENCE:** Storage replacement is feasible at an adapter/configuration boundary, but document/index metadata and tenant path conventions must be preserved.

## 12. API

- **FACT:** Quart API routes, admin routes, API tokens, and MCP server interfaces coexist.
- **FACT:** Admin login is `POST /api/v1/admin/login`; protected admin operations use Flask-Login/admin checks. Evidence: [`routes.py`](../../../repos/ragflow/admin/server/routes.py:1).
- **INFERENCE:** The API is suitable as a tenant-aware backend service, but its contracts are product/domain-specific rather than a minimal generic orchestration API.

## 13. Background jobs/usage

- **FACT:** Document indexing and ingestion use asynchronous task queues; server startup starts background chat channels. Evidence: [`document_service.py`](../../../repos/ragflow/api/db/services/document_service.py:1), [`ragflow_server.py`](../../../repos/ragflow/api/ragflow_server.py:1).
- **FACT:** Per-user document count limits are enforced through `MAX_FILE_NUM_PER_USER`. Evidence: [`document_service.py`](../../../repos/ragflow/api/db/services/document_service.py:1).
- **UNKNOWN:** Complete credit/billing ledger and token-level usage accounting.
- **INFERENCE:** Storage quotas can build on existing document limits, but full tenant billing requires cross-cutting instrumentation around model and ingestion jobs.

## 14. Deployment/scaling

- **FACT:** Runtime includes separate API/admin/background/MCP concerns and external search/object-store dependencies.
- **INFERENCE:** Horizontal scale requires shared relational database, object storage, search/index services, queue/task state, and consistent tenant index naming.
- **UNKNOWN:** Exact deployment manifests and benchmarked worker scaling were not exhaustively inspected.

## 15. Observability/testing

- **FACT:** Langfuse is declared and pytest configuration exists. Evidence: [`pyproject.toml`](../../../repos/ragflow/pyproject.toml:1).
- **INFERENCE:** Model/RAG tracing can be integrated, but coverage across tenant authorization and all retrieval backends remains a key audit concern.
- **UNKNOWN:** Effective test coverage for every tenant boundary and MCP host mode.

## 16. Coupling/extensibility

- **FACT:** Provider resolution, tenant authorization, file/document services, retrieval, memory, and MCP have named service boundaries.
- **INFERENCE:** Provider and external storage additions are moderate; replacing the database, tenant model, or retrieval engine is invasive because tenant IDs flow into relational predicates, index names, model access, memory, and API checks.
- **INFERENCE:** RAGFlow is more forkable as a specialized RAG product than as a generic orchestration library.

## 17. 13 modification tests

| Modification | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 2 | Add provider/model configuration and `LLMBundle` integration. |
| Another provider | 2 | Provider registry/model service is explicit but tenant configuration must be supported. |
| Persistent user memory | 1 | Memory entity, extraction, vector indexing, and policy already exist. |
| Project memory | 2 | Tenant/knowledge-base scoping exists; project semantics require domain mapping. |
| Context monitoring | 3 | Must instrument retrieval, memory, model bundle, streaming, and background jobs. |
| User-approved condensation | 3 | Existing forgetting/size policy is not evidence of approval workflow. |
| Credits | 4 | Billing ledger absent from inspected evidence; tenant/provider/jobs all involved. |
| Storage quotas | 2 | Existing per-user document limits and storage domain provide a base. |
| Custom external integration | 2 | API, MCP, tools, and admin/service seams exist. |
| MCP tool/server | 1 | Native MCP server and host multi-tenancy exist. |
| Independent backend/API in front | 2 | API/token/tenant boundaries are explicit, but domain mapping is substantial. |
| Frontend replacement | 2 | API/admin separation helps; product-specific data contracts remain. |
| Connecting to another candidate | 3 | API/MCP/provider boundaries are practical; direct data-model reuse is coupled. |

## 18. Evidence log

- **FACT:** Tenant/data model: [`db_models.py`](../../../repos/ragflow/api/db/db_models.py:1).
- **FACT:** Auth: [`apps/__init__.py`](../../../repos/ragflow/api/apps/__init__.py:1), [`auth.py`](../../../repos/ragflow/admin/server/auth.py:1).
- **FACT:** Tenant-scoped chat/provider/retrieval: [`conversation_service.py`](../../../repos/ragflow/api/db/services/conversation_service.py:1), [`dialog_service.py`](../../../repos/ragflow/api/db/services/dialog_service.py:1).
- **FACT:** File/document/index isolation: [`file_service.py`](../../../repos/ragflow/api/db/services/file_service.py:1), [`document_service.py`](../../../repos/ragflow/api/db/services/document_service.py:1).
- **FACT:** Memory taxonomy/pipeline: [`memory_message_service.py`](../../../repos/ragflow/api/db/joint_services/memory_message_service.py:1).
- **FACT:** MCP host mode: [`server.py`](../../../repos/ragflow/mcp/server/server.py:1).

## 19. Unknowns

- **UNKNOWN:** Complete billing/credits and token accounting.
- **UNKNOWN:** Full deployment manifests and measured scale limits.
- **UNKNOWN:** Complete test coverage of tenant predicates and all storage/search adapters.
- **UNKNOWN:** User-facing approval semantics for condensation/forgetting.
