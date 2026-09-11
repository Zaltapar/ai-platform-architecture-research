# Mem0 — memory and agent-state subsystem

## Selection rationale

**FACT:** Mem0 was selected because it has a source-visible memory lifecycle rather than merely storing chat history: add/extract, semantic search, get/list, update, history, delete, expiration, optional graph memory, and provider/vector-store factories. Its core implementation is [`Memory`](../../../repos/mem0/mem0/memory/main.py:17), and its HTTP server exposes the lifecycle directly in [`server/main.py`](../../../repos/mem0/server/main.py:367). **INFERENCE:** It is the strongest candidate for evaluating a replaceable long-term memory subsystem.

## Architecture and stack

**FACT:** The core is Python and supports synchronous/asynchronous memory APIs, configurable LLM, embedder, reranker, vector store, graph memory, SQLite-backed session/message state, and telemetry imports in [`memory/main.py`](../../../repos/mem0/mem0/memory/main.py:1). **FACT:** The optional server is FastAPI with Pydantic request models, JWT/API-key authentication, SQLAlchemy request/auth persistence, and a process-level configured memory instance in [`server/main.py`](../../../repos/mem0/server/main.py:160). **INFERENCE:** Mem0 can be used embedded as a library or deployed as a dedicated memory API, but those modes have different isolation and scaling properties.

## Repository structure

**FACT:** [`mem0`](../../../repos/mem0/mem0:1) contains core memory/config/factory/vector-store code; [`server`](../../../repos/mem0/server:1) contains the FastAPI service, auth, models, routers, and Alembic migrations; [`tests`](../../../repos/mem0/tests:1) contains memory, provider, security, and integration tests; [`integrations`](../../../repos/mem0/integrations:1) contains adapters/plugins; and [`examples`](../../../repos/mem0/examples:1) demonstrates agent-framework use.

## Core data model

**FACT:** Memory records are vector-store payloads containing data, identity dimensions (`user_id`, `agent_id`, `run_id`), hash/timestamps/expiration, and arbitrary non-reserved metadata; serialization is implemented in [`server/main.py`](../../../repos/mem0/server/main.py:385). **FACT:** The core rejects missing identity scope for creation and protects identity fields from arbitrary metadata in [`memory/main.py`](../../../repos/mem0/mem0/memory/main.py:134). **FACT:** The service also persists users, refresh-token JTIs, API keys, and request logs through server models/migrations; refresh-token persistence is visible in [`server/auth.py`](../../../repos/mem0/server/auth.py:56) and [`server/alembic/versions/005_refresh_token_jtis.py`](../../../repos/mem0/server/alembic/versions/005_refresh_token_jtis.py:24). **INFERENCE:** The memory data plane is primarily vector-store payloads, while server auth/audit-like data is relational.

## API/interface boundary

**FACT:** The HTTP contract includes configuration/provider discovery, create/list/get/search/update/history/delete/delete-all/reset in [`server/main.py`](../../../repos/mem0/server/main.py:322). Create requires at least one of user, agent, or run identity in [`server/main.py`](../../../repos/mem0/server/main.py:367). Search accepts structured filters, top-k, threshold, expiry visibility, and optional explanations in [`server/main.py`](../../../repos/mem0/server/main.py:452). **FACT:** The core factories expose provider registration/configuration; `LlmFactory.register_provider` mutates a provider map in [`factory.py`](../../../repos/mem0/mem0/utils/factory.py:35).

## Deployment and persistence

**FACT:** The server uses a configured memory instance and persists request/auth records through SQLAlchemy, while memory vectors are delegated to the selected vector store in [`server/main.py`](../../../repos/mem0/server/main.py:271). **FACT:** The project supports many vector-store implementations through [`vector_stores`](../../../repos/mem0/mem0/vector_stores:1), and provider selection is explicit in [`factory.py`](../../../repos/mem0/mem0/utils/factory.py:41). **INFERENCE:** Production deployment requires a durable vector backend plus compatible embedding/LLM services; the server’s relational database is not itself the semantic memory store.

## Security and multi-tenancy assumptions

**FACT:** Server authentication produces JWTs with subject and role in [`server/auth.py`](../../../repos/mem0/server/auth.py:56), and admin-only operations include configuration, unscoped listing, delete-all, and reset in [`server/main.py`](../../../repos/mem0/server/main.py:411). **FACT:** Memory scoping is implemented using caller-supplied user/agent/run filters and identity-payload protection in [`memory/main.py`](../../../repos/mem0/mem0/memory/main.py:143). **INFERENCE:** Mem0 provides scoped memory semantics, not a complete organization/workspace authorization authority. **UNKNOWN:** The inspected source did not establish a universal platform-tenant object, membership/role model, per-tenant key hierarchy, or database row-level isolation across every vector backend.

## Extension/plugin model

**FACT:** LLM, embedder, reranker, and vector-store factories map provider names to importable classes in [`factory.py`](../../../repos/mem0/mem0/utils/factory.py:35). New LLM providers can be registered by name/class path/config class in [`factory.py`](../../../repos/mem0/mem0/utils/factory.py:127). **FACT:** Tests show multiple vector-store adapters and framework integrations under [`integrations`](../../../repos/mem0/integrations:1). **INFERENCE:** Provider/vector replacement is a real adapter mechanism; replacing memory extraction policy or scope semantics is more invasive because it is encoded in core `Memory` behavior and API validation.

## Failure, retry, and background behavior

**FACT:** Add can invoke LLM extraction and vector persistence synchronously through the core memory path in [`memory/main.py`](../../../repos/mem0/mem0/memory/main.py:1); API exceptions are mapped to client/upstream errors in [`server/main.py`](../../../repos/mem0/server/main.py:214). **FACT:** Request logging is dispatched to an executor in [`server/main.py`](../../../repos/mem0/server/main.py:292). **FACT:** Memory history and delete/update are explicit lifecycle methods in [`server/main.py`](../../../repos/mem0/server/main.py:487). **UNKNOWN:** The inspected code did not establish a durable job queue for large-scale asynchronous ingestion or guaranteed transactional coordination across LLM extraction and vector writes.

## Observability and testing

**FACT:** The repository contains tests for memory lifecycle, scoping, metadata identity protection, temporal/decay behavior, performance notices, integrations, and server behavior; identity injection/update cases are covered in [`tests/memory/test_main.py`](../../../repos/mem0/tests/memory/test_main.py:421). **FACT:** Core telemetry is imported and request logs are persisted by the server in [`memory/main.py`](../../../repos/mem0/mem0/memory/main.py:51) and [`server/main.py`](../../../repos/mem0/server/main.py:271). **UNKNOWN:** End-to-end multi-replica consistency and vector-backend failure recovery were not executed in this wave.

## Upgradeability and coupling

**FACT:** Provider/vector adapters are separated by factories, but core memory logic directly orchestrates extraction, parsing, entity handling, scoring, session storage, and vector-store operations in [`memory/main.py`](../../../repos/mem0/mem0/memory/main.py:61). **INFERENCE:** Embedding/vector backend substitution is low-to-moderate coupling; replacing the memory ontology, extraction prompts, or scope model is high coupling. The optional server’s singleton-style configuration and request paths create additional service-level coupling.

## Integration level and fit

- **Likely integration level:** **2 — SDK/library** for in-process memory; **1 — HTTP/API** for a dedicated memory service; **3 — shared infrastructure/database** if the platform adopts Mem0’s vector collections or auth records as canonical.
- **Fit:** Strong replaceable subsystem behind a platform-owned `MemoryProvider` interface. The platform should own tenant/project IDs, consent, retention policy, deletion orchestration, and authorization, translating them into Mem0 filters and immutable identity fields.
- **Not provided:** **FACT/UNKNOWN:** Mem0 does not provide chat/conversation UI, workflow durability, general file/document ingestion, commercial billing, storage quota policy, or a complete workspace membership model. It is not a general agent runtime, even though `agent_id` and `run_id` scope memory.

## Modification difficulty experiments

| Experiment | Score | Reason |
|---|---:|---|
| Persistent user memory | 0–1 | Core add/search/update/delete/history lifecycle already exists. |
| Project-specific memory | 1–2 | Map project to `agent_id`/metadata, but platform authorization must remain external. |
| Add another vector store | 1–2 | Factory and `VectorStoreBase` adapter seam. |
| Add another LLM/embedder | 1–2 | Factory/config/provider class; custom extraction semantics may add tests. |
| User-approved condensation | 3 | No general approval workflow; needs platform state and context integration. |
| Cross-tenant isolation | 3–4 | Caller filters are not equivalent to a fail-closed tenant boundary; requires wrapper/API/auth enforcement. |
| Replace vector persistence model | 3 | Core payload/search/update/history semantics and all adapters are affected. |

## Bottom line

**INFERENCE:** Mem0 is highly reusable as a memory engine, especially when the target platform owns the policy envelope. Do not expose raw caller-controlled filters as the security boundary; derive them from authenticated platform context and keep Mem0 replaceable behind an adapter.
