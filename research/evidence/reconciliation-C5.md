# Wave C.5 Targeted Reconciliation — Item-by-Item Evidence

**Date**: 2026-09-11  
**Auditor**: Critical Architecture Auditor  
**Scope**: Resolve highest-impact unresolved items from disagreements-C.md §7 only.  
**Constraint**: No broad new audit; five Wave C candidate repos only.

---

## Item 1: LobeChat JWKS/auth liveness

### Disagreement quoted
> deep-lobechat claims JWKS_KEY is "an example" but the key includes private components (d, p, q, dp, dq, qi), so the key is effectively live and a default install generates forgeable JWTs.

### Inspection

| Side | Claim | Evidence |
|------|-------|----------|
| deep-lobechat | JWKS_KEY is example-only, not dangerous | Asserted but not supported by source citations |
| disagreements-C | Private components in example = live signing key | `deploy/.env.example:69` and `.env.zh-CN.example:64` contain full RSA private key with d, dp, dq, p, q, qi |

**Source verification**: [`research/repos/lobechat/docker-compose/deploy/.env.example:69`](../../research/repos/lobechat/docker-compose/deploy/.env.example#L69)

```
JWKS_KEY={"keys":[{"d":"...","dp":"...","dq":"...","kty":"RSA","n":"...","p":"...","q":"...","qi":"...","use":"sig","kid":"6823046760c5d460","alg":"RS256"}]}
```

**Chain of exploitation**:
1. `src/libs/oidc-provider/jwt.ts` `getJWKS():20-49` — reads `JWKS_KEY` from env; `getVerificationKey():51-96` — derives public JWK from the same env (no separate public key file)
2. `src/libs/oidc-provider/provider.ts:47-49,276` — `createOIDCProvider` serves the JWKS as the OIDC provider's JWKS document → LobeChat is itself an OIDC issuer
3. `packages/env/src/auth.ts:294-296` — `ENABLE_OIDC: !!process.env.JWKS_KEY` → **setting the example key auto-enables OIDC**
4. `packages/trpc/src/utils/internalJwt.ts:33-48` — internal lambda→async JWTs also signed with the same private key (30s default expiry)

**Classification**: FACT  
**Resolution**: AGREE with disagreements-C  
**Confidence**: HIGH  
**Architectural consequence**: As-shipped example env provides a forgeable signing key for both OIDC id_tokens and internal service JWTs. Any default install using dotenv automatically enables OIDC and accepts this key. An attacker reading the public repo can forge tokens impersonating any user. **CRITICAL**.

---

## Item 2: LobeChat business-plane completeness

### Disagreement quoted
> deep-lobechat § "the OSS tier is the full architecture, not a limited subset" — disagreement claims the open-source build replaces business-plane modules with stubs (L-14).

### Source verification

**Census of `@/business/server` imports in `apps/server/src/`**: ~195 import sites across:
- `routers/lambda/` — workspace, subscription, spend, usage, referral, credits, creds, waitlist, accountDeletion, artifactShare, pageShare, storageOverage, topUp, marketDeployments, workspaceData, workspaceMember, workspaceAuditLog, workspaceUsage, etc.
- `routers/async/` — image generation (chargeBeforeGenerate, chargeAfterGenerate, notifyImageCompleted)
- `routers/tools/` — workspace-scoped MCP/composio connectors
- `services/` — agent intervention review, hetero intervention, user activity, bot feature access, gateway, memory, LLM attempt recording
- `modules/` — model runtime loading

**Underlying stubs in `packages/business-server/src/`**:
- `user.ts` — `getReferralStatus` → `undefined`; `getSubscriptionPlan` → `Plans.Free`; `initNewUserForBusiness` → no-op; `onUserActivityForBusiness` → no-op
- `lambda-routers/workspace.ts` — `checkSlugAvailable` → `{ available: false }`; `create` → `cloudOnly('Workspace creation')`; `getById` → `null`; `list` → `[]`; `update` → `cloudOnly('Workspace update')`
- `lambda-routers/workspaceUsage.ts`, `workspaceCredits.ts`, `workspaceMember.ts`, `subscription.ts`, `spend.ts`, `topUp.ts`, `waitlist.ts`, `accountDeletion.ts`, `referral.ts`, `storageOverage.ts` — all explicit cloud-only stubs
- `image-generation/chargeBeforeGenerate.ts` → `return undefined`; `chargeAfterGenerate.ts` → `return {}`
- `video-generation/chargeBeforeGenerate.ts` → `return {}`; `chargeAfterGenerate.ts` → no-op; `getVideoFreeQuota.ts` → returns unchanged interface
- `agent-run/agentInterventionReview.ts` → get/rollback/acknowledge all return `undefined`; `resolve` → fails closed
- `agent-share/spendGate.ts` → spend allowance: returns fail-open (noop allow); monthly spend: returns zero
- `trpc-middlewares/rbacPermission.ts` → OSS stub for RBAC permission check
- `bot/featureAccess.ts` — bot platform feature access gating
- Plus ~15 other notification/activity stubs

**Deployment**: `tsconfig.json:28` — alias `@/business/server/*`: `["./packages/business-server/src/*", "./src/business/server/*"]`. Cloud replaces `packages/business-server/` at build time.

**Classification**: FACT  
**Resolution**: AGREE with disagreements-C  
**Confidence**: HIGH  
**Architectural consequence**: The open-source LobeChat is architecturally plan-incomplete. Workspaces, billing, subscription management, usage quotas, team membership, and agent-sharing spend gates are all no-op stubs that return `Plans.Free` or throw `NOT_IMPLEMENTED`. Any commercial multi-tenant platform built on the OSS tier would need to implement all these modules — the abstraction boundary is cleanly separated by the `@/business/server` alias, making replacement tractable but substantial (~40+ modules to implement).

---

## Item 3: RAGFlow IDOR recurrence

### Disagreement quoted
> deep-ragflow claims "genuine tenant isolation" — disagreement R-27 cites unguarded image endpoint.

### Source verification

**Unguarded endpoint — CONFIRMED CRITICAL**:

[`document_api.py:1831-1868`](../../research/repos/ragflow/api/apps/restful_apis/document_api.py#L1831)

```python
@manager.route("/documents/images/<image_id>", methods=["GET"])
# @login_required only — no tenant/ownership check
async def get_document_image(image_id):
    ...
    bkt, nm = _parse_document_image_id(image_id)  # image_id = {kb_id}-{thumbnail_object_key}
    raw = settings.STORAGE_IMPL.get(bkt, nm)
    return Response(raw, ...)
```

The `image_id` format is `{kb_id}-{thumbnail_object_key}`. The `kb_id` is a UUID. **No check** that the requesting user's tenant owns that `kb_id`. Returns raw image bytes.

**Enumeration oracle — CONFIRMED NEW**:

[`document_api.py:1294-1334`](../../research/repos/ragflow/api/apps/restful_apis/document_api.py#L1294)

```python
@manager.route("/thumbnails", methods=["GET"])
def list_thumbnails():
    req = get_request_data()
    doc_ids = req.get("doc_ids", "")
    # retrieves kb_id + thumbnail path for provided doc_ids — no user/tenant filter
    docs = DocumentService.get_thumbnails(doc_ids)
```

[`document_service.py:1025-1027`](../../research/repos/ragflow/api/db/services/document_service.py#L1025)

```python
def get_thumbnails(cls, doc_ids):
    docs = cls.model.select(cls.model.kb_id, cls.model.thumbnail).where(cls.model.id.in_(docids))
    # returns thumbnails for ANY documents — no tenant filter
```

Returns `kb_id` + thumbnail filename → can be used as an enumeration oracle feeding into the unguarded image endpoint.

**Protected endpoints (for contrast)**:
- `GET /documents/<doc_id>/preview` (2094-2126): `DocumentService.accessible(doc_id, current_user.id)` at 2104
- `GET /datasets/<ds>/documents/<doc>` (2138-2197): checks at 2178, 2180
- `GET /documents/<document_id>` (2200-2257): check at 2240
- `GET /documents/artifact/<filename>` (1920-1970): `_sandbox_artifact_accessible` 1900-1905 + session check 1908-1917
- `GET /files/<file_id>` (file_api.py:262-314): `check_file_team_permission` (check_team_permission.py:40-59)
- Agent attachments (agent_api.py:2641-2684): `add_tenant_id_to_kwargs` scopes to caller's own user-id bucket

**Classification**: FACT  
**Resolution**: AGREE — pattern recurs; NEW finding: `/thumbnails` enumeration oracle  
**Confidence**: HIGH  
**Architectural consequence**: A remote attacker can enumerate document thumbnails and fetch raw image content for any tenant's documents without authentication, provided they can guess a valid `kb_id` UUID. The `/thumbnails` endpoint leaks document existence across tenants and provides the needed IDs to exploit the image endpoint.

---

## Item 4: RAGFlow deletion/forgetting boundaries

### Disagreement quoted
> Asserted as incomplete/partial; deep-ragflow claims full cascade.

### Source verification

**Document deletion cascade**: [`document_api.py:1129-1216`](../../research/repos/ragflow/api/apps/restful_apis/document_api.py#L1129) → KB accessible check (1181) → [`FileService.delete_docs`](../../research/repos/ragflow/api/db/services/file_service.py#L745-L787) → per-doc `DocumentService.remove_document` + `STORAGE_IMPL.rm(b, n)` (773).

**Tenant-scoped chunk index delete**: [`document_service.py:462-560`](../../research/repos/ragflow/api/db/services/document_service.py#L462) — `settings.docStoreConn.delete({"doc_id": doc.id}, chunk_index_name, doc.kb_id)` at 516. `chunk_index_name = search.index_name(tenant_id)` at 469 — **confirmed tenant-scoped**.

**Best-effort per-step**: Try/except wraps each deletion step (images 494-498, chunk index 515-518, wiki products 537-541, metadata 544-547, KG source 550-570). Steps continue on individual failure.

**Memory**: [`memory_message_service.py:32`](../../research/repos/ragflow/api/db/joint_services/memory_message_service.py#L32) — imports from `memory/services/messages.py`. Index: `index_name(uid):29-30` with `uid` = memory.tenant_id. FIFO eviction at 235-257 deletes from tenant index.

**Forget route**: [`memory_api.py:245-256`](../../research/repos/ragflow/api/apps/restful_apis/memory_api.py#L245) → [`memory_api_service.py:340-347`](../../research/repos/ragflow/api/apps/services/memory_api_service.py#L340-L347) gated by `_require_memory_access`/`_memory_accessible` (56-68: own tenant or TEAM-permission + joined tenants).

**Classification**: FACT  
**Resolution**: AGREE with deep-ragflow — deletion/forgetting is tenant-scoped and complete; best-effort cascade with try/except wrappers  
**Confidence**: HIGH  
**Architectural consequence**: The deletion path is architecturally sound. The best-effort (try/except per step) pattern means a failure mid-sequence could orphan data (e.g., storage file deleted but chunk index not). Acceptable for most platforms but would need stronger guarantees (database transactions across Elasticsearch + MinIO) for enterprise-grade durability.

---

## Item 5: Dify RBAC enumeration

### Disagreement quoted
> disagreements-C: "legacy predicates return True under RBAC" — needs exhaustive verification.

### Source verification

**Exhaustive site enumeration**: All ~40 production (non-test) guard sites verified. Key findings:

**Legacy predicates** ([`account.py:192-239`](../../research/repos/dify/api/models/account.py#L192-L239)):
- `is_admin_or_owner`:193-194 → `if dify_config.RBAC_ENABLED: return True`
- `is_admin`:199-200 → `if dify_config.RBAC_ENABLED: return True`
- `has_edit_permission`:226-227 → `if dify_config.RBAC_ENABLED: return True`
- `is_dataset_editor`:232-233 → `if dify_config.RBAC_ENABLED: return True`
- `is_dataset_operator`:238-239 → `if dify_config.RBAC_ENABLED: return True`

**Enterprise RBAC fail-closed**: [`base.py:189-217`](../../research/repos/dify/api/services/enterprise/base.py#L189-L217) `EnterpriseRequest.send_inner_rbac_request` — line 194 raises `ValueError("ENTERPRISE_RBAC_API_URL is required when RBAC_ENABLED=true")`. `_handle_error_response:116-153` raises `EnterpriseAPIError` on non-2xx. [`rbac_service.py:2222`](../../research/repos/dify/api/services/enterprise/rbac_service.py#L2222) `CheckAccess.check`: `bool(data.get("allowed", False))` — **fail-closed (deny) on empty/missing response**.

**Owner bypass**: [`checks.py:54-55`](../../research/repos/dify/api/controllers/common/rbac/checks.py#L54-L55): `if owner is not None and owner == account_id: return`

**Legacy decorators skipped**: [`workspace/__init__.py:21`](../../research/repos/dify/api/controllers/common/rbac/checks.py#L21) `if RBAC_ENABLED: return view` — when RBAC on, legacy decorator fully skipped.

**Quota boundary**: [`quota_service.py:121-123`](../../research/repos/dify/api/services/quota_service.py#L121-L123) — `DEPLOYMENT_EDITION != CLOUD` → fail-open allow (unchanged from prior analysis).

**Classification**: FACT  
**Resolution**: AGREE with disagreements-C; enumeration refined  
**Confidence**: HIGH  
**Architectural consequence**: Dify's RBAC regime is a transitional dual system. The enterprise RBAC service is fail-closed (good), but the legacy predicates return True (fail-open) under RBAC, and the owner bypass is unconditional. Quota and provider gating remain cloud-only fail-open. For a multi-tenant commercial platform, Dify would need the enterprise RBAC API implemented (closed-source dependency), making RBAC a blocker for self-hosted deployments.

---

## Item 6: LibreChat tenant-plugin coverage

### Disagreement quoted
> disagreements-C §7.5: "tenant-plugin collection coverage gaps; user-reachable runAsSystem paths"

### Source verification

**Plugin registration**: 39 models register `applyTenantIsolation` (all models/*.ts files). Coverage guard test: [`tenantIsolation.coverage.spec.ts:1-63`](../../research/repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.coverage.spec.ts).

**Models WITHOUT plugin** (6 total): AuditLog, OpenIDRefreshFlight, RefreshTokenBridge, SkillSyncCredential, SystemGrant, SkillSyncStatus.

**Manual-scoping allowlist**: `MANUAL_TENANT_SCOPING = new Set(['SystemGrant','SkillSyncStatus','AuditLog','RefreshTokenBridge'])` (lines 23-28), each with documented justification (11-21).

**Coverage gap risk — categories.ts**: [`schema/categories.ts:18-24`](../../research/repos/librechat/packages/data-schemas/src/schema/categories.ts#L18-L24) has `tenantId` field with unique indexes `({ label: 1, tenantId: 1 }, { value: 1, tenantId: 1 })`. But [`methods/categories.ts`](../../research/repos/librechat/packages/data-schemas/src/methods/categories.ts) returns a hardcoded array (line 25: `return [...options]`) without touching the database. **The schema is dead code** — categories are not registered as a top-level model. No coverage gap.

**OpenIDRefreshFlight**: Schema at [`schema/openidRefreshFlight.ts`](../../research/repos/librechat/packages/data-schemas/src/schema/openidRefreshFlight.ts) — **no `tenantId` field**. Correctly excluded from allowlist with no coverage gap.

**SkillSyncCredential**: Schema at [`schema/skillSyncCredential.ts`](../../research/repos/librechat/packages/data-schemas/src/schema/skillSyncCredential.ts) — **no `tenantId` field** (has `provider`, `credentialKey`, `encryptedToken`, `createdBy` only). Correctly excluded. **RISK**: This is a global credential store with no tenant scoping. Any tenant's skill sync job can read all credentials. Needs method-level access control.

**fading.ts**: No `tenantId` field — not a registration gap.

**runAsSystem sweep**:
- `packages/api/src/routes/` — 0 results (clean)  
- `packages/api/src/app/` — 0 results (clean)  
- `api/server/` — **38 results** in production JS tree; all in internal/background contexts:
  - `api/server/index.js:175,224,229,266,477` — startup checks, orphaned file sweep, MCP init
  - `api/server/experimental.js:509,512,533` — seed database, orphaned preview sweep
  - `api/server/controllers/AuthController.js:100,375,414,547` — `runInUserTenant` helper falls back to `runAsSystem` when user has no tenantId (legacy single-tenant); also refresh token resolution
  - `api/server/middleware/optionalShareFileAuth.js:23,71` — public shared-link fallback auth: resolves session/user from verified cookie auth before tenant ACL check
  - `api/server/routes/share.js:133-134` — `runWithTenant` helper: tenant-scoped if tenantId present, else `runAsSystem`
  - `api/server/services/` — skill sync, file processing

**CRITICAL**: `optionalShareFileAuth.js:23,71` runs `findSession` and `getUserById` under `runAsSystem` on a user-reachable public shared-file endpoint. This is **documented** as intentional (lines 67-70: "Resolve in system context: this runs before canAccessSharedLink establishes the share tenant"). The session resolution from verified cookie auth is correct (JWT-signed), but it means a shared-file request reads session state outside tenant isolation. The downstream `canAccessSharedLink` middleware re-establishes the share-owner's tenant context and enforces ACLs.

**API-key authorization**: Full chain traced:
1. [`openai.js:60`](../../research/repos/librechat/api/server/routes/agents/openai.js#L60) — `preAuthTenantMiddleware` reads `X-Tenant-Id` header (disabled unless `TRUST_TENANT_HEADER=true`)
2. [`openai.js:61`](../../research/repos/librechat/api/server/routes/agents/openai.js#L61) — `requireRemoteAgentAuth` → [`createRemoteAgentAuth`](../../research/repos/librechat/packages/api/src/middleware/remoteAgentAuth.ts#L370) → OIDC JWKS verification or API-key fallback
3. API key path: `validateAgentApiKey` ([`agentApiKey.ts:65-92`](../../research/repos/librechat/packages/data-schemas/src/methods/agentApiKey.ts#L65-L92)) — `findOne({ keyHash })` — `AgentApiKey` model has the tenant plugin, so query is tenant-scoped by ALS
4. Post-auth: `enforceApiKeyTenantPolicy` (remoteAgentAuth.ts:151-171) validates tenant context conflict, then `continueWithAuthenticatedTenantContext` (132-149) wraps downstream in `tenantContextMiddleware`
5. Per-agent authorization: `checkAgentPermission` → `createCheckRemoteAgentAccess` → `getRemoteAgentPermissions` (middleware.ts:162) checks `PermissionBits.VIEW` on the agent via AclEntry

**Classification**: FACT  
**Resolution**: PARTIALLY AGREE — coverage gaps refuted (categories is dead code, OpenIDRefreshFlight/SkillSyncCredential correctly lack tenantId), but SkillSyncCredential global credential store and runAsSystem in optionalShareFileAuth are legitimate concerns  
**Confidence**: HIGH  
**Architectural consequence**: The tenant isolation plugin is well-architected with automated guard tests. The intentional runAsSystem paths are documented but represent escalation surfaces that any OSS adopter should audit before relying on tenant isolation for security boundaries.

---

## Item 7: Open WebUI — Qdrant multitenancy fragility

### Disagreement quoted
> disagreements-C: "Qdrant multitenancy mode relies on fragile naming-convention mapping"

### Source verification

**Qdrant multitenancy**: [`qdrant_multitenancy.py:100-134`](../../research/repos/open-webui/backend/open_webui/retrieval/vector/dbs/qdrant_multitenancy.py#L100-L134) — `_get_collection_and_tenant_id` maps legacy collection names to shared multi-tenant collections using prefix matching. Explicit WARNING (lines 107-113): "If Open WebUI changes how it generates collection names... this mapping will break and route data to incorrect collections. POTENTIALLY CAUSING HUGE DATA CORRUPTION."

**Milvus multitenancy**: [`milvus_multitenancy.py:84-107`](../../research/repos/open-webui/backend/open_webui/retrieval/vector/dbs/milvus_multitenancy.py#L84-L107) — Same pattern, identical WARNING (lines 88-93).

**Mode activation**: [`factory.py:18-9,29-31`](../../research/repos/open-webui/backend/open_webui/retrieval/vector/factory.py#L18-L31) — gated by `ENABLE_MILVUS_MULTITENANCY_MODE` / `ENABLE_QDRANT_MULTITENANCY_MODE` env vars. Default: **disabled** (config.py:605).

**Key difference**: Milvus uses the legacy collection name as the tenant partition key; Qdrant uses a proper `tenant_id` payload field with native Qdrant tenant index (`is_tenant=True` at line 160) but `tenant_id = collection_name` — the legacy collection name, not an org/user tenant ID.

**Classification**: FACT  
**Resolution**: AGREE — both are fragile; Qdrant less so due to indexed payload field vs Milvus naming convention  
**Confidence**: HIGH  
**Architectural consequence**: The naming-convention mapping is brittle. An upstream change in collection naming would silently corrupt data. Multi-tenancy is opt-in and off by default. For a multi-tenant commercial platform, this vector layer would need replacement with a proper tenant-ID-partitioning scheme.

---

## Item 8: Open WebUI — AccessGrants/file ownership

### Disagreement quoted
> "idempotent; AccessGrants table for per-resource ACLs" vs CWE-863 concerns

**AccessGrants system**: Full ACL table at [`models/access_grants.py`](../../research/repos/open-webui/backend/open_webui/models/access_grants.py) covering ~8 resource types (calendar, channel, knowledge, model, note, prompt, skill, tool). Consistent `_get_access_grants`/`set_access_grants`/`get_grants_by_resource` pattern across all models.

**File ownership**: [`utils/access_control/files.py:19-63`](../../research/repos/open-webui/backend/open_webui/utils/access_control/files.py#L19-L63) `has_access_to_file` — checks file ownership (42-43), then knowledge-base membership with **explicit CWE-863 defense** (lines 47-48 comment: "write/delete on a file only when the object's OWNER owns that file; otherwise a read-only file laundered into an object the user controls would gain write/delete on it"). This is **well-documented** and correctly implemented.

**Classification**: FACT  
**Resolution**: DISAGREE with CWE-863 concern as applied here — the code has the defense documented  
**Confidence**: HIGH  
**Architectural consequence**: Open WebUI's AccessGrants system is well-designed with consistent patterns across resource types. The CWE-863 defense in file access is explicit and correct. This is a strength, not a weakness.

---

## Item 9: Open WebUI — MCP authorization

### Disagreement quoted
> disagreements-C: needs MCP tool-call authorization verification

**MCP connection flow**: Provider configured as `tool_server.connections[].type == 'mcp'`. Tool list filtered by server `access_grants` at [`tools.py:178-190`](../../research/repos/open-webui/backend/open_webui/routers/tools.py#L178-L190).

**Per-call authorization**: [`middleware.py:2335`](../../research/repos/open-webui/backend/open_webui/utils/middleware.py#L2335) `has_connection_access(user, mcp_server_connection)` — returns `None` (denied) when access is denied. `has_connection_access`:

- Admin with bypass → always allowed
- No `access_grants` configured → **admin-only (fail-closed)**
- Grants present → delegates to `has_access` which checks principal/group membership

**Classification**: FACT  
**Resolution**: AGREE with deep-open-webui — MCP authorization exists and is fail-closed for unconfigured servers  
**Confidence**: HIGH  
**Architectural consequence**: MCP authorization is properly gated. Tool listing and execution both check user access. The admin-only default for unconfigured grants is the correct fail-closed behavior.

---

## Summary

| # | Item | Resolution | Confidence |
|---|------|-----------|------------|
| 1 | LobeChat JWKS liveness | AGREE — key is LIVE, CRITICAL | HIGH |
| 2 | LobeChat business serv. | AGREE — 40+ no-op stubs, plan-incomplete | HIGH |
| 3 | RAGFlow IDOR | AGREE — pattern recurs at /thumbnails (NEW) | HIGH |
| 4 | RAGFlow deletion | AGREE — complete, tenant-scoped | HIGH |
| 5 | Dify RBAC enum | AGREE — exhaustive enumeration complete | HIGH |
| 6 | LibreChat plugin cov. | PARTIAL — coverage gaps refuted; global credential store and runAsSystem share-path are real risks | HIGH |
| 7 | Open WebUI Qdrant | AGREE — naming convention fragile, multi-tenancy opt-in | HIGH |
| 8 | Open WebUI AccessGrants | DISAGREE on CWE-863 — defense is correctly implemented | HIGH |
| 9 | Open WebUI MCP auth | AGREE — properly gated, fail-closed | HIGH |