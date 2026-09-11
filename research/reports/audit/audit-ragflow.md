# Wave C Independent Audit — RAGFlow

**Auditor posture:** independent, skeptical. All claims independently re-verified against `research/repos/ragflow/` source. Labels: **FACT** (direct source evidence), **INFERENCE** (auditor reasoning), **UNKNOWN** (not verifiable in-repo).

---

## 1. Scope & Method

Independently inspected: auth layer (`api/apps/__init__.py`), tenant/user models (`db_models.py`, `user_service.py`), document REST API (`document_api.py`), knowledgebase access control (`knowledgebase_service.py`), index naming (`doc_metadata_service.py`, `chunk_api.py`, `search.index_name`), task concurrency (`task_executor_limiter.py`), memory subsystem keys (`memory_message_service.py`), LLM wrapper (`llm_service.py`).

---

## 2. Verified Architectural Facts

### 2.1 Tenant isolation is per-tenant *index naming*, not a scoping mechanism (FACT)

- Chunk/document indices are named per tenant: `index_name = search.index_name(tenant_id)` at [document_api.py:1767](research/repos/ragflow/api/apps/restful_apis/document_api.py:1767), [chunk_api.py:249](research/repos/ragflow/api/apps/restful_apis/chunk_api.py:249), [document_service.py:469](research/repos/ragflow/api/db/services/document_service.py:469); metadata index `f"ragflow_doc_meta_{tenant_id}"` at [doc_metadata_service.py:82](research/repos/ragflow/api/db/services/doc_metadata_service.py:82).
- Access decisions are service-level: `KnowledgebaseService.accessible(kb_id, user_id)` ([knowledgebase_service.py:569-590](research/repos/ragflow/api/db/services/knowledgebase_service.py:569)) and `_visibility_and_status_filter(joined_tenant_ids, user_id)` ([knowledgebase_service.py:118-128](research/repos/ragflow/api/db/services/knowledgebase_service.py:118)) — i.e., "does this user's tenant own this KB?" checked at selected call sites.
- **INFERENCE:** Isolation is "genuine" at the KB/chunk level *when the check is invoked*, but the tenant boundary is a naming convention + per-site ownership check, not a DB-level plugin (no LibreChat-style query injection). A single endpoint that forgets the `accessible()` call is an IDOR — and such an endpoint exists (see RF-1).

### 2.2 IDOR on document images (FACT — decisive new finding)

- `GET /documents/images/<image_id>` is guarded only by `@login_required(auth_types=[AUTH_JWT, AUTH_API, AUTH_BETA])` ([document_api.py:1831-1833](research/repos/ragflow/api/apps/restful_apis/document_api.py:1831)).
- The handler parses the composite ID into `(bucket, object_name)` and returns raw storage bytes with **no ownership or tenant check**: `parsed = _parse_document_image_id(image_id); bkt, nm = parsed; data = await thread_pool_exec(settings.STORAGE_IMPL.get, bkt, nm)` ([document_api.py:1856-1866](research/repos/ragflow/api/apps/restful_apis/document_api.py:1856)).
- `image_id` is `{dataset_id}-{thumbnail_object_key}` split on the first hyphen ([document_api.py:1845](research/repos/ragflow/api/apps/restful_apis/document_api.py:1845)) — the dataset_id is in the URL, so any authenticated user (or any holder of an API/beta token) who can guess/learn another tenant's dataset_id can retrieve its document page images.
- Contrast in the same file: artifact downloads do check ownership (`_sandbox_artifact_accessible(filename, user_id)`, [document_api.py:1900-1905](research/repos/ragflow/api/apps/restful_apis/document_api.py:1900)) — proving the pattern exists but was not applied here.

**Severity: HIGH.** This is a cross-tenant information disclosure (document page images can contain sensitive content, diagrams, PII) reachable via JWT, API, and beta auth types.

### 2.3 Credit system is dead code (FACT — stronger than the deep report's "ledger absent")

- The `Tenant` model carries `credit = IntegerField(default=512, index=True)` ([db_models.py:1163](research/repos/ragflow/api/db/db_models.py:1163)).
- `TenantService.decrease(user_id, num)` is defined ([user_service.py:219-222](research/repos/ragflow/api/db/services/user_service.py:219)) but a repo-wide search for `.decrease(` returns **zero call sites** — the credit column is never decremented, never checked against usage, and never surfaced as an entitlement.

**INFERENCE:** The deep report's "billing ledger absent from evidence" understates the finding: the schema *pretends* at a credit system (default 512 credits per tenant) while having **no** consumption, no check, no API. This is misleading for anyone assessing commercial readiness — it is not an absent feature, it is a vestigial fake one.

### 2.4 Multi-user is effectively single-user-per-tenant (FACT, carried from prior verification)

- `tenant_id == current_user.id` for self-created tenants (user_service tenant creation path); `UserTenantService` supports joining other tenants, but the product surface (API `tenant_id` resolution via `add_tenant_id_to_kwargs`, [api_utils.py:241-251](research/repos/ragflow/api/utils/api_utils.py:241)) is oriented around the *user's own* tenant.
- **INFERENCE:** "Stronger than user-only ownership" (deep report) is technically correct but practically weak: there is no meaningful workspace membership/role model, no per-resource ACLs, no admin/owner distinction at the API layer.

### 2.5 Concurrency control is per-process, not cluster-wide (FACT)

- `task_limiter = LoopLocalSemaphore(MAX_CONCURRENT_TASKS=5)`, `chunk_limiter`/`embed_limiter` = 1, `minio_limiter` = 10, `kg_limiter` = 2 ([task_executor_limiter.py:20-28](research/repos/ragflow/rag/svr/task_executor_limiter.py:20)) — semaphores live in the task-executor process's event loop.
- Chat concurrency similarly per-process: `chat_limiter = LoopLocalSemaphore(MAX_CONCURRENT_CHATS=10)` ([graphrag/utils.py:42](research/repos/ragflow/rag/graphrag/utils.py:42)).
- **INFERENCE:** Scaling task executors horizontally multiplies real concurrency (N × 5 tasks, N × 10 chats) — no distributed limiting, no per-tenant quotas. Under commercial load, tenant A's parse storm saturates shared embedding/LLM endpoints with no fairness or admission control.

### 2.6 CORS is open (FACT)

- `app = cors(app, allow_origin="*")` ([apps/__init__.py:62](research/repos/ragflow/api/apps/__init__.py:62)) — combined with cookie/JWT auth this widens the browser-based attack surface; API-key endpoints are the real surface, but `allow_origin="*"` is a red flag for any deployment.

---

## 3. Red-Flag Register

| # | Red flag | Evidence | Severity | Consequence | Likely remediation | Requires fork? |
|---|----------|----------|----------|-------------|--------------------|----------------|
| RF-1 | **IDOR on `/documents/images/<image_id>`** — no ownership/tenant check | [document_api.py:1831-1866](research/repos/ragflow/api/apps/restful_apis/document_api.py:1831) | **CRITICAL** | Cross-tenant disclosure of document page images via any auth type (JWT/API/beta) | Add `accessible()`-style ownership check on the dataset before storage read (pattern exists at [document_api.py:1900-1905](research/repos/ragflow/api/apps/restful_apis/document_api.py:1900)); audit all storage-read endpoints | No — small, local fix |
| RF-2 | **Dead credit system** — `credit` column default 512, `TenantService.decrease` never called | [db_models.py:1163](research/repos/ragflow/api/db/db_models.py:1163); [user_service.py:219-222](research/repos/ragflow/api/db/services/user_service.py:219); 0 call sites for `.decrease(` | HIGH | No metering, no quotas, no billing hook of any kind; misleading to stakeholders | Remove the column or build a real ledger at the platform plane (LiteLLM spend + entitlements) | No |
| RF-3 | Tenant boundary = index naming + per-site checks, no central enforcement | [doc_metadata_service.py:82](research/repos/ragflow/api/db/services/doc_metadata_service.py:82); [knowledgebase_service.py:569-590](research/repos/ragflow/api/db/services/knowledgebase_service.py:569) | HIGH | Every new endpoint is a potential IDOR; audit burden scales with feature count | Central authorization middleware + lint/test for tenant-scoped access | No, but substantial |
| RF-4 | Per-process semaphores, no cluster-wide or per-tenant concurrency/quotas | [task_executor_limiter.py:20-28](research/repos/ragflow/rag/svr/task_executor_limiter.py:20) | HIGH | Horizontal scaling multiplies load; noisy-tenant starvation of embedding/LLM endpoints | Distributed admission control (Redis-based) or route through platform job plane (Temporal) with per-tenant weights | No |
| RF-5 | `tenant_id == user.id` in practice; weak workspace/role model | prior verified reads of `user_service.py` tenant creation | MEDIUM | Cannot model teams/projects/customers as tenancy units | Platform-plane identity/tenancy; RAGFlow becomes an engine behind it | No |
| RF-6 | `cors(app, allow_origin="*")` | [apps/__init__.py:62](research/repos/ragflow/api/apps/__init__.py:62) | MEDIUM | Browser-origin attacks; token/cookie leakage surface | Restrict origins per deployment | No — trivial |
| RF-7 | `LLMBundle` sync/async bridging (thread + new event loops) around all model calls | [llm_service.py:407-473](research/repos/ragflow/api/db/services/llm_service.py:407) | MEDIUM | Thread/loop proliferation under load; subtle failure modes in streaming | Long-term: native async provider adapters; short-term: bounded thread pools | No |
| RF-8 | Heavy sync-in-async patterns (`thread_pool_exec`) across the API | [document_api.py:1860](research/repos/ragflow/api/apps/restful_apis/document_api.py:1860) and many peers | MEDIUM | Event-loop blocking risk at scale; hard to reason about backpressure | Incremental; accept for engine use | No |

---

## 4. Dimension Scores (0–5)

| Dimension | Score | Rationale |
|---|---|---|
| Architecture | 3 | Deep RAG domain layer; but monolithic Quart app + Peewee ORM + per-site security |
| Modularity | 2 | `rag/` vs `api/` split exists, but services, models, and controllers entangle; single giant `dialog_service.py` (~2300 lines) |
| Provider abstraction | 3 | Model factories per capability (chat/embd/rerank/tts/asr) are clean; per-tenant model binding |
| Memory | 5 | Best-in-set: first-class Memory objects, size accounting, session memory with vector store integration |
| RAG | 5 | Best-in-set: parsers, chunking, GraphRAG/advanced RAG, wiki compilation, SQL-on-documents |
| Agents | 3 | Dialog agent + MCP tool fetching (`get_mcp_tools`, [api_utils.py:668-698](research/repos/ragflow/api/utils/api_utils.py:668)); no durable agent runtime |
| Workflows | 2 | No workflow engine; task executors are parse/ingest workers only |
| Tenancy | 3 | Per-tenant index isolation is real; ownership checks per-site; IDOR present; no roles |
| API | 3 | Broad REST + OpenAI-ish endpoints; three auth types (JWT/API/beta) with inconsistent enforcement |
| Integrations | 2 | Docstore abstraction (ES/Infinity/OceanBase) is good; but app-level integration surface is thin |
| MCP/extensibility | 3 | MCP server config + tool fetching exists; no permission gating on MCP tools at call time verified |
| File/storage | 2 | Storage impl abstraction (MinIO/local/…) good; but IDOR on image reads + no deletion/forgetting guarantee verified |
| Context management | 3 | Dialog context + RAG assembly solid; no token-budget management at API level |
| Tests/docs | 2 | Large codebase, mixed test coverage; security paths (like RF-1) evidently untested |
| Deployment/scaling | 1 | Weakest of the five: ES + Infinity/OceanBase + Redis + task executors + sandbox; per-process limits make scale-out dangerous |
| Modification/forkability | 3 | Apache-2.0; Python; but enormous domain surface (2300-line services, 2500-line model files) makes surgical changes expensive |

**Complexity: 4/5.** Necessary: parser/chunker/GraphRAG depth (this is the product). Accidental: sync/async bridging layer, dead credit schema, per-process concurrency model, monolithic services.

---

## 5. Change-Blast-Radius Test (A–M)

- **A/B/C (LLM provider):** LOW-MED — per-tenant model binding + factories make provider swaps clean; but sync-bridge layer adds friction.
- **D (replace memory):** HIGH — RAGFlow's memory is a first-class product subsystem deeply coupled to docstore + Redis; replacing it is a product-level decision, not an adapter swap.
- **E (vector DB):** MEDIUM — docstore abstraction supports multiple backends; re-index pain is real.
- **F (file storage):** LOW-MED — `STORAGE_IMPL` abstraction.
- **G (replace auth):** MEDIUM — `_load_user` (JWT/API/beta) is a self-contained layer ([apps/__init__.py:144-229](research/repos/ragflow/api/apps/__init__.py:144)), replaceable but must preserve per-tenant API keys.
- **H (billing):** HIGH — nothing to hook into; build entire metering plane externally.
- **I (project-level permissions):** HIGH — no permission model exists beyond tenant ownership.
- **J/K (MCP, external apps):** MEDIUM.
- **L (replace frontend):** MEDIUM — API-first, but frontend is large.
- **M (custom backend wrapper):** MEDIUM — usable engine API, but you must re-implement the missing security/metering around it.

---

## 6. Disagreements with the Deep Report

See [disagreements-C.md](research/evidence/disagreements-C.md). Headlines:
1. Deep report: "Usage credits 4 — Billing ledger absent from evidence" → I **DISAGREE with the score and sharpen the finding**: it is not merely absent, it is *vestigial dead code* (`credit=512` default, `decrease()` with zero call sites). Score should be **0**.
2. Deep report: "genuine tenant/workspace isolation stronger than user-only ownership" → **PARTIALLY AGREE**: per-tenant index isolation is real, but the IDOR (RF-1) and per-site-only enforcement mean the *enforced* boundary is weaker than "genuine" implies.
3. Deep report missed the IDOR entirely → **UNVERIFIED-in-their-report / newly found by this audit**.

---

## 7. Commercial Suitability Verdict

**A best-in-class RAG engine that cannot be a multi-user platform.** If the platform decision is "which repo gives me the strongest document→knowledge pipeline," RAGFlow wins decisively on RAG and memory. But as a *foundation for a commercial multi-user product* it is the weakest of the five: no real permission model, an active cross-tenant IDOR, a fake credit system, per-process concurrency limits, and the heaviest deployment (ES + Infinity + Redis + task executors + sandbox + admin). The correct role is **RAG engine behind the platform**, not the platform itself.

**Confidence: HIGH** (RF-1, RF-2, RF-3, RF-4, RF-6 all re-verified at cited lines this wave).

---

## 8. Open Questions for Wave C.5

1. Audit **all** storage-read endpoints (`/documents/preview`, `/documents/<doc_id>`, thumbnails, artifact paths) for the same missing-ownership pattern — RF-1 may be one of several.
2. Verify whether `memory_{memory_id}` Redis keys and memory docstore indices are tenant-scoped on read/delete (size cache is keyed by memory_id only, [memory_message_service.py:317-340](research/repos/ragflow/api/db/joint_services/memory_message_service.py:317)).
3. Confirm the task-executor → Redis Streams claim: is a task's tenant propagated into the embedding call, and can a tenant's parse jobs starve shared LLM endpoints?
