# Letta forensic triage

## 1. Verdict snapshot

- **FACT:** The captured shallow commit [`5bcdd17`](../../../repos/letta/.git/HEAD) contains only 12 tracked files: policy/legal documents, contribution metadata, README, and one GitHub workflow. The Git tree contains no `pyproject.toml`, source package, API server, schema, migration, tests, or deployment implementation.
- **FACT:** A read-only Git tree inspection showed only `.github`, `AGENTS.md`, `AI_POLICY.md`, `CITATION.cff`, `CONTRIBUTING.md`, `LICENSE`, `PRIVACY.md`, `README.md`, `SECURITY.md`, and `TERMS.md`.
- **UNKNOWN:** Architecture, execution paths, data model, memory implementation, provider abstraction, tenancy, MCP, deployment, and modification costs cannot be established from the captured repository contents.
- **INFERENCE:** Any ranking of Letta against the other five repositories would be unreliable and must be excluded from evidence-based technical conclusions until a source-complete checkout is obtained.

## 2. Architecture & stack

- **UNKNOWN:** No source or manifest establishes runtime language, framework, dependency graph, process model, or service boundaries.
- **FACT:** Repository metadata alone is insufficient evidence for implementation claims.

## 3. Repo structure

- **FACT:** The captured tree is documentation/policy-dominant and has no application directories.
- **UNKNOWN:** Whether the commit is an intentionally documentation-only branch, a repository transition point, or an incomplete mirror cannot be determined from local objects.

## 4. Data model

- **UNKNOWN:** No schema, ORM model, migration, or persistence adapter is present.

## 5. AuthN/AuthZ/multi-tenancy

- **UNKNOWN:** No authentication middleware, authorization predicate, tenant/workspace model, API-key implementation, or membership relation is present.
- **FACT:** Product-user and tenant-isolation behavior cannot be assessed.

## 6. Provider abstraction

- **UNKNOWN:** No provider registry, adapter, model interface, or client implementation is present.

## 7. Conversation execution path

- **LOGIN UNKNOWN:** No login route or auth implementation is present.
- **CHAT UNKNOWN:** No conversation/session/message handler is present.
- **FILE UNKNOWN:** No upload, parser, storage, or indexing path is present.
- **API UNKNOWN:** No server router or API implementation is present.

## 8. Memory

- **UNKNOWN:** No evidence can establish short-term, user, project, semantic, episodic, procedural, or condensation memory.

## 9. RAG

- **UNKNOWN:** No ingestion, chunking, embedding, vector store, retriever, or citation implementation is present.

## 10. Agents/workflows/tools/MCP

- **UNKNOWN:** No agent runtime, workflow graph, tool interface, sandbox, or MCP client/server implementation is present.

## 11. Files/storage

- **UNKNOWN:** No file or object-storage implementation is present.

## 12. API

- **UNKNOWN:** No API surface, SDK, webhook, streaming protocol, or versioning implementation is present.

## 13. Background jobs/usage

- **UNKNOWN:** No queue, worker, scheduler, usage ledger, credits, quota, or billing implementation is present.

## 14. Deployment/scaling

- **UNKNOWN:** No Docker, compose, Helm, deployment, process, cache, or replica-safety implementation is present.

## 15. Observability/testing

- **UNKNOWN:** No application tests, metrics, tracing, logging, or error-reporting implementation is present.

## 16. Coupling/extensibility

- **UNKNOWN:** Coupling and extension points cannot be assessed without source.

## 17. 13 modification tests

| Modification | Score | Assessment |
|---|---:|---|
| Custom OpenAI-compatible provider | 5 | Source unavailable; cannot identify an extension seam. |
| Another provider | 5 | Source unavailable. |
| Persistent user memory | 5 | Source unavailable. |
| Project memory | 5 | Source unavailable. |
| Context monitoring | 5 | Source unavailable. |
| User-approved condensation | 5 | Source unavailable. |
| Credits | 5 | Source unavailable. |
| Storage quotas | 5 | Source unavailable. |
| Custom external integration | 5 | Source unavailable. |
| MCP tool/server | 5 | Source unavailable. |
| Independent backend/API in front | 5 | No API contract is available. |
| Frontend replacement | 5 | No backend contract is available. |
| Connecting to another candidate | 5 | No integration boundary is available. |

These are **UNKNOWN**, not claims that the real project is intrinsically invasive.

## 18. Evidence log

- **FACT:** Local `git ls-tree -r HEAD` returned 12 files and no implementation path.
- **FACT:** Local working tree is clean, so the absence is in the captured commit rather than an uncommitted deletion.
- **FACT:** `pyproject.toml` and expected source paths are absent.
- **UNKNOWN:** Whether another ref, full clone, submodule, Git LFS object, or upstream repository location contains the implementation.

## 19. Unknowns

All substantive technical sections remain UNKNOWN because the source-complete repository was not available. The blocker must be carried into the summary and Letta must not be shortlisted on implementation evidence.
