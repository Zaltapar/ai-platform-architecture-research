# Wave E — Consolidated Modification Blast Radius

**Date:** 2026-09-11 **Status:** Final research artifact (Wave E)

Scores: 0 = configuration/trivial, 1 = small extension, 2 = moderate development, 3 = substantial modification, 4 = invasive modification, 5 = architectural rewrite. **U** = UNKNOWN (evidence-limited).

**Evidence basis:** consolidated from Wave A1/A2 modification tests ([`triage-summary-A1.md`](../reports/forensic/triage-summary-A1.md) §modification signal, [`triage-summary-A2.md`](../reports/forensic/triage-summary-A2.md) §17), Wave B deep reports (§13 in each), audit disagreements register ([`disagreements-C.md`](../evidence/disagreements-C.md)). Some Deep Wave A2 reports are cross‑referenced below for Khoj, Flowise experiments not independently re‑pointed by Wave C.

---

## 1. Consolidated 13-modification-test table

| Test | Dify | AnythingLLM | LibreChat | Open WebUI | Flowise | Langflow ⚠ | Letta ⚠ | RAGFlow | Khoj | LobeChat |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1. Custom OpenAI-compatible provider | 1 | 0–1 | 1 | 0 | 1 | 1 | U | 2 | 1 | 1 |
| 2. Add another provider | 2 | 2 | 2 | 2 | 2 | 1 | U | 2 | 2 | 2 |
| 3. Persistent user memory | 3 | 1 | 1 | 1 | 3 | 2 | U | 1 | 1 | 2 |
| 4. Project-specific memory | 2 | 1 | 2 | 3 | 3 | 3 | U | 2 | 4 | 2 |
| 5. Context monitoring | 2 | 2 | 2 | 2 | 3 | 2 | U | 3 | 3 | 2 |
| 6. User-approved condensation | 3 | 2–3 | 2 | 2–3 | 3 | 3 | U | 3 | 3 | 3 |
| 7. Usage credits | 1 | 3 | 1–2 | 3 | 4 | 4 | U | 4 | 3 | 3 |
| 8. Storage quotas | 2 | 3 | 2 | 3 | 3 | 3 | U | 2 | 2 | 3 |
| 9. Custom external integration | 1–2 | 1–2 | 1–2 | 1–2 | 2 | 1 | U | 2 | 2 | 1 |
| 10. MCP server/tool | 1 | 1 | 1 | 1 | 1 | 1 | U | 1 | 2 | 1 |
| 11. Independent backend/API in front | 2 | 2 | 2 | 1 | 2 | 2 | U | 2 | 2 | 2 |
| 12. Frontend replacement | 2 | 2 | 1–2 | 1 | 2 | 2 | U | 2 | 2 | 2 |
| 13. Connect another candidate as RAG/memory | 3 | 2–3 | 1–2 | 1–2 | 3 | 3 | U | 3 | 3 | 2 |

⚠ = **evidence-limited**: Langflow partial Windows checkout; Letta source-unavailable shallow commit.

---

## 2. Per-test qualitative analysis

### Test 1 — Custom OpenAI-compatible provider

| Candidate | Score | Blast radius | Likely files/layers | Implementation cost | Confidence |
|---|---|---|---|---|---|
| Open WebUI | 0 | None (already supported) | `config.py`, `openai.py` per-user overrides | None (configuration) | HIGH |
| AnythingLLM | 0–1 | Small | `helpers/index.js` generic-openai switch | < 1d | HIGH |
| LibreChat | 1 | Small | `endpoints` / provider client package | < 1d | HIGH |
| Dify | 1 | Small | `ProviderManager` / model config | < 1d | HIGH |
| LobeChat | 1 | Small | `modelRuntime` initialization | < 1d | HIGH |
| Langflow | 1 | Small | Component bundle / OpenAI `ChatModel` | < 1d | MEDIUM (full checkout blocked) |
| Khoj | 1 | Small | `configure.py` base URL | < 1d | MEDIUM |
| RAGFlow | 2 | Moderate | `LLMBundle` + `tenant_model_service` | 1–2d | HIGH |
| Flowise | 1 | Small | Provider node configuration | < 1d | MEDIUM |

### Test 2 — Add another provider

All candidates score 1–2. Adding a new provider requires a client adapter, configuration path, and credential management in all cases. Langflow's component bundle system makes it cheapest (score 1). No candidate has a universal zero-code provider registry.

### Test 3 — Persistent user memory

| Candidate | Score | Blast radius | Confidence |
|---|---|---|---|
| LibreChat | 1 | Small (memory model and agent callback already exist) | HIGH |
| Open WebUI | 1 | Small (SQL memory + vector collections exist) | HIGH |
| RAGFlow | 1 | Small (full memory taxonomy exists) | HIGH |
| AnythingLLM | 1 | Small (`memories` table + injection module exist) | HIGH |
| Khoj | 1 | Small (`UserMemory` + APIs exist) | MEDIUM |
| LobeChat | 2 | Moderate (personal memory/tool seams exist; workspace/project memory not uniform) | HIGH |
| Langflow | 2 | Moderate (user/flow-scoped memory; no persistent project memory) | MEDIUM |
| Dify | 3 | Substantial (only token-buffer conversation memory; no generic persistent memory) | HIGH |
| Flowise | 3 | Substantial (only session-level memory nodes; no unified persistent memory) | MEDIUM |

### Test 4 — Project-specific memory

Any candidate without a first-class workspace/project tenant identity scores ≥3. Khoj (score 4) has the biggest gap because its data model is user-centric without project tenancy.

### Tests 5–6 — Context monitoring / user-approved condensation

Scores 2–3 for all candidates. LibreChat has the best starting point (token tracking, compaction), but no candidate has a complete cross-engine context-budget service with user approval. The platform `ContextBudgetService` must be built.

### Tests 7–8 — Usage credits / storage quotas

| Test | Score range | Candidates with best starting point | Candidates with worst gap |
|---|---|---|---|
| Credits | 1–4 | Dify (1 — credit pools exist, cloud-gated), LibreChat (1–2 — balances/transactions) | RAGFlow (4 — dead schema, zero call sites), Langflow (4 — none), Flowise (4 — none) |
| Storage quotas | 2–3 | Dify (2), LibreChat (2), RAGFlow (2) have partial; LobeChat (3 — stubbed), Flowise (3 — partial) | None below 3; all candidates need platform quota addition |

**Critical finding (from [`reconciliation-C5.md`](../evidence/reconciliation-C5.md)):** RAGFlow's credit system is not "absent" but **dead code** — `credit=512` default, `decrease()` never called. Its score is effectively 0 (build from scratch), not 4 (as if a starting point existed). Scored at 4 in this table because the modification initiative is "add usage credits from scratch," not "fix dead schema." Same applies to LobeChat's stubbed business modules.

### Test 9 — Custom external integration

All candidates score 1–2. Rest/webhook integrations are universally feasible through provider/tool/MCP seams. The variation is small because all candidates have some form of external API/tool boundary.

### Test 10 — MCP server/tool

All non-Letta candidates score 1. MCP is either natively supported (LibreChat, Dify, RAGFlow, Open WebUI, LobeChat, Langflow, Flowise, AnythingLLM) or has configuration-level paths. **No candidate makes MCP hard to add.**

### Tests 11–12 — Independent backend/API in front / Frontend replacement

| Candidate | Backend/API score | Frontend score | Notes |
|---|---|---|---|
| Open WebUI | 1 | 1 | Cleanest separation: separate Svelte frontend, FastAPI backend, OpenAI-compatible API |
| LibreChat | 2 | 1–2 | API routes explicit; frontend/backend separated |
| LobeChat | 2 | 2 | Dual backend surfaces; stronger Hono/tRPC API |
| Dify | 2 | 2 | Console/web/service API routes separated; frontend independent |
| RAGFlow | 2 | 2 | Quart API + admin/MCP APIs; frontend product-coupled |
| All others | 2 | 2 | API isolation achievable; product-specific contracts remain |

### Test 13 — Connect candidate as RAG/memory

| Candidate | Score | Best integration seam |
|---|---|---|
| LibreChat | 1–2 | External `rag_api` HTTP seam — best replaceability |
| Open WebUI | 1–2 | Vector factory (10 backends) |
| LobeChat | 2 | Document service / context-engine seam |
| RAGFlow | 3 | Deeply integrated; replaceable only at API boundary |
| Dify | 3 | Vector factory + dataset domain model |
| Langflow | 3 | Graph-component-composable RAG |
| Flowise | 3 | Node-based vector/retriever components |
| Khoj | 3 | User content/search adapters |
| AnythingLLM | 2–3 | Vector abstraction + collector |

---

## 3. Blast radius ranking (most to least invasive for a foundation-fork)

| Candidate | Avg. score (13 tests) | Blast radius assessment | Likely layers affected in a deep modification | Migration implication | Post-C.5 confidence |
|---|---|---|---|---|---|
| Dify | 1.92 | MODERATE–HIGH | Auth/RBAC, quota, provider, workflow, dataset, memory | Migrate must bypass closed ENTERPRISE RBAC API | HIGH |
| LibreChat | 1.46 | **LOWEST** | Tenant plugin default, credential store, agent checkpoint | Make strict mode default; wrap in platform adapter | HIGH |
| Open WebUI | 1.73 | MODERATE | Add tenancy layer, context service, billing — cross-cuts 770-line chat endpoint | Platform owns tenancy; Open WebUI becomes frontend-only | HIGH |
| RAGFlow | 2.19 | HIGH | IDOR fix, credit rebuild, tenant isolation hardening, per-process scaling | RAGFlow behind hard gateway; platform re-owns document auth | HIGH |
| LobeChat | 1.85 | HIGH (must rebuild business plane) | 40+ business modules, JWKS remediation, vector backend lock | Implement business modules; remediate env secrets | HIGH |
| AnythingLLM | 1.85 | MODERATE–HIGH | Tenancy hardening, storage quotas, credit ledger, collector paths | Additional auth/policy layer needed | MEDIUM |
| Khoj | 2.15 | HIGH (no project tenancy) | Add project model, propagate through all routes/adapters, rebuild context | Broadest data-model change of all candidates | MEDIUM |
| Langflow | 2.00 | MODERATE (partial evidence) | Auth, tenant, migration paths — full audit blocked | Must unblock checkout first | LOW–MEDIUM |
| Flowise | 2.31 | MODERATE–HIGH | Tenancy, credit, context, node-level isolation | Distributed scope — many nodes participate | MEDIUM |
| Letta | U | UNKNOWN | No source available | Cannot assess | N/A |

---

## 4. Cross-cutting modification patterns

1. **Provider addition is universally cheap (score 0–2)** for all candidates. Provider abstraction seams are consistently the most replaceable boundary. An OPENAI-compatible endpoint means any candidate can call LiteLLM.

2. **MCP/tool addition is universally cheap (score 1)** for all candidates with source. MCP is a well-established pattern across the ecosystem.

3. **Tenant/workspace addition is the most invasive cross-cutting concern.**
   - Candidates with existing tenant primitives (LibreChat, Dify, RAGFlow): moderate modification to harden.
   - Candidates without (Open WebUI, Khoj): substantial modification.
   - **Consistent INFERENCE across all candidates:** adding first-class platform tenancy is a 3–4 modification; building it at the platform plane avoids every candidate's gap.

4. **Credit/billing is the second-most invasive** (scores 1–4; effective 0 for RAGFlow due to dead schema). Only Dify has a real (cloud-gated) codebase. The platform must build billing; no candidate's code can be adopted as canonical.

5. **Frontend and API replacement** (scores 1–2) are consistently achievable for all candidates with source. Open WebUI is the easiest to replace the frontend; LibreChat and LobeChat have the cleanest API/backend separation.

---

## 5. Migration implications summary

| Decision path | Blast radius | Implications |
|---|---|---|
| Fork LibreChat as foundation | **Lowest** — tenant plugin exists; make strict default; wrap credential store | Migration: stabilize tenant; replace balance/credits with platform; wrap agent runtime |
| Fork Dify as foundation | HIGH — must implement ENTERPRISE RBAC API + rebuild quota for self-host | Closed dependency; each Dify version risks API contract changes |
| Fork LobeChat as foundation | HIGHEST — 40+ business modules to implement from stub contracts | JWKS must be remediated; business plane is a full rebuild |
| Build platform plane + API-wrap candidate(s) | **Lowest** for platform; MEDIUM for adapters | No permanent schema dependency; reversible at each boundary |
| Per-instance Open WebUI | LOW blast radius per instance | No shared tenancy; simplest if single-tenant topology is acceptable |

**Preserved finding (from [`integration-summary-D.md`](../reports/integration/integration-summary-D.md) §1):** platform-owned control plane + replaceable engines behind API boundaries is the lowest-regret architecture. No single candidate is safe as a fork foundation without substantial modification.