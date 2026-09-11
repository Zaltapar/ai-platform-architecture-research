# Wave C — Independent Architecture Audit Summary

**Scope:** exactly five candidates — LibreChat, Dify, RAGFlow, Open WebUI, LobeChat. Independently verified against source under `research/repos/`. Prior forensic reports (`deep-*.md`, `triage-*.md`) treated as **hypotheses**, not authority. Per-candidate detail: [audit-librechat.md](audit-librechat.md), [audit-dify.md](audit-dify.md), [audit-ragflow.md](audit-ragflow.md), [audit-open-webui.md](audit-open-webui.md), [audit-lobechat.md](audit-lobechat.md). Disputed claims: [disagreements-C.md](research/evidence/disagreements-C.md).

**Scope note (Langflow):** Langflow was excluded from Wave C because the deep forensic checkout was evidence-limited on Windows; its candidacy remains **conditional** and requires a complete checkout before any final decision.

Method note: `execute_command` was unavailable in this environment (DCG spawn failure); all verification was done via file listing, regex search, and direct file reads, with every load-bearing claim cited to file:line.

---

## 1. Independent Ranking (foundation suitability for a commercial, extensible, multi-user AI platform)

**1. LobeChat (score 59/80) — 2. LibreChat (54/80) — 3. Dify (51/80) — 4. RAGFlow (45/80) — 5. Open WebUI (43/80)**

> **Ranking note:** The order above is the raw feature/architecture score ranking, derived directly from the 16-dimension matrix totals in §1. It differs from the strict-tenancy/fork recommendation in §6 (LibreChat), which is based on which candidate's *open* code already contains a tenant primitive that can be made safe-by-default — not on the raw score total.

Rationale in one line each:
- **LobeChat** is the best-engineered codebase (typed monorepo, strongest MCP security, real file lifecycle, coherent deployment) but its **entire business plane (billing, quotas, storage caps, workspace management) is closed-source by design** — OSS ships typed no-op stubs. Highest complexity budget, and the fork must rewrite the one module that matters commercially.
- **LibreChat** is the only candidate with a *database-level* tenant-isolation mechanism (Mongoose plugin, query injection + mutation guards) that could survive a security audit — but it is **opt-in and default-off**, which is its fatal operational caveat. Best provider surface, cleanest RAG seam (external `rag_api`), real multi-service split.
- **Dify** has the deepest RAG/workflow/agent engine and a genuine tenant_id data model, but its two most commercial features — **fine-grained RBAC and quota/billing — are closed-source** (enterprise service / Cloud edition gate), and enabling RBAC degrades the open permission model.
- **RAGFlow** is the best RAG *engine* (parsers, GraphRAG, memory subsystem) but the weakest platform: **active cross-tenant IDOR** on document images, a **dead credit system** (`credit=512` default, `TenantService.decrease` with zero call sites), per-process concurrency limits, and the heaviest deployment.
- **Open WebUI** is the best *single-tenant appliance*: simplest deployment, best vector-store factory (10 backends), real per-resource ACLs (CWE-863-aware), but **no tenancy primitive at all** and a monolithic 770-line chat endpoint. Commercial use = instance-per-customer topology.

### 16-dimension scores (0–5)

| Dimension | LibreChat | Dify | RAGFlow | Open WebUI | LobeChat |
|---|---|---|---|---|---|
| Architecture | 3 | 4 | 3 | 2 | 4 |
| Modularity | 4 | 3 | 2 | 3 | 4 |
| Provider abstraction | 4 | 4 | 3 | 3 | 4 |
| Memory | 3 | 2 | **5** | 3 | 3 |
| RAG | 3 | 4 | **5** | 3 | 3 |
| Agents | 4 | 4 | 3 | 2 | 4 |
| Workflows | 2 | 3 | 2 | 1 | 2 |
| Tenancy | **4** | 4 | 3 | 4† | 4 |
| API | 4 | 3 | 3 | 3 | 4 |
| Integrations | 3 | 3 | 2 | 3 | 4 |
| MCP / extensibility | 4 | 3 | 3 | 3 | **5** |
| File / storage | 4 | 3 | 2 | 3 | **5** |
| Context management | 3 | 4 | 3 | 2 | 3 |
| Tests / docs | 3 | 2 | 2 | 3 | 2 |
| Deployment / scaling | 2 | 2 | 1 | 3 | 4 |
| Modification / forkability | 4 | 3 | 3 | 2 | 4 |
| **Total /80** | **54** | **51** | **45** | **43** | **59** |

† Open WebUI tenancy is scored for its *sharing/ACL* system quality; as an actual tenancy boundary it is **0/5** (no tenant unit exists — see red flag OW-1). It is kept at 4 in the table only because the AccessGrants/group primitives are the best per-resource ACL ergonomics in the set and would become tenancy-grade *under* a platform plane. If you treat tenancy strictly, Open WebUI's total drops from 43/80 to 39/80 (43 − 4: the tenancy dimension 4 → 0). Under either reading Open WebUI ranks last (5th by composite total at 43, below RAGFlow's 45; 39 under strict tenancy) — its *appliance*-role strengths (deployment simplicity, vector factory, API cleanliness) are noted qualitatively in the per-candidate audit and do not change the raw-score rank order.

### Complexity (1–5, necessary vs accidental)

| Candidate | Score | Necessary | Accidental |
|---|---|---|---|
| LibreChat | 4 | provider breadth; tenancy mechanism; file pipeline | chat-pipeline monolith; FerretDB legacy copies; duplicated strict-mode env reads |
| Dify | 4 | RAG depth; workflow engine; provider breadth | **dual permission regime** (legacy vs enterprise RBAC); call-sites-only enterprise features; Celery sprawl |
| RAGFlow | 4 | parser/chunker/GraphRAG depth (the product) | sync/async bridging layer; dead credit schema; per-process concurrency; 2300–2500-line monolithic services |
| Open WebUI | 3 | provider breadth; RAG breadth | monolithic `main.py` (3043 lines); naming-convention-coupled multitenancy mode |
| LobeChat | **5** | provider breadth; MCP security model; file lifecycle | **dual backend surfaces** (legacy Next.js handlers + new Hono/tRPC server); stub indirection layer; 75-package monorepo overhead |

---

## 2. Candidate-Specific Red Flags (decisive ones; full registers in per-candidate audits)

### LibreChat
| Flag | Severity | Evidence |
|---|---|---|
| Tenant isolation **default-off** (fail-open default; strict = opt-in env var) | HIGH | [policy.ts:49-51](research/repos/librechat/packages/data-schemas/src/tenant/policy.ts:49); [tenant.ts:125-127](research/repos/librechat/packages/api/src/middleware/tenant.ts:125); [.env.example:749-751](research/repos/librechat/.env.example:749) |
| `TRUST_TENANT_HEADER`/strict operator trap (boot warns, does not fail) | MEDIUM | [index.js:217-220](research/repos/librechat/api/server/index.js:217) |
| rag_api synchronous in-request HTTP dependency (no queue/idempotency visible) | MEDIUM | [createContextHandlers.js:25-33](research/repos/librechat/api/app/clients/prompts/createContextHandlers.js:25) |
| Chat pipeline monolith (client.js / BaseClient.js) | MEDIUM | [client.js:4359](research/repos/librechat/api/app/clients/client.js:4359) |

### Dify
| Flag | Severity | Evidence |
|---|---|---|
| **Fine-grained RBAC is a closed enterprise service**; with `RBAC_ENABLED=true` all legacy role predicates return `True` and enforcement delegates to `ENTERPRISE_RBAC_API_URL` | **CRITICAL** | [account.py:192-239](research/repos/dify/api/models/account.py:192); [checks.py:38-40](research/repos/dify/api/controllers/common/rbac/checks.py:38); [enterprise/base.py:193-194](research/repos/dify/api/services/enterprise/base.py:193) |
| QuotaService cloud-gated and **fail-open** (`DEPLOYMENT_EDITION != CLOUD` → allow) | HIGH | [quota_service.py:121-123](research/repos/dify/api/services/quota_service.py:121) |
| Permission checks scattered across hundreds of call sites, two coexisting regimes | HIGH | [datasets.py:75-76, 759-761](research/repos/dify/api/controllers/console/datasets/datasets.py:75); [flask_admission.py:70-72](research/repos/dify/api/controllers/console/flask_admission.py:70) |
| Three auth regimes (console JWT / service API keys / OpenAPI) | MEDIUM | [openapi/auth/verify.py:42-59](research/repos/dify/api/controllers/openapi/auth/verify.py:42) |

### RAGFlow
| Flag | Severity | Evidence |
|---|---|---|
| **IDOR: `GET /documents/images/<image_id>` has no ownership/tenant check** (JWT/API/beta auth all accepted) | **CRITICAL** | [document_api.py:1831-1866](research/repos/ragflow/api/apps/restful_apis/document_api.py:1831) vs ownership-checked artifact path at [document_api.py:1900-1905](research/repos/ragflow/api/apps/restful_apis/document_api.py:1900) |
| **Dead credit system**: `credit = IntegerField(default=512)`; `TenantService.decrease` defined but **zero call sites** | HIGH | [db_models.py:1163](research/repos/ragflow/api/db/db_models.py:1163); [user_service.py:219-222](research/repos/ragflow/api/db/services/user_service.py:219) |
| Tenant boundary = per-tenant index naming + per-site `accessible()` checks; no central enforcement | HIGH | [doc_metadata_service.py:82](research/repos/ragflow/api/db/services/doc_metadata_service.py:82); [knowledgebase_service.py:569-590](research/repos/ragflow/api/db/services/knowledgebase_service.py:569) |
| Per-process semaphores only (`LoopLocalSemaphore` 5/1/10/2; chat limiter 10) — scale-out multiplies load, no per-tenant fairness | HIGH | [task_executor_limiter.py:20-28](research/repos/ragflow/rag/svr/task_executor_limiter.py:20); [graphrag/utils.py:42](research/repos/ragflow/rag/graphrag/utils.py:42) |
| `cors(app, allow_origin="*")` | MEDIUM | [apps/__init__.py:62](research/repos/ragflow/api/apps/__init__.py:62) |

### Open WebUI
| Flag | Severity | Evidence |
|---|---|---|
| **No tenancy primitive at all** (roles + groups + per-resource AccessGrants only) | **CRITICAL** (multi-customer) | [users.py:45](research/repos/open-webui/backend/open_webui/models/users.py:45) |
| Milvus multitenancy mode **self-documented data-corruption risk** (parses internal collection-name conventions) | HIGH (if enabled) | [milvus_multitenancy.py:84-106](research/repos/open-webui/backend/open_webui/retrieval/vector/dbs/milvus_multitenancy.py:84) |
| Chat pipeline = 770-line endpoint in 3043-line `main.py` | HIGH | [main.py:1087-1858](research/repos/open-webui/backend/open_webui/main.py:1087) |
| Access enforcement per-call-site | MEDIUM | [access_control/files.py:19-124](research/repos/open-webui/backend/open_webui/utils/access_control/files.py:19) |
| No metering/billing surface (only session usage reporting) | MEDIUM | [main.py:2587-2609](research/repos/open-webui/backend/open_webui/main.py:2587) |

### LobeChat
| Flag | Severity | Evidence |
|---|---|---|
| **Entire business plane closed-source by design** — workspace creation `cloudOnly`/`NOT_IMPLEMENTED`; workspace usage stub `remainingBalance: 0`; spend router empty; file-storage checks no-op; precharge no-op; share spend gate always `{allowed: true}`; alias `@/business/server/*` → stub package | **CRITICAL** | [workspace.ts:46-82](research/repos/lobechat/packages/business-server/src/lambda-routers/workspace.ts:46); [workspaceUsage.ts:14-24](research/repos/lobechat/packages/business-server/src/lambda-routers/workspaceUsage.ts:14); [spend.ts:3](research/repos/lobechat/packages/business-server/src/lambda-routers/spend.ts:3); [file.ts:13-25](research/repos/lobechat/packages/business-server/src/lambda-routers/file.ts:13); [chargeBeforeGenerate.ts:41-43](research/repos/lobechat/packages/business-server/src/image-generation/chargeBeforeGenerate.ts:41); [spendGate.ts:39-61](research/repos/lobechat/packages/business-server/src/agent-share/spendGate.ts:39); [tsconfig.json:28](research/repos/lobechat/tsconfig.json:28) |
| **JWKS RSA private key embedded in `.env.example`** (liveness UNKNOWN) | HIGH | [.env.example:69](research/repos/lobechat/docker-compose/deploy/.env.example:69) |
| Vector backend locked to pgvector @ fixed 1024-dim in the app Postgres (no Qdrant/Milvus adapter) | MEDIUM | [rag.ts:19-132](research/repos/lobechat/packages/database/src/schemas/rag.ts:19); [docker-compose.yml:43-45](research/repos/lobechat/docker-compose/deploy/docker-compose.yml:43) |
| Optional Elasticsearch shipped with `xpack.security.enabled=false` | MEDIUM | [docker-compose.yml:157-169](research/repos/lobechat/docker-compose/deploy/docker-compose.yml:157) |
| Dual backend surfaces (legacy `src/app/(backend)` route handlers coexisting with `apps/server` Hono/tRPC) | MEDIUM | [Dockerfile](research/repos/lobechat/Dockerfile:1); [tsconfig.json:28](research/repos/lobechat/tsconfig.json:28) |

**Cross-cutting pattern:** all five candidates keep their *commercial* capabilities (fine-grained RBAC, quota/billing, workspace management) out of the open-source build — Dify via an enterprise API, LobeChat via typed stubs, the other three via simple absence. **None of the five can be a commercial multi-user foundation without a platform plane that the repo itself does not provide.**

---

## 3. Disputed Claims vs the Deep Reports

Full register with exact quotes and verdicts (AGREE / DISAGREE / PARTIALLY AGREE / UNVERIFIED): [disagreements-C.md](research/evidence/disagreements-C.md). Decisive items:

1. **deep-librechat** "fail-closed strict mode ... implemented" / "strongest explicit general-purpose tenant query boundary" → **PARTIALLY AGREE**: mechanism confirmed real and well-tested; but **default is non-strict (fail-open)** ([policy.ts:49-51](research/repos/librechat/packages/data-schemas/src/tenant/policy.ts:49), [tenant.ts:125-127](research/repos/librechat/packages/api/src/middleware/tenant.ts:125)) and the project's own e2e fixtures run `TENANT_ISOLATION_STRICT: 'false'` ([record.js:53](research/repos/librechat/e2e/setup/record.js:53)). The report omits the unsafe default.
2. **deep-dify** "Usage credits 1 — Credit pools and quota service already exist" → **DISAGREE with framing**: the quota lifecycle exists but is cloud-gated and fail-open ([quota_service.py:121-123](research/repos/dify/api/services/quota_service.py:121)); "already exists" overstates availability. The report also **missed the RBAC delegation finding entirely** (legacy predicates → `True` + closed enterprise service, [account.py:192-239](research/repos/dify/api/models/account.py:192)) — the single most important Dify fact for commercial use.
3. **deep-ragflow** "Usage credits 4 — Billing ledger absent from evidence" → **DISAGREE**: not absent, *dead code* (`credit=512` default, `decrease()` never called — [db_models.py:1163](research/repos/ragflow/api/db/db_models.py:1163), [user_service.py:219-222](research/repos/ragflow/api/db/services/user_service.py:219)). Score should be 0. The report also **missed the image-endpoint IDOR** ([document_api.py:1831-1866](research/repos/ragflow/api/apps/restful_apis/document_api.py:1831)) and overstated multi-user tenancy (`tenant_id == user.id` in practice).
4. **deep-open-webui** "Custom OpenAI-compatible provider 0" → **DISAGREE**: a full OpenAI-compatible provider path exists with per-user key/base overrides ([openai.py:320-345](research/repos/open-webui/backend/open_webui/utils/openai.py:320)). "Usage credits 3" → **PARTIALLY AGREE** (only session usage reporting exists; enforcement score should be 1). "No universal tenant_id boundary" → **AGREE**, and sharpened by the new milvus-multitenancy corruption WARNING ([milvus_multitenancy.py:84-106](research/repos/open-webui/backend/open_webui/retrieval/vector/dbs/milvus_multitenancy.py:84)), which the report missed.
5. **deep-lobechat** (stale) → **DISAGREE on architecture currency**: the report describes the old Next.js-route-handler architecture and leaves "usage-credit ledger, billing entitlement, storage quota aggregation" as UNKNOWN. The repo is now a pnpm monorepo with `apps/server` (Hono + tRPC) and `packages/business-server` **typed no-op stubs** overridden by the closed cloud repo ([tsconfig.json:28](research/repos/lobechat/tsconfig.json:28), [workspaceUsage.ts:14-24](research/repos/lobechat/packages/business-server/src/lambda-routers/workspaceUsage.ts:14)). The UNKNOWNs are resolved: usage *recording* is real OSS ([usage/index.ts:25-367](research/repos/lobechat/apps/server/src/services/usage/index.ts:25)); enforcement is closed.

---

## 4. Subsystem Integration Findings (vs A3 boundary recommendations)

Cross-checked against [subsystem-summary-A3.md](research/reports/forensic/subsystem-summary-A3.md).

| Subsystem | Recommended boundary | Fit with top candidates |
|---|---|---|
| **LiteLLM** (model plane) | Owns keys, spend keys, budgets, fallbacks | **LibreChat/Dify/Open WebUI/LobeChat all integrate cheaply** (OpenAI-compatible paths verified in each). RAGFlow is the outlier: per-tenant model binding + sync-bridge layer means LiteLLM works but adds latency/complexity. No candidate needs a fork for A/B/C. |
| **Mem0** (memory engine) | Replace/extend in-app memory | **None of the five has a swappable memory seam.** Best additive fit: LobeChat (context-engine with token accounting) and Open WebUI (self-contained `utils/memory.py` subsystem). RAGFlow's memory is a first-class product subsystem — replacing it is a product decision. Dify has essentially no memory module to replace (additive only). |
| **MCP Python SDK** (protocol adapter) | Adapter at platform edge | **LobeChat is the reference implementation** (fail-closed synced-tool allowlist, RFC 9728 PRM, default-deny share gate — [mcp.ts:84](research/repos/lobechat/apps/server/src/routers/tools/mcp.ts:84), [connector/oauth.ts:54](research/repos/lobechat/apps/server/src/services/connector/oauth.ts:54), [shareGate.ts](research/repos/lobechat/apps/server/src/services/aiAgent/shareGate.ts:1)). Open WebUI has a minimal in-process client ([mcp/client.py:59-147](research/repos/open-webui/backend/open_webui/utils/mcp/client.py:59)). RAGFlow fetches MCP tools but no verified call-time gating. Dify/LibreChat middling. |
| **Qdrant** (vector infra) | Swappable vector store behind adapter | **Open WebUI is the only candidate with a clean multi-backend vector factory** (10 backends incl. Qdrant + multitenancy mode — [factory.py:16-80](research/repos/open-webui/backend/open_webui/retrieval/vector/factory.py:16)). LibreChat's external rag_api seam is repointable. Dify's `core/rag` adapters are workable. **RAGFlow's docstore abstraction** (ES/Infinity/OceanBase) is real but not Qdrant-oriented. **LobeChat is locked to pgvector @1024-dim** — the only hard vector lock-in in the set. |
| **Temporal** (durable execution) | Long-running work outside the request | **No candidate uses durable execution.** All five do in-process/Celery/Redis-Streams work. LibreChat's rag_api + code-api are the most queue-shaped seams; Dify's Celery topology is the most mature (but vendor-shaped); RAGFlow's Redis Streams + per-process semaphores are the weakest at scale. Expect to add the Temporal plane *outside* whichever candidate is chosen, not inside it. |

**Cross-cutting integration risks (from A3, all still stand):** tenant propagation across subsystems (no candidate carries a tenant context through to vector/LLM/MCP calls in a verifiable, uniform way — each re-derives scoping at its own layer); duplicate policy planes (every candidate + the platform would each hold a permission model); MCP API-rename drift (LobeChat's synced-manifest model is the mitigation pattern worth copying); data privacy (RAGFlow's IDOR and Open WebUI's naming-convention multitenancy are the two live data-leak classes found).

---

## 5. Issues Requiring Targeted Verification in Wave C.5

**Security (priority 1):**
1. **RAGFlow IDOR sweep** — audit *all* storage-read endpoints (`/documents/preview`, `/documents/<doc_id>`, thumbnails, artifacts) for the missing-ownership pattern found at [document_api.py:1831-1866](research/repos/ragflow/api/apps/restful_apis/document_api.py:1831). RF-1 may be one of several.
2. **LobeChat JWKS key liveness** — determine whether the private key in [deploy/.env.example:69](research/repos/lobechat/docker-compose/deploy/.env.example:69) is a live signing key (check apps/auth for the JWKS public counterpart).
3. **LibreChat tenant-plugin coverage** — enumerate model collections *not* registered with `applyTenantIsolation` (unregistered = leak path); verify no user-reachable `runAsSystem` path.
4. **Dify RBAC bypass enumeration** — exhaustive grep of `if dify_config.RBAC_ENABLED` sites to list every code path where legacy gates read `True`.
5. **Open WebUI AccessGrants coverage** — sweep routers for resources reachable without grant checks; verify Qdrant multitenancy mode for milvus-style naming-convention fragility.
6. **LobeChat device-gateway** — verify per-device pairing approval and tool allowlist enforcement end-to-end.

**Architecture (priority 2):**
7. **LobeChat `@/business/server/*` import census** — size the closed surface (how many production call sites in `apps/server` depend on stubs).
8. **LobeChat workspace scoping** — confirm every data access in `apps/server` routes through `buildWorkspaceWhere` (unscoped query = leak).
9. **Open WebUI multi-replica safety** — do in-process background tasks (title gen, indexing) duplicate under horizontal scaling?
10. **RAGFlow memory tenancy** — verify `memory_{id}` Redis keys and memory docstore indices are tenant-scoped on read/delete ([memory_message_service.py:317-340](research/repos/ragflow/api/db/joint_services/memory_message_service.py:317)).

**Commercial (priority 3):**
11. **Dify `BillingService` contract** — document the enterprise billing API shape to size a custom backend vs. redirecting `QuotaService` to LiteLLM.
12. **LobeChat upgrade-drift measurement** — frequency of `business-server` contract changes across recent tags (drives fork-maintenance cost).

---

## 6. Overall Verdict

**No candidate is adoptable as-is as the commercial multi-user platform.** The consistent finding across all five: the *platform-critical* capabilities (fine-grained tenancy/permissions, quota/billing enforcement, durable multi-tenant management) are either closed-source (Dify RBAC/billing, LobeChat business plane) or absent (LibreChat, Open WebUI, RAGFlow) — and where the mechanisms that *are* open exist, they are unsafe-by-default (LibreChat strict mode off) or buggy (RAGFlow IDOR, Open WebUI milvus mapping).

The defensible architecture is therefore **platform plane + engine(s)**, not "pick one app":
- **Platform plane (you build):** identity/SSO, tenant + project model, RBAC, billing/quotas (LiteLLM spend keys + own ledger), durable jobs (Temporal), MCP policy (copy LobeChat's fail-closed synced-manifest + default-deny share-gate pattern).
- **Engine candidates by role:** RAGFlow or Dify as the RAG/workflow engine (behind a gateway that re-enforces tenancy — mandatory for RAGFlow until the IDOR is fixed); LibreChat as the chat+agents reference for tenant-scoped DB design; LobeChat as the reference for MCP security and precharge/recharge billing *shapes*; Open WebUI as the per-tenant appliance if that topology is acceptable.
- **If forced to pick one foundation to fork:** LibreChat — the only repo whose *open* code already contains the right multi-tenant primitive (DB-level scoping), with a single, well-scoped fix required to make it safe-by-default. LobeChat is the runner-up *only* if its MCP/file/agent surfaces outweigh committing to a private business-server rewrite.

**Unresolved disagreements carried to Wave C.5:** (a) LobeChat JWKS key liveness (UNKNOWN); (b) completeness of the Dify RBAC bypass list (representative set verified, exhaustive audit pending); (c) whether RAGFlow's IDOR pattern recurs at other storage endpoints (one confirmed, others unverified); (d) Open WebUI Qdrant multitenancy fragility (unverified); (e) LibreChat tenant-plugin coverage gaps (unverified).
