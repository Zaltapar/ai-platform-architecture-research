# Dify — Wave B Deep Forensic Investigation

## 1. Scope/evidence quality

**Scope.** Source-level review was limited to [`research/repos/dify`](../../../repos/dify:1), its API/web/runtime/deployment code, tests, migrations, and the orientation report [`triage-dify.md`](triage-dify.md:1). No product code was changed.

**FACT:** The checkout exposes a Flask/Python API, Next.js web application, SQLAlchemy/Alembic persistence, Redis/Celery workers, provider/plugin packages, agent/workflow services, MCP routes, and a separate runtime. Key anchors are [`api/pyproject.toml`](../../../repos/dify/api/pyproject.toml:6), [`docker/docker-compose.yaml`](../../../repos/dify/docker/docker-compose.yaml:7), and [`api/controllers`](../../../repos/dify/api/controllers:1).

**Evidence quality:** High for auth, tenant schema, chat entry, provider registry, RAG abstractions, quota, tools/MCP, and deployment. Medium for the complete provider invocation chain across every app mode and for user-facing long-term memory semantics. Those unresolved areas are marked UNKNOWN.

## 2. Architecture/data ownership

**FACT:** [`Account`](../../../repos/dify/api/models/account.py:89), [`Tenant`](../../../repos/dify/api/models/account.py:253), [`App`](../../../repos/dify/api/models/model.py:407), conversations/messages, datasets/documents/segments, agents, workflows, provider credentials, tool providers, uploaded files, and credit pools are persisted in the API database. Alembic history is under [`api/migrations/versions`](../../../repos/dify/api/migrations/versions:1).

**FACT:** API owns authorization, domain orchestration, provider configuration, RAG metadata, message/conversation state, and quota reservation. Web owns UI state and calls API route families. Celery/Redis own asynchronous tasks and queues; object storage/vector services own binary and index payloads.

**INFERENCE:** This is a modular monolith plus worker/runtime services. Domain identity is tenant-centered; replacement of the database or domain model would be invasive.

## 3. LOGIN trace

**FACT:** Enterprise web login enters [`LoginApi.post()`](../../../repos/dify/api/controllers/web/login.py:106), normalizes the email, calls `WebAppAuthService.authenticate`, maps authentication failures, then calls `WebAppAuthService.login` and returns an access token at [`login.py`](../../../repos/dify/api/controllers/web/login.py:108).

**FACT:** Request decorators enforce setup/edition and shared web identity context is applied by [`controllers/web/wraps.py`](../../../repos/dify/api/controllers/web/wraps.py:33). Workspace/account authorization is centralized through [`current_account_with_tenant()`](../../../repos/dify/api/controllers/common/wraps.py:20) and RBAC checks in [`checks.py`](../../../repos/dify/api/controllers/common/rbac/checks.py:33).

**FACT:** Tenant membership and role state are represented by `TenantAccountJoin` in [`account.py`](../../../repos/dify/api/models/account.py:253); app/dataset/provider resources carry tenant ownership. App visibility is further filtered by [`resolve_app_access_filter()`](../../../repos/dify/api/controllers/common/app_access.py:68).

**Boundary/state:** Login is synchronous request/DB/service work. Tenant selection is request identity/context state, not a separate frontend-only choice. Authorization is distributed through decorators, RBAC locators, and tenant predicates.

## 4. CHAT trace

**FACT:** Web chat enters [`ChatApi.post()`](../../../repos/dify/api/controllers/web/completion.py:208), validates app mode and optional conversation ownership through `ConversationService.get_conversation` at [`completion.py`](../../../repos/dify/api/controllers/web/completion.py:220), and calls `AppGenerateService.generate` at [`completion.py`](../../../repos/dify/api/controllers/web/completion.py:229).

**FACT:** Chat generation is delegated to [`ChatAppGenerator.generate()`](../../../repos/dify/api/core/app/apps/chat/app_generator.py:37) and [`ChatAppRunner`](../../../repos/dify/api/core/app/apps/chat/app_generator.py:14). Agent apps have a separate generator under [`agent_app/app_generator.py`](../../../repos/dify/api/core/app/apps/agent_app/app_generator.py:1).

**FACT:** Short-term state is held by conversation/message/message-file/message-agent-thought records in [`model.py`](../../../repos/dify/api/models/model.py:407). Context-window behavior is implemented by [`TokenBufferMemory`](../../../repos/dify/api/core/memory/token_buffer_memory.py:31). Dataset retrieval uses [`AbstractVectorFactory`](../../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29) and [`DatasetDocumentStore`](../../../repos/dify/api/core/rag/docstore/dataset_docstore.py:14).

**FACT:** Agent tool runtime receives tenant/app/user context in [`base_agent_runner.py`](../../../repos/dify/api/core/agent/base_agent_runner.py:55) and resolves tools through `ToolManager` at [`base_agent_runner.py`](../../../repos/dify/api/core/agent/base_agent_runner.py:145). MCP is represented by tool models and [`api/core/mcp`](../../../repos/dify/api/core/mcp:1).

**INFERENCE:** The actual chain is route → app/conversation validation → app generator/runner → token-buffer/context and dataset retrieval → model runtime/plugin invocation → agent/workflow/tool execution → streamed task/message state. Exact provider call for each mode is delegated below the generator and is not exhaustively traceable from one route.

**Async/state:** Streaming and task control cross into queue/task infrastructure; persistent message/conversation state is API DB-owned. Worker boundaries are asynchronous and Redis/Celery-backed.

## 5. FILE trace

**FACT:** Upload enters [`FileApi.post()`](../../../repos/dify/api/controllers/web/files.py:41), validates multipart presence/count/name, reads the stream, and calls `application_services().files.upload_file` at [`files.py`](../../../repos/dify/api/controllers/web/files.py:81).

**FACT:** Uploaded-file metadata and message/tool files are modeled in [`model.py`](../../../repos/dify/api/models/model.py:1909). Storage adapters are under [`extensions/storage`](../../../repos/dify/api/extensions/storage:1), with S3/Azure/GCS and other implementations. A database file-access controller participates in access checks at [`model.py`](../../../repos/dify/api/models/model.py:67).

**FACT:** Dataset ingestion owns documents, segments, child chunks and embeddings in [`dataset.py`](../../../repos/dify/api/models/dataset.py:166). Vector backend selection is registry/factory-based in [`vector_factory.py`](../../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29); ingestion may run through Celery workers.

**INFERENCE:** File flow is upload → storage/metadata → document parsing/chunking pipeline → embedding/provider call → vector backend/index → retrieval through dataset docstore/vector factory. Exact parser dispatch for every MIME type and every worker task was not exhaustively followed.

## 6. API trace

**FACT:** API families are separated into console, web, service API, inner API, files, workflow, triggers, and MCP under [`api/controllers`](../../../repos/dify/api/controllers:1). Service API credentials/end-users are modeled separately from console accounts in [`model.py`](../../../repos/dify/api/models/model.py:2112).

**FACT:** Shared wrappers derive account/tenant identity and invoke RBAC through [`wraps.py`](../../../repos/dify/api/controllers/common/wraps.py:19). Plugin backward-invocation endpoints pass tenant/user identity into model, embedding, rerank, tool, app, and file calls at [`plugin.py`](../../../repos/dify/api/controllers/inner_api/plugin/plugin.py:58).

**INFERENCE:** Public/service API request → token/end-user authentication → tenant/app authorization → application/service orchestration → provider/tool/plugin execution → response/stream and persistence. Public versus internal separation is explicit, but internal APIs require operational trust and correct tenant payloads.

## 7. Memory/context

**FACT:** Conversation history, messages, agent thoughts, workflow variables, and [`TokenBufferMemory`](../../../repos/dify/api/core/memory/token_buffer_memory.py:31) form short-term/context memory.

**FACT:** Datasets are knowledge memory, not a generic user-memory table. The inspected models did not establish a dedicated `user_memories` or `project_memories` lifecycle with extraction, prioritization, update, and deletion.

**UNKNOWN:** A complete product-level personal long-term memory subsystem and user-approved condensation workflow were not established. Token accounting exists in model/runtime structures, but user-facing remaining-context telemetry was not proven.

## 8. RAG

**FACT:** Datasets/documents/segments/child chunks/embeddings are first-class in [`dataset.py`](../../../repos/dify/api/models/dataset.py:166). [`DatasetDocumentStore`](../../../repos/dify/api/core/rag/docstore/dataset_docstore.py:14) reads/writes segment/chunk state.

**FACT:** Vector implementations are replaceable through [`AbstractVectorFactory`](../../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29) and packages under [`api/providers/vdb`](../../../repos/dify/api/providers/vdb:1).

**INFERENCE:** Vector storage is a real interface boundary, while dataset schemas, retrieval semantics, tenant/RBAC ownership, embeddings, and app configuration remain coupled to Dify’s domain model.

## 9. Agents/workflows/tools/MCP

**FACT:** Workflow definitions/runs/node executions/pause variables are persisted in [`workflow.py`](../../../repos/dify/api/models/workflow.py:209). Agent snapshots/revisions/workspace bindings are persisted in [`agent.py`](../../../repos/dify/api/models/agent.py:142).

**FACT:** Tool providers include built-in/API/workflow/MCP types in [`tools.py`](../../../repos/dify/api/models/tools.py:78). MCP controller/API and runtime code exist under [`controllers/mcp`](../../../repos/dify/api/controllers/mcp:1) and [`api/core/mcp`](../../../repos/dify/api/core/mcp:1).

**INFERENCE:** Agent/workflow/tool runtime is product-integrated, with app publishing, credentials, persistence, queueing, and quota dependencies. It is not a neutral embeddable agent library.

## 10. Auth/multi-tenancy

**FACT:** Tenant membership, tenant-owned app/dataset/provider records, tenant-scoped credential resolution, and RBAC resource locators are implemented. Provider credential resolution explicitly filters by tenant in [`runtime_credentials.py`](../../../repos/dify/api/controllers/inner_api/runtime_credentials.py:105).

**INFERENCE:** Isolation is genuine workspace/tenant isolation, but enforcement is application-level and distributed rather than one universal database RLS policy. Security tests/migrations should be audited during independent security review.

## 11. Storage/jobs/usage

**FACT:** Celery and Redis are direct dependencies; worker entrypoint is [`celery_entrypoint.py`](../../../repos/dify/api/celery_entrypoint.py:1). Compose separates API/worker dependencies in [`docker-compose.yaml`](../../../repos/dify/docker/docker-compose.yaml:74).

**FACT:** Credit pools and quota reservation/commit/release are implemented through [`QuotaService`](../../../repos/dify/api/services/quota_service.py:92) and `tenant_credit_pools` in [`model.py`](../../../repos/dify/api/models/model.py:2717).

**UNKNOWN:** Complete self-hosted billing entitlement integration and full storage quota enforcement were not established. OpenTelemetry dependencies are declared in [`pyproject.toml`](../../../repos/dify/api/pyproject.toml:36), but operational coverage was not measured.

## 12. Interface/coupling analysis

- **Provider:** real factory/protocol/plugin seam in [`provider_manager.py`](../../../repos/dify/api/core/provider_manager.py:17), but credentials, model settings, tenant records, app modes, and quota couple it to the product. Replaceability: moderate.
- **Memory:** token buffer is close to app generation; replacing the memory model is high blast radius. Replaceability: low/moderate.
- **RAG:** vector factory/provider registry is explicit. Replaceability: relatively high for vector backend, lower for dataset semantics.
- **Agent/workflow/MCP:** deeply linked to persistence, app publishing, queues, tools, credentials, and quota. Replaceability: low.
- **Auth/frontend/storage:** separately routed but contracts are product-specific; replacement requires broad API/resource mapping.

## 13. Modification tests

| Test | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 1 | Existing provider/runtime configuration and endpoint seam in [`provider_manager.py`](../../../repos/dify/api/core/provider_manager.py:50). |
| Another provider | 2 | Provider/plugin declarations exist; credentials, settings, model types, usage and tests must align. |
| Persistent user memory | 3 | New model/API/context/retrieval/update/delete lifecycle required beyond conversations. |
| Project-specific memory | 2 | Tenant/app/dataset ownership exists, but memory semantics and retrieval context must be added. |
| Context monitoring | 2 | Token buffer/runtime usage exists; API/UI telemetry is additional work. |
| User-approved condensation | 3 | Requires persisted proposal/approval state, generator changes, and frontend UX. |
| Usage credits | 1 | Credit pools and quota service already exist. |
| Storage quotas | 2 | File/storage records and quota hooks exist; aggregate policy/enforcement needs work. |
| Custom external integration | 1-2 | Plugin, service API, triggers, datasource and OAuth seams exist. |
| MCP server/tool | 1 | Native MCP models/routes/runtime exist. |
| Independent backend/API in front | 2 | Service API and token surfaces exist; identity/tenant mapping must be preserved. |
| Frontend replacement | 2 | Separate web/API helps, but product contracts are extensive. |
| Connect another candidate | 3 | RAG API/vector seam is practical; memory/agent/domain integration is substantial. |

## 14. Upgrade/forkability

**INFERENCE:** Dify is usable as a product/backend foundation but expensive as a deep fork. Provider/RAG/plugin additions can remain upstream-compatible; changing tenant, app, workflow, memory, or persistence semantics creates migration and runtime divergence.

**FACT:** API/web/runtime/worker boundaries improve deployment modularity, but schema migrations and plugin contracts create upgrade coordination requirements. Fork risk is high for domain-model changes and moderate for isolated providers/tools.

## 15. Risks/unknowns

- **UNKNOWN:** Complete provider invocation chain for all app modes.
- **UNKNOWN:** Full parser/chunker worker path for every file type.
- **UNKNOWN:** Generic long-term user/project memory lifecycle.
- **UNKNOWN:** Self-hosted commercial billing and complete storage quota enforcement.
- **RISK/INFERENCE:** Distributed tenant checks create authorization regression risk if new services omit tenant predicates.

## 16. Evidence index

Primary anchors: [`login.py`](../../../repos/dify/api/controllers/web/login.py:85), [`completion.py`](../../../repos/dify/api/controllers/web/completion.py:191), [`files.py`](../../../repos/dify/api/controllers/web/files.py:23), [`provider_manager.py`](../../../repos/dify/api/core/provider_manager.py:17), [`app_generator.py`](../../../repos/dify/api/core/app/apps/chat/app_generator.py:37), [`token_buffer_memory.py`](../../../repos/dify/api/core/memory/token_buffer_memory.py:31), [`vector_factory.py`](../../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29), [`dataset_docstore.py`](../../../repos/dify/api/core/rag/docstore/dataset_docstore.py:14), [`quota_service.py`](../../../repos/dify/api/services/quota_service.py:92), [`tools.py`](../../../repos/dify/api/models/tools.py:78), [`workflow.py`](../../../repos/dify/api/models/workflow.py:209), [`agent.py`](../../../repos/dify/api/models/agent.py:142), and [`docker-compose.yaml`](../../../repos/dify/docker/docker-compose.yaml:7).
