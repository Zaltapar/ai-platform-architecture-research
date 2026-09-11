# Open WebUI — Wave B Deep Forensic Investigation

## 1. Scope/evidence quality

**Scope.** Review limited to [`research/repos/open-webui`](../../../repos/open-webui:1), FastAPI backend, Svelte frontend, SQLAlchemy/Alembic models/migrations, retrieval/vector code, tests/deployment files, and [`triage-open-webui.md`](triage-open-webui.md:1).

**FACT:** Open WebUI is a Python FastAPI backend with Svelte frontend, SQLAlchemy persistence, configurable vector backends, optional Redis/session/socket infrastructure, function/tool plugins, and native MCP client support ([`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1), [`package.json`](../../../repos/open-webui/package.json:1)).

**Evidence quality:** High for chat entry/model access, files/RAG indexing, user memory, vector abstraction, auth sources, MCP and API routes. Medium for complete provider-specific call chains and distributed deployment behavior.

## 2. Architecture/data ownership

**FACT:** SQLAlchemy tables and repository helpers are under [`backend/open_webui/models`](../../../repos/open-webui/backend/open_webui/models:1), with Alembic history under [`migrations/versions`](../../../repos/open-webui/backend/open_webui/migrations/versions:1). Core entities include users, chats/messages, files/chat files, knowledge bases/access grants, memories, models, tools/functions, groups and automations.

**FACT:** FastAPI route handlers and middleware own authentication, model access, chat orchestration, file processing, memory routes and vector calls. Svelte owns UI state. SQL database owns metadata/message/memory state; selected vector client owns embeddings/chunks/search; Storage owns uploaded bytes.

**INFERENCE:** This is a modular monolith with optional external vector/cache/LLM services. The dominant authorization boundary is user/resource access rather than a universal tenant/workspace database partition.

## 3. LOGIN trace

**FACT:** Tokens are resolved from bearer authorization, cookie or configurable API-key header by [`asgi_middleware.py`](../../../repos/open-webui/backend/open_webui/utils/asgi_middleware.py:128) and auth utilities at [`auth.py`](../../../repos/open-webui/backend/open_webui/utils/auth.py:238).

**FACT:** JWT verification uses the configured secret/HS256 and verified/admin dependencies gate routes in [`auth.py`](../../../repos/open-webui/backend/open_webui/utils/auth.py:47) and [`auth.py`](../../../repos/open-webui/backend/open_webui/utils/auth.py:522). Local, LDAP and OAuth sign-in routes are under [`auths.py`](../../../repos/open-webui/backend/open_webui/routers/auths.py:716).

**FACT:** Core resource ownership is direct: chats have `user_id` in [`chats.py`](../../../repos/open-webui/backend/open_webui/models/chats.py:129), memories have `user_id` in [`memories.py`](../../../repos/open-webui/backend/open_webui/models/memories.py:15), and route handlers use verified-user dependencies and ownership/access checks.

**INFERENCE:** Login → verified user → route-specific owner/group/access check → resource selection. Groups/access grants enable collaboration but are not equivalent to a universal tenant/workspace selector.

## 4. CHAT trace

**FACT:** Chat requests enter `/api/chat/completions` or `/api/v1/chat/completions` at [`chat_completion()`](../../../repos/open-webui/backend/open_webui/main.py:1085), requiring `get_verified_user` at [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1090).

**FACT:** The route resolves the requested model, model metadata/access, fallback model, params, parent/chat/message IDs, chat variables, files, tool servers, automation and approval mode in [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1095) through [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1259).

**FACT:** Provider usage is requested in streaming mode with `include_usage` when model capability advertises usage at [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1173). Chat/message persistence is represented by [`Chat`](../../../repos/open-webui/backend/open_webui/models/chats.py:129), [`ChatFile`](../../../repos/open-webui/backend/open_webui/models/chats.py:217), and [`ChatMessage`](../../../repos/open-webui/backend/open_webui/models/chat_messages.py:127).

**FACT:** Retrieval context uses embedding/vector utilities in [`retrieval/utils.py`](../../../repos/open-webui/backend/open_webui/retrieval/utils.py:322), and built-in tools can query knowledge and user-memory collections in [`builtin.py`](../../../repos/open-webui/backend/open_webui/tools/builtin.py:3381). Tool/MCP resolution and access checks are in [`middleware.py`](../../../repos/open-webui/backend/open_webui/utils/middleware.py:2320).

**INFERENCE:** Route → verified user/model access → chat/file/variable/tool metadata → memory/RAG helpers → provider streaming → built-in/function/MCP tool execution → SQL/event persistence → frontend SSE/event response. Significant orchestration remains in `main.py` and middleware rather than one application service.

**Async/state:** FastAPI handlers are async; synchronous vector operations are wrapped by [`async_client.py`](../../../repos/open-webui/backend/open_webui/retrieval/vector/async_client.py:1) and file indexing is offloaded with `run_in_threadpool` at [`retrieval.py`](../../../repos/open-webui/backend/open_webui/routers/retrieval.py:2154). Background tasks/automations and websocket/session infrastructure are optional.

## 5. FILE trace

**FACT:** Upload enters [`upload_file()`](../../../repos/open-webui/backend/open_webui/routers/files.py:271), delegates to `upload_file_handler` at [`files.py`](../../../repos/open-webui/backend/open_webui/routers/files.py:315), validates extension/size, assigns UUID and owner tags, then calls `Storage.upload_file` through a thread at [`files.py`](../../../repos/open-webui/backend/open_webui/routers/files.py:355).

**FACT:** File metadata is inserted after storage in the same handler; the upload path exposes optional processing/background processing parameters. File deletion removes Storage bytes and vector records/collections through [`files.py`](../../../repos/open-webui/backend/open_webui/routers/files.py:1019).

**FACT:** Retrieval ingestion is `save_docs_to_vector_db()` in [`retrieval.py`](../../../repos/open-webui/backend/open_webui/routers/retrieval.py:1637). It performs duplicate hash checks, markdown/header or recursive/token/transformer splitting at [`retrieval.py`](../../../repos/open-webui/backend/open_webui/routers/retrieval.py:1687), creates embeddings using configured engine/base/key at [`retrieval.py`](../../../repos/open-webui/backend/open_webui/routers/retrieval.py:1778), then inserts vectors and metadata at [`retrieval.py`](../../../repos/open-webui/backend/open_webui/routers/retrieval.py:1824).

**INFERENCE:** File flow is request → Storage + SQL metadata → loader/parser → splitter → embedding function → selected vector collection → retrieval/search. The handler and indexing path cross synchronous threadpool boundaries; vector ownership is externalized behind a client façade.

## 6. API trace

**FACT:** OpenAI-compatible chat, model, embedding, Anthropic-style message/token-count, usage and analytics routes are in [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:874), [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1048), [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1085), and [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1974).

**FACT:** REST routers cover auth, chats/messages, files, knowledge/retrieval, memories, models, tools/functions, groups/users, automations, plugins, audio/images and integrations under [`routers`](../../../repos/open-webui/backend/open_webui/routers:1). API-key input is accepted by middleware/config at [`asgi_middleware.py`](../../../repos/open-webui/backend/open_webui/utils/asgi_middleware.py:128).

**INFERENCE:** API request → token source resolution → verified-user/admin/resource access → route orchestration → model/tool/vector execution → SQL/vector/event persistence → JSON/SSE response. Public OpenAI-compatible routes share the same backend and do not represent a separate independent service plane.

## 7. Memory/context

**FACT:** Persistent user memory is a SQL table with type/path/content/metadata and ownership in [`memories.py`](../../../repos/open-webui/backend/open_webui/models/memories.py:15). Routes create/update/search/reindex/reset both SQL and vector state; memory search uses collection `user-memory-{user.id}` at [`memories.py`](../../../repos/open-webui/backend/open_webui/routers/memories.py:331).

**FACT:** Memory has `user` and `context` types but the inspected model has no project/workspace foreign key. Context compaction-related parameters exist in chat configuration, but a complete user-approved condensation lifecycle was not established.

**FACT:** Token/usage information is requested from providers and exposed through analytics/token routes at [`analytics.py`](../../../repos/open-webui/backend/open_webui/routers/analytics.py:235).

**INFERENCE:** Personal memory is real and replaceable at the SQL/vector boundary. Project memory requires new ownership, access, vector naming, and context assembly semantics. Remaining-context monitoring would need route/provider/UI integration.

## 8. RAG

**FACT:** Knowledge bases own SQL records and file links in [`knowledge.py`](../../../repos/open-webui/backend/open_webui/models/knowledge.py:48); access grants are checked by knowledge routes/helpers. Indexing is centralized in [`save_docs_to_vector_db()`](../../../repos/open-webui/backend/open_webui/routers/retrieval.py:1637).

**FACT:** Vector backend selection is centralized in [`factory.py`](../../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:90), with the async façade in [`async_client.py`](../../../repos/open-webui/backend/open_webui/retrieval/vector/async_client.py:1). Multiple vector backends and external knowledge connectors are supported.

**INFERENCE:** RAG/vector replacement is one of Open WebUI’s strongest seams. Parser, chunk metadata, embedding configuration, collection naming, and retrieval response shape remain coupled to route/tool code.

## 9. Agents/workflows/tools/MCP

**FACT:** Persisted functions/tools are loaded by URL and can have global/user valves through [`functions.py`](../../../repos/open-webui/backend/open_webui/routers/functions.py:46). Built-in tools cover web, RAG, memory, image and file operations ([`builtin.py`](../../../repos/open-webui/backend/open_webui/tools/builtin.py:2564)).

**FACT:** MCP client uses the Python SDK, streamable HTTP, OAuth client/token storage and connection lifecycle in [`mcp/client.py`](../../../repos/open-webui/backend/open_webui/utils/mcp/client.py:59). Middleware resolves server access and connects tools at [`middleware.py`](../../../repos/open-webui/backend/open_webui/utils/middleware.py:2320).

**FACT:** Automations and scheduled runs are persisted/routes in [`automations.py`](../../../repos/open-webui/backend/open_webui/routers/automations.py:162).

**INFERENCE:** Tool/function/MCP extension is modular, but agent behavior is more tool/function-oriented and route-integrated than a separate durable workflow runtime.

## 10. Auth/multi-tenancy

**FACT:** User/resource ownership is explicit for chats, files, memories and knowledge access; groups/access grants add collaboration ([`groups.py`](../../../repos/open-webui/backend/open_webui/models/groups.py:37), [`access_grants.py`](../../../repos/open-webui/backend/open_webui/models/access_grants.py:1)).

**INFERENCE:** Open WebUI has strong user/resource authorization but no demonstrated universal `tenant_id` boundary equivalent to LibreChat’s plugin or RAGFlow’s tenant schema. A future workspace/project layer needs systematic schema and route changes.

**UNKNOWN:** Uniform enforcement of all group/access grants across every router and production database isolation behavior were not exhaustively audited.

## 11. Storage/jobs/usage

**FACT:** Redis session/socket infrastructure and websocket session pools exist at [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:2615) and [`socket/main.py`](../../../repos/open-webui/backend/open_webui/socket/main.py:143). Chat tasks and automations have control/run records at [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:2101) and [`automations.py`](../../../repos/open-webui/backend/open_webui/routers/automations.py:334).

**FACT:** Provider usage and analytics/token endpoints exist, but a commercial credit ledger/reservation/commit mechanism and first-class storage quota system were not established.

**INFERENCE:** Single-node deployment is simple; replicas require shared SQL, Redis/session/socket state, shared file storage and a production vector service. OTel/audit/event seams exist at [`events.py`](../../../repos/open-webui/backend/open_webui/events.py:1) and [`audit.py`](../../../repos/open-webui/backend/open_webui/utils/audit.py:1).

## 12. Interface/coupling analysis

- **Provider:** configurable model records and OpenAI-compatible base URLs are explicit in [`config.py`](../../../repos/open-webui/backend/open_webui/config.py:319); main chat route still owns substantial orchestration. Replaceability: moderate.
- **RAG:** vector factory/async façade/embedding function are real seams. Replaceability: relatively high.
- **Memory:** dedicated SQL/vector API; project semantics absent. Replaceability: moderate.
- **Auth:** FastAPI dependencies are explicit, but ownership/role checks are distributed. Replacing auth is invasive.
- **Tools/MCP:** explicit client/function routes, but metadata and middleware injection couple them to chat.
- **Frontend:** backend API is independently callable and Svelte is separate; SSE/event contracts remain product-specific.

## 13. Modification tests

| Test | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 0 | Base URL/key configuration already exists in [`config.py`](../../../repos/open-webui/backend/open_webui/config.py:319). |
| Another provider | 2 | Model/provider utilities exist, but discovery, streaming, tools, embeddings and usage need alignment. |
| Persistent user memory | 1 | SQL/vector CRUD, per-user collection, search/reindex/reset already exist. |
| Project-specific memory | 3 | Requires project/workspace schema, access, vector naming, routes and context assembly. |
| Context monitoring | 2 | Usage/token routes exist; remaining capacity requires provider/model metadata and UI. |
| User-approved condensation | 2-3 | Compaction fields/utilities exist, but durable proposal/approval state is additional. |
| Usage credits | 3 | Analytics exist; no established credit ledger/reservation path was found. |
| Storage quotas | 3 | File size/ownership processing exists; aggregate quota accounting/enforcement is absent. |
| Custom external integration | 1-2 | Functions/tools/OAuth/automations/webhooks and routes provide seams. |
| MCP server/tool | 1 | Native MCP client/OAuth/tool middleware exists. |
| Independent backend/API in front | 1 | OpenAI-compatible and broad REST surfaces already exist; auth forwarding is key. |
| Frontend replacement | 1 | Separate backend and Svelte client; preserve streaming/event contracts. |
| Connect another candidate | 1-2 | Vector and user-memory boundaries are practical; workspace semantics are missing. |

## 14. Upgrade/forkability

**INFERENCE:** Open WebUI is relatively forkable for frontend replacement, OpenAI-compatible upstreams, vector backends, functions and MCP. A fork that introduces universal tenancy, project memory, credits or storage quotas will touch many SQL models, dependencies, routers, middleware and frontend state.

**FACT:** Extensive Alembic history, tests, compose variants and plugin/tool surfaces improve maintainability, but the large central `main.py`/middleware orchestration increases merge conflict risk.

## 15. Risks/unknowns

- **UNKNOWN:** Complete provider call chain for every model/tool mode.
- **UNKNOWN:** Commercial credits, billing, storage quotas and robust tenant partitioning.
- **UNKNOWN:** Production object-storage defaults and distributed socket/session guarantees.
- **RISK/INFERENCE:** User-scoped defaults can cause security/design gaps if a future workspace boundary is added inconsistently.

## 16. Evidence index

Key anchors: [`auth.py`](../../../repos/open-webui/backend/open_webui/utils/auth.py:238), [`asgi_middleware.py`](../../../repos/open-webui/backend/open_webui/utils/asgi_middleware.py:128), [`main.py`](../../../repos/open-webui/backend/open_webui/main.py:1085), [`files.py`](../../../repos/open-webui/backend/open_webui/routers/files.py:271), [`retrieval.py`](../../../repos/open-webui/backend/open_webui/routers/retrieval.py:1637), [`factory.py`](../../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:90), [`memories.py`](../../../repos/open-webui/backend/open_webui/routers/memories.py:156), [`builtin.py`](../../../repos/open-webui/backend/open_webui/tools/builtin.py:3381), [`middleware.py`](../../../repos/open-webui/backend/open_webui/utils/middleware.py:2320), [`chats.py`](../../../repos/open-webui/backend/open_webui/models/chats.py:129), and [`migrations/versions`](../../../repos/open-webui/backend/open_webui/migrations/versions:1).
