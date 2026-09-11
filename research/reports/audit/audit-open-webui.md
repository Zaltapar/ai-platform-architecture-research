# Wave C Independent Audit — Open WebUI

**Auditor posture:** independent, skeptical. All claims independently re-verified against `research/repos/open-webui/` source. Labels: **FACT** (direct source evidence), **INFERENCE** (auditor reasoning), **UNKNOWN** (not verifiable in-repo).

---

## 1. Scope & Method

Independently inspected: auth/role model, access-grant system (`utils/access_control/`), vector factory and multitenancy modes, storage providers, MCP client, memory subsystem, `main.py` route surface, deployment files.

---

## 2. Verified Architectural Facts

### 2.1 No tenancy model — user/group/role only (FACT)

- Users have roles `pending / user / admin` (prior verified read of [users.py:45](research/repos/open-webui/backend/open_webui/models/users.py:45)); grouping is via Groups + per-resource **AccessGrants** (user or group grants with `read/write` permissions), not tenants.
- There is no `tenant_id` anywhere in the data model; "workspace" concepts (shared models, shared channels, shared chats) are **sharing mechanisms on a single-user-plane database**, not isolation boundaries.
- **INFERENCE:** The deep report's "no demonstrated universal tenant_id boundary" is **CONFIRMED**. The closest thing to multi-tenancy is AccessGrants + groups, which give per-resource ACLs *within one trust domain*. For a commercial multi-customer platform this is the wrong primitive: you cannot give customer B an isolated row-space, isolated API keys, isolated model provider pools, and isolated billing from customer A without building a whole tenancy layer on top — or running one Open WebUI instance per tenant (the "scale out per tenant" pattern).

### 2.2 Access-control code is mature and security-conscious *where it exists* (FACT)

- `has_access_to_file` implements a layered check: direct ownership → knowledge-base grants → channel membership → shared chats → shared workspace models, with an explicit **CWE-863 (authorization bypass via new resource) defense comment**: "An object confers write/delete on a file only when the object's OWNER owns that file; otherwise a read-only file laundered into an object the user controls would gain write/delete on it (CWE-863)" ([files.py:45-63](research/repos/open-webui/backend/open_webui/utils/access_control/files.py:45)).
- This is a *positive* signal: the team has actually reasoned about privilege-laundering bugs. But it is per-resource, call-site-invoked enforcement — the same "distributed checks" weakness as Dify, just without even a tenant unit to distribute across.

### 2.3 Vector store abstraction is the best in the set — 10 backends (FACT)

- `Vector.get_vector` factory dispatches to: Milvus (+ `ENABLE_MILVUS_MULTITENANCY_MODE` variant), Qdrant (+ `ENABLE_QDRANT_MULTITENANCY_MODE` variant), Pinecone, S3Vector, OpenSearch, pgvector, OpenGauss, MariaDB, Elasticsearch, Chroma, Oracle23AI, Weaviate ([factory.py:16-80](research/repos/open-webui/backend/open_webui/retrieval/vector/factory.py:16)).
- **INFERENCE:** This is exactly the A3-boundary shape we want: Qdrant is first-class, and the factory is a clean swap point. Replacing/adding a vector backend here is a LOW-cost change.

### 2.4 Milvus multitenancy mode is a **self-documented data-corruption risk** (FACT — new finding, not in deep report)

- The mapping from per-user collection names to shared multitenant collections relies on parsing Open WebUI's *own* naming conventions (`user-memory-` prefix, `file-` prefix, `web-search-` prefix, or 63-char hex hash), with an explicit WARNING: "If Open WebUI changes how it generates collection names ... this mapping will break and route data to incorrect collections. **POTENTIALLY CAUSING HUGE DATA CORRUPTION, DATA CONSISTENCY ISSUES AND INCORRECT DATA MAPPING INSIDE THE DATABASE**." ([milvus_multitenancy.py:84-106](research/repos/open-webui/backend/open_webui/retrieval/vector/dbs/milvus_multitenancy.py:84))
- **INFERENCE:** This is the strongest piece of evidence in any repo that a "multitenancy mode" flag can be a trap: the feature works *today* only because an internal naming convention happens to be stable. Any rename refactoring (frequent in this codebase) silently cross-contaminates users' memories/files in a shared collection. Anyone enabling `ENABLE_MILVUS_MULTITENANCY_MODE` in production is accepting this contract. Qdrant multitenancy mode should be checked for the same pattern before use (UNKNOWN — flagged for C.5).

### 2.5 Chat pipeline is a 770-line endpoint in a 3000+ line `main.py` (FACT)

- `@app.post('/api/chat/completions') async def chat_completion(...)` spans [main.py:1087-1858](research/repos/open-webui/backend/open_webui/main.py:1087) — a single function containing model selection, pipeline (RAG + function calling + TTS + memory + translation + citations) orchestration, and stream assembly.
- Storage is provider-abstracted (Local/S3/GCS/Azure, [provider.py:40-331](research/repos/open-webui/backend/open_webui/storage/provider.py:40)).
- Memory is a per-user file-backed/DB-backed subsystem with LLM-driven memory operations (`add_memory_context`, `review_memory_after_turn`, [memory.py:289-602](research/repos/open-webui/backend/open_webui/utils/memory.py:289)) — real but not a durable cross-app memory *engine*; it is a user-notes-with-LLM-curation feature.
- MCP: a simple client (`MCPClient.connect/list_tool_specs/call_tool`, [mcp/client.py:59-147](research/repos/open-webui/backend/open_webui/utils/mcp/client.py:59)) with per-user MCP server config; tool execution is in-process.
- **No billing, no quotas, no credits** anywhere (absence verified in prior sweeps); `/api/usage` reports token usage of the *current request session*, not spend/entitlements ([main.py:2587-2609](research/repos/open-webui/backend/open_webui/main.py:2587)).

---

## 3. Red-Flag Register

| # | Red flag | Evidence | Severity | Consequence | Likely remediation | Requires fork? |
|---|----------|----------|----------|-------------|--------------------|----------------|
| OW-1 | **No tenancy primitive at all** — single trust domain, sharing-based ACLs | [users.py:45](research/repos/open-webui/backend/open_webui/models/users.py:45); AccessGrants model | **CRITICAL** (for multi-customer use) | Cannot isolate customers; every isolation need (keys, models, data, billing) must be rebuilt or solved by 1-instance-per-tenant | Platform-plane tenancy above Open WebUI; or per-tenant instances with a control plane | No fork for instance-per-tenant; yes if in-app tenancy is required |
| OW-2 | Milvus multitenancy mode self-documented corruption risk | [milvus_multitenancy.py:84-106](research/repos/open-webui/backend/open_webui/retrieval/vector/dbs/milvus_multitenancy.py:84) | HIGH (if enabled) | Silent cross-user data contamination in shared collections on any naming-convention change | Do not enable; use per-user collections or Qdrant multitenancy after verification | No — refuse the flag |
| OW-3 | Chat pipeline monolith (770-line endpoint, 3000-line file) | [main.py:1087-1858](research/repos/open-webui/backend/open_webui/main.py:1087) | HIGH | Every pipeline behavior change (context ordering, failure modes, memory timing) is a high-risk edit to one giant function | Extract pipeline stages; this is the classic "framework became application" refactor | No, but fork-sized |
| OW-4 | Access enforcement is per-call-site (no central boundary) | [files.py:19-124](research/repos/open-webui/backend/open_webui/utils/access_control/files.py:19) pattern repeated across routers | MEDIUM | New resources must remember to wire AccessGrants; missed = cross-user leak | Central authorization middleware + coverage tests | No |
| OW-5 | No metering/billing surface | [main.py:2587](research/repos/open-webui/backend/open_webui/main.py:2587) only reports session usage | MEDIUM | Commercial multi-user needs full external plane | LiteLLM proxy (keys + budgets) + platform ledger | No |
| OW-6 | Frontend/backend coupling: SvelteKit frontend calls backend internals; API is stable but undocumented contract for streaming | prior verified reads | MEDIUM | Frontend replacement is possible but the streaming/event contract must be re-implemented | Document/stabilize API; treat frontend as swappable consumer | No |
| OW-7 | SQLite default + single-process async job model | `main.py` lifespan tasks, [tasks.py](research/repos/open-webui/backend/open_webui/tasks.py:1) | MEDIUM | Horizontal scaling of the API process requires careful handling of in-process tasks (title gen, indexing) | Postgres (supported) + external task runner; or keep stateless + queue | No |

---

## 4. Dimension Scores (0–5)

| Dimension | Score | Rationale |
|---|---|---|
| Architecture | 2 | One FastAPI app + monolithic `main.py`; good sub-packages but no layering discipline |
| Modularity | 3 | `retrieval/vector` factory, storage providers, and access_control utils are genuinely swappable; rest is monolith |
| Provider abstraction | 3 | Broad model provider support incl. OpenAI-compatible ([openai.py:320-345](research/repos/open-webui/backend/open_webui/utils/openai.py:320)); no unified credential/metering plane |
| Memory | 3 | Per-user LLM-curated memory is real; not a durable engine; per-user file paths |
| RAG | 3 | Knowledge bases + RAG pipeline + 10 vector backends; depth below Dify/RAGFlow |
| Agents | 2 | Function calling + MCP; no agent runtime, no tool permission model beyond user-level |
| Workflows | 1 | No workflow engine at all |
| Tenancy | 4 (as a *sharing* model) / **1 (as tenancy)** | Scored **4** here reflects that the AccessGrants/group ACL system is well-built for a *single-tenant* multi-user deployment — the best per-resource ACL ergonomics in the set — but it is not tenancy. See red flag OW-1 for the tenancy verdict. |
| API | 3 | Clean REST + OpenAI-compat `/api/chat/completions`; undocumented streaming contract |
| Integrations | 3 | Ollama/OpenAI/Anthropic/MCP/pipelines are first-class |
| MCP/extensibility | 3 | Per-user MCP servers + pipelines (in-process Python functions) — pipelines are a real extension point but execution is in-process |
| File/storage | 3 | 4 storage providers; access-grant-protected files; forgetting = per-object delete |
| Context management | 2 | Pipeline assembles context but with no visible token-budget control |
| Tests/docs | 3 | Decent test coverage in access_control (incl. CWE-863 regression), docs are product-level |
| Deployment/scaling | 3 | Single container + optional Postgres/Milvus/Qdrant is the *simplest* deployment in the set; scaling = Postgres + external vector + stateless API (task caveats apply) |
| Modification/forkability | 2 | Permissive license, but the monolith makes *surgical* modification expensive; frequent upstream churn |

**Complexity: 3/5.** Necessary: provider breadth, RAG breadth. Accidental: the monolithic `main.py`, per-user collection-naming conventions that the multitenancy mode parses.

---

## 5. Change-Blast-Radius Test (A–M)

- **A/B/C (LLM provider changes):** LOW — OpenAI-compatible path + provider registry; LiteLLM drop-in is the easiest in the set.
- **D (replace memory):** MEDIUM — memory is a self-contained utils subsystem; swappable but loses the built-in UX.
- **E (add vector DB):** LOW — best factory in the set.
- **F (replace file storage):** LOW — provider pattern.
- **G (replace auth):** MEDIUM — JWT HS256 + OAuth + SSO exist; no enterprise IdP-grade RBAC.
- **H (billing):** HIGH (absence) — build entire plane externally (LiteLLM budgets + ledger).
- **I (project-level permissions):** HIGH — groups + AccessGrants approximate it; real project isolation = tenancy gap.
- **J/K (MCP, external apps):** LOW-MED — MCP client + pipelines exist.
- **L (replace frontend):** MEDIUM — stable REST, churning contract.
- **M (custom backend wrapper):** MEDIUM — API-first enough to wrap; pipeline extension (in-process Python) pulls custom code *into* the process, a coupling to avoid for a commercial backend.

---

## 6. Disagreements with the Deep Report

See [disagreements-C.md](research/evidence/disagreements-C.md). Headlines:
1. Deep report: "Custom OpenAI-compatible provider 0" → **DISAGREE**: Open WebUI has a full OpenAI-compatible provider path (`utils/openai.py`, `OPENAI_API_BASE_URL`/`OPENAI_API_KEY` user-level overrides verified at [openai.py:320-345](research/repos/open-webui/backend/open_webui/utils/openai.py:320)). The score appears to have measured something else (perhaps *custom provider class extensibility*); the claim as written is false.
2. Deep report: "Usage credits 3" → **PARTIALLY AGREE**: there is *no* credits/quota/entitlement system (only session usage reporting). "3" overstates; honest score is 1 (usage *reporting* exists, enforcement does not).
3. Deep report: "no demonstrated universal tenant_id boundary" → **AGREE**, and sharpened with OW-2 (the multitenancy *flag* is a documented corruption risk — the deep report missed it).

---

## 7. Commercial Suitability Verdict

**The best single-team / single-tenant deployment, the weakest multi-customer foundation.** Open WebUI is the fastest to stand up, has the most vendor-neutral vector layer, a real per-resource ACL system, and a permissive license. But it has no tenancy, no billing, no workflow engine, and a monolithic chat endpoint; its one "multitenancy" flag is self-documented as corruption-prone. The only defensible commercial topology is **instance-per-customer behind a control plane** (or a platform plane that re-implements tenancy/billing and treats Open WebUI as a per-tenant appliance). As *the* platform foundation: not viable. As an embeddable chat appliance: strong.

**Confidence: HIGH** (OW-1..OW-4 re-verified at cited lines this wave).

---

## 8. Open Questions for Wave C.5

1. Verify Qdrant multitenancy mode (`qdrant_multitenancy.py`) for the same naming-convention fragility as Milvus.
2. Confirm whether in-process background tasks (title generation, indexing) are safe under multi-replica API deployment (duplicate execution?).
3. Sweep all routers for AccessGrants coverage gaps (resources added without grant checks) — the files.py CWE-863 comment shows the class of bug the team is fighting; a full coverage audit is needed.
