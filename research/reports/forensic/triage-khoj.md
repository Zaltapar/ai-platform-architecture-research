# Khoj forensic triage

## 1. Verdict snapshot

- **FACT:** Khoj combines Django 5.1 persistence/authentication with FastAPI routers, provider/model configuration, personal knowledge ingestion, agents, long-term user memory, scheduled automation, MCP, search tools, and S3 storage. Evidence: [`pyproject.toml`](../../../repos/khoj/pyproject.toml:1), [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **FACT:** Persistent `UserMemory`, `Conversation`, `FileObject`, `Entry`, `Agent`, and subscription models exist. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1).
- **INFERENCE:** Khoj is a strong personal knowledge/assistant product and reusable user-memory subsystem, but it is less clearly designed as a generic multi-tenant workspace platform than RAGFlow or LobeChat.
- **INFERENCE:** Provider, storage, and external-tool additions are approachable; replacing the Django product/data model or adding project tenancy has higher blast radius.

## 2. Architecture & stack

- **FACT:** Dependencies include FastAPI, Django, pgvector, OpenAI/Anthropic/Google clients, MCP, APScheduler/Django APScheduler, storage integrations, and search/tool providers. Evidence: [`pyproject.toml`](../../../repos/khoj/pyproject.toml:38).
- **FACT:** Configuration combines Django and FastAPI behavior, registers chat, agent, automation, model, memory, content, and Notion routers, and conditionally enables auth/subscription routes. Evidence: [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **INFERENCE:** The architecture is a hybrid product server rather than a pure FastAPI service or library.

## 3. Repo structure

- **FACT:** `src/khoj/database` contains models/adapters; `routers` contains API surfaces; `processor` contains chat/search/tools; `telemetry` contains usage instrumentation; tests cover agents, users, providers, memory, and knowledge bases.
- **FACT:** Separate adapters exist for conversations, files, entries, and user memory. Evidence: [`database/adapters/__init__.py`](../../../repos/khoj/src/khoj/database/adapters/__init__.py:1).
- **INFERENCE:** Adapter boundaries improve reuse, but domain behavior remains anchored to Django models and Khoj service conventions.

## 4. Data model

- **FACT:** Models include `KhojUser`, `ChatModel`, `Agent`, `Conversation`, `FileObject`, `Entry`, `UserMemory`, `Subscription`, and related configuration entities. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1).
- **FACT:** User memory search is associated with a user and optional agent. Evidence: [`database/adapters/__init__.py`](../../../repos/khoj/src/khoj/database/adapters/__init__.py:1).
- **INFERENCE:** The data model is user-centric and supports agent-scoped personalization; a first-class organization/project tenant entity was not established in inspected source.

## 5. AuthN/AuthZ/multi-tenancy

- **FACT:** Middleware supports regular user bearer/session-style authentication, Khoj API user tokens, client applications, and anonymous mode. Evidence: [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **FACT:** Chat, content, and memory routers expose authenticated operations; tests cover agent ownership/privacy and knowledge-base access behavior. Evidence: [`api_chat.py`](../../../repos/khoj/src/khoj/routers/api_chat.py:1), [`api_content.py`](../../../repos/khoj/src/khoj/routers/api_content.py:1), [`api_memories.py`](../../../repos/khoj/src/khoj/routers/api_memories.py:1), [`test_agents.py`](../../../repos/khoj/tests/test_agents.py:1).
- **INFERENCE:** Khoj provides real user/resource authorization and privacy boundaries, but ordinary user accounts should not be reported as proper multi-tenancy. Explicit organization/workspace isolation and cross-user tenant membership were not established.
- **UNKNOWN:** Whether every content/search/background path applies identical user predicates.

## 6. Provider abstraction

- **FACT:** Configuration initializes an OpenAI-compatible chat client using a configurable API base URL. Evidence: [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **FACT:** Chat model/provider models and tests cover multiple providers. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1), [`tests/conftest.py`](../../../repos/khoj/tests/conftest.py:1).
- **INFERENCE:** Provider addition is moderate-to-low effort when following existing model configuration/client contracts; changing the provider registry and persistence model is more coupled.

## 7. Conversation execution path

- **LOGIN FACT:** Configure registers authentication middleware and auth routes when not anonymous. Evidence: [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **CHAT FACT:** Chat router provides history, sessions, sharing/forking, title/feedback operations, WebSocket chat, and POST chat. Evidence: [`api_chat.py`](../../../repos/khoj/src/khoj/routers/api_chat.py:1).
- **CHAT INFERENCE:** Authenticated request → user/agent/conversation resolution → search/memory context through helpers/adapters → configured model/tool execution → persisted conversation/stream response.
- **FACT:** Search helpers include document search and command rate limiting. Evidence: [`helpers.py`](../../../repos/khoj/src/khoj/routers/helpers.py:1).
- **UNKNOWN:** Exact ordering of memory retrieval versus RAG versus tool execution for every agent mode.

## 8. Memory

- **FACT:** `UserMemory` is a persistent model described as long-term memory derived from user-agent conversation. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:855).
- **FACT:** Memory list/update/delete APIs exist and memory adapters scope search to user and optional agent. Evidence: [`api_memories.py`](../../../repos/khoj/src/khoj/routers/api_memories.py:1), [`database/adapters/__init__.py`](../../../repos/khoj/src/khoj/database/adapters/__init__.py:1).
- **INFERENCE:** Persistent user memory is a first-class subsystem; project memory is not clearly first-class because the inspected identity model is user/agent rather than workspace/project.
- **UNKNOWN:** Explicit semantic/episodic taxonomy and user-approved context condensation.

## 9. RAG

- **FACT:** Content APIs support authenticated upload, deletion, listing, size/type, and conversion. Evidence: [`api_content.py`](../../../repos/khoj/src/khoj/routers/api_content.py:1).
- **FACT:** Knowledge-base fixtures and search helpers exist; pgvector is a dependency. Evidence: [`pyproject.toml`](../../../repos/khoj/pyproject.toml:38), [`tests/conftest.py`](../../../repos/khoj/tests/conftest.py:1), [`helpers.py`](../../../repos/khoj/src/khoj/routers/helpers.py:1).
- **INFERENCE:** RAG ingestion/retrieval is integrated with user content and search adapters; replacing the vector/search backend is feasible but not free because entries, processors, and user scoping participate.
- **UNKNOWN:** Complete parser/chunker/embed/index path for every supported content type from inspected anchors.

## 10. Agents/workflows/tools/MCP

- **FACT:** Agent model/API support exists, including ownership/privacy tests and knowledge-base access behavior. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1), [`test_agents.py`](../../../repos/khoj/tests/test_agents.py:1).
- **FACT:** Tools include online search across Exa, Firecrawl, SearXNG, Google, and Serper; code execution supports sandboxed Python and E2B. Evidence: [`online_search.py`](../../../repos/khoj/src/khoj/processor/tools/online_search.py:1), [`run_code.py`](../../../repos/khoj/src/khoj/processor/tools/run_code.py:1).
- **FACT:** MCP is a dependency and router/configuration surface is present. Evidence: [`pyproject.toml`](../../../repos/khoj/pyproject.toml:38), [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **INFERENCE:** Agents/tools are practical product extension points, but workflow graph modularity is less explicit than Langflow/Flowise.

## 11. Files/storage

- **FACT:** File objects are persisted; content routes manage authenticated user content; generated/user images can be uploaded to S3 buckets. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1), [`api_content.py`](../../../repos/khoj/src/khoj/routers/api_content.py:1), [`storage.py`](../../../repos/khoj/src/khoj/routers/storage.py:1).
- **INFERENCE:** Storage replacement is moderate because object records, content processing, quotas/size reporting, and generated assets must remain consistent.
- **UNKNOWN:** Complete quota enforcement across all file and generated-media paths.

## 12. API

- **FACT:** Routers cover `/api`, `/api/chat`, `/api/agents`, `/api/automation`, `/api/model`, `/api/memories`, `/api/content`, and `/api/notion`. Evidence: [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **FACT:** Chat supports WebSocket and POST interfaces; content/memory APIs are authenticated. Evidence: [`api_chat.py`](../../../repos/khoj/src/khoj/routers/api_chat.py:1), [`api_content.py`](../../../repos/khoj/src/khoj/routers/api_content.py:1), [`api_memories.py`](../../../repos/khoj/src/khoj/routers/api_memories.py:1).
- **INFERENCE:** Khoj can sit behind an independent backend, but Django identity/session/API-token semantics must be mapped rather than bypassed casually.

## 13. Background jobs/usage

- **FACT:** APScheduler/Django APScheduler are dependencies and automation routes exist. Evidence: [`pyproject.toml`](../../../repos/khoj/pyproject.toml:38), [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **FACT:** Telemetry stores usage in SQLite and can send events to PostHog. Evidence: [`telemetry.py`](../../../repos/khoj/src/khoj/telemetry/telemetry.py:1).
- **FACT:** Subscription model/routes are conditionally enabled when billing is configured. Evidence: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1), [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **INFERENCE:** Usage accounting and billing seams exist, but a full tenant credit/quota ledger was not established.

## 14. Deployment/scaling

- **FACT:** The hybrid Django/FastAPI stack, scheduler, vector database, storage, and external tools create multiple runtime dependencies.
- **INFERENCE:** Horizontal scaling requires shared database/vector storage/object storage and careful scheduler deduplication; WebSockets and background automation add operational concerns.
- **UNKNOWN:** Complete deployment manifests and replica-safe scheduler strategy.

## 15. Observability/testing

- **FACT:** Telemetry and optional PostHog are implemented; tests create users/providers/agents/memory/knowledge bases and cover privacy behavior. Evidence: [`telemetry.py`](../../../repos/khoj/src/khoj/telemetry/telemetry.py:1), [`conftest.py`](../../../repos/khoj/tests/conftest.py:1), [`test_agents.py`](../../../repos/khoj/tests/test_agents.py:1).
- **INFERENCE:** User/agent authorization has meaningful test coverage, while provider/RAG/tool integration coverage is likely more heterogeneous.

## 16. Coupling/extensibility

- **FACT:** Adapters separate conversations/files/entries/memory; provider and tool modules are named extension areas.
- **INFERENCE:** Adding an external tool/provider is relatively modular. Adding project tenants, replacing Django persistence, or changing identity semantics is coupled across models, adapters, routers, processors, and tests.
- **INFERENCE:** Khoj is forkable as a personal assistant product, but custom platform requirements may create a long-lived product fork.

## 17. 13 modification tests

| Modification | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 1 | Configurable base URL/client exists. |
| Another provider | 2 | Provider/model model and processor paths are explicit. |
| Persistent user memory | 1 | `UserMemory` and APIs/adapters already exist. |
| Project memory | 4 | No established workspace/project identity was found; requires broad model/auth/context work. |
| Context monitoring | 3 | Requires instrumentation across chat, search, memory, tools, and providers. |
| User-approved condensation | 3 | Memory APIs exist, but approval/condensation semantics are not established. |
| Credits | 3 | Subscription/telemetry seams exist, but full usage ledger is not established. |
| Storage quotas | 2 | Content size/listing and file objects provide a base; complete enforcement needs audit. |
| Custom external integration | 2 | Routers, tools, Notion, MCP, and client APIs provide seams. |
| MCP tool/server | 2 | MCP dependency/configuration exists; exact server extension path needs deeper trace. |
| Independent backend/API in front | 2 | FastAPI/Django APIs are explicit, but identity mapping is required. |
| Frontend replacement | 2 | API routes and WebSocket/POST chat support help; product sessions/contracts remain. |
| Connecting to another candidate | 3 | API/tool/provider boundaries are practical; direct model reuse is coupled. |

## 18. Evidence log

- **FACT:** Hybrid application/API setup: [`configure.py`](../../../repos/khoj/src/khoj/configure.py:1).
- **FACT:** Models and user memory: [`models/__init__.py`](../../../repos/khoj/src/khoj/database/models/__init__.py:1).
- **FACT:** Chat API: [`api_chat.py`](../../../repos/khoj/src/khoj/routers/api_chat.py:1).
- **FACT:** Content/memory APIs: [`api_content.py`](../../../repos/khoj/src/khoj/routers/api_content.py:1), [`api_memories.py`](../../../repos/khoj/src/khoj/routers/api_memories.py:1).
- **FACT:** Adapters/privacy tests: [`database/adapters/__init__.py`](../../../repos/khoj/src/khoj/database/adapters/__init__.py:1), [`test_agents.py`](../../../repos/khoj/tests/test_agents.py:1).
- **FACT:** Tools/telemetry: [`online_search.py`](../../../repos/khoj/src/khoj/processor/tools/online_search.py:1), [`telemetry.py`](../../../repos/khoj/src/khoj/telemetry/telemetry.py:1).

## 19. Unknowns

- **UNKNOWN:** First-class workspace/project/organization tenancy.
- **UNKNOWN:** Complete ingestion/chunking/indexing path for all file types.
- **UNKNOWN:** Semantic/episodic memory taxonomy and user-approved condensation.
- **UNKNOWN:** Complete credits, storage quota, and replica-safe scheduler behavior.
- **UNKNOWN:** Exact MCP server/client implementation boundaries.
