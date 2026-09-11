# Langflow — Wave B Deep Forensic Investigation

## 1. Scope/evidence quality

**Scope.** Review limited to [`research/repos/langflow`](../../../repos/langflow:1), materialized `src/backend`, `src/lfx`, `src/bundles`, SDK, tests/deployment assets, and [`triage-langflow.md`](triage-langflow.md:1).

**CRITICAL EVIDENCE LIMITATION — FACT:** The checkout is partial because Windows filename-length limits prevented complete working-tree materialization; Git objects were fetched but unavailable paths cannot be treated as inspected source. The login route, complete migrations/schema inventory and some serving-plane paths therefore remain UNKNOWN.

**FACT:** Available source shows FastAPI backend, `lfx` graph/runtime/component layer, SDK, provider bundles, database services, Redis/cache/queue options, MCP, Sentry and telemetry ([`pyproject.toml`](../../../repos/langflow/pyproject.toml:1), [`main.py`](../../../repos/langflow/src/backend/base/langflow/main.py:1)).

**Evidence quality:** High for graph/build execution, provider bundles, flow/user authorization helpers, memory query scoping, file sandbox/scaling warnings, MCP bootstrap and tests that are materialized. Low/medium for login-to-token and complete database migration paths.

## 2. Architecture/data ownership

**FACT:** `src/backend` owns FastAPI routes, auth helpers, SQLModel/SQLAlchemy services, flows/users/permissions/messages/files/jobs/memory-base services, graph build APIs and worker tasks. `src/lfx` owns newer runtime/component abstractions. `src/bundles` owns provider/tool/vector/memory components. SDK owns client-side API contracts.

**FACT:** Flow graph definitions are database/API artifacts; execution carries flow/user/session context. Build jobs, event streams and checkpoints own runtime state; external Redis/storage/queue are required for reliable multi-worker deployment per preflight warnings.

**INFERENCE:** Langflow is a graph authoring/orchestration platform. It is more reusable as an execution/component engine than as a ready-made tenant product.

## 3. LOGIN trace

**UNKNOWN:** The complete login route/token issuance chain could not be fully traced because of the partial checkout and unavailable paths.

**FACT:** Available route/build helpers receive authenticated user identity and use user-scoped flow queries. [`list_flows()`](../../../repos/langflow/src/backend/base/langflow/helpers/flow.py:60) rejects missing user ID and selects flows with `Flow.user_id == uuid_user_id` at [`flow.py`](../../../repos/langflow/src/backend/base/langflow/helpers/flow.py:65).

**FACT:** Flow-folder queries require both the requested flow and listed flows to belong to the same user at [`flow.py`](../../../repos/langflow/src/backend/base/langflow/helpers/flow.py:91). The triage anchor [`flow.py`](../../../repos/langflow/src/backend/base/langflow/helpers/flow.py:1) also documents caller-supplied user IDs as an IDOR risk requiring server-derived identity.

**FACT:** Agentic file reads resolve per-user sandbox roots and hide unauthorized paths in [`files_router.py`](../../../repos/langflow/src/backend/base/langflow/agentic/api/files_router.py:1).

**INFERENCE:** Available evidence supports auth → server-derived user identity → user/resource-scoped flow selection. It does not prove a universal organization/tenant workspace boundary.

## 4. CHAT trace

**FACT:** Graph execution/build enters the FastAPI build API in [`build.py`](../../../repos/langflow/src/backend/base/langflow/api/build.py:1). It imports graph construction from DB/data, applies tweaks, resolves `Flow`, user/job/task/memory services, streams events, handles cancellation and supports background tasks.

**FACT:** Build API event handling includes graph vertices, streamed messages, output projection, checkpoints and resume logic. Checkpoint restoration deliberately avoids re-running persisted providers/tools and re-runs only dropped opaque producers at [`build.py`](../../../repos/langflow/src/backend/base/langflow/api/build.py:130).

**FACT:** Graph component execution can use model, memory, retriever, vector store, tool and MCP components. Provider bundle example [`OpenAIModelComponent.build_model()`](../../../repos/langflow/src/bundles/openai/src/lfx_openai/components/openai/openai_chat_model.py:98) constructs a LangChain `ChatOpenAI` model with configurable base URL/key and streaming usage.

**FACT:** Core memory retrieval queries messages by flow/session/context/user in [`memory.py`](../../../repos/langflow/src/backend/base/langflow/memory.py:24), including `flow_id` and `user_id` predicates at [`memory.py`](../../../repos/langflow/src/backend/base/langflow/memory.py:35).

**FACT:** Starter RAG is graph-wired through file/splitter/retriever/vector-store nodes in [`vector_store_rag.py`](../../../repos/langflow/src/backend/base/langflow/initial_setup/starter_projects/vector_store_rag.py:1).

**INFERENCE:** Chat-like path is authenticated flow lookup → graph build/execution → component-level memory/RAG/context → model/tool/MCP components → event/SSE output → message/checkpoint/job state. Langflow does not have one central chat service equivalent to Dify; graph topology owns orchestration.

**Async/state:** Build requests are async and can stream NDJSON/SSE-like events; background jobs use worker/task services. Checkpoints serialize graph state but opaque model/tool outputs may be dropped and regenerated.

## 5. FILE trace

**FACT:** Agentic file routes enforce per-user sandbox roots in [`files_router.py`](../../../repos/langflow/src/backend/base/langflow/agentic/api/files_router.py:1). Production preflight requires external storage for replicas in [`preflight.py`](../../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1).

**FACT:** RAG file processing is not one immutable service pipeline: starter flows connect file loader, splitter, retriever and vector-store components in [`vector_store_rag.py`](../../../repos/langflow/src/backend/base/langflow/initial_setup/starter_projects/vector_store_rag.py:1). Bundles include document/chunk/vector components; available tests show component-level chunk configuration such as [`chunk_docling_document.py`](../../../repos/langflow/src/bundles/docling/src/lfx_docling/components/docling/chunk_docling_document.py:122).

**INFERENCE:** File flow is flow-dependent: upload/file component → parser/loader → splitter/chunker → embedding component → vector store → retriever node → graph context. There is no evidence of a single centralized tenant-wide indexing pipeline.

**UNKNOWN:** Complete storage upload route, parser dispatch, index lifecycle and tenant scope for every supported vector backend due to partial checkout and graph variability.

## 6. API trace

**FACT:** FastAPI build/execution endpoints support flow IDs, execution payloads, streaming, cancellation, background build jobs and event retrieval; test helpers call `api/v1/build/{flow_id}/flow` and `/events` in [`build_utils.py`](../../../repos/langflow/src/backend/tests/unit/build_utils.py:17).

**FACT:** SDK exposes flow-run contracts and tests use `/api/v1/run/{flow_id}` in [`test_testing.py`](../../../repos/langflow/src/sdk/tests/test_testing.py:277).

**FACT:** MCP routers/services are registered during application startup in [`main.py`](../../../repos/langflow/src/backend/base/langflow/main.py:1); preflight validates MCP/security posture in [`preflight.py`](../../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1).

**INFERENCE:** API request → auth/user/flow authorization → graph build/execution → component model/tool/vector calls → job/event/checkpoint state → streamed/JSON response. An independent backend can front these endpoints but must preserve flow IDs, user identity, event formats, cancellation and checkpoint semantics.

## 7. Memory/context

**FACT:** Memory query builder filters by sender/session/context/flow/user and rejects invalid ordering fields in [`memory.py`](../../../repos/langflow/src/backend/base/langflow/memory.py:24). Default flow context is derived from current executing graph at [`memory.py`](../../../repos/langflow/src/backend/base/langflow/memory.py:141), preventing history leakage on colliding session IDs.

**FACT:** Starter memory graph exists in [`memory_chatbot.py`](../../../repos/langflow/src/backend/base/langflow/initial_setup/starter_projects/memory_chatbot.py:1). Provider bundles include persistent external memory options such as Valkey chat memory tested in [`test_valkey_chat.py`](../../../repos/langflow/src/bundles/valkey/tests/test_valkey_chat.py:45).

**FACT:** Agentic assistant conversation buffer is in-memory/restart-volatile in [`conversation_buffer.py`](../../../repos/langflow/src/backend/base/langflow/agentic/services/conversation_buffer.py:1).

**INFERENCE:** Langflow memory is graph/component-selected and can be user/flow/session scoped; it is not one centralized semantic/episodic/project memory service. Project memory requires explicit identity propagation through graph inputs, memory components and vector stores.

**UNKNOWN:** Global token accounting, user-facing context condensation and approval workflow. Component diversity makes a universal policy nontrivial.

## 8. RAG

**FACT:** RAG is assembled from graph nodes; starter vector RAG explicitly wires file, splitter, retriever and vector store in [`vector_store_rag.py`](../../../repos/langflow/src/backend/base/langflow/initial_setup/starter_projects/vector_store_rag.py:1).

**FACT:** Provider/vector/memory bundles are independently packaged, and component outputs/inputs are graph contracts.

**INFERENCE:** RAG replacement is easy when replacing a graph component or vector provider; replacing all RAG semantics is more involved because saved flows encode component types, parameters, serialization and output shapes. Tenant isolation depends on graph/component configuration rather than one global index policy.

## 9. Agents/workflows/tools/MCP

**FACT:** Langflow’s workflow is the graph itself. Build execution supports tools, agentic services, streaming, cancellation, durable checkpoints and background jobs in [`build.py`](../../../repos/langflow/src/backend/base/langflow/api/build.py:1).

**FACT:** Provider/tool/vector/memory bundles are explicit workspace/package seams. MCP services initialize/register at startup in [`main.py`](../../../repos/langflow/src/backend/base/langflow/main.py:1), with security posture checks in [`preflight.py`](../../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1).

**INFERENCE:** Node/component extension is Langflow’s strongest capability. MCP is serving-plane integrated and operationally coupled to auth, SSE/eventing, security and worker topology.

## 10. Auth/multi-tenancy

**FACT:** Available helpers consistently use server-derived user IDs for flow selection and message memory predicates; agent files use per-user sandbox roots.

**INFERENCE:** Resource isolation is real at user/flow/file level, but this evidence does not establish a complete organization/tenant hierarchy or universal cross-service partitioning.

**UNKNOWN:** Workspace/organization isolation across all APIs, vector indexes, job queues, checkpoints, agents and MCP artifacts. This is a mandatory Wave C audit item.

## 11. Storage/jobs/usage

**FACT:** Worker task logic exists in [`worker.py`](../../../repos/langflow/src/backend/base/langflow/worker.py:1), and build APIs can schedule background tasks. Preflight warns that in-memory queues do not support multi-worker UI/MCP event delivery and in-memory cache is not shared across pods in [`preflight.py`](../../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1) and [`__main__.py`](../../../repos/langflow/src/backend/base/langflow/__main__.py:1).

**FACT:** Production checks cover database, storage, encryption secret, pgVector, MCP, cache and queue posture in [`preflight.py`](../../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1).

**UNKNOWN:** Built-in credits/billing ledger, complete provider-token accounting, aggregate storage quota policy, and production autoscaling guarantees.

**INFERENCE:** Single-worker/local storage is suitable for development; reliable multi-replica deployment requires external storage, Redis/shared cache, queue and durable event/checkpoint state.

## 12. Interface/coupling analysis

- **Provider:** bundles/components and `LCModelComponent` boundaries are explicit; adding providers is highly replaceable.
- **RAG:** graph component boundary is explicit; saved-flow serialization and output types couple replacements.
- **Memory:** component-level and database helper contracts are explicit, but no universal memory service; project memory is a propagation problem.
- **Agent/workflow:** graph runtime/checkpoints/eventing are central and hard to replace without rewriting execution semantics.
- **Auth/persistence:** route/service identity and flow ownership are distributed; full replacement is invasive.
- **Frontend/API:** SDK/FastAPI separation enables replacement, but event/flow/build contracts are extensive.

## 13. Modification tests

| Test | Score | Evidence-based assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 1 | OpenAI bundle accepts editable base URL/key and SSRF-safe client construction. |
| Another provider | 1 | Provider bundles are explicit package/component seams. |
| Persistent user memory | 2 | User/flow memory exists; policy and durable component selection must be integrated. |
| Project-specific memory | 3 | Project identity must propagate through flows, components, storage and retrieval. |
| Context monitoring | 2 | Build/telemetry boundaries exist; component diversity adds instrumentation work. |
| User-approved condensation | 3 | Requires shared context policy, persisted approval state and UI/API changes. |
| Usage credits | 4 | No established billing ledger; cross-cuts execution, users, jobs and providers. |
| Storage quotas | 3 | File/storage boundaries exist; enforcement must cover local/external and graph paths. |
| Custom external integration | 1 | Components, tools, SDK and API are direct seams. |
| MCP server/tool | 1 | MCP is integrated, subject to security and worker topology. |
| Independent backend/API in front | 2 | FastAPI/SDK boundary is usable; auth/flow/stream context must be preserved. |
| Frontend replacement | 2 | API/SDK separation helps; authoring/runtime contracts remain. |
| Connect another candidate | 3 | Best through API/provider/vector/tool boundaries, not shared internals. |

## 14. Upgrade/forkability

**FACT:** Compatibility aliases map older `langflow` imports to `lfx` paths in [`langflow/__init__.py`](../../../repos/langflow/src/backend/base/langflow/__init__.py:1).

**INFERENCE:** The alias/migration layer reduces immediate breakage but increases long-term custom-component and fork maintenance risk. Provider/component additions can stay modular; changing graph serialization, runtime checkpoints, event formats, auth or persistence creates substantial divergence.

**CRITICAL UNKNOWN:** Partial checkout prevents complete migration/test/deployment audit; any fork decision requires a Linux/full-checkout independent audit first.

## 15. Risks/unknowns

- **FACT:** Partial Windows checkout blocks complete source verification.
- **UNKNOWN:** Full login/token chain and migrations/schema inventory.
- **UNKNOWN:** Universal tenant/workspace isolation across queues, vectors, jobs and agents.
- **UNKNOWN:** Global context token/condensation policy and billing/credits.
- **RISK/INFERENCE:** In-memory queue/cache plus opaque checkpoint outputs can produce replica divergence, duplicate billing or repeated side effects if external state is not configured.

## 16. Evidence index

Key anchors: [`build.py`](../../../repos/langflow/src/backend/base/langflow/api/build.py:1), [`flow.py`](../../../repos/langflow/src/backend/base/langflow/helpers/flow.py:60), [`memory.py`](../../../repos/langflow/src/backend/base/langflow/memory.py:24), [`vector_store_rag.py`](../../../repos/langflow/src/backend/base/langflow/initial_setup/starter_projects/vector_store_rag.py:1), [`openai_chat_model.py`](../../../repos/langflow/src/bundles/openai/src/lfx_openai/components/openai/openai_chat_model.py:98), [`files_router.py`](../../../repos/langflow/src/backend/base/langflow/agentic/api/files_router.py:1), [`worker.py`](../../../repos/langflow/src/backend/base/langflow/worker.py:1), [`preflight.py`](../../../repos/langflow/src/backend/base/langflow/cli/preflight.py:1), [`main.py`](../../../repos/langflow/src/backend/base/langflow/main.py:1), [`build_utils.py`](../../../repos/langflow/src/backend/tests/unit/build_utils.py:17), and [`test_valkey_chat.py`](../../../repos/langflow/src/bundles/valkey/tests/test_valkey_chat.py:45).
