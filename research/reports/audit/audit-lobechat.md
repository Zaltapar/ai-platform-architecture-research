# Wave C Independent Audit — LobeChat

**Auditor posture:** independent, skeptical. All claims independently re-verified against `research/repos/lobechat/` source. Labels: **FACT** (direct source evidence), **INFERENCE** (auditor reasoning), **UNKNOWN** (not verifiable in-repo).

---

## 1. Scope & Method

Independently inspected: monorepo layout (apps/, packages/, tsconfig path aliases), the `packages/business-server` stub layer, usage recording service, agent-quota router, deployment compose + env, MCP connector/share-gate execution paths, file deletion paths, RAG schema (pgvector), Dockerfile.

---

## 2. Verified Architectural Facts

### 2.1 The repository is a pnpm monorepo with a **cloud/OSS business seam** — the decisive finding (FACT)

- `tsconfig.json` maps the alias `@/business/server/*` to `packages/business-server/src/*` first ([tsconfig.json:28](research/repos/lobechat/tsconfig.json:28)).
- `packages/business-server/src/lambda-routers/` contains **typed no-op stubs** with explicit comments that the closed cloud repo overrides the same import paths:
  - **Workspace management is cloud-only.** `workspace.ts` defines `cloudOnly(feature)` which throws `NOT_IMPLEMENTED`; `create` → `cloudOnly('Workspace creation')` ("Workspace creation is a cloud-only feature"), `list` → `[]`, `getById` → `null` ([workspace.ts:46-82](research/repos/lobechat/packages/business-server/src/lambda-routers/workspace.ts:46)).
  - **Workspace usage/balance is stubbed zero.** `workspaceUsageRouter.getCurrentUsage` returns `{ remainingBalance: 0, since: null, subscription: null, until: null, usageByType: [] }` with the comment "Cloud overrides this at the same path with the real workspaceUsageRouter." ([workspaceUsage.ts:14-24](research/repos/lobechat/packages/business-server/src/lambda-routers/workspaceUsage.ts:14)).
  - **Spend router is empty:** `spendRouter = router({})` ([spend.ts:3](research/repos/lobechat/packages/business-server/src/lambda-routers/spend.ts:3)).
  - **File storage checks are empty no-ops:** `businessFileUploadCheck` / `businessFileTransferStorageCheck` are `async () => {}` ([file.ts:13-25](research/repos/lobechat/packages/business-server/src/lambda-routers/file.ts:13)).
  - **Generation charging is a no-op:** `chargeBeforeGenerate(_params) { return undefined; }` ([chargeBeforeGenerate.ts:41-43](research/repos/lobechat/packages/business-server/src/image-generation/chargeBeforeGenerate.ts:41)).
  - **Agent-share spend gate never refuses:** `checkAgentShareSpendAllowance` → `{ allowed: true }`, `getAgentShareMonthlySpend` → `null` ([spendGate.ts:39-61](research/repos/lobechat/packages/business-server/src/agent-share/spendGate.ts:39)).

**INFERENCE (the single most important fact in this audit):** the open-source LobeChat server implements the *entire platform shape* — workspaces, memberships, invitations, audit logs (schema at [workspace.ts:19-195](research/repos/lobechat/packages/database/src/schemas/workspace.ts:19)) — but **every enforcement point (billing, quotas, storage caps, workspace lifecycle) is a commercial feature delivered as a typed contract stub that the closed cloud repo overrides at the same import path**. Self-hosted LobeChat therefore:
  - has **no spend caps** (precharge/reconciliation logic in the OSS `image-generation`/`video-generation` routers calls into stubs that never charge),
  - has **no storage quota enforcement** (upload/transfer checks are empty),
  - **cannot create workspaces via the business router** (cloud-only),
  - reports `remainingBalance: 0` to any workspace usage UI.
This is a *deliberate architecture*, not an accident — the stubs exist precisely to type the cloud contract. The consequence for a commercial adopter: you are adopting a product whose **monetization and multi-tenant-management surface is closed-source by design**, and the OSS build is a single-organization personal/teammate product. Forking would require writing the entire business-server implementation yourself (which the vendor has deliberately kept private).

### 2.2 Usage *recording* is real OSS; usage *enforcement* is not (FACT)

- `UsageRecordService` (real, OSS) reads `messages.usage` + `metadata.usage` scoped by `buildWorkspaceWhere({ userId, workspaceId })`, and computes spend/tokens/tps/ttft aggregations ([usage/index.ts:25-367](research/repos/lobechat/apps/server/src/services/usage/index.ts:25)).
- `agentQuota.ts` is a real OSS router for **provider-account quota load-balancing** (ingest snapshot from desktop sampler, recordUsage, account bindings, `selectAccountForAgent`) — note this is quota over *your own* provider API keys, not customer spend ([agentQuota.ts](research/repos/lobechat/apps/server/src/routers/lambda/agentQuota.ts:1)).
- **INFERENCE:** LobeChat will *measure* what it spends (good telemetry foundation for LiteLLM/ledger integration) but will never *limit* it. The precharge/recharge pattern (`chargeBeforeGenerate` + `spendOrigin` `agentShare` attribution + reconciliation tests) is fully wired in OSS against a stub that returns `undefined` — so the *shape* of the billing integration is visible and reusable, which lowers the cost of writing your own business-server.

### 2.3 MCP boundary is the strongest in the set (FACT)

- `mcpRouter.callTool` hard-blocks at execution time any tool not present in the synced manifest — "so the MCP server is never actually called" — and stdio transport is rejected on the web deployment ([mcp.ts:84+](research/repos/lobechat/apps/server/src/routers/tools/mcp.ts:84)).
- Connectors store **encrypted credentials**, support `mcpConnectionType`, and implement **RFC 9728 Protected Resource Metadata** OAuth discovery ([connector/oauth.ts:54](research/repos/lobechat/apps/server/src/services/connector/oauth.ts:54)).
- `connector/exec.ts` enforces that a tool exists in the synced list and is not disabled; device-only endpoints (stdio/local-URL) are rejected when a device gateway is configured.
- **Shared-agent gate is default-deny:** `SHARE_VISITOR_ALLOWED_IDENTIFIERS` allowlist, per-API grants, MD5-hashed generated names for long/non-ASCII identifiers, and `humanIntervention: 'required'` for needs-approval tools ([shareGate.ts](research/repos/lobechat/apps/server/src/services/aiAgent/shareGate.ts:1)).
- Device gateway tunnels stdio/LAN MCP from the server to user devices ([deviceGateway](research/repos/lobechat/apps/server/src/services/deviceGateway/index.ts:1)).
- **INFERENCE:** This is the most security-conscious MCP implementation audited: fail-closed allowlists, encrypted credential storage, OAuth PRM, transport restriction by deployment type. For the "MCP + external applications" requirement, LobeChat is the reference implementation in this set.

### 2.4 RAG is pgvector + ParadeDB in one Postgres — fixed 1024-dim (FACT)

- RAG schema: `chunks`, `unstructuredChunks`, `embeddings`, `documentChunks` tables in [rag.ts](research/repos/lobechat/packages/database/src/schemas/rag.ts:19), embeddings stored via pgvector; deploy compose pins `paradedb/paradedb:latest-pg17` with `shared_preload_libraries=pg_search` for FTS ([docker-compose.yml:43-45](research/repos/lobechat/docker-compose/deploy/docker-compose.yml:43)).
- **INFERENCE:** One-Postgres-everything is operationally attractive (LobeChat's deployment is the most coherent of the five), but **vector backend is not swappable** — there is no Qdrant/Milvus adapter, and the 1024-dimension is fixed by schema. Any requirement for a separate scalable vector store (per A3 boundary: Qdrant as vector infra) means forking the RAG subsystem.

### 2.5 Deployment & secrets hygiene (FACT)

- Compose: `lobe` (lobehub/lobehub, :3210) + ParadeDB Postgres + Redis + RustFS (S3) + `rustfs-init` (minio/mc bootstrap) + SearXNG + optional `elasticsearch` profile ([docker-compose.yml](research/repos/lobechat/docker-compose/deploy/docker-compose.yml:1)).
- The optional Elasticsearch node is shipped with **`xpack.security.enabled=false`** and an explicit "Intentionally no `ports`" internal-only comment ([docker-compose.yml:157-169](research/repos/lobechat/docker-compose/deploy/docker-compose.yml:157)) — acceptable only while unpublishable; a config slip exposes an unauthenticated ES.
- **A full RSA JWKS private key is embedded in `.env.example`** (`JWKS_KEY={"keys":[{"d":"PVoFyqyrGstB8wU52S7gqqQQdZLtin_...` ([.env.example:69](research/repos/lobechat/docker-compose/deploy/.env.example:69), also in the zh-CN variant). If this key is real and used by anyone who copied the sample (or if it is the vendor's actual signing key), tokens can be forged; at minimum it is a serious secrets-hygiene red flag. **UNKNOWN:** whether the key is a placeholder with no public counterpart in the codebase.
- Dockerfile builds a Next.js standalone server (Node 24), strips desktop routes ([Dockerfile](research/repos/lobechat/Dockerfile:1)).
- File deletion is real: `fileService.deleteFile(s)` removes S3 objects when no longer referenced ([file.ts:772-774, 873-874, 888, 930-932](research/repos/lobechat/apps/server/src/routers/lambda/file.ts:772)).

---

## 3. Red-Flag Register

| # | Red flag | Evidence | Severity | Consequence | Likely remediation | Requires fork? |
|---|----------|----------|----------|-------------|--------------------|----------------|
| LB-1 | **Entire business plane (billing, quotas, storage caps, workspace lifecycle) is closed-source by design; OSS ships typed no-op stubs** | [workspaceUsage.ts:14-24](research/repos/lobechat/packages/business-server/src/lambda-routers/workspaceUsage.ts:14); [workspace.ts:46-82](research/repos/lobechat/packages/business-server/src/lambda-routers/workspace.ts:46); [spend.ts:3](research/repos/lobechat/packages/business-server/src/lambda-routers/spend.ts:3); [file.ts:13-25](research/repos/lobechat/packages/business-server/src/lambda-routers/file.ts:13); [spendGate.ts:39-61](research/repos/lobechat/packages/business-server/src/agent-share/spendGate.ts:39); [tsconfig.json:28](research/repos/lobechat/tsconfig.json:28) | **CRITICAL** | A commercial multi-user deployment of self-hosted LobeChat has **zero** spend/storage enforcement and no workspace management API; the product is architecturally a single-organization personal assistant | Write your own `business-server` implementation against the visible typed contracts (precharge/recharge/`spendOrigin` shapes are all in OSS), or gate the product behind an external billing plane that the OSS code does not consult | **Yes, effectively** — you are re-implementing the vendor's private module |
| LB-2 | JWKS RSA **private key** shipped in `.env.example` | [.env.example:69](research/repos/lobechat/docker-compose/deploy/.env.example:69) | **HIGH** (UNKNOWN if key is live) | If live: full token-forgery by anyone with the example file; if placeholder: still a hygiene failure that trains operators to copy real keys into example files | Rotate key; ship generated-placeholder with instructions | No — config/secrets fix |
| LB-3 | Vector backend locked to pgvector @ 1024-dim in the app Postgres | [rag.ts:19-132](research/repos/lobechat/packages/database/src/schemas/rag.ts:19); [docker-compose.yml:43-45](research/repos/lobechat/docker-compose/deploy/docker-compose.yml:43) | MEDIUM | Conflicts with A3 boundary (Qdrant as swappable vector infra); large-scale RAG co-locates with OLTP | Fork the RAG package or accept pgvector scale limits | Yes, for vector swap |
| LB-4 | Optional ES runs with security disabled | [docker-compose.yml:157-169](research/repos/lobechat/docker-compose/deploy/docker-compose.yml:157) | MEDIUM | Port misconfiguration → unauthenticated full-text index of user content | Enable xpack security or remove the profile from default docs | No |
| LB-5 | Two coexisting backend surfaces: new `apps/server` (Hono + tRPC) and legacy `src/app/(backend)` route handlers | [Dockerfile](research/repos/lobechat/Dockerfile:1) (builds standalone incl. both); [tsconfig.json:28](research/repos/lobechat/tsconfig.json:28) alias ordering | MEDIUM | Duplicate request paths, duplicated auth context resolution, upgrade churn between Next.js route handlers and the new server; deep report's architecture description is already stale | Track vendor migration; do not build on `src/app/(backend)` | No (but pin versions) |
| LB-6 | Device-gateway tunneling extends the trust boundary to user devices | [deviceGateway](research/repos/lobechat/apps/server/src/services/deviceGateway/index.ts:1) | MEDIUM | A compromised user device can execute stdio MCP tools reachable from the platform; per-device authorization model must be audited | Audit device pairing/approval flow; restrict tool allowlists per device | No (verification needed) |
| LB-7 | Provider-account quota system is personal (your own keys), not customer-facing | [agentQuota.ts](research/repos/lobechat/apps/server/src/routers/lambda/agentQuota.ts:1) | LOW | Not reusable as a billing mechanism; do not mistake it for one | N/A | No |

---

## 4. Dimension Scores (0–5)

| Dimension | Score | Rationale |
|---|---|---|
| Architecture | 4 | Cleanest monorepo of the set (75 packages, clear apps/ + packages/ boundary, typed tRPC contracts everywhere) |
| Modularity | 4 | Package boundaries are real and typed; docked for business-plane lock-in and dual backend surfaces |
| Provider abstraction | 4 | `model-runtime` covers ~85 providers behind one interface; custom model gateway is an adapter add |
| Memory | 3 | Chat memory + agent memory exist; no durable cross-workspace memory engine |
| RAG | 3 | Solid in-Postgres RAG (chunks/embeddings/FTS); not swappable, fixed dimension |
| Agents | 4 | Rich agent runtime, MCP, computer-use runtime (tool-runtime), share gating |
| Workflows | 2 | No durable workflow engine; agents are conversational, not step-graph based |
| Tenancy | 4 | Real workspace/membership/invitation/audit-log *schema* + `buildWorkspaceWhere` scoping; but workspace *management* is cloud-stubbed and there is no per-tenant isolation beyond workspace membership |
| API | 4 | tRPC + REST with typed contracts; stable surface; OpenAI-ish endpoints |
| Integrations | 4 | Connectors (encrypted creds + OAuth PRM), SearXNG, device gateway |
| MCP/extensibility | 5 | Best-in-set: fail-closed synced-tool allowlist, default-deny share gate, RFC 9728, transport policy |
| File/storage | 5 | S3-backed, provider abstracted, real deletion/forgetting paths, usage accounting — best verified file lifecycle in the set |
| Context management | 3 | `context-engine` with token accounting exists; assembly logic is in-model-runtime |
| Tests/docs | 2 | Reconciliation tests for precharge exist (impressive) but docs are product-level; the stub layer is documented *only in code comments* |
| Deployment/scaling | 4 | Most coherent single-node compose (lobe+pg+redis+rustfs+searxng) + optional ES + production Grafana/OTel/Tempo stack; scaling = add replicas + Redis (state largely in Postgres) |
| Modification/forkability | 4 | TypeScript, typed contracts, monorepo tooling; docked because the *interesting* module (business) is the one you must rewrite |

**Complexity: 5/5 (highest in set).** Necessary: provider breadth, MCP security model, file lifecycle. Accidental: dual backend surfaces (legacy `src/app/(backend)` + new `apps/server`), the stub indirection layer, 75-package monorepo overhead. The complexity is *well-organized* but the largest — expect the highest onboarding and upgrade-tracking cost.

---

## 5. Change-Blast-Radius Test (A–M)

- **A/B/C (LLM provider changes / gateway):** LOW — model-runtime abstraction is excellent.
- **D (replace memory):** MEDIUM — memory lives in model-runtime/context-engine; no first-class swap seam.
- **E (add vector DB):** **HIGH** — RAG is hardwired to pgvector in the app DB; no adapter.
- **F (replace file storage):** LOW — S3 provider pattern.
- **G (replace auth):** MEDIUM — Better Auth in apps/auth + tRPC `authedProcedure`; OIDC configured, replaceable but touches the workspace-auth middleware (`wsCompatProcedure`).
- **H (billing):** **HIGH** — the integration *shapes* exist (precharge, reconciliation, `spendOrigin`) but the enforcement module is the closed one you must write; good news: the contracts are typed and visible.
- **I (project-level permissions):** MEDIUM — workspace membership/roles schema exists; the management APIs are cloud-stubbed, so you implement against real schema.
- **J/K (MCP, external apps):** LOW — best-in-set.
- **L (replace frontend):** MEDIUM — SPA assets are built separately (`_spa`, `_spa-share`, `_spa-workbench`); API is tRPC-typed (consumer codegen possible) but Next.js-specific behaviors are interleaved.
- **M (custom backend wrapper):** MEDIUM — apps/server is a standalone Hono/tRPC service you can call directly; but business features you expect will 404/NOT_IMPLEMENTED in OSS.

---

## 6. Disagreements with the Deep Report

See [disagreements-C.md](research/evidence/disagreements-C.md). Headlines:
1. Deep report described the architecture as "Next.js backend route handlers own auth context, workspace resolution, model runtime initialization" and left "Complete usage-credit ledger, billing entitlement, storage quota aggregation" as **UNKNOWN** → **DISAGREE/stale**: the repo has since migrated to a pnpm monorepo with `apps/server` (Hono + tRPC) and a `packages/business-server` stub layer. The UNKNOWNs are now **resolved**: usage *recording* is real OSS; billing/quota/storage *enforcement* is **closed-source by design** (typed no-op stubs). The report's modification-test scores ("Usage credits 3", "Storage quotas 3", "Independent backend/API in front 2") were guesses against the wrong architecture; against the actual architecture they should read: credits-enforcement **0** (stub), storage-quota **0** (stub), independent-backend **3** (apps/server is genuinely standalone and callable).
2. Deep report: "Frontend replacement 2" → **PARTIALLY AGREE** with different reasoning: the API is typed and standalone-server-based (easier than the report assumed), but the tRPC + Next.js standalone coupling keeps it at 2–3.

---

## 7. Commercial Suitability Verdict

**The best-architected codebase in the set, and the worst fit for a commercial multi-user product *as shipped*.** LobeChat's engineering quality is the highest here: typed contracts, the strongest MCP security model, the most coherent deployment, real file lifecycle, real usage telemetry. But its most commercially relevant module — billing, quotas, storage caps, workspace management — is **deliberately closed-source**, stubbed out in OSS. Adopting LobeChat for a commercial multi-user platform means committing to a **fork with a private rewrite of `business-server`** from day one, on top of the highest complexity budget in the set. It is the right choice only if (a) you value its MCP/file/agent surfaces enough to carry the rewrite, and (b) you accept the upgrade-drift risk of tracking a fast-moving monorepo. For a lower-risk path, treat LobeChat as a **reference implementation** for MCP gating and precharge/recharge billing shapes rather than as the foundation.

**Confidence: HIGH** (all load-bearing stub claims re-verified at cited lines this wave; JWKS key liveness is the only UNKNOWN).

---

## 8. Open Questions for Wave C.5

1. Determine whether the JWKS key in `.env.example` is live (check for a corresponding public key/JWKS endpoint in the auth app; attempt token verification against a test deployment).
2. Map the *complete* list of `@/business/server/*` import sites in `apps/server` to size the closed surface precisely (20 stub files seen; how many production code paths depend on them?).
3. Verify workspace-scoped query coverage: is every data access in `apps/server` going through `buildWorkspaceWhere`, or are there unscoped queries (leak paths)?
4. Confirm whether the device-gateway pairing flow enforces per-device tool allowlists end-to-end (LB-6).
