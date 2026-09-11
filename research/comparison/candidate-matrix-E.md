# Wave E — Normalized Candidate Comparison Matrix

**Date:** 2026-09-11
**Status:** Final research artifact (Wave E). Synthesis only — no production code, no repository modifications, no invented evidence.
**Scope:** Ten product candidates (Dify, AnythingLLM, LibreChat, Open WebUI, Flowise, Langflow, Letta, RAGFlow, Khoj, LobeChat) plus five subsystem candidates (LiteLLM, Mem0, MCP Python SDK, Qdrant, Temporal).

**Evidence base:**
- Wave C independent audit scores (source-verified): [`audit-summary-C.md`](../reports/audit/audit-summary-C.md)
- Wave C.5 reconciliation (9/9 items resolved): [`reconciliation-summary-C5.md`](../reports/audit/reconciliation-summary-C5.md), [`reconciliation-C5.md`](../evidence/reconciliation-C5.md)
- Wave B deep forensics: [`deep-summary-B.md`](../reports/forensic/deep-summary-B.md)
- Wave A1/A2 triage: [`triage-summary-A1.md`](../reports/forensic/triage-summary-A1.md), [`triage-summary-A2.md`](../reports/forensic/triage-summary-A2.md)
- Wave A3 subsystem forensics: [`subsystem-summary-A3.md`](../reports/forensic/subsystem-summary-A3.md)

---

## 1. Score definitions

### Capability dimensions (0–5, higher is better)

| Score | Meaning |
|---:|---|
| 0 | Not established in evidence / absent / non-functional |
| 1 | Minimal or prototype; material gaps; not product-usable |
| 2 | Basic; user-scoped or partial; significant gaps for a commercial multi-tenant platform |
| 3 | Solid; product-usable with clear, documented limits |
| 4 | Strong; production-quality with explicit, testable seams |
| 5 | Best-in-class in the set; explicit, tested, replaceable seams |
| U | UNKNOWN — evidence unavailable (evidence-limited candidate) |
| N/A | Dimension not applicable to this class of component |

**Basis of scores (row provenance):**

- **Audited rows (LibreChat, Dify, RAGFlow, Open WebUI, LobeChat):** Wave C independent audit, source-verified file:line evidence. These are the highest-confidence rows.
- **Triage-derived rows (AnythingLLM, Flowise, Khoj, Langflow):** Normalized to the same 0–5 scale from Wave A1/A2 and per-repository triage evidence. **Not independently re-audited in Wave C** → treat as MEDIUM/LOW confidence for decision purposes.
- **Letta:** All dimensions **U** — the captured shallow commit contains only 12 non-implementation files (policy/legal docs, README, one workflow). No implementation evidence exists. **Explicitly flagged: evidence-limited; every score below is UNKNOWN, not a low score.**
- **Langflow:** **Explicitly flagged: evidence-limited** — partial Windows checkout caused by filename-length limits; login/token chain, full migrations, and complete tenant/job paths remain UNKNOWN. Its scores reflect what was verifiable and must not be read as a complete audit.

### Complexity (1–5, higher is more complex)

Composite of **necessary complexity** (intrinsic to the product domain) and **accidental complexity** (avoidable design overhead). For audited rows the breakdown is taken from [`audit-summary-C.md`](../reports/audit/audit-summary-C.md); for triage rows it is an INFERENCE from repository structure evidence.

### Post-C.5 composite adjustments (audited candidates only)

The Wave C.5 reconciliation applied composite-level adjustments that do not map to single-dimension deltas (risk/severity re-weighting): LobeChat 59→58 (live JWKS key + confirmed business void), LibreChat 54→55 (tenant plugin held up; coverage gaps refuted), Dify 51 (no delta), RAGFlow 45 (no net delta), Open WebUI 43→44 (AccessGrants/MCP auth stronger than first read). The dimension table below shows the **base** Wave C scores so totals remain internally consistent; the adjusted composites are reported alongside.

---

## 2. Product candidate matrix (10 candidates × 16 dimensions)

| Dimension (0–5) | Dify | AnythingLLM | LibreChat | Open WebUI | Flowise | Langflow ⚠ | Letta ⚠ | RAGFlow | Khoj | LobeChat |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Architecture | 4 | 2 | 3 | 2 | 3 | 3 | U | 3 | 2 | 4 |
| Modularity / low coupling | 3 | 2 | 4 | 3 | 3 | 4 | U | 2 | 3 | 4 |
| Provider abstraction | 4 | 3 | 4 | 3 | 3 | 4 | U | 3 | 3 | 4 |
| Memory | 2 | 3 | 3 | 3 | 2 | 2 | U | **5** | 4 | 3 |
| RAG | 4 | 3 | 3 | 3 | 3 | 3 | U | **5** | 3 | 3 |
| Agents | 4 | 3 | 4 | 2 | 3 | 3 | U | 3 | 3 | 4 |
| Workflows | 3 | 2 | 2 | 1 | 4 | 4 | U | 2 | 2 | 2 |
| Multi-tenancy | 4 | 2 | **4**† | 4‡ | 2 | 1 | U | 3 | 1 | 4 |
| API quality | 3 | 3 | 4 | 3 | 3 | 4 | U | 3 | 3 | 4 |
| External integrations | 3 | 3 | 3 | 3 | 3 | 3 | U | 2 | 3 | 4 |
| MCP / extensibility | 3 | 3 | 4 | 3 | 3 | 3 | U | 3 | 2 | **5** |
| Files / storage | 3 | 2 | 4 | 3 | 3 | 3 | U | 2 | 3 | **5** |
| Context management | 4 | 2 | 3 | 2 | 2 | 2 | U | 3 | 2 | 3 |
| Docs / tests | 2 | 3 | 3 | 3 | 2 | 2 | U | 2 | 3 | 2 |
| Deployment / scalability | 2 | 1 | 2 | 3 | 2 | 1 | U | 1 | 2 | 4 |
| Modification / forkability | 3 | 3 | 4 | 2 | 3 | 3 | U | 3 | 2 | 4 |
| **Total (of 80)** | **51** | **40** | **54** | **43** | **44** | **45** ⚠ | **U** | **45** | **41** | **59** |
| **Post-C.5 adjusted composite (audited only)** | 51 | — | 55 | 44 | — | — | U | 45 | — | 58 |

⚠ = **evidence-limited** (see §1). † LibreChat tenancy is the strongest *mechanism* (DB-level query scoping) but **strict mode is default-off** (fail-open default); the 4 scores the mechanism, and the unsafe default is carried as red flag LC-1. ‡ Open WebUI tenancy 4 scores the *sharing/ACL primitive quality* (best per-resource ACL ergonomics in the set, CWE-863-aware); as an actual tenancy boundary it is **0/5** — no tenant unit exists (red flag OW-1). Under strict-tenancy reading Open WebUI totals 39/80. Either way it ranks last among audited candidates.

### Row provenance and confidence

| Candidate | Score basis | Confidence | Key evidence |
|---|---|---|---|
| Dify | Wave C audit (source-verified) | HIGH | [`deep-dify.md`](../reports/forensic/deep-dify.md), [`audit-dify.md`](../reports/audit/audit-dify.md), RBAC/quota findings in [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 5 |
| AnythingLLM | Wave A1 triage, normalized | MEDIUM | [`triage-anythingllm.md`](../reports/forensic/triage-anythingllm.md) |
| LibreChat | Wave C audit + C.5 | HIGH | [`deep-librechat.md`](../reports/forensic/deep-librechat.md), [`audit-librechat.md`](../reports/audit/audit-librechat.md), [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) item 6 |
| Open WebUI | Wave C audit + C.5 | HIGH | [`deep-open-webui.md`](../reports/forensic/deep-open-webui.md), [`audit-open-webui.md`](../reports/audit/audit-open-webui.md), [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) items 7–9 |
| Flowise | Wave A2 triage, normalized | MEDIUM | [`triage-flowise.md`](../reports/forensic/triage-flowise.md) |
| Langflow ⚠ | Wave B deep + A2 triage, **partial checkout** | LOW–MEDIUM | [`deep-langflow.md`](../reports/forensic/deep-langflow.md), [`triage-langflow.md`](../reports/forensic/triage-langflow.md) |
| Letta ⚠ | **No implementation source available** | N/A (all U) | [`triage-letta.md`](../reports/forensic/triage-letta.md) |
| RAGFlow | Wave C audit + C.5 | HIGH | [`deep-ragflow.md`](../reports/forensic/deep-ragflow.md), [`audit-ragflow.md`](../reports/audit/audit-ragflow.md), [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) items 3–4 |
| Khoj | Wave A2 triage, normalized | MEDIUM | [`triage-khoj.md`](../reports/forensic/triage-khoj.md) |
| LobeChat | Wave C audit + C.5 | HIGH | [`deep-lobechat.md`](../reports/forensic/deep-lobechat.md), [`audit-lobechat.md`](../reports/audit/audit-lobechat.md), [`reconciliation-C5.md`](../evidence/reconciliation-C5.md) items 1–2 |

---

## 3. Complexity (1–5, higher is more complex)

| Candidate | Score | Necessary complexity | Accidental complexity |
|---|---:|---|---|
| Dify | 4 | RAG depth; workflow engine; provider breadth | **Dual permission regime** (legacy vs enterprise RBAC); call-sites-only enterprise features; Celery sprawl |
| AnythingLLM | 3 | Provider breadth; RAG; agent plugins | Single-container default; central provider/vector switches; local-storage default |
| LibreChat | 4 | Provider breadth; tenancy mechanism; file pipeline | Chat-pipeline monolith (`client.js`/`BaseClient.js`); FerretDB legacy copies; duplicated strict-mode env reads |
| Open WebUI | 3 | Provider breadth; RAG breadth | Monolithic `main.py` (3043 lines); naming-convention-coupled multitenancy mode |
| Flowise | 3 | Graph runtime; component ecosystem | Process-local MCP toolkit state; distributed auth/storage semantics |
| Langflow ⚠ | 4 | Graph runtime; provider bundles; SDK | `langflow`→`lfx` compatibility alias layer; in-memory queue/cache topology |
| Letta ⚠ | U | No evidence | No evidence |
| RAGFlow | 4 | Parser/chunker/GraphRAG depth (the product) | Sync/async bridging; dead credit schema; per-process concurrency; 2300–2500-line monolithic services |
| Khoj | 3 | Personal knowledge product; user memory | Django/FastAPI hybrid; user-centric model without project tenancy |
| LobeChat | **5** | Provider breadth; MCP security model; file lifecycle | **Dual backend surfaces** (legacy Next.js handlers + `apps/server` Hono/tRPC); stub indirection layer; ~75-package monorepo overhead |

---

## 4. Subsystem candidate matrix (5 candidates)

Dimensions are the Wave A3 comparative dimensions (0–5, higher better). **Operational complexity is inverted: higher = more operational burden.** Scores are FACT-grounded from [`subsystem-summary-A3.md`](../reports/forensic/subsystem-summary-A3.md) §3.

| Subsystem | Modularity | API quality | Replaceability | Production readiness | Multi-tenancy | Extensibility | Operational complexity (↑=heavier) |
|---|---:|---:|---:|---:|---:|---:|---:|
| LiteLLM | 4 | 5 | 4 | 4 | 3 | 5 | 3 |
| Mem0 | 4 | 4 | 4 | 3 | 2 | 4 | 2 |
| MCP Python SDK | 5 | 4 | 5 | 4 | 1 (intentional — it provides none) | 5 | 2 |
| Qdrant | 4 | 5 | 4 | 5 | 3 (service RBAC, not application tenancy) | 3 | 4 |
| Temporal | 3 | 5 | 3 | 5 | 4 (namespaces) | 4 | **5** |

**Evidence anchors:**
- LiteLLM: SDK/provider/gateway boundaries, router strategies, callbacks — [`subsystem-litellm.md`](../reports/forensic/subsystem-litellm.md). INFERENCE: gateway persistence/auth/budget coupling lowers replaceability from a perfect score.
- Mem0: full add/search/update/history/delete lifecycle, scoped identities, factories, server auth — [`subsystem-mem0.md`](../reports/forensic/subsystem-mem0.md). INFERENCE: caller-derived identity filters and incomplete tenant authority lower multi-tenancy/production scores.
- MCP Python SDK: server/client/transports/auth/extensions — [`subsystem-mcp-python-sdk.md`](../reports/forensic/subsystem-mcp-python-sdk.md). Its low multi-tenancy score is **by design** (protocol layer, not authority).
- Qdrant: typed REST/gRPC point/query contracts, storage/RBAC layers — [`subsystem-qdrant.md`](../reports/forensic/subsystem-qdrant.md). Production-ready infrastructure but operationally heavy; not an application tenant authority.
- Temporal: durable workflows, persistence factories, namespaces, retries — [`subsystem-temporal.md`](../reports/forensic/subsystem-temporal.md). Highest production readiness **and** highest operational complexity and migration cost.

**Boundary rule (from A3, preserved in E):** none of the five subsystems is a product-foundation candidate. They are replaceable engines/sidecars behind platform-owned interfaces; the platform owns canonical identity, tenancy, policy, billing, and audit.

---

## 5. Ranking summary

### Product candidates (composite, of 80)

| Rank | Candidate | Base total | Post-C.5 composite | Confidence | Status |
|---:|---|---:|---|---|---|
| 1 | LobeChat | 59 | 58 | HIGH (audited) | Best-engineered; **business plane closed by design** + confirmed live JWKS risk |
| 2 | LibreChat | 54 | 55 | HIGH (audited) | Strongest open tenant primitive; strict mode default-off |
| 3 | Dify | 51 | 51 | HIGH (audited) | Deepest RAG/workflow; **closed enterprise RBAC dependency**; fail-open quota |
| 4 | Langflow ⚠ | 45 | — | LOW–MEDIUM (evidence-limited) | Best graph/provider modularity; full audit blocked |
| 4 | RAGFlow | 45 | 45 | HIGH (audited) | Best RAG engine; **recurring IDOR**; dead credit system |
| 6 | Flowise | 44 | — | MEDIUM (triage) | Practical graph/component platform; tenancy distributed |
| 7 | Open WebUI | 43 | 44 | HIGH (audited) | Best single-tenant appliance; **no tenancy primitive** |
| 8 | Khoj | 41 | — | MEDIUM (triage) | Strong personal memory; no project tenancy |
| 9 | AnythingLLM | 40 | — | MEDIUM (triage) | Approachable fork; weakest isolation/scaling posture |
| 10 | Letta ⚠ | U | U | N/A | **Not rankable** — source unavailable in captured commit |

**Cross-cutting FACT (P-1, from [`disagreements-C.md`](../evidence/disagreements-C.md) §6):** all five audited products keep their *commercial* capabilities (fine-grained RBAC, quota/billing, workspace management) out of the open-source build — Dify via a closed enterprise API, LobeChat via typed stubs, the other three via absence. **No candidate is adoptable as-is as a commercial multi-user foundation.**

### Subsystems (role, not rank)

| Subsystem | Role in target architecture | Confidence |
|---|---|---|
| LiteLLM | Model-access engine behind `ModelGateway` | HIGH |
| Mem0 | Memory engine behind `MemoryProvider` | HIGH |
| MCP Python SDK | Protocol runtime inside a platform-owned MCP gateway | HIGH |
| Qdrant | Vector/index infrastructure behind `RagProvider` | HIGH |
| Temporal | Durable execution engine behind `JobService` | HIGH |

---

## 6. Evidence limitations carried into this matrix

1. **Letta — source unavailable (FACT):** captured commit `5bcdd17` contains only 12 tracked non-implementation files; no manifest, source, schema, or tests. All dimensions are UNKNOWN by construction.
2. **Langflow — incomplete Windows checkout (FACT):** filename-length limits blocked full materialization; login/token chain, migrations, and complete tenant/job scope are UNKNOWN.
3. **LibreChat external RAG (FACT):** vector/parser internals live in an external `rag_api` service, not the Node repo.
4. **LobeChat parser/index internals and full billing ledger (FACT/UNKNOWN):** context-engine mapping proven; complete parser/chunker/index path not traced; usage *recording* is real OSS while usage *enforcement* is closed (typed stubs).
5. **Shallow clones everywhere (FACT):** release cadence, tag-based upgradeability, and historical migration evolution are lower-confidence for every candidate.
6. **AnythingLLM/Flowise/Khoj (FACT):** normalized from triage, not independently re-audited in Wave C; their rows are MEDIUM confidence at best.
7. No final decision in this package relies on unverified claims; UNKNOWNs are labeled and never silently scored.
