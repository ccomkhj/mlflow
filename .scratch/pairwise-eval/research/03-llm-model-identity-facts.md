# Where does a trace record which LLM it called? (ticket 03)

Research for the pairwise-evaluation map. Vocabulary follows `.scratch/pairwise-eval/GLOSSARY.md`: **Model** means the LLM an agent calls (such as `claude-sonnet-5-5`), never a LoggedModel.

Snapshot: MLflow master `7a010ed01` (paths are repo-relative). External sources were fetched on 2026-10-10:

- Claude Code docs: `code.claude.com/docs/en/*.md`
- OpenAI Codex source: `openai/codex@c3d3b142d10f4316b46e35aad7e5317e7e506cb7`
- Codex docs: `learn.chatgpt.com`, which `developers.openai.com/codex/*` now 308-redirects to
- OTel GenAI semconv: `open-telemetry/semantic-conventions-genai@6fd0d763a092db245a54104fc1124031eb3c51d2`
- LiteLLM: `BerriAI/litellm@5282aed09b568c7c4f9c07c39ccd72c8c3135b11`

The file reports facts only. Where a statement is my inference from code rather than something a source states, it is labelled **(inference)**.

## Answer

- **There is one normalized per-span Model field, `mlflow.llm.model`, with `mlflow.llm.provider` beside it.** Both are defined at `mlflow/tracing/constant.py:129-131`. MLflow relies on them for:
  - cost calculation
  - the SQL `spans.model_name` / `model_provider` columns and the daily cost rollups
  - the `span_model_name` metric dimension
  - the gateway budget
  - the UI span panel
  - OTel export

  There is **no trace-level Model field.** `TraceMetadataKey` has `TOKEN_USAGE` and `COST`, but no model key (`constant.py:5-37`).
- **`mlflow.llm.model` holds one value and does not say whether it is the request Model or the response Model.** Each integration chooses:
  - **Response first, falling back to request:** OpenAI autolog, and server-side OTel translation (`gen_ai.response.model` before `gen_ai.request.model`).
  - **Request (configured) Model:** Anthropic, Gemini, LangChain, Bedrock and Gateway.
  - **Not set on the client at all:** LiteLLM, Codex, and the Python Claude Agent SDK path write only a bare `model` attribute. The server fills `mlflow.llm.model` later only if it can find `model` in the inputs or outputs.

  On export MLflow writes `mlflow.llm.model` back out as `gen_ai.request.model` (`mlflow/tracing/export/genai_semconv/translator.py:108-110`).
- **OTel GenAI semconv defines two separate attributes.** `gen_ai.request.model` is "the exact name of the model requested". `gen_ai.response.model` is "the exact name of the model actually used". These conventions have moved out of `opentelemetry.io` into the `semantic-conventions-genai` repo.
- **Claude Code's native OTel `claude_code.llm_request` span records only the request Model**, in `model` and in `gen_ai.request.model` (the same value). It has no `gen_ai.response.model`. MLflow maps it to `mlflow.llm.model` and `mlflow.llm.provider=anthropic`; I verified this by running the translator.
- **MLflow's Claude Code hook tracer (now a TypeScript plugin) sets `mlflow.llm.model`** from each assistant message's `message.model` in the session transcript. It does this for subagent transcripts too, so every LLM span carries its own Model.
- **Codex's native OTLP spans carry the Model only on the `turn` span**, as `model` = the requested model slug. The `codex.sampling` span has no Model. A server-side reroute to a different Model shows up only as a `ModelReroute` event. MLflow has no Codex translator, so these spans get no `mlflow.llm.model` (verified). MLflow's Codex notify-hook tracer stamps every span with the first `turn_context.model` it finds in the rollout file.
- **One trace can involve several Models, and Claude Code's docs say so directly** ("one session can span multiple models"). Sources of a second Model:
  - subagents that pick their own Model
  - fallback chains, which switch the Model for one turn
  - automatic content-based fallback
  - `opusplan`, which uses Opus in plan mode and Sonnet afterwards
  - Haiku-class "background functionality"

  In Claude Code's OTel traces, every `llm_request` span has its own `model`. Subagent spans nest under the parent's `claude_code.tool` span and carry `agent_id` and `query_source(_safe)`. In Codex, `review_model` and `agents.default_subagent_model` run in separate threads.
- **Model identifiers are not canonical across providers.** MLflow's model catalog is keyed per provider. For example, `anthropic.json` has `claude-sonnet-4-5` and `claude-sonnet-4-5-20250929`, while `bedrock.json` uses `anthropic.claude-…-v1:0`. Lookup strips any `provider/` prefix.

## 1. MLflow's own Model attributes

### 1.1 Constants

- **`SpanAttributeKey.MODEL = "mlflow.llm.model"`**, commented "the model name extracted from span inputs/attributes" (`mlflow/tracing/constant.py:129-130`).
- **`SpanAttributeKey.MODEL_PROVIDER = "mlflow.llm.provider"`** (`constant.py:131`).
- **`SpanAttributeKey.LLM_COST = "mlflow.llm.cost"`** (`constant.py:128`).
- **`GenAiSemconvKey.REQUEST_MODEL` and `RESPONSE_MODEL`** mirror the OTel names (`constant.py:392-398`). MLflow has no separate request/response pair of its own.
- **`TraceMetadataKey` has no Model key.** It does have trace-level `TOKEN_USAGE` (`mlflow.trace.tokenUsage`) and `COST` (`mlflow.trace.cost`), which are sums across spans and not broken down by Model (`constant.py:5-37`; aggregation at `mlflow/tracing/utils/__init__.py:325-340`).
- **`SpanMetricDimensionKey` has `SPAN_MODEL_NAME` and `SPAN_MODEL_PROVIDER`** (`constant.py:288-297`).
- **`Span.model_name`** is a property returning `mlflow.llm.model` (`mlflow/entities/span.py:228-230`).

### 1.2 How `mlflow.llm.model` gets filled

**Client side.** `set_span_model_attribute(span, inputs)` copies `inputs["model"]` when it is a string (`mlflow/tracing/utils/__init__.py:924-939`). Autologgers call it or set the key directly (section 3).

**Server side, when spans are stored through `log_spans`.** `SqlAlchemyStore` calls `translate_span_when_storing` (`mlflow/store/tracking/sqlalchemy_store.py:5743`). That function:

1. Fills `mlflow.llm.model` and `mlflow.llm.provider` only if they are absent (`mlflow/tracing/otel/translation/__init__.py:118-125`).
2. Finds the model name (`__init__.py:295-329`) by trying each translator's `MODEL_NAME_KEYS` in `_TRANSLATORS` order (`__init__.py:37-50`), then `mlflow.spanInputs["model"]`, then `mlflow.spanOutputs["model"]`.
3. Computes `mlflow.llm.cost` when usage and a Model are both present (`__init__.py:127-146`).

Translator key lists:

| Translator | Model keys, in priority order | Provider key |
|---|---|---|
| GenAI semconv | `gen_ai.response.model`, then `gen_ai.request.model` (`genai_semconv.py:52-55`) | `gen_ai.provider.name` |
| OpenInference | `llm.response.model_name`, `llm.model_name`, `llm.request.model_name`, `embedding.model_name` (`open_inference.py:50-57`) | `llm.provider` |
| Laminar | same as GenAI semconv (`laminar.py:29-30`) | `gen_ai.system` |
| Traceloop | none of its own | `gen_ai.system` (`traceloop.py:53`) |
| Claude Code | **none** (`claude_code.py:39-49`) | none |

No translator reads a bare `model` attribute.

**Raw attributes are kept.** `sanitize_attributes` only re-decodes values (`__init__.py:483-508`). So after normalization, a span still carries the original `gen_ai.request.model` / `gen_ai.response.model` next to the single `mlflow.llm.model`.

**Empirical check.** I ran `translate_span_when_storing` from this worktree on synthetic attribute sets:

| Input attributes | Resulting `mlflow.llm.model` / `mlflow.llm.provider` |
|---|---|
| Claude Code `llm_request` (`model`, `gen_ai.request.model`, `gen_ai.system`) | `claude-sonnet-5-5` / `anthropic` |
| `gen_ai.request.model=gpt-4o`, `gen_ai.response.model=gpt-4o-2024-08-06` | `gpt-4o-2024-08-06`, the response Model, wins |
| Codex-style bare `model` only | nothing |
| LiteLLM-style bare `model` plus `spanOutputs.model` | the outputs value |

### 1.3 What relies on `mlflow.llm.model`

- **Cost.**
  - `calculate_span_cost` reads `MODEL`, `CHAT_USAGE` and `MODEL_PROVIDER` (`tracing/utils/__init__.py:343-356`).
  - `calculate_cost_by_model_and_token_usage` skips `gateway:/` and `endpoints:/` names (`:361-370`). It uses LiteLLM's `cost_per_token` if LiteLLM is importable, and MLflow's bundled catalog otherwise (`:389-392`). It tries the provider first and then falls back to no provider (`:414-440`).
  - Cost is computed client-side only for Databricks and SageMaker tracking URIs (`:943-954`, called from `entities/span.py:1248-1249`). For every other backend it is computed server-side at store time (section 1.2).
- **SQL analytics.**
  - `SqlSpan.model_name` and `model_provider` are denormalized columns "used by span cost analytics queries" (`mlflow/store/tracking/dbmodels/models.py:2303-2310`). They are populated from `mlflow.llm.model` / `mlflow.llm.provider` at write time (`sqlalchemy_store.py:5782-5787`), truncated to 500 chars.
  - `SqlSpanCostDailyRollup` groups by `model_name` and `model_provider` (`models.py:1021-1052`).
  - The `span_model_name` dimension maps to those columns (`mlflow/store/tracking/utils/sql_trace_rollups.py:790-794`).
  - Migration `75868b020152` backfills them from the same attribute keys (`mlflow/store/db_migrations/versions/75868b020152_add_sql_trace_analytics_schema.py:45-46`).
- **Trace search.** `span.model_name` is not a search key; the supported span keys are `name`, `type` and `status` (`mlflow/utils/search_utils.py:1784`). `span.attributes.<key>` is accepted (`search_utils.py:1931-1945`). It is implemented as a LIKE or RLIKE over the span's JSON `content` (`sqlalchemy_store.py:10642-10670`).
- **AI Gateway budget.** It reads `MODEL`, `MODEL_PROVIDER`, `CHAT_USAGE` and `LLM_COST` from the gateway trace's child spans (`mlflow/gateway/tracing_utils.py:103-108`, `mlflow/gateway/budget.py:101-119`).
- **UI.** The span detail panel reads `mlflow.llm.model` (`mlflow/server/js/src/shared/web-shared/model-trace-explorer/constants.ts:25`, `ModelTraceExplorer.utils.tsx:569`). The docs say it is "Shown in the span detail panel as the model used for the LLM call" (`docs/docs/genai/tracing/opentelemetry/attribute-mapping.mdx:36-37`).
- **OTel export.** It is written as `gen_ai.request.model`, and `mlflow.llm.provider` as `gen_ai.provider.name` (`mlflow/tracing/export/genai_semconv/translator.py:108-114`).
- **Docs on cost failures.** Cost shows $0.00 when "the model name recorded on the span has no matching entry in LiteLLM's price table" (`docs/docs/genai/tracing/token-usage-cost/index.mdx:225`). DSPy cost is unsupported "because the model name may not be available in the span attributes" (`docs/docs/genai/tracing/integrations/listing/dspy.mdx:171`).

### 1.4 Model catalog

- **Format.** `mlflow/utils/model_catalog/<provider>.json` files are bundled. Each has `schema_version` and `models`, keyed by the provider's own model ID. A remote copy is fetched first from `MLFLOW_MODEL_CATALOG_URI` (default: the GitHub release `model-catalog/latest`) with a TTL cache (`mlflow/utils/providers.py:256-334`, `mlflow/environment_variables.py:1880-1891`).
- **Lookup.** `_lookup_model_info` strips everything up to the first `/` (`bare_model = model.split("/", 1)[-1]`). It then looks in the given provider's file, or scans the bundled files when no provider is given (`providers.py:336-352`). `cost_per_token` prices from that entry (`providers.py:355-410`).
- **Keys are provider-specific.**
  - `anthropic.json` has `claude-sonnet-4-5`, `claude-sonnet-4-5-20250929`, `claude-sonnet-5` and `claude-sonnet-5-5`.
  - `bedrock.json` uses IDs like `anthropic.claude-3-5-sonnet-20241022-v2:0`.

  I read these keys from the bundled files.
- **The TypeScript Claude Code plugin mirrors this lookup** for its own cost estimate. It loads catalog rates and retries a lookup with any trailing `-YYYYMMDD` removed (`libs/typescript/integrations/claude-code/src/modelCatalog.ts:1-29`, `pricing.ts:45-54`).

## 2. OTel GenAI semantic conventions

- **The GenAI conventions have moved.** `opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/` and the `registry/attributes/gen-ai/` page say the attributes "have moved to the OpenTelemetry GenAI semantic conventions repository": https://github.com/open-telemetry/semantic-conventions-genai. In the old registry the attributes are marked Deprecated with "Moved" (https://github.com/open-telemetry/semantic-conventions/blob/main/docs/registry/attributes/gen-ai.md). MLflow code still links the old URLs (`constant.py:393`, `genai_semconv.py:53`).
- **The inference span is `gen_ai.client.inference`** (status: Development). Its span name SHOULD be `{gen_ai.operation.name} {gen_ai.request.model}` (https://github.com/open-telemetry/semantic-conventions-genai/blob/6fd0d763a092db245a54104fc1124031eb3c51d2/docs/gen-ai/client-inference.md).
  - `gen_ai.request.model` is **Conditionally Required, "If available"**: "If the model is supplied by a vendor, then the value must be the exact name of the model requested."
  - `gen_ai.response.model` is **Recommended**: "The name of the GenAI model that provided the response. If the model is supplied by a vendor, then the value must be the exact name of the model actually used."
  - Example values are `gpt-4` for the request and `gpt-4-0613` for the response.
  - The same definitions appear in `docs/gen-ai/gen-ai-spans.md` and `docs/registry/attributes/gen-ai.md` (both Development).
- **Provider.** `gen_ai.provider.name` is Required on the inference span. The registry says `gen_ai.request.model`, `gen_ai.response.model` and `server.address` "may help identify the actual system in use" when one API shape fronts several providers (`semantic-conventions/docs/registry/attributes/gen-ai.md`, the `gen_ai.provider.name` note).
- **Scope.** Each inference span describes one call. The conventions have no trace-level Model attribute.

## 3. Integration sample: which attribute, request or response Model

| Integration | Attribute written at capture | Request or response Model | `mlflow.llm.provider` | Source |
|---|---|---|---|---|
| OpenAI autolog (chat, responses, completions, embeddings) | `mlflow.llm.model` | **Response** `output.model` first, else request `inputs.model`. The code comments that the Azure request `model` may be a deployment name. | not set | `mlflow/openai/utils/chat_schema.py:26-46`, `:132-190`; `mlflow/openai/autolog.py:400`. OpenAI documents the response `model` as "The model used for the chat completion" (https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create). |
| OpenAI Agents SDK tracer | Generation span: `mlflow.llm.model = span_data.model`. Response span: all `Response` fields as attributes, including a bare `model`. | SDK-reported / response | not set | `mlflow/openai/_agent_tracer.py:235-241`, `:284-289` |
| Anthropic autolog | `mlflow.llm.model` from `inputs.model` | **Request** | `"anthropic"`, hard-coded | `mlflow/anthropic/autolog.py:133-140`. The Messages API response also carries `model` (https://platform.claude.com/docs/en/api/messages/create, "Returns"), but MLflow does not read it. |
| Gemini autolog | `mlflow.llm.model` from `inputs.model` or `Chat._model` | **Request** | `"gemini"` | `mlflow/gemini/autolog.py:138-140`, `:187` |
| LangChain | `mlflow.llm.model` from `metadata.ls_model_name`, then `invocation_params.model`, then `model_name` (set at LLM start) | **Request** (configured) | `ls_provider`, else `_type` with `chat-` / `-chat` stripped | `mlflow/langchain/langchain_tracer.py:398-417`. `on_llm_end` adds usage but no Model (`:499-504`). |
| LiteLLM | Tracing is LiteLLM's own `MlflowLogger` (MLflow only adds the `"mlflow"` callback, `mlflow/litellm/__init__.py:45-47`). It writes a bare `model = kwargs["model"]` and `custom_llm_provider` (the latter only without `standard_logging_object`). Span outputs are the LiteLLM response object. | Request at capture. Server-side fallback reads `spanOutputs.model`. | not set | litellm `litellm/integrations/mlflow.py:44-49`, `:162-218`; `tests/litellm/test_litellm_autolog.py:57` asserts `attributes["model"]` |
| Bedrock | `mlflow.llm.model = kwargs["modelId"]` | **Request** | the text before the first `.` in `modelId` | `mlflow/bedrock/_autolog.py:194-201`, `:233-239`. **(inference)** A region-prefixed ID such as `us.amazon.nova-lite-v1:0` (used in `tests/bedrock/test_bedrock_autolog.py:216`) would give provider `us`. |
| MLflow AI Gateway provider spans | `mlflow.llm.model` = the endpoint's configured model name | Configured | provider name | `mlflow/gateway/providers/base.py:361-369` |
| Claude Code hook tracer (TS plugin) | `model` and `mlflow.llm.model` = assistant `message.model` from the transcript | The Model on the recorded assistant message | not set | `libs/typescript/integrations/claude-code/src/tracing.ts:186-211`, `:428-440` |
| Claude Agent SDK, TypeScript (`createTracedQuery`) | `model` and `mlflow.llm.model` = `msg.message.model`, else the `system/init` model | Assistant message, else session init | not set | `libs/typescript/integrations/claude-code/src/liveTracing.ts:135-170` |
| Claude Agent SDK, Python (`mlflow.anthropic.autolog` → `process_sdk_messages`) | bare `model` and `inputs.model` = `AssistantMessage.model` (only on text-only assistant messages) | Assistant message | not set; server-side fallback reads `inputs.model` | `mlflow/claude_code/tracing.py:284-301`; `mlflow/anthropic/autolog.py:62-64` |
| Claude Code native OTel | `model`, `gen_ai.request.model` (same value), `gen_ai.system=anthropic` | **Request only** | from `gen_ai.system` | Section 4.1 |
| Codex native OTel | `turn` span: `model` = requested slug | **Request** | none | Section 5.1 |
| Codex notify-hook tracer (TS) | bare `model` and `inputs.model` | The first `turn_context.model` in the rollout | not set | Section 5.3 |

**Consistency.** Bare `model` is common, but it is not a key MLflow normalizes from. Only `mlflow.llm.model`, the OTel and OpenInference keys, and `inputs.model` / `outputs.model` are read (section 1.2).

## 4. Claude Code

### 4.1 Native OTel traces (first-party docs)

Source: https://code.claude.com/docs/en/monitoring-usage

**Enabling.** Tracing is beta and needs `CLAUDE_CODE_ENABLE_TELEMETRY=1`, `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` and `OTEL_TRACES_EXPORTER` (`#traces-beta`).

**Span hierarchy** (`#span-hierarchy`):

- Each user prompt starts a `claude_code.interaction` root span.
- `claude_code.llm_request` and `claude_code.tool` spans are its children.
- "When the Agent tool, or legacy Task tool, spawns a subagent, the subagent's API and tool spans nest under the parent's `claude_code.tool` span."

**`claude_code.llm_request` attributes** (`#span-attributes`):

| Attribute | Meaning |
|---|---|
| `model` | "Model identifier" |
| `gen_ai.system` | "Always `anthropic`" |
| `gen_ai.request.model` | "Same value as `model`" |
| `query_source` | Subsystem that issued the request, such as `repl_main_thread` or a subagent name. Only with detailed beta tracing. |
| `query_source_safe` | Bounded form, for example `agent.builtin.general-purpose`. Always emitted, v2.1.268+. |
| `agent_id` | "Absent on the main session" |
| `parent_agent_id` | The agent that spawned this one |
| `effort` | Effort level |
| `speed` | `fast` or `normal` |
| `llm_request.context` | `interaction`, `tool` or `standalone`, depending on the parent span |
| `gen_ai.response.id` | Same as `request_id` |
| token counts | Usage for the request |

- **No response Model.** The documented table has no `gen_ai.response.model` attribute.
- **No `gen_ai.usage.*`.** "Claude Code doesn't set `gen_ai.usage.*` attributes" (`#map-input-tokens-to-opentelemetry-genai-semantic-conventions`).
- **Events and metrics also carry `model`**, but they travel as OTLP logs and metrics, not spans. The `claude_code.api_request` event has `model` ("Model used") and `query_source` (`#api-request-event`). The cost and token counters have `model`, plus `query_source` in {`main`, `subagent`, `auxiliary`} (`#cost-counter`, `#token-counter`). `claude_code.subagent_completed` has `model` ("the model the subagent was resolved to run"), `final_model` and `model_swapped` (`#subagent-completed-event`).
- **MLflow ingests only spans.** Its OTLP receiver is mounted only at `/v1/traces` (`mlflow/server/otel_api.py:104`). **(inference)** So Claude Code's log events and metrics, including `final_model` and `model_swapped`, do not reach MLflow.

### 4.2 MLflow ingestion of Claude Code OTel spans

- **No model keys of its own.** `ClaudeCodeTranslator` maps `span.type` to span types and reads bare token keys. It defines no `MODEL_NAME_KEYS` (`mlflow/tracing/otel/translation/claude_code.py:39-49`).
- **The Model comes from other translators.** `mlflow.llm.model` is still derived from `gen_ai.request.model` by the GenAI-family translators. `mlflow.llm.provider=anthropic` comes from `gen_ai.system` via `TraceloopTranslator` (section 1.2, empirical check). The repo's test also feeds `gen_ai.request.model` on an `llm_request` span (`tests/tracing/otel/test_span_translation.py:1104-1121`).
- **Recognized as an OTLP client.** `claude-code` is on the OTLP `service.name` allowlist (`otel_api.py:59-73`).

### 4.3 MLflow's Claude Code hook tracer and Agent SDK paths

- **The Python hook runtime is retired.** `mlflow/claude_code/hooks.py:1-16` is now a no-op shim telling users "MLflow Claude tracing has moved to the marketplace plugin runtime". `mlflow autolog claude` installs the npm plugin `@mlflow/claude-code` from `libs/typescript/integrations/claude-code` (`mlflow/claude_code/plugin.py:22-26`, `.claude-plugin/marketplace.json`). The plugin runs on the `Stop` hook (`libs/typescript/integrations/claude-code/hooks/hooks.json`).
- **Per-LLM-span Model.** `getLogicalAssistantMessage` merges streamed entries with one `message.id` and takes the first non-`unknown` `message.model` (`src/tracing.ts:186-211`). Each `llm` span gets `model` and `mlflow.llm.model` from that, plus `mlflow.message.format=anthropic` (`:428-440`). The plugin sets no provider.
- **Subagents get their own Model.** Subagent spans come from the subagent's own transcript at `<session>/subagents/agent-<agentId>.jsonl` (`src/tracing.ts:260-276`, `:335-371`). Each subagent LLM span carries that transcript's `message.model` under an AGENT wrapper span named `subagent_<subagent_type>` (`:282-300`). Claude Code's hook docs confirm the location: the subagent transcript is "stored in a nested `subagents/` folder" (https://code.claude.com/docs/en/hooks, SubagentStop input).
- **The transcript format is undocumented.** The hooks reference describes `transcript_path` only as "Path to conversation JSON" (https://code.claude.com/docs/en/hooks, common input fields). The `message.model` layout MLflow reads is not documented there.
- **The plugin computes cost itself.** It computes cost client-side on every backend. Its comment explains that the OSS server's store-time computation "never sees these spans because the TS SDK exports trace data as an artifact blob" (`src/tracing.ts:444-451`). The TypeScript SDK's default exporter does call `createTrace` then `uploadTraceData` (`libs/typescript/core/src/exporters/mlflow.ts:218-223`). An opt-in path, `MLFLOW_ENABLE_ASYNC_TRACE_LOGGING`, posts OTLP to `/v1/traces` instead (`libs/typescript/core/src/core/provider.ts:56-66`, `core/utils/env.ts:44-51`). **(inference)** On the default path, the server-side `mlflow.llm.model` backfill in section 1.2 does not run for TypeScript-tracer spans.

## 5. Codex

### 5.1 Native OTel (first-party docs and source)

**Docs.** Codex's `[otel]` config exports log events, with `otel.trace_exporter` and `otel.metrics_exporter` as separate keys (https://learn.chatgpt.com/docs/config-file/config-advanced, "Observability and telemetry"; https://learn.chatgpt.com/docs/config-file/config-reference).

- "Event metadata includes service name, CLI version, env tag, conversation id, model, …".
- `codex.conversation_starts` carries "model, reasoning settings, …".
- Metrics carry a default `model` tag ("name of the model used").
- The docs do not list trace span attributes.

**Source** (`openai/codex@c3d3b142`):

- **The `turn` span carries the Model.** Every turn opens a span with `model = %turn_context.model_info().slug`, plus `thread.id`, `turn.id`, reasoning effort and token usage fields (`codex-rs/core/src/tasks/mod.rs:354-368`). Its `otel.name` is the task's span name, such as `session_task.turn`, `session_task.review` or `session_task.compact` (`core/src/tasks/regular.rs:37`, `review.rs:51`, `compact.rs:26`).
- **The per-request span does not.** `codex.sampling` (with `stream_request` under it) has `codex.turn.phase`, `conversation.id` and `turn.id`, but no Model (`core/src/session/turn.rs:2550-2576`).
- **Events carry `model` and `slug`.** Log events and trace-safe events add both fields to every record (`codex-rs/otel/src/events/shared.rs:14-50`). `model` is the collaboration mode's selected model and `slug` is the resolved model-info slug (`core/src/session/step_settings.rs:166-171`). Trace export keeps all spans except `h2`, `tokio` and `runtime` ones, plus events from the `codex_otel.trace_safe` target (`codex-rs/otel/src/provider.rs:341-349`, `targets.rs`).
- **Request vs response Model.** When the server reports a different Model (`ResponseEvent::ServerModel`), Codex compares it with the requested slug. On a mismatch it emits `EventMsg::ModelReroute { from_model, to_model, reason }` (`core/src/session/turn.rs:2894-2906`, `core/src/session/mod.rs:4054-4082`, `codex-rs/protocol/src/protocol.rs:2230-2234`). **(inference)** The `turn` span's `model` field is therefore the requested Model, not the served one.

### 5.2 MLflow ingestion of Codex OTLP

- **Accepted but not translated.** `codex_cli_rs` and `codex_vscode` are on the `service.name` allowlist (`mlflow/server/otel_api.py:62-73`), but there is no Codex translator (see `_TRANSLATORS`, `translation/__init__.py:37-50`).
- **The bare `model` on the `turn` span is not normalized** (empirical check, section 1.2).
- **MLflow's docs describe these traces as internals.** They capture "internal execution details (websocket connections, request handling, tool scheduling) rather than clean conversation flows" (`docs/docs/genai/tracing/integrations/listing/codex.mdx`, "OTLP Tracing").

### 5.3 MLflow's Codex notify-hook tracer (`@mlflow/codex`)

- **Where the Model comes from.** `getModel(records)` scans the **whole** rollout file and returns the first `model` string on a `session_meta` or `turn_context` record (`libs/typescript/integrations/codex/src/transcript.ts:151-163`). It is called with all records, not only the last turn's (`src/tracing.ts:75-82`).
- **The rollout format.** In Codex's protocol, `TurnContextItem` has `model: String` (`codex-rs/protocol/src/protocol.rs:3417-3440`). MLflow's `SessionMetaPayload` type has `model_provider` but no `model` (`libs/typescript/integrations/codex/src/types.ts:42-50`).
- **What gets stamped.** The same value goes on the root span (`attributes: { model, 'service.name': 'codex' }`) and on every `llm_call` span (`attributes: { model }`, `inputs: { model, messages }`) (`src/tracing.ts:98-104`, `:120-130`, `:258-267`). The tracer does not write `mlflow.llm.model` or a provider.
- **(inference)** If the Model changes mid-session, later turns are still stamped with the earliest `turn_context` Model. A review or subagent thread's Model is not captured.

## 6. Can one trace involve several Models?

Yes. Every source below is first-party.

**Claude Code: the docs say so.** "Per-model breakdowns of commits can only be approximated … since one session can span multiple models. Filter … to rows where `query_source` is `"main"` so auxiliary and subagent requests don't attribute the session's commits to a model that didn't make them" (https://code.claude.com/docs/en/monitoring-usage#alerting-and-segmentation).

**Claude Code: where extra Models come from** (https://code.claude.com/docs/en/sub-agents, https://code.claude.com/docs/en/model-config, https://code.claude.com/docs/en/env-vars, https://code.claude.com/docs/en/costs, https://code.claude.com/docs/en/hooks):

- **Subagents.** A subagent's Model is resolved in this order: the per-invocation `model` parameter, then the definition's `model` frontmatter, then `CLAUDE_CODE_SUBAGENT_MODEL`, then the main conversation's Model (`sub-agents#choose-a-model`). The built-in `claude-code-guide` helper uses Haiku and `statusline-setup` uses Sonnet (`sub-agents#built-in-subagents`, the "Other" tab). The Agent tool's PostToolUse `tool_response` reports `resolvedModel` and, after a mid-run swap, `modelsUsed` such as `["claude-sonnet-4-5", "claude-haiku-4-5"]` (`hooks`, Agent tool).
- **Fallback model chains.** With `--fallback-model` or `fallbackModel`, Claude Code switches to a fallback when the primary is overloaded or unavailable. "The switch lasts for the current turn only." Chains also apply to subagents (`model-config#fallback-model-chains`).
- **Automatic model fallback.** Requests flagged by safety classifiers re-run on a fallback Model, for example Sonnet 5.5 → Sonnet 5 for cybersecurity. "After a fallback, the session continues on the fallback model" (`model-config#automatic-model-fallback`).
- **`opusplan`.** It uses `opus` in plan mode, "then switches to `sonnet` for execution" (`model-config#opusplan-model-setting`).
- **Background functionality.** `ANTHROPIC_DEFAULT_HAIKU_MODEL` is "also used for background functionality". `ANTHROPIC_SMALL_FAST_MODEL` (deprecated) names the "Haiku-class model for background tasks" (`env-vars`, `model-config#environment-variables`). Background work includes conversation summarization and a "background small/fast-model request that generates a session title" (`costs#background-token-usage`; `env-vars`, `CLAUDE_CODE_DISABLE_TERMINAL_TITLE`).
- **Model-switch hooks.** `PostModelSwitch` fires on user switches, automatic fallback, `opusplan` plan-mode transitions and resume, but not for one-turn fallback-chain substitutions (`hooks#postmodelswitch`).

**Claude Code: how it shows up in spans.**

- **Native OTel.** Each `claude_code.llm_request` has its own `model` / `gen_ai.request.model`. Subagent requests nest under the parent's `claude_code.tool` span and carry `agent_id`, `parent_agent_id` and `query_source(_safe)` (`monitoring-usage#span-hierarchy`, `#span-attributes`). **(inference)** `llm_request.context = standalone` means a request without an interaction or tool parent; such a request would not sit inside the interaction's trace.
- **Open question.** The docs do not say whether a fallback-chain switch inside one `llm_request` (which records `attempt` and retries) changes that span's `model`.
- **MLflow hook tracer.** Each `llm` span carries its own transcript `message.model`, so main and subagent spans can differ within one trace (section 4.3). Requests that never reach the transcript are not traced by this path. **(inference)** Which background requests are absent is not documented.

**Codex.**

- **Review uses its own thread.** `/review` runs a sub-agent thread on `review_model`, "defaults to the current session model" (config reference; `core/src/tasks/review.rs:123-127`).
- **Spawned agents.** `agents.default_subagent_model` is the "Default model for spawned agents. An explicit spawn model takes precedence" (config reference).
- **Spawn spans can join the parent trace.** A spawned thread starts under a `thread_spawn` span parented from the W3C trace context it was given (`core/src/session/mod.rs:540-543`). Each thread's `turn` spans carry their own `model` (section 5.1).
- **Server reroutes** produce a `ModelReroute` event, not a span attribute (section 5.1).

**MLflow side.** Several Models in one trace appear only as different `mlflow.llm.model` values on different spans. Nothing aggregates or records them at trace level (sections 1.1 and 1.3).

## 7. Not verified / open

- **Python `log_spans` export.** Whether the Python SDK's OSS export always goes through `log_spans`, and so through the server-side backfill, was not traced beyond the docstring at `tracing/utils/__init__.py:943-954`.
- **Codex trace levels.** Whether TRACE-level Codex spans such as `codex.sampling` are actually exported depends on Codex's subscriber level configuration, which I did not trace.
- **Claude Code transcript.** The transcript JSONL schema, including which background requests it records, is not documented by Anthropic.
- **Gemini response Model.** I did not check whether Gemini responses carry a response-side Model field. MLflow uses the request Model only.
