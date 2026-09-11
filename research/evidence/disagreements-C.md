# Wave C — Disagreements Register (Deep Reports vs Independent Audit)

Companion to [audit-summary-C.md](../reports/audit/audit-summary-C.md). Verdicts: **AGREE** / **DISAGREE** / **PARTIALLY AGREE** / **UNVERIFIED**. Every disputed claim is quoted exactly from the forensic report, then contradicted or confirmed with independently re-verified source evidence (file:line). Source repos: `research/repos/`.

---

## 1. deep-librechat.md

### D-1. "Fail-closed strict mode ... implemented" presented without its default state
- **Exact claim (line 93):** "Tenant isolation is not merely a user field: AsyncLocalStorage tenant context, query middleware, write stamping, fail-closed strict mode, aggregate filtering, and mutation guards are implemented in `tenantIsolation.ts`."
- **Verdict: PARTIALLY AGREE.**
- **Evidence:** The mechanism is real and exactly as described: `tenantId` absent + `TENANT_ISOLATION_STRICT=true` → throws; absent + strict off → "passes through (transitional/pre-tenancy)" ([tenantIsolation.ts:96-104](../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96)); hooks cover find/findOne/distinct/update/delete/count/replace ([tenantIsolation.ts:148-159](../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:148)).
- **What the report omits:** strictness is *not* the default. `isTenantIsolationStrict()` is `process.env.TENANT_ISOLATION_STRICT === 'true'` with malformed values "defaulting to non-strict mode" ([policy.ts:48-63](../repos/librechat/packages/data-schemas/src/tenant/policy.ts:48)); the request middleware documents "Non-strict (default) → passes through with user/request context only" ([tenant.ts:125-127](../repos/librechat/packages/api/src/middleware/tenant.ts:125)); `.env.example` ships `# TENANT_ISOLATION_STRICT=false` commented out ([.env.example:749-751](../repos/librechat/.env.example:749)); and the project's *own* e2e fixtures run `TENANT_ISOLATION_STRICT: 'false'` ([record.js:53](../repos/librechat/e2e/setup/record.js:53), [playwright.config.mock.ts:111](../repos/librechat/e2e/playwright.config.mock.ts:111)).
- **Consequence of the omission:** a reader concludes LibreChat is safe-by-default. It is safe-by-configuration. For a commercial multi-tenant deployment this is a HIGH-severity operational gap (red flag LC-1), not a footnote.

### D-2. "strongest explicit general-purpose tenant query boundary among these six"
- **Exact claim (line 97):** "LibreChat has the strongest explicit general-purpose tenant query boundary among these six, while user/resource ACLs remain a second independent authorization layer."
- **Verdict: AGREE (mechanism), with the D-1 caveat attached.** Among the five audited, no other repo implements DB-level query scoping at all (Dify/RAGFlow/Open WebUI/LobeChat all enforce in application code at call sites). The superlative stands *for the mechanism*; it does not extend to default safety.

### D-3. Modification-test "Custom OpenAI-compatible provider | 1"
- **Exact claim (line 122):** score 1, "Custom endpoint/OpenAI client boundary exists."
- **Verdict: UNVERIFIED (low materiality).** The report scores *effort to add* (1 = cheap), consistent with my provider-abstraction score of 4; no factual conflict, noting only that the 0–5 scale here is inverted vs my dimension scale.

### D-4. "UNKNOWN: ... Storage quota policy exists in pieces but aggregate tenant quota enforcement needs product work"
- **Exact claim (line 107).**
- **Verdict: AGREE.** My independent sweep found no quota/entitlement enforcement surface in-repo; consistent.

---

## 2. deep-dify.md

### D-5. "Usage credits 1 — Credit pools and quota service already exist"
- **Exact claim (line 117):** "Usage credits | 1 | Credit pools and quota service already exist." (Supporting FACT at line 95: "Credit pools and quota reservation/commit/release are implemented through `QuotaService` and `tenant_credit_pools` in `model.py`.")
- **Verdict: DISAGREE (framing overstates availability).**
- **Evidence:** The lifecycle code exists, but it is gated and fail-open. `QuotaService.reserve` begins with: `if dify_config.DEPLOYMENT_EDITION != DeploymentEdition.CLOUD: logger.debug("Quota billing is unavailable outside the Cloud edition; allowing request for %s", tenant_id); return QuotaCharge(success=True, charge_id=None, ...)` ([quota_service.py:121-123](../repos/dify/api/services/quota_service.py:121)). The class docstring says it "Orchestrates quota reserve / commit / release lifecycle via BillingService" — a proprietary billing backend ([quota_service.py:91-92](../repos/dify/api/services/quota_service.py:91)).
- **Consequence:** "already exist" implies a self-host operator gets quota enforcement. They get the opposite: *no* enforcement, by explicit code path, outside Dify Cloud. The honest statement: "a well-shaped quota API exists that refuses to run outside the vendor's cloud." A commercial forker must either stand up a compatible billing backend or redirect this class (~1 class, narrow seam, but proprietary contract).

### D-6. The report's central RBAC finding is **absent**
- **Exact claim (line 89):** "Isolation is genuine workspace/tenant isolation, but enforcement is application-level and distributed rather than one universal database RLS policy."
- **Verdict: PARTIALLY AGREE — the report is true but materially incomplete, and the omission is the single most important Dify fact for commercial use.**
- **Evidence (not in the report):** with `RBAC_ENABLED=true`, *every* legacy permission predicate on the account object unconditionally returns `True`: `is_admin_or_owner` ([account.py:192-194](../repos/dify/api/models/account.py:192)), `is_admin` ([198-200](../repos/dify/api/models/account.py:198)), `has_edit_permission` ([226-227](../repos/dify/api/models/account.py:226)), `is_dataset_editor` ([231-233](../repos/dify/api/models/account.py:231)), `is_dataset_operator` ([237-239](../repos/dify/api/models/account.py:237)). The fine-grained checks delegate to a **closed enterprise service**: "ENTERPRISE_RBAC_API_URL is required when RBAC_ENABLED=true" ([enterprise/base.py:193-194](../repos/dify/api/services/enterprise/base.py:193)); the open-side checks no-op when RBAC is off ([checks.py:38-40](../repos/dify/api/controllers/common/rbac/checks.py:38)). Invited members are forced to `NORMAL` role under RBAC ([account_service.py:1833](../repos/dify/api/services/account_service.py:1833)).
- **Consequence:** enabling the advertised RBAC flag on self-host (a) turns every legacy gate permissive in code paths not re-wired to `RBACService`, and (b) makes all real enforcement depend on a non-open-source endpoint. The fine-grained RBAC Dify advertises is a commercial feature; the OSS repo ships only call sites. The report's line-89 sentence ("enforcement is application-level and distributed") is true for the *legacy* path but never mentions the RBAC path at all.

### D-7. "Isolation is genuine workspace/tenant isolation" (affirmative half of line 89)
- **Verdict: AGREE.** Tenant model + tenant-scoped records re-confirmed (e.g., tenant-scoped credential resolution; role-gated dataset permission checks at [datasets.py:759-761](../repos/dify/api/controllers/console/datasets/datasets.py:759)).

---

## 3. deep-ragflow.md

### D-8. "Usage credits 4 — Billing ledger absent from evidence"
- **Exact claim (line 126):** "Usage credits | 4 | Billing ledger absent from evidence; cross-cuts tenant/provider/jobs." (Supporting UNKNOWN at line 105: "Complete credit/billing ledger, reservation/commit semantics ... not established.")
- **Verdict: DISAGREE (direction and severity).**
- **Evidence:** the ledger is not merely "absent from evidence" — it is **present as dead code**. `Tenant.credit = IntegerField(default=512, index=True)` ([db_models.py:1163](../repos/ragflow/api/db/db_models.py:1163)); `TenantService.decrease(cls, user_id, num)` is defined ([user_service.py:219-222](../repos/ragflow/api/db/services/user_service.py:219)) but a repo-wide search for `.decrease(` returns **zero call sites** — the credit is never decremented, never checked, never exposed.
- **Consequence:** a "4" (reading: cheap to add credits) misrepresents the situation; there is a vestigial fake credit system that misleads stakeholders into believing metering exists. Correct score: **0** (nothing to hook into; build entirely at the platform plane).

### D-9. "genuine tenant/workspace isolation stronger than user-only ownership"
- **Exact claim (line 95).**
- **Verdict: PARTIALLY AGREE, overstated.**
- **Evidence for:** per-tenant index naming (`search.index_name(tenant_id)` at [document_api.py:1767](../repos/ragflow/api/apps/restful_apis/document_api.py:1767), [chunk_api.py:249](../repos/ragflow/api/apps/restful_apis/chunk_api.py:249); `f"ragflow_doc_meta_{tenant_id}"` at [doc_metadata_service.py:82](../repos/ragflow/api/db/services/doc_metadata_service.py:82)); `KnowledgebaseService.accessible()` ownership checks ([knowledgebase_service.py:569-590](../repos/ragflow/api/db/services/knowledgebase_service.py:569)).
- **Evidence against (not in the report):** (a) **IDOR** — `GET /documents/images/<image_id>` is guarded only by `@login_required(auth_types=[AUTH_JWT, AUTH_API, AUTH_BETA])` and returns raw storage bytes with **no ownership/tenant check** ([document_api.py:1831-1866](../repos/ragflow/api/apps/restful_apis/document_api.py:1831)), while the sibling artifact endpoint *does* check ownership (`_sandbox_artifact_accessible`, [document_api.py:1900-1905](../repos/ragflow/api/apps/restful_apis/document_api.py:1900)) — proving the pattern existed and was skipped. (b) In practice `tenant_id == user.id` for self-created tenants, so "workspace" is a weak term.
- **Consequence:** "genuine ... stronger than user-only ownership" overstates an enforced boundary that is per-site-invoked and demonstrably breached at least once.

### D-10. Concurrency/scale claims
- **Exact claim (line 105, UNKNOWN):** "production worker scaling guarantees" not established.
- **Verdict: AGREE, and I can be stronger (FACT):** concurrency control is per-process — `task_limiter = LoopLocalSemaphore(5)`, chunk/embed = 1, minio = 10, kg = 2 ([task_executor_limiter.py:20-28](../repos/ragflow/rag/svr/task_executor_limiter.py:20)); chat limiter 10 per process ([graphrag/utils.py:42](../repos/ragflow/rag/graphrag/utils.py:42)). Scaling task executors horizontally multiplies real concurrency with no per-tenant admission control. This upgrades the UNKNOWN to a quantified risk (red flag RF-4).

---

## 4. deep-open-webui.md

### D-11. "Custom OpenAI-compatible provider | 0"
- **Exact claim (line 118):** "Custom OpenAI-compatible provider | 0 | Base URL/key configuration already exists in `config.py`." (Score 0 read as "trivial/already possible"; the rationale sentence contradicts the usual scale reading — the claim as *written* is incoherent and I treat it as "already supported, no work".)
- **Verdict: PARTIALLY AGREE / DISAGREE on the number.**
- **Evidence:** a full OpenAI-compatible provider path exists: `utils/openai.py` builds request targets with per-user `OPENAI_API_BASE_URL`/key overrides ([openai.py:320-345](../repos/open-webui/backend/open_webui/utils/openai.py:320)); model records expose OpenAI-compatible base URLs ([config.py:319](../repos/open-webui/backend/open_webui/config.py:319)). So the capability is real and first-class. The report's own rationale ("Base URL/key configuration already exists") supports a score of *already-supported*, which its "0" apparently encodes — the register is inconsistent with the other reports' scale (where 1 = cheap). Net: **the underlying fact agrees; the score is mis-scaled or mis-keyed.**

### D-12. "no demonstrated universal tenant_id boundary"
- **Exact claim (line 93).**
- **Verdict: AGREE (confirmed), sharpened.** No tenant unit exists (roles pending/user/admin at [users.py:45](../repos/open-webui/backend/open_webui/models/users.py:45); AccessGrants + groups as the only isolation-ish primitives). **New finding the report missed:** the one *named* multitenancy feature is self-documented as corruption-prone — `MilvusClient._get_collection_and_resource_id` "WARNING: This mapping relies on current Open WebUI naming conventions ... this mapping will break and route data to incorrect collections. POTENTIALLY CAUSING HUGE DATA CORRUPTION ..." ([milvus_multitenancy.py:84-106](../repos/open-webui/backend/open_webui/retrieval/vector/dbs/milvus_multitenancy.py:84)). The report's "UNKNOWN: ... production database isolation behavior" is answered negatively by the vendor's own code comment.

### D-13. "Usage credits 3 — Analytics exist; no established credit ledger/reservation path was found"
- **Exact claim (line 124).**
- **Verdict: PARTIALLY AGREE (fact correct, score high).** No ledger/reservation path exists — confirmed; the only usage surface is session-level `/api/usage` ([main.py:2587-2609](../repos/open-webui/backend/open_webui/main.py:2587)). For *enforcement* the score should be 1 (reporting exists, control does not), matching how LibreChat/LobeChat/Dify are scored in this audit.

### D-14. Access-control maturity
- **Verdict: AGREE + positive finding.** The report notes "ownership/role checks are distributed" (line 110). I confirm distributed enforcement **and** found the team has explicitly fixed a privilege-laundering class with a documented CWE-863 defense in `has_access_to_file` ("a read-only file laundered into an object the user controls would gain write/delete on it (CWE-863)", [files.py:45-63](../repos/open-webui/backend/open_webui/utils/access_control/files.py:45)) — evidence of security maturity the report did not capture.

---

## 5. deep-lobechat.md

### D-15. Architecture description is stale
- **Exact claims (lines 95-97):** "Workspace existence/membership is checked server-side by `workspace.ts` in `src/app/(backend)/webapi/_utils/workspace.ts`. Chat/model/document/agent routes consume authenticated identity and workspace context." The report's modification tests and UNKNOWNs (lines 109, 132-133, 151) treat "usage-credit ledger, billing entitlement, storage quota aggregation" as **UNKNOWN**.
- **Verdict: DISAGREE (stale).**
- **Evidence:** the repo is now a pnpm monorepo (`apps/` + ~75 `packages/`), with a new `apps/server` (Hono + tRPC) backend that coexists with the legacy `src/app/(backend)` surface (both are built into the Docker image — [Dockerfile](../repos/lobechat/Dockerfile:1) removes only `src/app/desktop` and the desktop trpc route, not the rest). The report's anchors (`src/app/(backend)/...`) describe the *old* architecture.
- **The UNKNOWNs are resolved:** the repo contains `packages/business-server`, a typed stub layer mapped by the `@/business/server/*` tsconfig alias ([tsconfig.json:28](../repos/lobechat/tsconfig.json:28)):
  - workspace creation is `cloudOnly` → throws `NOT_IMPLEMENTED` ("Workspace creation is a cloud-only feature"); `list` → `[]`; `getById` → `null` ([workspace.ts:46-82](../repos/lobechat/packages/business-server/src/lambda-routers/workspace.ts:46));
  - workspace usage stub returns `remainingBalance: 0` with comment "Cloud overrides this at the same path with the real workspaceUsageRouter" ([workspaceUsage.ts:14-24](../repos/lobechat/packages/business-server/src/lambda-routers/workspaceUsage.ts:14));
  - `spendRouter = router({})` ([spend.ts:3](../repos/lobechat/packages/business-server/src/lambda-routers/spend.ts:3)); file storage checks are empty no-ops ([file.ts:13-25](../repos/lobechat/packages/business-server/src/lambda-routers/file.ts:13)); `chargeBeforeGenerate` returns `undefined` ([chargeBeforeGenerate.ts:41-43](../repos/lobechat/packages/business-server/src/image-generation/chargeBeforeGenerate.ts:41)); the agent-share spend gate always returns `{ allowed: true }` ([spendGate.ts:39-61](../repos/lobechat/packages/business-server/src/agent-share/spendGate.ts:39)).
  - Meanwhile usage *recording* is real OSS: `UsageRecordService` over `messages.usage` with workspace scoping ([usage/index.ts:25-367](../repos/lobechat/apps/server/src/services/usage/index.ts:25)).
- **Consequence:** "UNKNOWN: billing/credits/quotas" is no longer an unknown — it is **closed-source by design**, delivered as typed contracts the cloud repo overrides. This converts LobeChat's adoption model from "adopt and add features" to "fork and privately rewrite the business plane." The report's modification scores ("Usage credits 3", "Storage quotas 3", "Independent backend/API in front 2") were assessed against the wrong architecture; against the actual one they should read: credits-enforcement **0** (stubbed, cloud-only), storage-quota **0** (stubbed), independent-backend **3** (`apps/server` is a genuinely standalone Hono/tRPC service).

### D-16. "Public file preview intentionally omits user filtering by ID" (line 97)
- **Verdict: AGREE (confirmed as designed), with the report's own caveat ("requires separate threat modeling") standing as a C.5 item.** Shared-link file previews remain an explicit ownership exception in the new server surface too.

### D-17. "Forks that change auth/workspace/database/context-engine semantics face high rebase cost" (line 142)
- **Verdict: AGREE, and it is understated.** The rebase cost now includes tracking a *closed* module's contract (the business-stub layer) — the fork's heaviest new surface is precisely the one that will drift against an implementation you cannot see.

---

## 6. Cross-report pattern findings (no single report owns these)

| # | Pattern | Verdict on collective reporting |
|---|---|---|
| P-1 | **All five repos gate their commercial capabilities out of OSS** (Dify: enterprise RBAC + Cloud billing; LobeChat: business-server stubs; LibreChat/Open WebUI/RAGFlow: absence). No deep report states this as a unifying finding; each treated its subject's gap as local. **NEW.** |
| P-2 | **No candidate carries tenant context uniformly to subsystem edges** (vector/LLM/MCP calls re-derive scoping per layer). A3's "tenant propagation" risk is confirmed empirically in all five. |
| P-3 | **The one live cross-tenant data-exposure class found is an unguarded storage-read endpoint** (RAGFlow image IDOR). The sibling-checked-artifact pattern in the same file shows the fix is known but not applied repo-wide; a full sweep is required (C.5 item 1). |

---

## 7. Unresolved items carried to Wave C.5

1. LobeChat JWKS key in [deploy/.env.example:69](../repos/lobechat/docker-compose/deploy/.env.example:69) — liveness UNKNOWN (no public counterpart located in-repo).
2. Dify — exhaustive `if dify_config.RBAC_ENABLED` site enumeration (representative set verified; full list not audited).
3. RAGFlow — does the image-IDOR pattern recur at other storage-read endpoints?
4. Open WebUI — Qdrant multitenancy mode fragility check (milvus analogue unverified); multi-replica in-process task duplication.
5. LibreChat — tenant-plugin collection coverage gaps; user-reachable `runAsSystem` paths.
6. LobeChat — complete census of `@/business/server/*` import sites; workspace-scoping coverage in `apps/server`.
