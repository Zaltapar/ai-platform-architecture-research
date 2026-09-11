# Open WebUI — Source-Level Forensic Triage

## 1. Verdict snapshot

- **FACT:** Open WebUI is a Python FastAPI backend with a Svelte frontend, SQLAlchemy persistence, configurable vector backends, Redis/session/socket support, function/tool plugins, and native MCP client/tool-server support ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1), [`src`](../../repos/open-webui/src:1)).
- **FACT:** It has first-class users, chats, files, knowledge bases, per-user memories, groups/access grants, model records, tools/functions, automations, and OpenAI/Anthropic-compatible HTTP routes.
- **INFERENCE:** Best fit for a replaceable frontend/API product or a personal-AI/chat frontend foundation with strong provider/RAG configurability.
- **INFERENCE:** Less suitable than Dify/LibreChat as a ready-made project/workspace tenant backend: its core persistence is strongly user-scoped, while workspace/tenant semantics are not the dominant primary boundary in the inspected model layer.

## 2. Architecture & stack (evidence: file paths)

- **FACT:** Backend is FastAPI/Starlette-style Python with dependency-injected route auth and async SQLAlchemy sessions ([`backend/open_webui/routers/chats.py`](../../repos/open-webui/backend/open_webui/routers/chats.py:1), [`backend/open_webui/internal/db.py`](../../repos/open-webui/backend/open_webui/internal/db.py:1)).
- **FACT:** Frontend is Svelte under [`src`](../../repos/open-webui/src:1), built with Vite/Svelte configuration ([`package.json`](../../repos/open-webui/package.json:1), [`vite.config.ts`](../../repos/open-webui/vite.config.ts:1)).
- **FACT:** SQLAlchemy models and Alembic migrations are under [`backend/open_webui/models`](../../repos/open-webui/backend/open_webui/models:1) and [`backend/open_webui/migrations/versions`](../../repos/open-webui/backend/open_webui/migrations/versions:1).
- **FACT:** Vector retrieval is abstracted behind `VECTOR_DB_CLIENT` and an async facade; backends include Chroma, Qdrant, Milvus, pgvector, OpenSearch, Pinecone, Weaviate, MariaDB vector, Valkey, and Oracle23AI ([`backend/open_webui/retrieval/vector/factory.py`](../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:1)).
- **INFERENCE:** It is a modular monolith with optional external vector/cache/LLM services rather than a multi-service orchestration platform like Dify.

## 3. Repo structure map

- [`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1): application bootstrap, OpenAI-compatible routes, lifecycle, OAuth/MCP initialization.
- [`backend/open_webui/routers`](../../repos/open-webui/backend/open_webui/routers:1): FastAPI route modules for auth, chats, files, knowledge, memory, models, tools, functions, automations, users, groups, retrieval, audio, and integrations.
- [`backend/open_webui/models`](../../repos/open-webui/backend/open_webui/models:1): SQLAlchemy tables and repository/table helpers.
- [`backend/open_webui/retrieval`](../../repos/open-webui/backend/open_webui/retrieval:1): loaders, embedding functions, chunking/retrieval utilities, vector DB clients, external knowledge connectors.
- [`backend/open_webui/utils`](../../repos/open-webui/backend/open_webui/utils:1): auth, tools, MCP, middleware, sessions, storage, model/provider helpers.
- [`src`](../../repos/open-webui/src:1), [`static`](../../repos/open-webui/static:1): Svelte frontend/static assets.
- [`docker-compose*.yaml`](../../repos/open-webui/docker-compose.yaml:1): deployment variants for API/data/GPU/OTel/Playwright.

## 4. Data model (table list with schema file locations)

- **FACT:** SQLAlchemy table definitions are in the model modules; Alembic migration history is under [`backend/open_webui/migrations/versions`](../../repos/open-webui/backend/open_webui/migrations/versions:1).
- Identity/auth: `user` in [`models/users.py`](../../repos/open-webui/backend/open_webui/models/users.py:45), OAuth sessions in [`models/oauth_sessions.py`](../../repos/open-webui/backend/open_webui/models/oauth_sessions.py:23), API-key/auth records in auth/config models.
- Conversations/messages: `chat` and `chat_file` in [`models/chats.py`](../../repos/open-webui/backend/open_webui/models/chats.py:129), `chat_message` in [`models/chat_messages.py`](../../repos/open-webui/backend/open_webui/models/chat_messages.py:127), `message` and `message_reaction` in [`models/messages.py`](../../repos/open-webui/backend/open_webui/models/messages.py:19).
- Files/knowledge: `file` in [`models/files.py`](../../repos/open-webui/backend/open_webui/models/files.py:18), `knowledge`, `knowledge_directory`, and `knowledge_file` in [`models/knowledge.py`](../../repos/open-webui/backend/open_webui/models/knowledge.py:48).
- Memory: `memory` with `user_id`, type, path, content, metadata, and timestamps in [`models/memories.py`](../../repos/open-webui/backend/open_webui/models/memories.py:15).
- Providers/models/tools: model records in [`models/models.py`](../../repos/open-webui/backend/open_webui/models/models.py:117), tool records in [`models/tools.py`](../../repos/open-webui/backend/open_webui/models/tools.py:20), function records in [`models/functions.py`](../../repos/open-webui/backend/open_webui/models/functions.py:19).
- Access/automation/social: groups/members, access grants, folders/tags, automations/runs, calendar, channels, notes, skills, prompts, feedback, and shared chats under [`models`](../../repos/open-webui/backend/open_webui/models:1).

## 5. AuthN / AuthZ / multi-tenancy (with isolation evidence)

- **FACT:** Auth supports JWT bearer tokens, cookie tokens, a configurable custom API-key header, and OAuth/OIDC providers; the ASGI middleware resolves tokens from authorization, cookie, or API-key sources ([`backend/open_webui/utils/asgi_middleware.py`](../../repos/open-webui/backend/open_webui/utils/asgi_middleware.py:128), [`backend/open_webui/utils/auth.py`](../../repos/open-webui/backend/open_webui/utils/auth.py:238)).
- **FACT:** JWT uses `WEBUI_SECRET_KEY` and HS256; verified/admin dependencies enforce role gates ([`backend/open_webui/utils/auth.py`](../../repos/open-webui/backend/open_webui/utils/auth.py:47), [`backend/open_webui/utils/auth.py`](../../repos/open-webui/backend/open_webui/utils/auth.py:522)).
- **FACT:** Local sign-in/sign-up, LDAP, OAuth configuration, signout, and user API-key routes are under [`routers/auths.py`](../../repos/open-webui/backend/open_webui/routers/auths.py:716).
- **FACT:** Most core records are directly user-scoped: chats carry `user_id`, files carry owner/user fields, memories carry `user_id`, and route handlers use `get_verified_user` plus ownership checks ([`models/chats.py`](../../repos/open-webui/backend/open_webui/models/chats.py:129), [`models/memories.py`](../../repos/open-webui/backend/open_webui/models/memories.py:21)).
- **FACT:** Groups, access grants, shared folders/chats, knowledge access grants, and model/tool/function access provide resource-level collaboration ([`models/groups.py`](../../repos/open-webui/backend/open_webui/models/groups.py:37), [`models/access_grants.py`](../../repos/open-webui/backend/open_webui/models/access_grants.py:1)).
- **INFERENCE:** This is strong user/resource authorization, but not the same as a first-class tenant/workspace boundary: the inspected primary models do not show a universal `tenant_id` partition comparable to LibreChat’s tenant plugin. Project/workspace memory and database tenant isolation require additional design.

## 6. Provider abstraction

- **FACT:** Main model routing loads configured models and checks per-model access before invoking the provider path in [`chat_completion()`](../../repos/open-webui/backend/open_webui/main.py:1085).
- **FACT:** OpenAI-compatible upstream access is configuration-driven via `OPENAI_API_BASE_URL`/key lists and related endpoint config ([`backend/open_webui/config.py`](../../repos/open-webui/backend/open_webui/config.py:319)).
- **FACT:** Anthropic compatibility and provider-specific utilities exist ([`backend/open_webui/utils/anthropic.py`](../../repos/open-webui/backend/open_webui/utils/anthropic.py:1)); Ollama/OpenAI-compatible routes and external model connections are exposed through main/config/model modules.
- **FACT:** Model entries are persisted as configurable wrappers with params, metadata, access grants, and base model references ([`backend/open_webui/models/models.py`](../../repos/open-webui/backend/open_webui/models/models.py:117)).
- **INFERENCE:** Custom OpenAI-compatible APIs are configuration-level when their wire format matches; adding a fundamentally different provider is moderate development because model discovery, streaming, tools, embeddings, and usage behavior must align.

## 7. Conversation execution path (file:line chain)

- **FACT:** Chat requests enter `POST /api/chat/completions` or `/api/v1/chat/completions` and require `get_verified_user` ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1085)).
- **FACT:** The route resolves `model_id`, model metadata/access, params, parent/chat/message IDs, variables, tool servers, files, and feature metadata before downstream generation ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1095)).
- **FACT:** It requests provider usage in streaming mode when the selected model advertises usage capability ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1173)).
- **FACT:** Chats/messages are persisted through SQLAlchemy `Chat`, `ChatFile`, `ChatMessage`, and `Message` tables ([`backend/open_webui/models/chats.py`](../../repos/open-webui/backend/open_webui/models/chats.py:129), [`backend/open_webui/models/chat_messages.py`](../../repos/open-webui/backend/open_webui/models/chat_messages.py:127)).
- **FACT:** Retrieval context is built through `retrieval/utils.py`, using an embedding function and vector client; model tools can query knowledge/memory vectors ([`backend/open_webui/retrieval/utils.py`](../../repos/open-webui/backend/open_webui/retrieval/utils.py:488), [`backend/open_webui/tools/builtin.py`](../../repos/open-webui/backend/open_webui/tools/builtin.py:2564)).
- **INFERENCE:** Execution is route → user/model access → context/files/tools/retrieval → provider streaming → persistence/events; substantial orchestration is in `main.py`, middleware, tools, and retrieval utilities rather than a single replaceable application service.

## 8. Memory

- **FACT:** Persistent user memory is a dedicated SQLAlchemy table with CRUD methods and explicit user ownership ([`backend/open_webui/models/memories.py`](../../repos/open-webui/backend/open_webui/models/memories.py:15), [`backend/open_webui/models/memories.py`](../../repos/open-webui/backend/open_webui/models/memories.py:50)).
- **FACT:** Memory content is embedded into a per-user vector collection named `user-memory-{user.id}`; add/update/search/reindex/reset routes manage both SQL and vector state ([`backend/open_webui/routers/memories.py`](../../repos/open-webui/backend/open_webui/routers/memories.py:156), [`backend/open_webui/routers/memories.py`](../../repos/open-webui/backend/open_webui/routers/memories.py:331)).
- **FACT:** Memory has `user` and `context` types, path metadata, and content, but no project/workspace foreign key in the inspected table ([`backend/open_webui/models/memories.py`](../../repos/open-webui/backend/open_webui/models/memories.py:21)).
- **INFERENCE:** Personal long-term memory is real and replaceable at the model/vector boundary; project-specific memory would require new ownership/linking semantics and context assembly.

## 9. RAG

- **FACT:** Files are uploaded through [`routers/files.py`](../../repos/open-webui/backend/open_webui/routers/files.py:270), persisted in `file`, then parsed/embedded/indexed through retrieval utilities and the vector client.
- **FACT:** Knowledge bases own `knowledge`, `knowledge_file`, and directories; access grants are checked through knowledge/router/model helpers ([`models/knowledge.py`](../../repos/open-webui/backend/open_webui/models/knowledge.py:48), [`routers/knowledge.py`](../../repos/open-webui/backend/open_webui/routers/knowledge.py:300)).
- **FACT:** `save_docs_to_vector_db` generates embeddings and inserts chunks into a selected vector backend ([`backend/open_webui/routers/retrieval.py`](../../repos/open-webui/backend/open_webui/routers/retrieval.py:1637)).
- **FACT:** Vector backend selection is centralized in a factory with synchronous clients wrapped by an async facade ([`backend/open_webui/retrieval/vector/factory.py`](../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:1), [`backend/open_webui/retrieval/vector/async_client.py`](../../repos/open-webui/backend/open_webui/retrieval/vector/async_client.py:1)).
- **FACT:** External knowledge retrieval supports Qdrant, Milvus, and pgvector-specific connections ([`backend/open_webui/retrieval/external.py`](../../repos/open-webui/backend/open_webui/retrieval/external.py:89)).
- **INFERENCE:** RAG/vector replacement is one of Open WebUI’s strongest extension points; parser/embedding contract changes still affect file processing and retrieval utilities.

## 10. Agents / workflows / tools / MCP

- **FACT:** Tools are persisted and exposed through tool/function routers; functions can be loaded/synced from URLs and have global/user valves ([`backend/open_webui/routers/functions.py`](../../repos/open-webui/backend/open_webui/routers/functions.py:46)).
- **FACT:** Built-in tools perform web/RAG/memory/image/file operations and are injected into chat execution ([`backend/open_webui/tools/builtin.py`](../../repos/open-webui/backend/open_webui/tools/builtin.py:2564)).
- **FACT:** Automations and scheduled runs are first-class routes/models ([`backend/open_webui/routers/automations.py`](../../repos/open-webui/backend/open_webui/routers/automations.py:162)).
- **FACT:** Native MCP client support uses the Python MCP SDK, streamable HTTP transport, OAuth client provider/token storage, and a dedicated MCP utility package ([`backend/open_webui/utils/mcp/client.py`](../../repos/open-webui/backend/open_webui/utils/mcp/client.py:10)).
- **INFERENCE:** Open WebUI has a strong plugin/tool/MCP surface, but its agent runtime is more tool/function oriented than Dify’s explicit workflow/agent product model in the inspected source.

## 11. Files & storage

- **FACT:** File metadata is persisted in `file`; chat-file links include user/chat/message/file ownership fields ([`backend/open_webui/models/files.py`](../../repos/open-webui/backend/open_webui/models/files.py:18), [`backend/open_webui/models/chats.py`](../../repos/open-webui/backend/open_webui/models/chats.py:217)).
- **FACT:** File routes expose upload, list, search, content, processing status, rename, and deletion APIs ([`backend/open_webui/routers/files.py`](../../repos/open-webui/backend/open_webui/routers/files.py:270)).
- **FACT:** Storage defaults/configuration are in the backend data directory with configurable external/object storage paths in the file utility layer; exact production object-storage matrix was not exhaustively inspected.
- **FACT:** Deleting files removes associated vector records/collections through `ASYNC_VECTOR_DB_CLIENT` ([`backend/open_webui/routers/files.py`](../../repos/open-webui/backend/open_webui/routers/files.py:1019)).
- **INFERENCE:** User file ownership is explicit; storage quota and tenant-wide aggregation are not first-class in the inspected core.

## 12. API surface

- **FACT:** OpenAI-compatible model, embedding, chat-completion, and Anthropic-style message/token-count routes are in [`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:874), [`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1048), and [`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1974).
- **FACT:** REST routers cover auth, chats, messages/channels, files, knowledge, retrieval, memories, models, tools/functions, users/groups, automations, pipelines, plugins, audio, images, and integrations ([`backend/open_webui/routers`](../../repos/open-webui/backend/open_webui/routers:1)).
- **FACT:** API keys can be enabled/configured and are accepted through the custom header middleware ([`backend/open_webui/env.py`](../../repos/open-webui/backend/open_webui/env.py:796), [`backend/open_webui/utils/asgi_middleware.py`](../../repos/open-webui/backend/open_webui/utils/asgi_middleware.py:128)).
- **FACT:** Usage endpoint and analytics/token routes exist ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:2587), [`backend/open_webui/routers/analytics.py`](../../repos/open-webui/backend/open_webui/routers/analytics.py:235)).

## 13. Background jobs & usage accounting

- **FACT:** Redis/session/socket infrastructure exists, including optional Redis session store and websocket session pool ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:2615), [`backend/open_webui/socket/main.py`](../../repos/open-webui/backend/open_webui/socket/main.py:143)).
- **FACT:** Chat tasks can be listed/stopped, and automations have run records/routes ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:2101), [`backend/open_webui/routers/automations.py`](../../repos/open-webui/backend/open_webui/routers/automations.py:334)).
- **FACT:** Streaming provider usage is requested and analytics expose token routes ([`backend/open_webui/main.py`](../../repos/open-webui/backend/open_webui/main.py:1173), [`backend/open_webui/routers/analytics.py`](../../repos/open-webui/backend/open_webui/routers/analytics.py:235)).
- **UNKNOWN:** A commercial credit ledger, reservation/commit mechanism, billing hooks, or storage quota system was not established; classify as **NONE established** beyond usage/analytics.

## 14. Deployment & scaling

- **FACT:** Multiple Compose variants exist for base, API, data, GPU, AMD GPU, OTel, Playwright, and auxiliary services ([`docker-compose.yaml`](../../repos/open-webui/docker-compose.yaml:1), [`docker-compose.api.yaml`](../../repos/open-webui/docker-compose.api.yaml:1), [`docker-compose.data.yaml`](../../repos/open-webui/docker-compose.data.yaml:1)).
- **FACT:** Database/vector configuration supports SQLite-like local deployment plus PostgreSQL/pgvector, Qdrant, Milvus, and other external backends ([`backend/open_webui/config.py`](../../repos/open-webui/backend/open_webui/config.py:501), [`backend/open_webui/config.py`](../../repos/open-webui/backend/open_webui/config.py:645)).
- **INFERENCE:** A single backend can be deployed simply; multi-replica scaling requires shared DB, Redis/session/socket state, shared file storage, and a production vector backend.
- **UNKNOWN:** Kubernetes/native autoscaling guarantees were not established from the inspected repository files.

## 15. Observability & testing

- **FACT:** OTel compose/config variants and event publication/audit utilities exist ([`docker-compose.otel.yaml`](../../repos/open-webui/docker-compose.otel.yaml:1), [`backend/open_webui/events.py`](../../repos/open-webui/backend/open_webui/events.py:1), [`backend/open_webui/utils/audit.py`](../../repos/open-webui/backend/open_webui/utils/audit.py:1)).
- **FACT:** Python tests exist under [`test`](../../repos/open-webui/test:1), and migration history is extensive ([`backend/open_webui/migrations/versions`](../../repos/open-webui/backend/open_webui/migrations/versions:1)).
- **FACT:** Source-level tests cover auth, memory, knowledge/RAG, tools, routers, and vector operations in relevant modules.
- **INFERENCE:** Core product paths have meaningful testing, but complete coverage and production observability depth were not measured.

## 16. Coupling & extensibility assessment

- Provider coupling: **2-3/5** — OpenAI-compatible config/model records are clear, but main chat route performs substantial orchestration.
- RAG coupling: **2/5** — vector factory, async facade, embedding function factory, and external knowledge adapters are explicit ([`retrieval/vector/factory.py`](../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:1)).
- Memory coupling: **2/5** — dedicated SQL/vector management and routes; project memory is absent as a primary concept.
- Auth/authorization coupling: **3/5** — FastAPI dependencies are explicit, but many routers directly enforce ownership/roles.
- Agent/tool/MCP coupling: **3/5** — functions/tools/MCP are modular, but built-in tools and chat route metadata are integrated.
- **FACT:** Extension mechanisms include functions loaded from URL, tools with valves, pipelines, model wrappers, external knowledge, OpenAI-compatible routes, and MCP client connections ([`routers/functions.py`](../../repos/open-webui/backend/open_webui/routers/functions.py:105), [`utils/mcp/client.py`](../../repos/open-webui/backend/open_webui/utils/mcp/client.py:10)).

## 17. Modification tests table (0-5)

| Test | Score | Evidence-based reason |
|---|---:|---|
| Custom OpenAI-compatible API | 0 | `OPENAI_API_BASE_URL`/key configuration and compatible chat route are already present ([`backend/open_webui/config.py`](../../repos/open-webui/backend/open_webui/config.py:319)). |
| Add another provider | 2 | Provider/config utility boundaries exist, but model discovery, streaming, tools, and access need implementation. |
| Persistent user memory | 1 | Dedicated SQL table, CRUD routes, embeddings, per-user vector collection, search/reindex/reset already exist. |
| Project-specific memory | 3 | Memory is user-only in the inspected schema; needs project/workspace model, authorization, vector naming, and context changes. |
| Context monitoring | 2 | Token counting and usage analytics routes exist, but remaining-capacity display requires chat/UI/provider metadata. |
| User-approved condensation | 2-3 | Chat summary/context-compaction fields/utilities exist, but durable approval UX and policy need implementation. |
| Usage credits | 3 | Usage analytics exist, but no established credit ledger/reservation/billing model was found. |
| Storage quotas | 3 | File ownership and processing exist, but aggregate quota accounting/enforcement is not established. |
| Custom external integration | 1-2 | Functions/tools, OAuth, webhooks, automations, calendar, external knowledge, and API routes exist. |
| Add MCP server/tool | 1 | Native Python MCP client, streamable HTTP, OAuth, and tool integration exist ([`backend/open_webui/utils/mcp/client.py`](../../repos/open-webui/backend/open_webui/utils/mcp/client.py:10)). |
| Independent backend/API in front | 1 | OpenAI-compatible and broad REST APIs are already exposed; auth/API-key forwarding is the main integration concern. |
| Replace frontend | 1 | Backend routes are independently exposed and frontend is separate Svelte code. |
| Connect another candidate as RAG/memory | 1-2 | Vector factory/async facade and per-user memory vector contract provide clean integration points; project-memory semantics remain absent. |

## 18. Evidence log: key file:line citations

- [`FastAPI chat route`](../../repos/open-webui/backend/open_webui/main.py:1085)
- [`OpenAI-compatible models route`](../../repos/open-webui/backend/open_webui/main.py:874)
- [`OpenAI-compatible embeddings`](../../repos/open-webui/backend/open_webui/main.py:1048)
- [`User table`](../../repos/open-webui/backend/open_webui/models/users.py:45)
- [`Chat table`](../../repos/open-webui/backend/open_webui/models/chats.py:129)
- [`Chat file table`](../../repos/open-webui/backend/open_webui/models/chats.py:217)
- [`Memory table`](../../repos/open-webui/backend/open_webui/models/memories.py:15)
- [`Knowledge table`](../../repos/open-webui/backend/open_webui/models/knowledge.py:48)
- [`JWT/API auth`](../../repos/open-webui/backend/open_webui/utils/auth.py:238)
- [`Token source middleware`](../../repos/open-webui/backend/open_webui/utils/asgi_middleware.py:128)
- [`Provider base URL config`](../../repos/open-webui/backend/open_webui/config.py:319)
- [`Vector factory`](../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:1)
- [`RAG indexing`](../../repos/open-webui/backend/open_webui/routers/retrieval.py:1637)
- [`Memory vector search`](../../repos/open-webui/backend/open_webui/routers/memories.py:331)
- [`MCP client`](../../repos/open-webui/backend/open_webui/utils/mcp/client.py:10)
- [`Alembic migrations`](../../repos/open-webui/backend/open_webui/migrations/versions:1)

## 19. Unknowns / could not establish

- **UNKNOWN:** Exact default database engine/value for every deployment variant beyond the config/migration paths inspected.
- **UNKNOWN:** Whether all resource routes enforce group/access grants uniformly; source shows many explicit checks but this was not exhaustively audited.
- **UNKNOWN:** Complete provider invocation chain for every configured upstream and every tool-calling mode.
- **UNKNOWN:** Commercial credit/billing/storage-quota mechanisms; none were established in the inspected source.
- **UNKNOWN:** Production distributed deployment guarantees and external object-storage defaults.
- **UNKNOWN:** Release cadence was only signaled by the repository changelog/migration history, not measured from git tags.
