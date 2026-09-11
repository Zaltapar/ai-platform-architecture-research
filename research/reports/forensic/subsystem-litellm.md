# LiteLLM — model routing/provider abstraction

## Selection rationale

**FACT:** LiteLLM was selected because the repository contains both a provider-normalizing Python SDK and an OpenAI-compatible gateway with routing, fallbacks, budgets, rate limits, persistence, callbacks, tests, and deployment assets. The repository explicitly separates proxy/gateway concerns from SDK/provider concerns in [`ARCHITECTURE.md`](../../../repos/litellm/ARCHITECTURE.md:9). **INFERENCE:** This makes it the strongest Wave A candidate for testing whether model access can remain an independently replaceable platform subsystem.

## Architecture and stack

**FACT:** The SDK exposes synchronous/asynchronous completion entry points in [`main.py`](../../../repos/litellm/litellm/main.py:389) and provider handlers under [`llms`](../../../repos/litellm/litellm/llms:1). The gateway is an async Python service in [`proxy_server.py`](../../../repos/litellm/litellm/proxy/proxy_server.py:11010), with FastAPI/HTTP, Redis caching/coordination, PostgreSQL/Prisma persistence, and Prometheus deployment configuration in [`docker-compose.yml`](../../../repos/litellm/docker-compose.yml:1). **FACT:** [`ARCHITECTURE.md`](../../../repos/litellm/ARCHITECTURE.md:9) describes the gateway → SDK → provider path and the authentication, routing, cost, and asynchronous spend-log stages.

## Repository structure

**FACT:** [`litellm`](../../../repos/litellm/litellm:1) contains the SDK, [`litellm/llms`](../../../repos/litellm/litellm/llms:1) contains provider families, [`litellm/router_strategy`](../../../repos/litellm/litellm/router_strategy:1) contains routing strategies, [`litellm/proxy`](../../../repos/litellm/litellm/proxy:1) contains the gateway, [`migrations`](../../../repos/litellm/migrations:1) and [`schema.prisma`](../../../repos/litellm/schema.prisma:1) define gateway persistence, [`tests`](../../../repos/litellm/tests:1) contains unit/e2e/provider tests, and [`helm`](../../../repos/litellm/helm:1)/[`docker-compose.yml`](../../../repos/litellm/docker-compose.yml:1) provide deployment assets.

## Core data model

**FACT:** The Prisma schema uses PostgreSQL and defines credentials, proxy models, agents, organizations, teams, projects, budgets, keys, and spend-oriented relationships in [`schema.prisma`](../../../repos/litellm/schema.prisma:1). Budget records include model restrictions, parallel-request limits, TPM/RPM limits, and reset metadata in [`schema.prisma`](../../../repos/litellm/schema.prisma:12). **INFERENCE:** LiteLLM’s gateway data model is more than a stateless proxy: it is an API-key, organization, project, budget, and spend control plane.

## API/interface boundary

**FACT:** The SDK’s stable conceptual boundary is `completion`/`acompletion` in [`main.py`](../../../repos/litellm/litellm/main.py:389), while `Router.completion` and `Router.acompletion` provide model-list routing in [`router.py`](../../../repos/litellm/litellm/router.py:681). Provider-specific transformation/transport implementations sit behind [`BaseLLMHTTPHandler`](../../../repos/litellm/litellm/llms/custom_httpx/llm_http_handler.py:313). **FACT:** Custom provider callbacks use [`CustomLogger`](../../../repos/litellm/litellm/integrations/custom_logger.py:65), and the repository’s architecture says new proxy hooks register through the hook mechanism in [`ARCHITECTURE.md`](../../../repos/litellm/ARCHITECTURE.md:144).

## Deployment and persistence

**FACT:** The compose topology runs LiteLLM, PostgreSQL, and Prometheus; the proxy receives `DATABASE_URL` and persists PostgreSQL data in a named volume in [`docker-compose.yml`](../../../repos/litellm/docker-compose.yml:1). **FACT:** The architecture identifies Redis for API-key cache, rate counters, and spend queues and PostgreSQL for persistent records in [`ARCHITECTURE.md`](../../../repos/litellm/ARCHITECTURE.md:156). **INFERENCE:** A production deployment should treat Redis and PostgreSQL as shared control-plane infrastructure rather than optional implementation details when gateway features are enabled.

## Security and multi-tenancy assumptions

**FACT:** The schema models organizations, teams, projects, keys, object permissions, allowed models, and budgets in [`schema.prisma`](../../../repos/litellm/schema.prisma:86). **FACT:** Gateway authentication and key authorization are explicit stages in [`ARCHITECTURE.md`](../../../repos/litellm/ARCHITECTURE.md:22). **INFERENCE:** LiteLLM offers resource/account controls suitable for a platform’s model-access plane. **UNKNOWN:** This triage did not establish that every application-level tenant, file, conversation, or memory object would be isolated by LiteLLM; it is not a general SaaS tenant database.

## Extension/plugin model

**FACT:** Providers are organized as separate handler/configuration modules, and a custom LLM extension example implements `CustomLLM.completion` in [`custom_handler.py`](../../../repos/litellm/litellm/proxy/example_config_yaml/custom_handler.py:8). **FACT:** `Router` accepts model lists and routing settings in [`router.py`](../../../repos/litellm/litellm/router.py:681), while logger/hook classes provide callbacks in [`custom_logger.py`](../../../repos/litellm/litellm/integrations/custom_logger.py:65). **INFERENCE:** Adding an OpenAI-compatible endpoint is usually configuration or a small adapter; adding a provider with unusual semantics may require transformation, cost, streaming, and error mapping work.

## Failure, retry, and background behavior

**FACT:** Router retry policies are resolved by exception class/model group in [`get_retry_from_policy.py`](../../../repos/litellm/litellm/router_utils/get_retry_from_policy.py:31), and fallback handlers select alternate model groups in [`fallback_event_handlers.py`](../../../repos/litellm/litellm/router_utils/fallback_event_handlers.py:532). E2E tests prove timeout and context-window fallback behavior in [`test_reliability_retries_e2e.py`](../../../repos/litellm/tests/e2e/router/test_reliability_retries_e2e.py:1). **FACT:** Spend updates are asynchronously queued/batched according to the architecture flow in [`ARCHITECTURE.md`](../../../repos/litellm/ARCHITECTURE.md:60). **INFERENCE:** Retry semantics are a major part of the subsystem contract and must be preserved when wrapping LiteLLM.

## Observability and testing

**FACT:** The repository has provider unit tests, router reliability e2e tests, quota/rate-limit tests, callback tests, and a Prometheus compose service; examples include [`test_reliability_retries_e2e.py`](../../../repos/litellm/tests/e2e/router/test_reliability_retries_e2e.py:65) and [`prometheus.yml`](../../../repos/litellm/prometheus.yml:1). **UNKNOWN:** This triage did not measure coverage or validate all provider-specific production paths.

## Upgradeability and coupling

**FACT:** The SDK/provider boundary is explicit, but the gateway couples auth, budgets, routing, callbacks, cost calculation, Redis, and Prisma schema. The router itself imports cache, logging, token, cost, provider, and strategy modules in [`router.py`](../../../repos/litellm/litellm/router.py:38). **INFERENCE:** SDK-only adoption is relatively replaceable; gateway adoption creates a strategic dependency on LiteLLM’s key/budget/spend schema and operational behavior. **UNKNOWN:** Release-to-release compatibility of all provider transformations was not established from shallow history.

## Integration level and fit

- **Likely integration level:** **2 — SDK/library** for provider normalization and routing; **1 — HTTP/API** for the gateway; **3 — shared infrastructure/database** if its budgets, keys, or spend tables become authoritative.
- **Fit:** Strong as an independently owned model-access service or SDK adapter. Keep platform tenant, conversation, memory, billing entitlement, and policy records outside LiteLLM; pass a platform request/tenant correlation context into callbacks and gateway keys.
- **Not provided:** **FACT/UNKNOWN:** LiteLLM does not provide the target platform’s chat UI, conversation persistence, RAG corpus lifecycle, general memory, file tenancy, workflow engine, or product authorization model. Its organization/team/project model should not be mistaken for complete application tenancy.

## Modification difficulty experiments

| Experiment | Score | Reason |
|---|---:|---|
| Custom OpenAI-compatible endpoint | 0–1 | Existing OpenAI-like/custom handler paths in [`openai_like/handler.py`](../../../repos/litellm/litellm/llms/openai_like/chat/handler.py:223). |
| Add another provider | 2 | Provider handler/config/transformation plus cost/error/stream tests. |
| Context/token monitoring | 1–2 | Token utilities and usage/cost metadata exist, but product UI is external. |
| Usage credits | 1–2 | Budget/spend schema exists, but platform entitlements remain external. |
| MCP/tool integration | 1–2 | MCP client/proxy paths exist, but tool authorization is not a complete platform policy layer. |
| Replace gateway persistence | 4 | Prisma schema, auth, budgets, spend logs, management APIs, and migrations are cross-cutting. |

## Bottom line

**INFERENCE:** LiteLLM is a high-value replaceable subsystem when bounded to model invocation, routing, retries, cost/usage events, and provider translation. It should not own the commercial platform’s canonical tenant or application data model.
