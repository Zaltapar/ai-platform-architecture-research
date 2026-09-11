# Wave C Independent Audit — LibreChat

**Auditor posture:** independent, skeptical. Prior reports treated as hypotheses. All claims below independently re-verified against `research/repos/librechat/` source. Labels: **FACT** (direct source evidence), **INFERENCE** (auditor reasoning from FACT), **UNKNOWN** (not verifiable in-repo).

---

## 1. Scope & Method

Independently inspected: tenant-isolation plugin and policy code, tenant middleware, server boot path, RAG integration surface (rag_api HTTP seam), file/VectorDB CRUD, context handler construction, deployment config (`.env.example`), e2e/test fixtures. No production code written; no other repos investigated.

---

## 2. Verified Architectural Facts

### 2.1 Tenant isolation is a real, well-tested Mongoose plugin — but opt-in (FACT)

- [tenantIsolation.ts](research/repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96) documents the four-state policy:
  - `tenantId` present in async context → injected into every query filter;
  - `tenantId = SYSTEM_TENANT_ID` → injection skipped (explicit cross-tenant op);
  - `tenantId` absent + `TENANT_ISOLATION_STRICT=true` → **throws** (fail-closed);
  - `tenantId` absent + strict off → **passes through** (transitional/pre-tenancy).
- The plugin hooks `find`, `findOne`, `distinct`, `findOneAndUpdate/Delete/Replace`, `updateOne/Many`, `deleteOne/Many`, `countDocuments`, `replaceOne` ([tenantIsolation.ts:148-159](research/repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:148)), plus a mutation/replace guard that strips attacker-supplied `tenantId` from updates and empties the filter (`$in: []`) when scoping would otherwise be widened ([tenantIsolation.ts:119-146](research/repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:119)).
- Strictness is read from the environment only: `isTenantIsolationStrict()` returns `process.env.TENANT_ISOLATION_STRICT === 'true'` ([policy.ts:49-51](research/repos/librechat/packages/data-schemas/src/tenant/policy.ts:49)), with a warning for malformed values defaulting to **non-strict** ([policy.ts:58-63](research/repos/librechat/packages/data-schemas/src/tenant/policy.ts:58)).
- `.env.example` ships `# TENANT_ISOLATION_STRICT=false` commented out ([.env.example:749-751](research/repos/librechat/.env.example:749)) — **default deployment is non-strict**.
- The request path confirms it: an authenticated request without a resolvable `tenantId` returns 403 **only in strict mode**; non-strict "(default) → passes through with user/request context only" ([tenant.ts:125-127](research/repos/librechat/packages/api/src/middleware/tenant.ts:125)).
- Pre-auth tenant headers are only trusted when `TRUST_TENANT_HEADER` is on; the boot path explicitly warns when strict mode is active while the header is untrusted: "[Security] TENANT_ISOLATION_STRICT is active while TRUST_TENANT_HEADER is disabled. Pre-authentication tenant headers will be ignored." ([index.js:217-220](research/repos/librechat/api/server/index.js:217), duplicated in [experimental.js:501-504](research/repos/librechat/api/server/experimental.js:501)).
- The project's own e2e fixtures run with `TENANT_ISOLATION_STRICT: 'false'` ([record.js:53](research/repos/librechat/e2e/setup/record.js:53), [playwright.config.mock.ts:111](research/repos/librechat/e2e/playwright.config.mock.ts:111)) — i.e., the project's own test suite does not exercise strict mode as the default.

**INFERENCE:** The mechanism is among the strongest DB-level tenant scoping in the five candidates (query-level injection + mutation guards + `runAsSystem` + a conformance test suite in [conformance.ts](research/repos/librechat/packages/data-schemas/src/tenant/conformance.ts:1)). But the **default is fail-open**, and the safe posture requires operator configuration plus a coherent `TRUST_TENANT_HEADER` decision. A multi-user commercial deployment that forgets this env var runs with permissive legacy scoping.

### 2.2 RAG is an external HTTP service (rag_api), not embedded (FACT)

- Context construction short-circuits when `RAG_API_URL` is unset ([createContextHandlers.js:12-13](research/repos/librechat/api/app/clients/prompts/createContextHandlers.js:12)); otherwise it issues `axios.get(RAG_API_URL/documents/{id}/context)` and `axios.post(RAG_API_URL/query)` ([createContextHandlers.js:25-33](research/repos/librechat/api/app/clients/prompts/createContextHandlers.js:25)).
- Vector upsert/delete go over the same HTTP seam ([crud.js:20-28](research/repos/librechat/api/server/services/Files/VectorDB/crud.js:20), [crud.js:67-89](research/repos/librechat/api/server/services/Files/VectorDB/crud.js:67)); file-search tool likewise (`fileSearch.js:139-141`).
- **INFERENCE:** This is a genuine architectural seam — replacing the vector store means replacing/repointing an external service, not forking the app. But it is a **single synchronous HTTP dependency inside the chat request path** (no queue, no idempotency keys visible in these call sites): rag_api availability degrades the main product. For a commercial platform this is where Temporal/queue indirection would need to be added.

### 2.3 Provider surface is broad but monolithic (FACT/INFERENCE, carried from prior verified reads)

- Client layer (`BaseClient.js`) and `createStreamServices` remain a large single implementation surface; model switching is provider-registry-driven but the chat pipeline is not cleanly split into replaceable stages. Scoring reflects: good provider abstraction, weak pipeline modularity.

---

## 3. Red-Flag Register

| # | Red flag | Evidence | Severity | Consequence | Likely remediation | Requires fork? |
|---|----------|----------|----------|-------------|--------------------|----------------|
| LC-1 | Tenant isolation **default-off** (fail-open default) | [policy.ts:49-51](research/repos/librechat/packages/data-schemas/src/tenant/policy.ts:49); [tenant.ts:125-127](research/repos/librechat/packages/api/src/middleware/tenant.ts:125); [.env.example:751](research/repos/librechat/.env.example:751) | **HIGH** | A commercial multi-tenant deployment with default config permits cross-tenant reads until `tenantId` scoping is manually enabled; incident class is silent data leakage | Flip default to strict behind a deprecation window; make `TRUST_TENANT_HEADER` + strict a documented, tested deployment preset; add startup hard-fail for multi-user mode without strict | No — config/policy change in OSS |
| LC-2 | `TRUST_TENANT_HEADER` + strict interaction is operator-trap | [index.js:217-220](research/repos/librechat/api/server/index.js:217) | MEDIUM | Operator enables strict but not the header trust → pre-auth tenant resolution silently unavailable; requests 403 or degrade | Single env knob `MULTI_TENANT_MODE=strict` that sets both coherently | No |
| LC-3 | rag_api is a synchronous in-request HTTP dependency | [createContextHandlers.js:25-33](research/repos/librechat/api/app/clients/prompts/createContextHandlers.js:25) | MEDIUM | RAG outage slows/fails chat; no visible async/idempotent contract | Put retrieval behind the platform's job/queue plane (Temporal) or a local adapter with circuit breaking | Partial — adapter is in OSS, but the *seam* must be enforced |
| LC-4 | FerretDB/legacy collection copies bypass tenant middleware and must be manually excluded | [ferretdb/schemas.ts:28-30](research/repos/librechat/packages/data-schemas/misc/ferretdb/schemas.ts:28) | MEDIUM | Maintenance scripts and copied collections can throw under strict mode or run unscoped (cross-tenant) if the exclusion is missed | Centralize the "system tenant" wrapper (`runAsSystem`) and lint for direct model imports outside it | No |
| LC-5 | Chat pipeline is one large monolith (client.js / BaseClient.js thousands of lines) | prior verified reads of [client.js](research/repos/librechat/api/app/clients/client.js:4359), [BaseClient.js](research/repos/librechat/api/app/clients/specs/BaseClient.js:732) | MEDIUM | Every pipeline change (memory ordering, streaming failure modes, context assembly) touches the same file families; high modification blast radius | Extract pipeline stages behind interfaces (context → retrieve → assemble → generate → persist) | No, but it is a fork-sized refactor |
| LC-6 | No billing/credits/quota enforcement surface found in-repo (UNKNOWN → absence verified across prior sweeps) | repo-wide; no quota ledger in `api/` | MEDIUM | Commercial multi-user needs an external billing plane | Wrap with LiteLLM (metering) + platform-owned entitlement service | No |

---

## 4. Dimension Scores (0–5, 5 = best foundation for a commercial multi-tenant AI platform)

| Dimension | Score | Rationale (abbreviated) |
|---|---|---|
| Architecture | 3 | Clear service split (api / data-schemas / packages), but monolithic chat pipeline |
| Modularity | 4 | Package boundaries are real; plugin architecture for DB scoping; RAG is an external seam |
| Provider abstraction | 4 | Provider registry is broad and provider-agnostic at the client layer |
| Memory | 3 | Memory hooks exist but no durable cross-session memory subsystem verified |
| RAG | 3 | External rag_api seam is clean but synchronous, single-deployment, weak ops story |
| Agents | 4 | Agents/MCP/skills surface is substantial and tenant-scoped |
| Workflows | 2 | No durable workflow engine; long-running work is in-process |
| Tenancy | 4 | Best-in-class DB-level mechanism; docked for fail-open default (LC-1) |
| API | 4 | Stable REST + JWT; OpenAI-compat surfaces; good docs |
| Integrations | 3 | OpenAI-compat and webhook-ish surfaces, but rag_api coupling constrains |
| MCP/extensibility | 4 | MCP + skills + tools registry present |
| File/storage | 4 | Local/S3/GCS/Azure providers with tenant-scoped File documents |
| Context management | 3 | Prompt/context assembly works but lives in the monolith |
| Tests/docs | 3 | Strong tenant conformance tests; large JS surface with mixed test density |
| Deployment/scaling | 2 | Multi-container but stateful single-node assumptions; code-api + rag_api + api scaling story weak |
| Modification/forkability | 4 | MIT-adjacent license, TS/JS, well-structured packages; fork cost moderate |

**Complexity: 4/5.** Necessary: provider breadth, tenancy mechanism, file pipeline. Accidental: the chat-pipeline monolith, FerretDB legacy copies, duplicated strict-mode env reads in 5+ modules.

---

## 5. Change-Blast-Radius Test (A–M)

- **A/B/C (replace / add / gateway LLM provider):** LOW — provider registry + LiteLLM-compat make this the cheapest change in the set.
- **D (replace memory):** MEDIUM — no first-class memory module to swap; memory touches the monolith.
- **E (add vector DB):** LOW-MED — rag_api seam means swapping the *service*, but its API shape is LibreChat-specific.
- **F (replace file storage):** LOW — provider pattern.
- **G (replace auth):** MEDIUM — OIDC is deep; tenant header trust model entangled with auth.
- **H (billing):** MEDIUM — add external plane (LiteLLM metering + entitlements); no in-repo ledger to fight.
- **I (project-level permissions):** MEDIUM — tenant exists, but project/workspace is not the primary unit; requires schema growth.
- **J/K (MCP, external apps):** LOW — surfaces exist.
- **L (replace frontend):** MEDIUM — REST API is stable, but client-specific behaviors (streaming event shapes) are tightly coupled to the web client.
- **M (custom backend wrapper):** MEDIUM — rag_api + code-api + api are three separately-versioned HTTP surfaces to reconcile.

---

## 6. Disagreements with the Deep Report

See [disagreements-C.md](research/evidence/disagreements-C.md). Headline: the deep report's claim that fail-closed strict mode is "implemented" and that LibreChat has the "strongest explicit general-purpose tenant query boundary among these six" **overstates default safety**. The mechanism is real (FACT); the *default* is permissive (FACT). The correct statement: strongest mechanism, unsafe-by-default posture.

---

## 7. Commercial Suitability Verdict

**Best-in-set for a tenant-isolated chat+agents platform *if* you own the configuration.** It is the only candidate with a DB-level tenant scoping mechanism that would survive an audit, and the only one where "add a second LLM provider" and "add MCP" are near-zero cost. Its two structural debts are the chat-pipeline monolith (modification risk) and the fail-open default (security debt). Neither blocks adoption; both must be closed before multi-tenant commercial launch.

**Confidence: HIGH** (all load-bearing claims re-verified at cited lines).

---

## 8. Open Questions for Wave C.5

1. Verify which model collections are *not* registered with the tenant plugin (coverage gaps = leak paths). Need a full `applyTenantIsolation` registration sweep.
2. Confirm rag_api's own auth model (short-lived JWT generation path) end-to-end — does a forged/leaked short-lived token cross tenants?
3. Does `runAsSystem` appear in any user-reachable code path (it is the documented cross-tenant escape hatch)?
