# Wave A1 — Four-Repository Forensic Triage Summary

## Comparative verdict

| Rank | Repository | Best architectural role | Primary evidence | Main concern |
|---:|---|---|---|---|
| 1 | **LibreChat** | Commercial multi-user chat/agent backend foundation | Explicit Mongo tenant-isolation plugin, ACL/resource permissions, persistent memory, balances/transactions, package-level handlers, external RAG boundary, native MCP/agent runtime ([`tenantIsolation.ts`](../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96), [`tenantContext.ts`](../../repos/librechat/packages/data-schemas/src/config/tenantContext.ts:19), [`files/text.ts`](../../repos/librechat/packages/api/src/files/text.ts:82)) | Agent/job/MCP subsystem is complex; Mongo and distributed stream/checkpoint infrastructure become strategic dependencies |
| 2 | **Dify** | Full AI application/workflow/RAG platform or orchestration engine | First-class tenants/workspaces, provider/plugin runtime, workflow/agent persistence, many vector adapters, MCP, Celery/Redis, quota/credit pools ([`provider_manager.py`](../../repos/dify/api/core/provider_manager.py:17), [`model.py`](../../repos/dify/api/models/model.py:2717), [`vector_factory.py`](../../repos/dify/api/core/rag/datasource/vdb/vector_factory.py:29)) | High domain coupling and substantial fork surface; general personal/project memory is not established as a dedicated subsystem |
| 3 | **Open WebUI** | Replaceable frontend/API and personal-AI/RAG subsystem | Clean FastAPI routers, SQLAlchemy models, per-user memory + vector collections, vector factory/async facade, OpenAI-compatible routes, native MCP client, replaceable Svelte frontend ([`main.py`](../../repos/open-webui/backend/open_webui/main.py:1085), [`memories.py`](../../repos/open-webui/backend/open_webui/models/memories.py:15), [`factory.py`](../../repos/open-webui/backend/open_webui/retrieval/vector/factory.py:1)) | User/resource isolation is strong, but universal tenant/workspace isolation and commercial credits/quotas are not primary built-ins |
| 4 | **AnythingLLM** | Approachable workspace chat/RAG/agent product or focused fork | Direct Node flow, Prisma workspace schema, global/workspace memory, generic OpenAI adapter, vector switch, agent plugins, MCP hypervisor ([`stream.js`](../../repos/anythingllm/server/utils/chats/stream.js:20), [`schema.prisma`](../../repos/anythingllm/server/prisma/schema.prisma:431), [`hypervisor/index.js`](../../repos/anythingllm/server/utils/MCP/hypervisor/index.js:21)) | Default single-container/local-storage architecture; tenant isolation and commercial usage/storage accounting require substantial cross-cutting work |

## Shortlist recommendation for Wave B

### 1. LibreChat — recommended first deep forensic target

**FACT:** LibreChat has the clearest explicit tenant boundary: AsyncLocalStorage tenant context, a Mongoose isolation plugin that injects filters across query/write operations, strict fail-closed behavior, and coverage tests for model/plugin coverage ([`tenantContext.ts`](../../repos/librechat/packages/data-schemas/src/config/tenantContext.ts:19), [`tenantIsolation.ts`](../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.ts:96), [`tenantIsolation.spec.ts`](../../repos/librechat/packages/data-schemas/src/models/plugins/tenantIsolation.spec.ts:656)). It also has dedicated memory, balances/transactions, projects, ACLs, files, agent queues/checkpoints, and an external RAG seam. **INFERENCE:** It offers the best starting point for a future commercial multi-user platform if its Mongo/agent architecture is acceptable. Wave B should trace tenant propagation through every model, agent checkpoint/job path, file ownership path, balance/usage path, and MCP authorization path.

### 2. Dify — second deep forensic target

**FACT:** Dify has the broadest integrated product architecture: tenant/workspace roles, model/provider plugins, workflow execution, agents, RAG pipelines, MCP, file adapters, Celery/Redis, and credit/quota entities ([`account.py`](../../repos/dify/api/models/account.py:21), [`workflow.py`](../../repos/dify/api/models/workflow.py:209), [`tools.py`](../../repos/dify/api/models/tools.py:299), [`quota_service.py`](../../repos/dify/api/services/quota_service.py:92)). **INFERENCE:** It may be the strongest complete-platform candidate, but its app/workflow/provider/plugin domain is more tightly coupled than LibreChat’s package-level seams. Wave B should determine whether its tenant and quota services can be adopted independently and whether personal/project memory can be added without a fork.

### 3. Open WebUI — targeted deep forensic candidate

**FACT:** Open WebUI has clean, inspectable FastAPI/SQLAlchemy boundaries, strong per-user memory, configurable vector backends, OpenAI-compatible APIs, functions/tools, and MCP. **INFERENCE:** It is an excellent candidate for frontend/API reuse or as a personal-AI/RAG subsystem, but less ready as the central workspace/tenant/billing authority. Wave B should test whether groups/access grants can be elevated into true workspace tenants, and whether chat context, memory, and knowledge can be project-partitioned without invasive route changes.

### 4. AnythingLLM — focused deep forensic candidate only

**FACT:** AnythingLLM is highly approachable and already has global/workspace memory, many provider/vector adapters, agents, MCP, and Prisma migrations. **INFERENCE:** Its direct code path makes experiments easy, but its default single-container/local-storage deployment and middleware/query-based isolation make it a weaker commercial multi-tenant foundation without a substantial hardening layer. Wave B should only prioritize it if implementation simplicity and Node.js ownership outweigh tenant/scaling requirements.

## Cross-cutting finding

**The major architectural divide is not feature count; it is isolation and replacement boundaries.** LibreChat is the only triage target with a source-visible, reusable tenant-isolation mechanism that automatically scopes database operations and fails closed in strict mode. Dify has the strongest integrated application/workflow/usage architecture but couples many concerns through its tenant/app/provider/plugin runtime. Open WebUI and AnythingLLM are easier to adapt for personal AI, RAG, and provider substitution, but their primary boundaries are user/resource or workspace middleware rather than a complete commercial tenant/credit/storage control plane. Therefore, Wave B should compare LibreChat and Dify first, then use Open WebUI as a modular subsystem/reference and AnythingLLM as an implementation-simplicity counterpoint.

## Modification signal comparison

| Capability | LibreChat | Dify | Open WebUI | AnythingLLM |
|---|---:|---:|---:|---:|
| Custom OpenAI-compatible provider | 1 | 1 | 0 | 0-1 |
| Add provider | 2 | 2 | 2 | 2 |
| Persistent user memory | 1 | 3 | 1 | 1 |
| Project memory | 2 | 2 | 3 | 1 |
| Context monitoring/condensation | 2 | 2-3 | 2-3 | 2-3 |
| Usage credits | 1-2 | 1 | 3 | 3 |
| Storage quotas | 2 | 2 | 3 | 3 |
| External integration | 1-2 | 1-2 | 1-2 | 1-2 |
| MCP server/tool | 1 | 1 | 1 | 1 |
| Independent API in front | 2 | 2 | 1 | 2 |
| Replace frontend | 1-2 | 2 | 1 | 2 |
| Connect candidate as RAG/memory | 1-2 | 3 | 1-2 | 2-3 |

Scores are preliminary triage classifications, not implementation estimates. They reflect source-visible seams and blast radius, not feature marketing claims.

## Evidence and uncertainty

The four repository reports contain the detailed FACT/INFERENCE/UNKNOWN classifications and file-level evidence:

- [`triage-librechat.md`](triage-librechat.md)
- [`triage-dify.md`](triage-dify.md)
- [`triage-open-webui.md`](triage-open-webui.md)
- [`triage-anythingllm.md`](triage-anythingllm.md)

The clones were shallow (`depth 1`), so release cadence, historical migration evolution, and tag-based upgradeability remain lower-confidence until Wave B inspects git metadata or a broader history. No repository outside the four assigned targets was cloned or analyzed.
