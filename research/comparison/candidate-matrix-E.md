# Wave E — Normalized Candidate Comparison Matrix

**Date:** 2026-09-11
**Status:** Final research artifact (Wave E). Synthesis only — no production code, no repository modifications, no invented evidence.
**Scope:** Ten product candidates (Dify, AnythingLLM, LibreChat, Open WebUI, Flowise, Langflow, Letta, RAGFlow, Khoj, LobeChat) plus five subsystem candidates (LiteLLM, Mem0, MCP Python SDK, Qdrant, Temporal).

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

## 3. Ranking summary

| Rank | Candidate | Base total | Post-C.5 composite | Confidence | Status |
|---:|---|---:|---:|---|---|
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

**Cross-cutting FACT:** all five audited products keep their *commercial* capabilities (fine-grained RBAC, quota/billing, workspace management) out of the open-source build — Dify via a closed enterprise API, LobeChat via typed stubs, the other three via absence. **No candidate is adoptable as-is as a commercial multi-user foundation.**

## 4. Subsystem candidates

| Subsystem | Modularity | API quality | Replaceability | Production readiness | Multi-tenancy | Extensibility | Operational complexity |
|---|---:|---:|---:|---:|---:|---:|---:|
| LiteLLM | 4 | 5 | 4 | 4 | 3 | 5 | 3 |
| Mem0 | 4 | 4 | 4 | 3 | 2 | 4 | 2 |
| MCP Python SDK | 5 | 4 | 5 | 4 | 1 | 5 | 2 |
| Qdrant | 4 | 5 | 4 | 5 | 3 | 3 | 4 |
| Temporal | 3 | 5 | 3 | 5 | 4 | 4 | **5** |

**Boundary rule:** none of the five subsystems is a product-foundation candidate. They are replaceable engines/sidecars behind platform-owned interfaces; the platform owns canonical identity, tenancy, policy, billing, and audit.

## 5. Evidence limitations

1. Letta source unavailable; all dimensions are UNKNOWN.
2. Langflow checkout incomplete on Windows; login/token, migrations, and tenant/job scope remain UNKNOWN.
3. LibreChat RAG internals live in an external `rag_api` service.
4. LobeChat parser/index internals and complete billing ledger remain partially UNKNOWN.
5. Shallow clones reduce confidence in release and migration history.
6. AnythingLLM/Flowise/Khoj rows are triage-derived, not independently re-audited in Wave C.
7. UNKNOWNs are labeled and never silently scored.
