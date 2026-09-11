# Dify — Source-Level Forensic Triage

## 1. Verdict snapshot

- **FACT:** Dify is a multi-service AI application platform: Flask/Python API, Next.js web UI, Celery/Redis workers, plugin/provider packages, a separate agent runtime, and optional vector-database services ([`api/pyproject.toml`](../../repos/dify/api/pyproject.toml:6), [`docker/docker-compose.yaml`](../../repos/dify/docker/docker-compose.yaml:7)).
- **FACT:** It has the strongest native workspace/tenant, provider, workflow, agent, RAG, MCP, usage-credit, and deployment surface of the four candidates.
- **INFERENCE:** Best fit as a product/backend foundation when a future platform is willing to adopt Dify's domain model and service boundaries.
- **INFERENCE:** Worst fit as a lightweight embeddable library; replacing its app model, provider runtime, or persistence layer would be invasive.

## 2. Architecture & stack (evidence: file paths)

- **FACT:** Python API uses Flask, Flask-RESTX, SQLAlchemy models, Celery, Redis, PostgreSQL/MySQL-compatible configuration, and OpenTelemetry dependencies ([`api/controllers/console/auth/login.py`](../../repos/dify/api/controllers/console/auth/login.py:1), [`api/pyproject.toml`](../../repos/dify/api/pyproject.toml:6)).
- **FACT:** Web is a separate Next.js/TypeScript application under [`web`](../../repos/dify/web/package.json:1); API route families are split into console, web, service, MCP, files, and workflow controllers ([`api/controllers`](../../repos/dify/api/controllers:1)).
- **FACT:** Agent execution is split between Python orchestration and [`dify-agent-runtime`](../../repos/dify/dify-agent-runtime/README.md:1), a Go runtime for shell/runtime utilities.
- **INFERENCE:** Runtime layout is a modular monolith plus worker/runtime services, not a single process.

## 3. Repo structure map

- [`api`](../../repos/dify/api:1): Flask API, SQLAlchemy models, services, controllers, RAG, providers, plugins, Celery tasks, migrations.
- [`web`](../../repos/dify/web:1): Next.js frontend and client-side service layer.
- [`api/providers`](../../repos/dify/api/providers:1): provider-specific vector backends and related packages.
- [`dify-agent`](../../repos/dify/dify-agent:1): Python agent backend/runtime client-facing code.
- [`dify-agent-runtime`](../../repos/dify/dify-agent-runtime:1): Go sandbox/runtime service.
- [`docker`](../../repos/dify/docker:1), [`sdks`](../../repos/dify/sdks:1), [`e2e`](../../repos/dify/e2e:1): deployment, SDK, and end-to-end assets.

## 4. Data model (table list with schema file locations)

- **FACT:** SQLAlchemy declarative models are the primary schema source; Alembic migrations are under [`api/migrations/versions`](../../repos/dify/api/migrations/versions:1).
- Accounts/tenancy: `accounts`, `tenants`, `tenant_account_joins`, `invitation_codes`, plugin permissions in [`api/models/account.py`](../../repos/dify/api/models/account.py:89).
- Apps/conversations: `apps`, `app_model_configs`, `conversations`, `messages`, `message_files`, `message_feedbacks`, `message_agent_thoughts`, `api_requests`, `upload_files` in [`api/models/model.py`](../../repos/dify/api/models/model.py:407).
- Knowledge/RAG: `datasets`, `documents`, `document_segments`, `child_chunks`, `embeddings`, dataset bindings/permissions in [`api/models/dataset.py`](../../repos/dify/api/models/dataset.py:166).
- Agents/workflows: `agents`, agent snapshots/config revisions/workspaces in [`api/models/agent.py`](../../repos/dify/api/models/agent.py:142); `workflows`, `workflow_runs`, `workflow_node_executions`, pause/variable tables in [`api/models/workflow.py`](../../repos/dify/api/models/workflow.py:209).
- Providers/tools/MCP: provider credential/model tables in [`api/models/provider.py`](../../repos/dify/api/models/provider.py:38); tool provider/MCP/model-invoke/file tables in [`api/models/tools.py`](../../repos/dify/api/models/tools.py:78).
- Usage: `tenant_credit_pools` in [`api/models/model.py`](../../repos/dify/api/models/model.py:2717), plus quota/credit orchestration in [`api/services/quota_service.py`](../../repos/dify/api/services/quota_service.py:92).

## 5. AuthN / AuthZ / multi-tenancy (with isolation evidence)

- **FACT:** Local password login is exposed by [`LoginApi.post()`](../../repos/dify/api/controllers/console/auth/login.py:120) and delegates to account authentication services; access/refresh/CSRF tokens are cookie-managed ([`api/controllers/console/auth/login.py`](../../repos/dify/api/controllers/console/auth/login.py:52)).
- **FACT:** OAuth routes exist under [`api/controllers/console/auth/oauth.py`](../../repos/dify/api/controllers/console/auth/oauth.py:1); Flask-Login user identity and tenant role are represented by [`Account`](../../repos/dify/api/models/account.py:89).
- **FACT:** Tenant isolation is first-class: `TenantAccountJoin` associates users to tenants and roles, while app/dataset/provider records carry tenant ownership ([`api/models/account.py`](../../repos/dify/api/models/account.py:253), [`api/models/dataset.py`](../../repos/dify/api/models/dataset.py:166)).
- **INFERENCE:** This is real workspace/tenant isolation rather than merely user accounts, although enforcement is distributed across service/controller query code rather than one universal database row-level-security layer.
- **FACT:** API tokens, end users, sites, and service APIs are modeled separately ([`api/models/model.py`](../../repos/dify/api/models/model.py:2112), [`api/models/model.py`](../../repos/dify/api/models/model.py:2293), [`api/controllers/service_api`](../../repos/dify/api/controllers/service_api:1)).

## 6. Provider abstraction

- **FACT:** Provider configuration is persisted per tenant and cached by tenant/source in [`ProviderManager`](../../repos/dify/api/core/provider_manager.py:72).
- **FACT:** The model runtime is obtained through a factory/protocol boundary (`ModelProviderFactory`, provider entities, runtime protocol) ([`api/core/provider_manager.py`](../../repos/dify/api/core/provider_manager.py:19), [`api/core/provider_manager.py`](../../repos/dify/api/core/provider_manager.py:50)).
- **FACT:** Plugin model-provider declarations are supported ([`api/core/provider_manager.py`](../../repos/dify/api/core/provider_manager.py:37)).
- **INFERENCE:** Adding a provider is usually a plugin/package/configuration task, but custom behavior that crosses model settings, billing, or plugin daemon contracts has a larger blast radius.
- **Modification classification:** custom OpenAI-compatible endpoint **1** when using an existing generic/OpenAI-compatible provider path; another provider **2-3** depending on whether a plugin declaration is sufficient.

## 7. Conversation execution path (file:line chain)

- **FACT:** Web chat enters [`ChatApi.post()`](../../repos/dify/api/controllers/web/completion.py:191), validates the app mode/conversation, and calls `AppGenerateService.generate` ([`api/controllers/web/completion.py`](../../repos/dify/api/controllers/web/completion.py:207)).
- **FACT:** Chat generation is implemented by [`ChatAppGenerator.generate()`](../../repos/dify/api/core/app/apps/chat/app_generator.py:37), which uses app configuration, file-upload configuration, queue management, and [`ChatAppRunner`](../../repos/dify/api/core/app/apps/chat/app_generator.py:14).
- **FACT:** The generator's domain objects include `Conversation`, `Message`, `App`, and account/end-user identities ([`api/core/app/apps/chat/app_generator.py`](../../repos/dify/api/core/app/apps/chat/app_generator.py:29)).
- **FACT:** RAG uses `Vector`, lazy tenant-specific embeddings, and a registry-selected vector factory ([`api/core/rag/datasource/vdb/vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29), [`api/core/rag/datasource/vdb/vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:64)).
- **FACT:** Agent apps can delegate streamed execution to the agent backend rather than an in-process ReAct loop ([`api/core/app/apps/agent_app/app_generator.py`](../../repos/dify/api/core/app/apps/agent_app/app_generator.py:1)).
- **INFERENCE:** Response persistence and streaming are coordinated through the message-based generator/queue/service stack; exact provider invocation is delegated through the runtime/plugin model layer.

## 8. Memory

- **FACT:** Short-term conversation memory is represented by `conversations`, `messages`, message chains, agent thoughts, and workflow conversation variables ([`api/models/model.py`](../../repos/dify/api/models/model.py:1132), [`api/models/workflow.py`](../../repos/dify/api/models/workflow.py:1521)).
- **FACT:** Token-window memory exists as [`TokenBufferMemory`](../../repos/dify/api/core/memory/token_buffer_memory.py:31).
- **FACT:** Document/project knowledge is implemented as datasets, segments, child chunks, and embeddings, but no separate general-purpose `user_memories` or `project_memories` table was established in this triage.
- **UNKNOWN:** A complete product-level long-term personal-memory extraction/update lifecycle was not established from the inspected files; report as **NONE established** beyond conversation/workflow memory and knowledge datasets.

## 9. RAG

- **FACT:** Upload/document processing is modeled by datasets/documents/segments/child chunks and pipeline execution tables ([`api/models/dataset.py`](../../repos/dify/api/models/dataset.py:485), [`api/models/dataset.py`](../../repos/dify/api/models/dataset.py:1656)).
- **FACT:** `DatasetDocumentStore` reads/writes SQL-backed document segments and child chunks ([`api/core/rag/docstore/dataset_docstore.py`](../../repos/dify/api/core/rag/docstore/dataset_docstore.py:14)).
- **FACT:** Vector storage is replaceable through `AbstractVectorFactory` and a backend registry ([`api/core/rag/datasource/vdb/vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29), [`api/core/rag/datasource/vdb/vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:12)).
- **FACT:** The repository contains many vector provider packages, including Qdrant, Milvus, Weaviate, Chroma, PGVector/PGVecto-RS, Elasticsearch/OpenSearch, TiDB, and others ([`api/providers/vdb`](../../repos/dify/api/providers/vdb:1)).
- **INFERENCE:** RAG is one of Dify's most replaceable major subsystems, though dataset schemas and retrieval service contracts remain coupled.

## 10. Agents / workflows / tools / MCP

- **FACT:** Workflow schema/run/node execution persistence is substantial ([`api/models/workflow.py`](../../repos/dify/api/models/workflow.py:209)).
- **FACT:** Agent configuration, snapshots, revisions, workspace bindings, and agent thoughts are persisted ([`api/models/agent.py`](../../repos/dify/api/models/agent.py:142)).
- **FACT:** Tool providers include built-in, API, workflow, MCP, tool invokes, files, and published apps ([`api/models/tools.py`](../../repos/dify/api/models/tools.py:78)).
- **FACT:** Native MCP client/server/session code is under [`api/core/mcp`](../../repos/dify/api/core/mcp:1), with MCP controller routes under [`api/controllers/mcp`](../../repos/dify/api/controllers/mcp:1).
- **INFERENCE:** Agent/workflow/tool support is product-grade and deeply integrated with app publishing, credentials, persistence, and quota accounting.

## 11. Files & storage

- **FACT:** Upload files and message/tool files are modeled in SQLAlchemy ([`api/models/model.py`](../../repos/dify/api/models/model.py:1909)).
- **FACT:** Storage adapters include S3, Azure Blob, Google Cloud, Tencent COS, Huawei OBS, Aliyun OSS, Supabase, Oracle OCI, OpenDAL, and others ([`api/extensions/storage`](../../repos/dify/api/extensions/storage:1)).
- **FACT:** File access is controlled through a database file-access controller referenced by model code ([`api/models/model.py`](../../repos/dify/api/models/model.py:67)).
- **FACT:** Docker configuration exposes local middleware/vector/storage services and environment-backed object storage ([`docker/docker-compose.yaml`](../../repos/dify/docker/docker-compose.yaml:8)).
- **INFERENCE:** Storage backend replacement is moderately replaceable through adapters, but signed file URLs, message files, tools, and workflow files create cross-cutting dependencies.

## 12. API surface

- **FACT:** REST API families are separated into console, web, service API, inner API, files, workflow, trigger, and MCP controllers ([`api/controllers`](../../repos/dify/api/controllers:1)).
- **FACT:** Web chat, conversation, message, file, and workflow endpoints are explicit ([`api/controllers/web/completion.py`](../../repos/dify/api/controllers/web/completion.py:191), [`api/controllers/web/conversation.py`](../../repos/dify/api/controllers/web/conversation.py:47), [`api/controllers/web/message.py`](../../repos/dify/api/controllers/web/message.py:64)).
- **FACT:** API tokens and service API wrappers are persisted and routed separately ([`api/models/model.py`](../../repos/dify/api/models/model.py:2293), [`api/controllers/service_api`](../../repos/dify/api/controllers/service_api:1)).
- **FACT:** Rate-limit logs and invoke/quota errors exist in the web generation path ([`api/models/dataset.py`](../../repos/dify/api/models/dataset.py:1485), [`api/controllers/web/completion.py`](../../repos/dify/api/controllers/web/completion.py:251)).

## 13. Background jobs & usage accounting

- **FACT:** Celery is a direct dependency and the API has Celery entrypoint/healthcheck files ([`api/pyproject.toml`](../../repos/dify/api/pyproject.toml:10), [`api/celery_entrypoint.py`](../../repos/dify/api/celery_entrypoint.py:1)).
- **FACT:** Redis is a direct dependency and is used for queues/caches/provider configuration ([`api/pyproject.toml`](../../repos/dify/api/pyproject.toml:23), [`api/core/provider_manager.py`](../../repos/dify/api/core/provider_manager.py:41)).
- **FACT:** Credit pools and quota reservation/commit/release are implemented ([`api/models/model.py`](../../repos/dify/api/models/model.py:2717), [`api/services/quota_service.py`](../../repos/dify/api/services/quota_service.py:92)).
- **INFERENCE:** Dify is unusually ready for commercial usage controls; exact billing entitlement service boundaries include external/hosting integrations not fully inspected.

## 14. Deployment & scaling

- **FACT:** Compose defines shared API/worker environment and dependencies including databases, Redis, vector stores, SSRF proxy, MinIO, etcd, and Milvus options ([`docker/docker-compose.yaml`](../../repos/dify/docker/docker-compose.yaml:7)).
- **FACT:** API and worker configuration are separated through compose anchors ([`docker/docker-compose.yaml`](../../repos/dify/docker/docker-compose.yaml:74)).
- **INFERENCE:** Stateless API scaling is plausible, while Celery/Redis, database, vector backend, object storage, and agent runtime must be shared and correctly configured.
- **UNKNOWN:** Production Kubernetes autoscaling behavior was not validated beyond repository deployment assets.

## 15. Observability & testing

- **FACT:** OpenTelemetry instrumentation covers Flask, Celery, Redis, SQLAlchemy, HTTPX, and related components ([`api/pyproject.toml`](../../repos/dify/api/pyproject.toml:36)).
- **FACT:** CI includes API/web tests, migration tests, vector tests, agent tests, Docker tests, and e2e workflows ([`.github/workflows`](../../repos/dify/.github/workflows:1)).
- **FACT:** API tests and vector-provider unit/integration tests exist ([`api/tests`](../../repos/dify/api/tests:1), [`api/providers/vdb/vdb-qdrant/tests`](../../repos/dify/api/providers/vdb/vdb-qdrant/tests:1)).
- **INFERENCE:** Core paths are substantially tested, but full coverage/quality was not measured.

## 16. Coupling & extensibility assessment

- Provider/auth coupling: **3/5** — explicit provider manager/plugin/runtime boundaries, but tenant credentials, model settings, billing/features, and app configs cross-reference each other ([`api/core/provider_manager.py`](../../repos/dify/api/core/provider_manager.py:17)).
- RAG coupling: **2/5** — explicit vector factory/backend registry and provider packages ([`api/core/rag/datasource/vdb/vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29)).
- Memory coupling: **4/5** for replacing the whole memory model; token buffer is embedded in app generation ([`api/core/memory/token_buffer_memory.py`](../../repos/dify/api/core/memory/token_buffer_memory.py:31)).
- Agent/workflow coupling: **4/5** — persistence, queue, plugin, app publishing, runtime, and tool schemas are interconnected.
- Extensibility: **FACT:** plugins, provider packages, tool/MCP models, APIs, SDKs, and workflow nodes are real extension mechanisms; this is not merely configuration ([`api/core/plugin`](../../repos/dify/api/core/plugin:1), [`api/providers`](../../repos/dify/api/providers:1)).

## 17. Modification tests table (0-5)

| Test | Score | Evidence-based reason |
|---|---:|---|
| Custom OpenAI-compatible API | 1 | Provider configuration/runtime supports configurable endpoints; existing generic provider path likely suffices ([`api/core/provider_manager.py`](../../repos/dify/api/core/provider_manager.py:50)). |
| Add another provider | 2 | Plugin/provider declaration and model runtime contracts exist; provider credentials/settings must be integrated ([`api/core/provider_manager.py`](../../repos/dify/api/core/provider_manager.py:37)). |
| Persistent user memory | 3 | Conversation/token memory exists, but a dedicated personal-memory lifecycle was not established; would touch models, context, retrieval, APIs. |
| Project-specific memory | 2 | Tenant/app/dataset/workspace-like ownership exists; dedicated memory semantics would extend dataset/app context. |
| Context monitoring | 2 | Token buffer and model usage structures exist, but user-facing capacity telemetry requires API/UI changes ([`api/core/memory/token_buffer_memory.py`](../../repos/dify/api/core/memory/token_buffer_memory.py:31)). |
| User-approved condensation | 3 | Token buffer exists, but approval/state/UI/persistence workflow is not evident; needs generator and frontend changes. |
| Usage credits | 1 | Credit pool and quota service already exist ([`api/models/model.py`](../../repos/dify/api/models/model.py:2717), [`api/services/quota_service.py`](../../repos/dify/api/services/quota_service.py:92)). |
| Storage quotas | 2 | Tenant/file/storage models and quota hooks exist, but a complete storage quota policy was not established. |
| Custom external integration | 1-2 | Service APIs, triggers, data-source OAuth/API bindings, and plugin endpoints exist ([`api/models/source.py`](../../repos/dify/api/models/source.py:14)). |
| Add MCP server/tool | 1 | Native MCP models, client/session code, controller routes, and tool providers exist ([`api/models/tools.py`](../../repos/dify/api/models/tools.py:299)). |
| Independent backend/API in front | 2 | Explicit service API and API-token surfaces exist; proxy must preserve Dify auth/app contracts ([`api/controllers/service_api`](../../repos/dify/api/controllers/service_api:1)). |
| Replace frontend | 2 | API is separately organized from Next.js web; product-specific contracts remain substantial ([`web`](../../repos/dify/web:1), [`api/controllers`](../../repos/dify/api/controllers:1)). |
| Connect another candidate as RAG/memory | 3 | RAG has vector-factory boundaries; memory replacement is less explicit and app-generation coupling is high. |

## 18. Evidence log: key file:line citations

- [`Account` and tenant roles](../../repos/dify/api/models/account.py:21)
- [`Account` table](../../repos/dify/api/models/account.py:89)
- [`App`, conversation, message tables](../../repos/dify/api/models/model.py:407)
- [`Dataset` and document tables](../../repos/dify/api/models/dataset.py:166)
- [`Workflow` tables](../../repos/dify/api/models/workflow.py:209)
- [`Agent` tables](../../repos/dify/api/models/agent.py:142)
- [`ProviderManager`](../../repos/dify/api/core/provider_manager.py:17)
- [`ChatApi.post()`](../../repos/dify/api/controllers/web/completion.py:191)
- [`ChatAppGenerator`](../../repos/dify/api/core/app/apps/chat/app_generator.py:37)
- [`AbstractVectorFactory`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29)
- [`DatasetDocumentStore`](../../repos/dify/api/core/rag/docstore/dataset_docstore.py:14)
- [`TokenBufferMemory`](../../repos/dify/api/core/memory/token_buffer_memory.py:31)
- [`QuotaService`](../../repos/dify/api/services/quota_service.py:92)
- [`MCP tool table`](../../repos/dify/api/models/tools.py:299)
- [`Docker shared services`](../../repos/dify/docker/docker-compose.yaml:7)

## 19. Unknowns / could not establish

- **UNKNOWN:** Exact default-branch commit/release cadence and tag history were not independently measured after the shallow clone.
- **UNKNOWN:** Complete provider invocation call chain from every app mode to each plugin/runtime implementation was not exhaustively traced.
- **UNKNOWN:** Whether Dify's current cloud billing service can be used independently in a self-hosted commercial product.
- **UNKNOWN:** A dedicated generic long-term personal-memory subsystem with extraction, prioritization, update, and deletion semantics was not established; treat it as absent for this triage.
- **UNKNOWN:** Full storage-quota enforcement and production autoscaling guarantees were not established.
