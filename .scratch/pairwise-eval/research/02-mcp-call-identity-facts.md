# 02: How do traces identify an MCP server call? (findings)

Research for ticket `issues/02-mcp-call-identity-facts.md`. Gathered 2026-10-10 against MLflow master `7a010ed01`. Paths without a host are repo-relative to MLflow. External sources are pinned to the tag or commit named in **Sources**. This file reports facts only. It recommends no design.

## Answer

- **No tracer records a version for an MCP server.** No MLflow integration, and neither Claude Code's nor Codex's native OTel output, puts `serverInfo.version` (or any other MCP server version) on a span. So an MCP server Component's Variant cannot be read from today's traces.
- **What tracers do record is the client-side server name and the tool name, with uneven coverage:**
  - **Claude Code (MLflow plugin and Agent SDK paths):** only the combined tool name, `mcp__<server>__<tool>`.
  - **Claude Code native OTel spans:** a `tool_name`, but no server attribute. The server and tool names appear only on OTel *log events*, only with `OTEL_LOG_TOOL_DETAILS=1`, and MLflow does not ingest log events.
  - **Codex native OTel:** an `mcp.tools.call` span with `mcp.server.name`, `tool.name`, and `mcp.server.origin`. The origin is only the URL's scheme, host and port, or `stdio`. MLflow has no Codex translator, so these land as raw attributes.
  - **Pydantic AI 1.x autolog:** the `MCPServer` object's public primitive attributes, including `url` or `command`/`args`.
  - **Pydantic AI 2.x autolog:** tool name and args only.
  - **OpenAI Agents autolog:** drops the SDK's `mcp_data.server` field.
  - **LangChain autolog:** the server name appears only if `langchain-mcp-adapters` was asked to prefix tool names.
- **MCP spec:**
  - **Up to 2025-11-25:** `InitializeResult.serverInfo` is a required `Implementation` with required `name` and `version` strings. No version format is specified, and the value is self-reported.
  - **Current revision (2026-07-28):** the `initialize` handshake is removed. `serverInfo` moves to an optional (SHOULD) `_meta["io.modelcontextprotocol/serverInfo"]` on each result, described as self-reported, unverified and "for display, logging, and debugging".
  - **SDK defaults:** the MCP Python SDK v2.3.0 reports an empty `version` when a server sets none. v1.30.0 substituted the `mcp` package's own version.
- **Resolving a span to the registry:**
  - **Name:** the registry identifies servers by `namespace/slug` plus a semver `version` taken from `server_json.version`. No trace field carries either.
  - **Client-side name:** set by whoever configures the client. MLflow's own "Connect" instructions turn `com.acme/full-mcp-server` into `com-acme-full-mcp-server`, a lossy mapping that carries no version.
  - **Tools:** the tools snapshot is optional, holds raw MCP tool names, and has no search filter.
  - **Endpoints:** `MCPAccessEndpoint.url` is not unique, has no index or search filter, and may target a moving alias.
  - **`serverInfo`:** the registry does not store it, and MLflow's tool discovery reads it during `initialize` and then discards it.
- **`link_mcp_server_versions_to_trace(trace_id, mcp_servers: list[MCPServerVersion]) -> None` and `get_mcp_server_versions_for_trace(trace_id) -> list[MCPServerVersion]`:**
  - **Status:** declared in #24380 ("Trace linking will come at a later date").
  - **Design:** RFC 0004 describes an explicit, after-the-fact link that a runtime or server creates when it already knows the `{workspace, name, version}`. It is not inferred from span contents.
  - **Closed prototype:** PR #24066 (never merged) stored the link as `entity_associations` rows with `destination_type="mcp_server_version"` and `destination_id="<name>/<version>"`.
  - **Access endpoints:** the RFC says they "describe direct access only" and defines no trace-to-endpoint API.

## 1. What identifying information sits on an MCP tool span

Summary. "—" means the information is not on the span.

| Tracer | Client-side server name | Tool name | Server URL / command | `serverInfo.name` / `.version` |
| - | - | - | - | - |
| Claude Code, MLflow Stop-hook plugin (TS) | Only inside the tool name (`mcp__<server>__<tool>`) | `tool_name` attribute and span name `tool_<name>` | — | — |
| Claude Agent SDK, MLflow Python autolog | Only inside the tool name | Same as above | — | — |
| Claude Agent SDK, MLflow TS `tracedClaudeAgent` wrapper | Inside the tool name. Also in root-span `inputs.options.mcpServers` keys when the caller passes them | Same as above | Only if passed in `options.mcpServers` (root span, not the tool span) | — (available from the SDK, not recorded) |
| Claude Code native OTel, through MLflow `ClaudeCodeTranslator` | — on spans (logs-only, flag-gated) | `tool_name` (ungated per docs) | — | — |
| Codex native OTel (no MLflow translator) | `mcp.server.name` | `tool.name` | `mcp.server.origin` (scheme://host:port or `stdio`), `server.address`, `server.port` | — |
| Pydantic AI 1.x autolog (`MCPServer.call_tool`) | `id` / `tool_prefix` if set | Input `name` | `url` (HTTP) or `command` + `args` (stdio) as span attributes | — (filtered out) |
| Pydantic AI 2.x autolog (`MCPToolset.direct_call_tool`) | — | Input `name` | — | — |
| OpenAI Agents SDK autolog | — (SDK's `mcp_data.server` dropped) | Span name | — | — |
| LangChain / LangGraph + `langchain-mcp-adapters` | Only if `tool_name_prefix=True` (`<server>_<tool>`) | Span name | — | — |

### 1.1 Claude Code: MLflow's hook-based tracer

- `mlflow/claude_code/` no longer contains the hook runtime. `hooks.py` is a no-op shim saying tracing "has moved to the marketplace plugin runtime" (`mlflow/claude_code/hooks.py:1-16`). `mlflow autolog claude` installs the plugin `mlflow-tracing@mlflow-plugins` from `libs/typescript/integrations/claude-code` (`mlflow/claude_code/plugin.py:22-55`).
- The plugin registers only a `Stop` hook (`libs/typescript/integrations/claude-code/hooks/hooks.json`). The hook input type carries only `session_id` and `transcript_path` (`libs/typescript/integrations/claude-code/src/types.ts:102-105`). The tracer re-reads the transcript JSONL.
- **Transcript fields read:** the `tool_use` block's `id`, `name` and `input` (`src/types.ts:19-24`).
- **TOOL span shape:** named `tool_${toolName}`, with inputs `toolUse.input` and attributes `{tool_name, tool_id}` only (`src/tracing.ts:474-485`).
- **Trace metadata:** session, user, working directory, permission mode and Claude Code version (`src/tracing.ts:585-604`). There is no MCP field.
- The plugin source contains no MCP-specific handling. A `grep -i mcp` over `libs/typescript/integrations/claude-code/src` matches only the comments in `_internal.ts` / `liveTracing.ts` about sanitizing `mcpServers` callbacks.
- **Claude Agent SDK, Python:** `mlflow.anthropic.autolog()` patches `ClaudeSDKClient` and hands the collected messages to `process_sdk_messages` (`mlflow/anthropic/autolog.py:21-72`). TOOL spans get attributes `{"tool_name", "tool_id"}` only (`mlflow/claude_code/tracing.py:312-321`). The root span input is `{"prompt"}` only (`mlflow/claude_code/tracing.py:377-382`). The init `SystemMessage` is not read.
- **Claude Agent SDK, TS wrapper (`tracedClaudeAgent`):**
  - Tool spans have the same `{tool_name, tool_id}` shape (`src/liveTracing.ts:181-187`).
  - The wrapper records the caller's `options` (sanitized of functions) as the root span's `inputs.options` (`src/liveTracing.ts:107-113`, `src/_internal.ts:95-121`). If the caller passes `mcpServers`, its keys (client-side names) and configs (command/url) land on the **root** span, not on the tool spans.
  - From the init message it reads only `session_id`, `model`, `cwd`, `permissionMode` and `claude_code_version` (`src/liveTracing.ts:62-70`, `134-140`). It ignores the init message's `mcp_servers` list.
- **Data the Agent SDK exposes but MLflow does not record:**
  - The init message carries `mcp_servers: {name, status, source?}[]`.
  - `mcpServerStatus()` returns `McpServerStatus` with an optional `serverInfo: {name: string; version: string}` and `config` (the transport config).
  - `PreToolUse` and other tool hooks receive `mcp_server: {name, source}` for MCP tools (from Agent SDK v0.3.274 / Claude Code v2.1.274).
  - Sources: Agent SDK TS reference, sections `SDKSystemMessage`, `McpServerStatus`, `McpServerProvenance`; hooks reference, "PreToolUse input".

### 1.2 Claude Code: tool naming and native OTel

- **Naming:** Claude Code docs say "MCP tools follow the naming pattern `mcp__<server>__<tool>`", for example `mcp__memory__create_entities` (hooks reference, "Match MCP tools").
- **Plugin-bundled servers:** these use `mcp__plugin_<plugin-name>_<server-name>__<tool>`, "where any character outside `A-Z`, `a-z`, `0-9`, `_`, and `-` is replaced with `_`" (MCP docs, "Plugin MCP tool names"). The same replacement is documented for the server segment of `/mcp__servername__promptname` (MCP docs, prompts section).
- **Observation from this session:** the tools exposed in this Claude Code session follow the same shapes, for example `mcp__claude_ai_Notion__notion-search` and `mcp__plugin_linear_linear__get_issue`.
- **Docs caveat:** the docs warn against relying on the name for trust: "Base trust decisions on `source`, not on `name` or the `mcp__<server>__` tool-name prefix" (Agent SDK TS reference, `McpServerProvenance`).
- **Native OTel `claude_code.tool` span attributes** (monitoring docs, "Traces (beta) → Span attributes"):
  - The span carries `tool_name` (ungated), `tool_name_safe`, `tool_use_id` / `gen_ai.tool.call.id`, `duration_ms` and `result_tokens`.
  - Detail-gated attributes (`file_path`, `full_command`, `skill_name`, `subagent_type`) cover only built-in tools.
  - There is no MCP server, URL or version attribute on the span. `tool_name_safe` is described as the "form of `tool_name` that carries no user-chosen names … MCP tool names appear as `mcp_other`", which implies `tool_name` itself carries the user-chosen MCP name. The docs do not spell out its exact format for MCP tools (inference).
- **OTel log events:** the MCP identity lives on log events, not spans (monitoring docs, "Tool result event", "Tool decision event", "MCP server connection event", "Audit MCP activity"):
  - **`tool_result`:** has `mcp_server_scope`. With `OTEL_LOG_TOOL_DETAILS=1`, `tool_parameters` includes `mcp_server_name` and `mcp_tool_name`. Without the flag, `tool_name` is "redacted to the literal `"mcp_tool"` for user-configured servers".
  - **`tool_decision`:** has `tool_source` (`"mcp"` / `"sdk_host_builtin_mcp"` / `"builtin"`) and the same `tool_parameters` behaviour.
  - **`mcp_server_connection`:** has `status`, `transport_type`, `server_scope`, and `server_name` ("Configured server name", only with `OTEL_LOG_TOOL_DETAILS=1`).
  - None of the documented events or spans carries a server URL, server version or `serverInfo`.
- **Trace context:** "Outbound HTTP MCP requests carry `traceparent`" when tracing is active and Claude Code talks directly to the Anthropic API (monitoring docs, "Traces (beta)").
- **MLflow translator:** `ClaudeCodeTranslator` maps `claude_code.tool` to TOOL. Its fallback input reads only `_TOOL_DETAIL_KEYS = ["full_command", "file_path", "skill_name", "subagent_type"]` (`mlflow/tracing/otel/translation/claude_code.py:36`, `111-128`). It reads no MCP key.
- **MLflow OTLP endpoint:** it is a traces endpoint only (`otel_router` prefix `OTLP_TRACES_PATH`, `mlflow/server/otel_api.py:104`). Claude Code's log events, where the MCP names live, do not reach MLflow.

### 1.3 Codex

Sources: openai/codex at tag `rust-v0.162.1`.

- **Model-visible tool names:** Codex keeps the raw server and tool names separate. `ToolInfo.server_name` is the "Raw MCP server name used for routing", and `tool.name` is "sent back to the MCP server". The model sees a sanitized `callable_namespace` / `callable_name`, optionally prefixed with `mcp__` (`codex-rs/codex-mcp/src/tools.rs:24-60`, `228-234`). Hooks see `mcp__<namespace>__<name>` (`codex-rs/core/src/tools/handlers/mcp.rs:46-47`, `115-137`).
- **The `mcp.tools.call` span:** every MCP call runs inside a `tracing` span named `mcp.tools.call` (`codex-rs/core/src/mcp_tool_call.rs:594-604`, `658-702`) with:
  - `otel.kind = "client"`, `rpc.system = "jsonrpc"`, `rpc.method = "tools/call"`;
  - `mcp.server.name` and `tool.name`;
  - `mcp.server.origin`, `mcp.transport` and `mcp.connector.id/name`;
  - `tool.call_id`, `conversation.id`, `session.id` and `turn.id`;
  - `server.address` and `server.port`, filled from the origin (`record_server_fields`, `mcp_tool_call.rs:704-717`).
- **Field sources:** `server` and `tool` come from `McpInvocation`: "Name of the MCP server as defined in the config" and "Name of the tool as given by the MCP server" (`codex-rs/protocol/src/protocol.rs:2686-2693`).
- **`mcp.server.origin`:** this is `McpServerOrigin`: `"stdio"` for stdio servers, or for streamable HTTP the URL's `origin().ascii_serialization()` (scheme, host, port; **no path**) (`codex-rs/codex-mcp/src/server.rs:421-445`).
- **Export:** Codex's OTel trace layer exports every span except `h2` ones (`codex-rs/otel/src/provider.rs:341-350`), when `otel.trace_exporter` is configured (Codex config reference, `otel.trace_exporter`).
- **`codex.tool_result` event:** emitted on two channels (`codex-rs/otel/src/tool_result.rs:23-81`; macro split in `codex-rs/otel/src/events/shared.rs:14-62`):
  - **Log channel:** carries `mcp_server` and `mcp_server_origin` (from `McpHandler::telemetry_tags`, `codex-rs/core/src/tools/handlers/mcp.rs:441-447`).
  - **Trace channel:** carries only `tool_origin = "mcp" | "builtin"` and `mcp_tool: bool`.
- **`codex.conversation_starts`:** logs the configured `mcp_servers` names on the log channel and only `mcp_server_count` on the trace channel (`codex-rs/otel/src/events/session_telemetry.rs:688-713`).
- **`serverInfo`:** Codex keeps the `initialize` result's `server_info` in memory (`ManagedClient.server_info`, `codex-rs/codex-mcp/src/rmcp_client.rs:124`, `1005-1016`). None of the span or event fields above includes it.
- **MLflow side:**
  - There is no Codex translator. The registered translators are listed in `mlflow/tracing/otel/translation/__init__.py:38-56`, and none is for Codex.
  - The OTLP endpoint only allowlists `codex_cli_rs` / `codex_vscode` as service names (`mlflow/server/otel_api.py:62-73`).
  - Ingested OTLP spans keep all attributes verbatim (`mlflow/entities/span.py:562-565`). A Codex `mcp.tools.call` span would therefore arrive in MLflow with its `mcp.server.name` / `tool.name` / `mcp.server.origin` attributes untranslated. This was not tested end to end here.

### 1.4 Pydantic AI autolog

- **1.x:**
  - `mlflow.pydantic_ai.autolog` patches `pydantic_ai.mcp.MCPServer.call_tool` and `list_tools` (`mlflow/pydantic_ai/__init__.py:191`).
  - The span is named `<Class>.<method>` (for example `MCPServerStdio.call_tool`), typed TOOL, with inputs `{name, args}` (`mlflow/pydantic_ai/autolog.py:141-159`, `429-430`; `tests/pydantic_ai/test_pydanticai_mcp_tracing.py:68-112`).
  - Span attributes come from `_extract_safe_attributes(instance)`: every public, non-callable attribute whose value is a `str/int/float/bool`, or a dict, list or tuple of such values, or a dataclass instance (`mlflow/pydantic_ai/autolog.py:24`, `27-74`, `77-86`, `481-489`).
  - For pydantic-ai v1.107.7 `MCPServer`, that covers `id`, `tool_prefix`, `timeout` and so on; `command`, `args`, `env`, `cwd` on `MCPServerStdio`; and `url`, `headers` on HTTP servers (`pydantic_ai_slim/pydantic_ai/mcp.py` at v1.107.7: lines 684-764, 866-879, 1430-1443, 1591-1600). The unit test asserts `{"name": "my_server", "url": "http://localhost:8080"}` is kept (`tests/pydantic_ai/test_pydanticai_tracing.py:410-414`).
  - `env` and `headers` dicts of strings pass the filter, so secrets in them would be recorded.
- **`server_info` in 1.x:** pydantic-ai 1.x exposes `MCPServer.server_info` (set from `initialize` `result.serverInfo`, `mcp.py:885-892`, `1248-1252` at v1.107.7). Its type is `mcp.types.Implementation`, a Pydantic `BaseModel` (MCP Python SDK v1.30.0 `src/mcp/types.py:231`, `263`). It is neither a primitive nor a dataclass, so MLflow's filter drops it.
- **2.x:** `MCPToolset.direct_call_tool` gets a span `MCPToolset.direct_call_tool` with inputs `{name, args}` and no attributes. `list_tools` likewise gets no attributes (`mlflow/pydantic_ai/autolog_v2.py:210-242`, `461-476`; `tests/pydantic_ai/test_pydanticai_v2_tracing.py:418-474`). No server identity is recorded at all.

### 1.5 OpenAI Agents SDK autolog

- **What the SDK provides** (openai-agents-python v0.23.1):
  - `FunctionSpanData` has an `mcp_data` field, which `invoke_mcp_tool` sets to `{"server": get_mcp_server_log_name(server.name)}` (`src/agents/tracing/span_data.py:135-166`; `src/agents/mcp/util.py:834-843`).
  - `MCPListToolsSpanData` (`type == "mcp_tools"`) carries `server` and `result` (`span_data.py:428-451`).
  - Server names default to `"stdio: <command>"`, `"sse: <url>"` or `"streamable_http: <url>"` when the user passes no name (`src/agents/mcp/server.py:2070`, `2214`, `2393`). The log-name helper strips URL credentials, query and fragment (`src/agents/mcp/_logging.py`).
- **What MLflow keeps:**
  - `_parse_span_data` reads only `input`/`output` for function spans and never reads `mcp_data` (`mlflow/openai/_agent_tracer.py:215-258`).
  - `mcp_tools` is not in `_SPAN_TYPE_MAP`, so it becomes CHAIN, named "Unknown" (no `name` attribute), with no inputs, outputs or attributes (`_agent_tracer.py:53-60`, `140-149`, `202-212`).
  - The span name is the function tool name, which is `mcp_<server>__<tool>` only when the agent sets `include_server_in_tool_names` (`src/agents/mcp/util.py:330-337`, `448-451`).

### 1.6 LangChain / LangGraph with `langchain-mcp-adapters`

- MLflow has no MCP-specific LangChain code. `MlflowLangchainTracer.on_tool_start` names the span with the tool name and stores the callback `metadata` as a span attribute (`mlflow/langchain/langchain_tracer.py:637-673`).
- `langchain-mcp-adapters` (main `52a4535`) builds a `StructuredTool` whose `name` is the raw MCP tool name, or `f"{server_name}_{tool.name}"` only when `tool_name_prefix=True`. Its `metadata` holds the MCP tool `annotations` plus `_meta`, not the server (`langchain_mcp_adapters/tools.py:510-535`).
- LangChain forwards `self.metadata` into the tool callback (`langchain_core/tools/base.py:1054`, `1073`, `1182`, `1201` at langchain master `b060a74`). So MLflow LangChain spans carry the annotations, but carry the server name only via the optional prefix.

### 1.7 `mlflow/mcp` decorator and MLflow's own MCP code

- `@mlflow_mcp(tool_name=...)` only stores a tool name on a Click command, so the `mlflow mcp` *server* can register it (`mlflow/mcp/decorator.py:27-53`). It does no tracing.
- The server is `FastMCP(name="Mlflow MCP", tools=...)` with no `version` argument (`mlflow/mcp/server.py:240-243`).
- **Registry tool discovery:** discovery opens a `fastmcp.Client` (which runs `initialize`) and returns only `list_tools()` results (`mlflow/genai/mcp_tool_discovery.py:96-127`). The `serverInfo` returned during `initialize` is not stored.

## 2. MCP spec: what `serverInfo` contains, and whether a version is guaranteed

- **Revisions up to 2025-11-25:**
  - `InitializeResult` has required `protocolVersion`, `capabilities` and `serverInfo: Implementation` (`schema/2025-11-25/schema.ts:279-293`).
  - `Implementation` extends `BaseMetadata` (`name: string` required, `title?`) and `Icons`, with **required `version: string`**, plus optional `description` and `websiteUrl` (`schema.ts:528-571`).
  - The schema puts no format on `version`.
- **Current revision, 2026-07-28:**
  - The versioning page names it current (`docs/docs/2026-07-28/learn/versioning.mdx:27`).
  - The changelog says it removes the `initialize` handshake: "servers SHOULD identify themselves in each result's `_meta` (`io.modelcontextprotocol/serverInfo`)". It adds a mandatory `server/discover` RPC (`docs/specification/2026-07-28/changelog.mdx:14-16`).
  - In the schema, `ResultMetaObject["io.modelcontextprotocol/serverInfo"]?: Implementation` is optional. The doc comment reads: "Servers SHOULD include this field on every response unless specifically configured not to … The value is self-reported by the server and is not verified by the protocol. It is intended for display, logging, and debugging. Clients SHOULD NOT use it to change their behavior" (`schema/2026-07-28/schema.ts:143-158`).
  - `Implementation.version` is still required inside the object (`schema.ts:976-1000`). `DiscoverResult` itself has no `serverInfo` field (`schema.ts:678-697`).
- **Is a version guaranteed?**
  - Up to 2025-11-25, a `version` string field is required, but its content is free-form and self-reported.
  - In 2026-07-28, the whole `serverInfo` object is SHOULD, not MUST.
  - SDK defaults vary. The MCP Python SDK v2.3.0 reports `version=""` for an unversioned server: "An unversioned server reports an empty `version`; the SDK never substitutes its own" (`src/mcp/server/lowlevel/server.py:640-653`). v1.30.0 fell back to the `mcp` package version (`server.py:183`).
- **Recorded in any trace above?** No. Pydantic AI 1.x has it in memory but MLflow filters it out (§1.4). Codex keeps it in memory but does not emit it (§1.3). The Claude Agent SDK exposes it via `mcpServerStatus()`, but no MLflow tracer calls that (§1.1). Claude Code's documented OTel output has no such field (§1.2).

## 3. Could a call be resolved to a registry entry and an `MCPServerVersion`?

What the registry holds:

- **`MCPServer.name`:**
  - It must be `<reverse-dns namespace>/<slug>` (`mlflow/entities/mcp_server.py:41-80`).
  - `register_mcp_server` takes it from `server_json["name"]` and the version from `server_json["version"]`, which must parse as semver (`mlflow/genai/mcp_servers.py:163-170`).
  - The RFC says the registry *enforces* the upstream "SHOULD follow semantic versioning" (RFC 0004, "Version string conventions").
- **`MCPServerVersion`:** holds `name`, `version`, `server_json`, `status`, `tools: list[MCPTool] | None`, `aliases`, `tags` and `source` (`mlflow/entities/mcp_server_version.py:20-34`).
  - `MCPTool.name` is the MCP tool name as the server lists it (`mlflow/entities/mcp_server.py:85-128`).
  - `tools` is auto-discovered only from the first usable `server_json.remotes[]` URL. It is `None` when discovery fails, is disabled (`MLFLOW_ENABLE_MCP_TOOL_DISCOVERY`, default `True`), or there is no remote (`mlflow/genai/mcp_tool_discovery.py:152-240`; `mlflow/environment_variables.py:728-730`).
  - Version search filters accept only `name`, `version`, `status`, `created_at` and `last_updated_at`. There is no tool-name filter (`mlflow/utils/search_utils.py:2797-2811`).
- **`MCPAccessEndpoint`:** holds `server_name`, `url`, `transport_type`, and exactly one of `server_version` or `server_alias` (`mlflow/entities/mcp_access_endpoint.py:17-29`; `abstract_mixin.py:267-291`).
  - The table has no unique constraint or index on `url`. Indexes cover `(workspace, server_name)`, `(…, server_version)` and `(…, server_alias)` (`mlflow/store/tracking/dbmodels/models.py:4618-4680`).
  - Endpoint search filters accept `status`, `server_name`, `transport_type` and timestamps, but not `url` (`mlflow/utils/search_utils.py:2814-2825`).
  - An alias-targeted endpoint resolves to whichever version the alias points at when it is read (RFC 0004, "Binding target"). The docs say "endpoint follows alias as it moves between versions" (`docs/docs/genai/mcp-registry/connect-to-servers.mdx:50`).
  - `register_mcp_server(..., create_access_endpoints_from_remotes=True)` creates one endpoint per `server_json.remotes[]` URL, pinned to the new version (`mlflow/genai/mcp_servers.py:197-214`).
- **Client-side name produced by MLflow's own "Connect" instructions:**
  - `deriveClientName(registryName)` replaces every character outside `[A-Za-z0-9_-]` with `-`, collapses repeated dashes and lowercases. For example, `com.acme/full-mcp-server` becomes `com-acme-full-mcp-server` (`mlflow/server/js/src/mcp-registry/installInstructions.ts:60-67`; tests `installInstructions.test.ts:117-136`).
  - That name is used as the `claude mcp add` server name and as the `.mcp.json` key (`installInstructions.ts:52-54`, `256`, `699-708`).
  - Package installs pin the package version from `server_json.packages[].version` into the command (`@x.y.z` for npm, `==` for PyPI, `:` for OCI) (`installInstructions.ts:190-203`). No version is in the server name.

What would have to match, given what traces carry:

| Signal on the trace | Registry field it would have to match | Facts that limit the match |
| - | - | - |
| Client-side server name (Claude Code `mcp__<server>__`, Codex `mcp.server.name`, OpenAI Agents `mcp_data.server`, Pydantic AI `id`) | `MCPServer.name` | Chosen by whoever configured the client. MLflow's own instructions emit a lossy transform (`.` and `/` both become `-`). It never carries a version. Claude Code also rewrites the plugin-server segment. |
| Tool name(s) | `MCPServerVersion.tools[].name` | The snapshot may be `None`. There is no search filter on tool names. Different servers, and different versions of one server, can expose the same tool names. Claude Code and OpenAI Agents (when prefixing) embed the server name in the tool name. |
| Endpoint URL | `MCPAccessEndpoint.url` or `server_json.remotes[].url` | Only Codex (origin only, no path) and Pydantic AI 1.x (full `url`) put it on spans. There is no unique constraint, index or search filter on `url`. An alias-targeted endpoint's version depends on when it is read. `remotes[].url` may be a template with variables (RFC 0004, `server_json` section; `installInstructions.ts:229-251`). |
| Launch command / package (stdio) | `server_json.packages[]` (identifier, version) | Only Pydantic AI 1.x records `command`/`args`. No tracer above records it for Claude Code or Codex spans. |
| `serverInfo.name` / `.version` | Nothing stored. The closest are `server_json.name` / `server_json.version` | Not recorded by any tracer (§2). The registry does not persist `serverInfo`. `server_json` is publisher metadata in the registry format (`namespace/slug`), while `serverInfo` is whatever the running implementation self-reports. Nothing ties the two values together. |

## 4. The declared-but-unimplemented trace-link methods

Exact signatures (`mlflow/store/tracking/mcp_server_registry/abstract_mixin.py:403-418`):

```python
    # --- Trace linking ---

    def link_mcp_server_versions_to_trace(
        self,
        trace_id: str,
        mcp_servers: list[MCPServerVersion],
    ) -> None:
        """Link MCP server versions to a trace via entity associations."""
        raise NotImplementedError(self.__class__.__name__)

    def get_mcp_server_versions_for_trace(
        self,
        trace_id: str,
    ) -> list[MCPServerVersion]:
        """Retrieve MCP server versions linked to a trace."""
        raise NotImplementedError(self.__class__.__name__)
```

- **Stubs:** both the REST and SQL mixins raise `NotImplementedError` (`rest_mixin.py:366-379`; `sqlalchemy_mixin.py:1009-1022`). A grep of `mlflow/`, `tests/` and `docs/` finds no other reference.
- **Introducing commit:** `git log -S link_mcp_server_versions_to_trace` gives `14414ab00`, "Add MCP registry backend and clients (#24380)" by Matthew Prahl, merged 2026-07-15.
  - The PR body says: "This adds the MCP registry backend client APIs. The UI changes will come separately. **Trace linking will come at a later date.**"
  - It links issue #22625 and RFC 0004 (https://github.com/mlflow/mlflow/pull/24380). The PR's comments and review threads contain no trace-linking discussion.
- **Issue #22625 (closed):** "Longer term, this registry could be used to link MCP usage in LLM traces" (https://github.com/mlflow/mlflow/issues/22625).
- **RFC 0004 intent** (mlflow/rfcs `rfcs/0004-mcp-registry/0004-mcp-registry.md` at `7859983`, "Trace linking" section):
  - "The source of truth is a trace-to-MCP-version association created when a runtime or server knows which registered `{workspace, name, version}` handled a request. This supports both trace-to-MCP and MCP-to-traces lookup."
  - "Users can trace MCP usage without the registry … but those traces may only capture raw endpoint details or ad hoc server names. The registry adds a governed canonical `{workspace, name, version}` identity."
  - "Looking up an MCP server in the registry does not by itself create a trace association."
  - "For after-the-fact association, or for runtimes that know the canonical `{workspace, name, version}` only after request handling begins, an explicit API is provided" (example `client.link_mcp_server_versions_to_trace(trace_id=..., mcp_servers=[mcp_server_version])`). The association is scoped to the caller's workspace.
  - The planned UI is an "MCP Servers" tab next to "Prompts" in the trace view.
  - Phase 1 lists "Trace linking: trace-to-MCP-version associations and `link_mcp_server_versions_to_trace()` API". The impact table says "No changes to existing tables".
- **How `MCPAccessEndpoint` relates** (RFC 0004 calls it `MCPAccessBinding`):
  - "Access bindings describe direct access only; this RFC does not define trace-to-binding APIs or any gateway-managed binding entity. A future MLflow MCP Gateway could use its own gateway-managed deployment record … to determine which governed server version a request should resolve to, then record traces against that resolved `{workspace, name, version}`."
  - The future gateway path: "gateway request → gateway-managed entity → governed alias or version → `{workspace, name, version}`."
  - The link methods take `MCPServerVersion` objects, not endpoints.
- **Closed, unmerged prototype:** https://github.com/mlflow/mlflow/pull/24066 by `fege`, against branch `mcp-registry`, opened 2026-06-17 and closed 2026-07-27. It implemented:
  - a new `EntityAssociationType.MCP_SERVER_VERSION = "mcp_server_version"` (`mlflow/entities/entity_type.py`);
  - SQL `link_mcp_server_versions_to_trace`, which validates that the trace exists and that each `(name, version)` exists and is not `deleted`, then inserts deduplicated `SqlEntityAssociation(source_type=TRACE, source_id=trace_id, destination_type=MCP_SERVER_VERSION, destination_id=f"{name}/{version}")` rows;
  - `get_mcp_server_versions_for_trace`, which splits `destination_id` with `rsplit("/", 1)`;
  - REST routes `POST /traces/link-versions` and `GET /traces/mcp-server-versions`;
  - public `mlflow.genai.link_mcp_server_versions_to_trace` / `get_mcp_server_versions_for_trace`.
  - The only PR comments are bot comments (DCO failure). No closing rationale is recorded.
- **Docs:** `docs/docs/genai/mcp-registry/*.mdx` (index, register, versions/aliases, connect) never mentions traces or trace linking (grep for "trace" over those four files: no matches).

## Open points (not verified here)

- The Claude Code transcript JSONL format is not documented in the sources read. Whether it records MCP server identity beyond the tool name was not checked; MLflow's parser reads only `name`/`id`/`input` (§1.1).
- The exact `tool_name` value on the native `claude_code.tool` span for MCP tools is inferred from the `tool_name_safe` description. It was not observed.
- No end-to-end run checked that Codex's `mcp.tools.call` span reaches the MLflow OTLP endpoint and how it is typed there.

## Sources

MLflow (master `7a010ed01`): files cited inline. PRs and issues: https://github.com/mlflow/mlflow/pull/24380, https://github.com/mlflow/mlflow/pull/24066, https://github.com/mlflow/mlflow/issues/22625.

- RFC 0004, MCP Registry (mlflow/rfcs `7859983`): https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md
- Claude Code docs (fetched 2026-10-10):
  - https://code.claude.com/docs/en/monitoring-usage (Traces (beta); Tool result, Tool decision and MCP server connection events; Audit MCP activity)
  - https://code.claude.com/docs/en/hooks (Match MCP tools; PreToolUse input)
  - https://code.claude.com/docs/en/mcp (Plugin MCP tool names; MCP prompts)
  - https://code.claude.com/docs/en/agent-sdk/typescript (`SDKSystemMessage`, `McpServerStatus`, `McpServerProvenance`)
- OpenAI Codex, source at `rust-v0.162.1` (https://github.com/openai/codex/tree/rust-v0.162.1):
  - `codex-rs/core/src/mcp_tool_call.rs`, `codex-rs/core/src/tools/handlers/mcp.rs`
  - `codex-rs/codex-mcp/src/{tools.rs,server.rs,rmcp_client.rs}`
  - `codex-rs/otel/src/{tool_result.rs,events/shared.rs,events/session_telemetry.rs,provider.rs,targets.rs}`
  - `codex-rs/protocol/src/{protocol.rs,tool_name.rs}`
  - Config reference, `otel.*` and `mcp_servers.*`: https://developers.openai.com/codex/config-reference (redirects to https://learn.chatgpt.com/docs/config-file/config-reference)
- MCP specification (modelcontextprotocol/modelcontextprotocol main):
  - `schema/2025-11-25/schema.ts`, `schema/2026-07-28/schema.ts`
  - `docs/specification/2026-07-28/changelog.mdx`, `docs/docs/2026-07-28/learn/versioning.mdx`
  - https://modelcontextprotocol.io/specification/2026-07-28/changelog
- MCP Python SDK: `src/mcp/server/lowlevel/server.py` at `v2.3.0` and `v1.30.0`; `src/mcp/types.py` at `v1.30.0` (https://github.com/modelcontextprotocol/python-sdk)
- Pydantic AI `pydantic_ai_slim/pydantic_ai/mcp.py` at `v1.107.7` (https://github.com/pydantic/pydantic-ai)
- OpenAI Agents SDK at `v0.23.1`: `src/agents/tracing/span_data.py`, `src/agents/mcp/{util.py,server.py,_logging.py}` (https://github.com/openai/openai-agents-python)
- `langchain-mcp-adapters` `langchain_mcp_adapters/tools.py` at `52a4535`; `langchain_core/tools/base.py` at langchain `b060a74`
