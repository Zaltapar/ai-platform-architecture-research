# MCP Python SDK — MCP runtime and tool integration

## Selection rationale

**FACT:** The official Python SDK was selected instead of a gateway product because the architecture question needs the lowest-level replaceable MCP boundary: protocol types, server/client sessions, transports, auth hooks, lifecycle context, tools/resources/prompts, extensions, and tests. The package identifies itself as the Model Context Protocol SDK in [`pyproject.toml`](../../../repos/mcp-python-sdk/pyproject.toml:1). **INFERENCE:** This reveals what a platform must own around MCP rather than conflating protocol support with tenancy or tool governance.

## Architecture and stack

**FACT:** The SDK is Python >=3.10, Pydantic-based, uses AnyIO, Starlette, SSE, Streamable HTTP, JSON-RPC/MCP types, JWT/OAuth helpers, and optional OpenTelemetry in [`pyproject.toml`](../../../repos/mcp-python-sdk/pyproject.toml:114). **FACT:** The high-level server is [`MCPServer`](../../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:157), while low-level protocol server code is under [`lowlevel`](../../../repos/mcp-python-sdk/src/mcp/server/lowlevel:1). **FACT:** The former `FastMCP` module is intentionally removed/renamed to `MCPServer` in [`fastmcp.py`](../../../repos/mcp-python-sdk/src/mcp/server/fastmcp.py:1).

## Repository structure

**FACT:** [`src/mcp-types`](../../../repos/mcp-python-sdk/src/mcp-types:1) contains generated/wire models; [`src/mcp/server`](../../../repos/mcp-python-sdk/src/mcp/server:1) contains server/runtime/transports/auth; [`src/mcp/client`](../../../repos/mcp-python-sdk/src/mcp/client:1) contains client/session/auth; [`examples`](../../../repos/mcp-python-sdk/examples:1) contains server/client stories; and [`tests`](../../../repos/mcp-python-sdk/tests:1) contains transport, auth, interaction, race, and protocol conformance tests.

## Core data model

**FACT:** Protocol data is typed as JSON-RPC/MCP wire models such as tools, call results, resources, prompts, capabilities, and request parameters imported by [`MCPServer`](../../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:13). **FACT:** Streamable HTTP sessions have session IDs, event IDs, JSON/SSE response modes, and an injectable [`EventStore`](../../../repos/mcp-python-sdk/src/mcp/server/streamable_http.py:105) for resumability. **INFERENCE:** The SDK’s durable domain model is protocol/session state, not application records, conversations, users, or tenants.

## API/interface boundary

**FACT:** `MCPServer` accepts tools, resources, extensions, lifespan, authentication providers, token verifiers, and auth settings in [`server.py`](../../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:157). **FACT:** Tool/resource/prompt registration is delegated to managers imported in [`server.py`](../../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:81). **FACT:** Extension hooks use `Extension`, `MethodBinding`, request handlers, and tool-call composition in [`server.py`](../../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:63). **FACT:** Server transport supports stdio, SSE, and Streamable HTTP; Streamable HTTP’s transport constructor exposes event storage, security settings, retry interval, and idle timeout in [`streamable_http.py`](../../../repos/mcp-python-sdk/src/mcp/server/streamable_http.py:150).

## Deployment and persistence

**FACT:** The SDK is a Python package with a CLI entry point in [`pyproject.toml`](../../../repos/mcp-python-sdk/pyproject.toml:31), and HTTP examples build Starlette applications, e.g. [`oauth/server.py`](../../../repos/mcp-python-sdk/examples/stories/oauth/server.py:1). **FACT:** `EventStore` is an abstract interface, not a built-in durable database implementation, in [`streamable_http.py`](../../../repos/mcp-python-sdk/src/mcp/server/streamable_http.py:116). **INFERENCE:** A production platform must provide process hosting, session/event persistence, horizontal routing, secrets, and operational deployment around the SDK.

## Security and multi-tenancy assumptions

**FACT:** The SDK contains bearer authentication middleware, OAuth authorization-server provider interfaces, token verification, auth settings, and request-state security imports in [`server.py`](../../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:57). **FACT:** Tests cover protected-resource metadata, OAuth discovery, issuer/resource validation, token storage, client registration, and scope step-up in [`tests/client/test_auth.py`](../../../repos/mcp-python-sdk/tests/client/test_auth.py:270). **INFERENCE:** The SDK provides protocol authentication primitives and extension points. **UNKNOWN:** It does not establish application tenant/workspace membership, per-tenant tool catalogs, billing, quotas, or an authorization policy engine; those must be implemented by the host platform/server.

## Extension/plugin model

**FACT:** Tools, resources, prompts, lifecycle contexts, middleware/extensions, custom auth providers, token verifiers, event stores, subscription buses, and transport security settings are explicit constructor/interface seams in [`server.py`](../../../repos/mcp-python-sdk/src/mcp/server/mcpserver/server.py:57). **FACT:** Examples show low-level `list_tools`/`call_tool` implementations in [`tools/server_lowlevel.py`](../../../repos/mcp-python-sdk/examples/stories/tools/server_lowlevel.py:31). **INFERENCE:** The SDK is highly extensible at protocol/runtime level, but it does not discover or govern arbitrary third-party plugins by itself.

## Failure, retry, and background behavior

**FACT:** Streamable HTTP handles long-running bidirectional communication, request-scoped response streams, cancellation, idle timeout, and optional resumability in [`streamable_http.py`](../../../repos/mcp-python-sdk/src/mcp/server/streamable_http.py:150). **FACT:** OAuth client tests explicitly validate retrying the original request after token acquisition and reject unsafe redirects/issuer mismatches in [`tests/client/test_auth.py`](../../../repos/mcp-python-sdk/tests/client/test_auth.py:1249). **UNKNOWN:** The SDK does not provide a durable background job queue, workflow retry ledger, execution scheduler, or at-least-once business-task semantics; tool handlers and the host must supply those.

## Observability and testing

**FACT:** Development dependencies include pytest, pyright, coverage, Logfire, and OpenTelemetry in [`pyproject.toml`](../../../repos/mcp-python-sdk/pyproject.toml:56). **FACT:** Tests cover transports, session hosting, Streamable HTTP races, auth, OAuth conformance, and interaction requirements; examples include [`test_streamable_http.py`](../../../repos/mcp-python-sdk/tests/interaction/transports/test_streamable_http.py:244). **INFERENCE:** Protocol behavior is strongly tested, but host-specific tool side effects, tenant authorization, and production queue behavior remain outside SDK coverage.

## Upgradeability and coupling

**FACT:** Protocol types and transports are separated into packages/modules, and extension/auth/event-store interfaces are explicit. **FACT:** The major-version migration warning in [`fastmcp.py`](../../../repos/mcp-python-sdk/src/mcp/server/fastmcp.py:1) shows that public names and APIs can change materially. **INFERENCE:** Using the protocol interfaces through a narrow adapter is replaceable; embedding high-level `MCPServer` internals throughout a product increases upgrade cost, especially around major protocol/API changes.

## Integration level and fit

- **Likely integration level:** **2 — SDK/library** for an in-process MCP client/server; **1 — HTTP/API** when the platform hosts MCP servers behind its own gateway; **3 — shared infrastructure** only if the platform supplies shared event/session storage.
- **Fit:** Excellent protocol adapter/runtime boundary. Use platform-owned tool registration records, project/tenant policy, credential references, approval UI, audit events, rate limits, and sandbox execution around it.
- **Not provided:** **FACT/UNKNOWN:** No complete MCP gateway, tenant registry, tool marketplace, secrets vault, sandbox, durable job engine, billing, vector store, or application-level permission model is provided by the SDK.

## Modification difficulty experiments

| Experiment | Score | Reason |
|---|---:|---|
| Add an MCP server/tool | 0–1 | Register a tool/resource/prompt or implement low-level handlers. |
| OAuth-protected MCP endpoint | 1–2 | Auth provider/token verifier and Starlette transport seams exist. |
| Durable resumable sessions | 2–3 | Implement `EventStore` and external session routing. |
| Tenant-aware tool authorization | 2–3 | Host must derive policy from authenticated project context and enforce before calls. |
| Background tool execution | 3 | Requires external queue/workflow and result correlation. |
| Replace transport | 2–3 | Protocol/session assumptions span client/server transport modules and tests. |

## Bottom line

**INFERENCE:** The SDK should be treated as a narrow, replaceable protocol implementation. It materially lowers MCP integration cost, but adopting it does not eliminate the need for a platform-owned MCP control plane and security boundary.
