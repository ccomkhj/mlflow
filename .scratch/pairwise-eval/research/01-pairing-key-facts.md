# Ticket 01: What can identify a task across traces today?

Research date: 2026-10-10. MLflow source: master `7a010ed01`. Paths are repo-relative unless they are URLs. Vocabulary follows `.scratch/pairwise-eval/GLOSSARY.md`.

Some claims were also checked empirically. Those checks used Claude Code 2.1.296, codex-cli 0.162.0, and a throwaway `mlflow server` 3.16.1 from PyPI (sqlite backend). Each check is marked **[verified]**.

## Answer

- **`mlflow.genai.evaluate` stamps nothing on a trace that identifies the dataset record.** With a `predict_fn`, the only per-row tag is `mlflow.eval.requestId`, and its value is a fresh `uuid4()` on every call. A record's `dataset_record_id` is in the DataFrame but is never read. The dataset id and digest are logged on the run, not on traces. `mlflow.sourceRun` identifies the evaluation run, not the record.
- **One mechanism does carry a user-chosen record identifier onto the trace: the row's `tags` column.** `evaluate` copies `tags` onto the produced trace as trace tags, for every input format (EvaluationDataset record tags, list of dicts, DataFrame). Trace tags can be filtered with `tag.<key> = '<value>'`.
- **Exception, static rows only (no `predict_fn`, no `trace` column):** in this case `mlflow.eval.requestId` equals the row's `request_id` column if one is given. Otherwise it equals `sha256(str(inputs))`. This tag is filterable.
- **Claude Code and Codex traces sent over OTLP:** a user can attach an arbitrary key without changing the agent by setting `OTEL_RESOURCE_ATTRIBUTES`. On ingestion, MLflow saves every OTLP resource attribute (except `telemetry.sdk.*` and `mlflow.*`) as a **trace tag**, which `search_traces` can filter. **[verified]** for both agents.
  - Codex also accepts `otel.span_attributes` (`-c` override), which lands on every span.
  - Both agents can send an `x-mlflow-run-id` header, which links each ingested trace to an MLflow run (`run_id = ...` filter).
- **The hook-based `mlflow autolog claude` plugin has no way to add a custom tag or metadata entry.** It writes a fixed set of metadata keys. `update_current_trace` needs an in-process active span, so it cannot reach either agent. After the fact, only tags can be added (`set_trace_tag`, `mlflow traces set-tag`). Metadata is immutable.
- **No existing field identifies a "task".**
  - `mlflow.trace.session` identifies a conversation. Claude Code can choose its value with `--session-id <uuid>`, but refuses to reuse one ("Session ID … is already in use") **[verified]**, so two runs cannot share it.
  - The request preview is not a valid `search_traces` key **[verified]**.
  - `mlflow.traceInputs` metadata is filterable but truncated to 250 characters.
  - Dataset records do have an input hash (`sha256(json.dumps(inputs, sort_keys=True))`), but it lives only in the `evaluation_dataset_records` table and never reaches a trace.

## 1. What `mlflow.genai.evaluate` stamps on each trace

### Code path

1. `evaluate` → `_run_harness` (`mlflow/genai/evaluation/base.py:303`). For a managed EvaluationDataset, the harness gets `df = data.to_df()` (`base.py:407`). A list or DataFrame becomes `mlflow.data.from_pandas(df=data, name=..., digest=precomputed_digest)` (`base.py:412`).
2. The dataset entity (name and digest) is logged as a **run input**: `_log_dataset_input(mlflow_dataset, run_id, model_id)` (`base.py:424`, body at `base.py:446-458`). This writes nothing to traces.
3. `harness.run` builds one `EvalItem` per row via `EvalItem.from_dataset_row` (`mlflow/genai/evaluation/harness.py:678`).
4. `EvalItem.from_dataset_row` (`mlflow/genai/evaluation/entities.py:125-174`):
   - `request_id = row.get("request_id")` (`entities.py:154`). If that is missing, it falls back to `hashlib.sha256(str(<first non-None of inputs, outputs, trace, expectations>)).hexdigest()` (`entities.py:155-164`). This hashes Python `str()` of the dict, so it depends on key order. It is not the store's JSON input hash.
   - Reads `tags` (`entities.py:146`) and `source` (`entities.py:149`).
   - **Never reads `dataset_record_id`.** `EvaluationDataset.to_df()` emits a `dataset_record_id` column (`mlflow/entities/evaluation_dataset.py:525`), but nothing in `mlflow/genai/evaluation/` or `mlflow/genai/utils/` refers to it (grep for `dataset_record_id|record_id` over `mlflow/genai/` outside `datasets/` turns up only labeling and review-queue code).
5. `_run_predict` (`harness.py:809-866`) takes one of three branches:
   - **With `predict_fn`:** `eval_request_id = str(uuid.uuid4())` (`harness.py:824`), set as the prediction context's `request_id` (`harness.py:826`). The span processor turns this into the trace tag `mlflow.eval.requestId` (`mlflow/tracing/processor/base_mlflow.py:328-330`; key at `mlflow/tracing/constant.py:42`). The value is random per call, so the same record evaluated twice gets two unrelated values.
   - **With a `trace` column:** the trace is cloned into the experiment (`harness.py:844`) or linked to the run (`harness.py:862`). The original trace's tags and metadata are carried along.
   - **Static rows (inputs and outputs only):** `create_minimal_trace` (`harness.py:866`) runs under `Context(request_id=eval_item.request_id, is_evaluate=True)` (`mlflow/genai/utils/trace_utils.py:1038`). In this one case, `mlflow.eval.requestId` is the user's `request_id` column or the `str(inputs)` hash. If the record's `source.source_data` has a `session_id`, it is copied to `mlflow.trace.session` (`trace_utils.py:1021-1048`).
6. `_run_score` copies the row's `tags` onto the trace: `for key in tags.keys() - IMMUTABLE_TAGS: mlflow.set_trace_tag(...)` (`harness.py:889-896`). This runs even when there are no scorers. The regression test `test_evaluate_with_empty_scorers_logs_dataset_tags` asserts that a row tag such as `{"case_id": "case-1"}` appears on the trace (`tests/genai/evaluate/test_evaluation.py:370-391`).
7. Run-level stamping:
   - `mlflow.sourceRun` metadata is set to the active run (`base_mlflow.py:306-307`; key at `constant.py:8`).
   - Traces are linked to the run (`batch_link_traces_to_run`, `harness.py:744`).
   - Under `@mlflow.test`, every trace gets the tags `mlflow.test.name` and `mlflow.test.case_id`. Here `case_id` is the pytest parametrize id, not a dataset record (`harness.py:635-659`; `mlflow/pytest/plugin.py:34-38`; `mlflow/pytest/session.py:18-20`).
8. Other paths:
   - Scope-level injection: `mlflow.tracing.context(metadata=..., tags=...)` adds tags and metadata to every trace created in scope (`mlflow/tracing/context.py:39-136`; applied at `base_mlflow.py:318-320`, `335-337`). However, the predict pool submits jobs with plain `self._pool.submit(...)` (`harness.py:326`), unlike the scorer pool, which uses `contextvars.copy_context().run` (`harness.py:986`). So a `context()` opened around `evaluate()` is not carried into predict threads. This is inferred from the code and was not run.
   - Calling it *inside* `predict_fn` works, but `predict_fn` receives only `inputs` (`harness.py:831`).

### Which candidates are on the trace, and can `search_traces` filter them?

| Candidate | On the trace? | Filterable? |
|---|---|---|
| Dataset record id (`dataset_record_id`) | No (dropped, see step 4) | n/a |
| Dataset id / name / digest | No; logged as a run input only (`base.py:424`) | n/a |
| `mlflow.sourceRun` (metadata) | Yes; eval run id | Yes: `run_id = '<id>'` is rewritten to `metadata.mlflow.sourceRun` (`mlflow/utils/search_utils.py:1807`). The SQL store ORs it with run-link associations (`mlflow/store/tracking/sqlalchemy_store.py:4215-4237`, `10566-10574`) |
| `mlflow.eval.requestId` (tag) | Yes; random uuid4 with `predict_fn`, row `request_id` or `str(inputs)` hash for static rows | Yes: `tag.\`mlflow.eval.requestId\` = '...'` (used in `tests/genai/evaluate/test_evaluation.py:1227-1230`) |
| Row `tags` (user-chosen) | Yes; copied as trace tags (`harness.py:889-896`) | Yes: `tag.<key> = / != / LIKE / ILIKE / RLIKE / IS [NOT] NULL` (`search_utils.py:1740`). **No `IN`** for tags **[verified]**: `Invalid comparator 'IN'` |
| Input hash (store) | No; only `evaluation_dataset_records.input_hash` (`sqlalchemy_store.py:8201-8202`; unique per dataset at `dbmodels/models.py:2091`). `DatasetRecord` does not expose it (`mlflow/entities/dataset_record.py:30-42`) | n/a |
| `mlflow.traceInputs` (metadata) | Yes; root-span inputs truncated to 250 chars (`base_mlflow.py:354-363`, `373-380`; limit at `constant.py:190`) | Yes, as generic metadata: `metadata.\`mlflow.traceInputs\` = / LIKE ...` |
| Request preview | Yes (`trace_info.request_preview` column) | **No.** `request_preview` is not in `VALID_SEARCH_ATTRIBUTE_KEYS` (`search_utils.py:1699-1715`), so the parser rejects it (`search_utils.py:418-421`). **[verified]**: HTTP 400 `Invalid attribute key 'request_preview'` |
| `client_request_id` | Not set by `evaluate` | Yes, including `IN` (`search_utils.py:1708`, `1792-1798`) **[verified]** |

Search grammar summary: identifiers are `tag`, `metadata`/`request_metadata`, `attribute`/`trace`, `span`, `feedback`, `expectation`, and `issue` (`search_utils.py:1756-1781`).
- Any tag or metadata key can be filtered (`sqlalchemy_store.py:10577-10633`).
- `metadata.mlflow.trace.session` maps to the indexed `session_id` column (`sqlalchemy_store.py:10544-10557`).
- `span.attributes.<k>` does a LIKE over the span content JSON (`sqlalchemy_store.py:10642-10671`). `=` does not match; `LIKE '%v%'` does **[verified]**.

## 2. Attaching an arbitrary tag or metadata to Claude Code and Codex traces without changing the agent

### Claude Code, hook-based (`mlflow autolog claude`)

- `mlflow autolog claude` installs the npm plugin `@mlflow/claude-code` from the `mlflow-plugins` marketplace (`mlflow/claude_code/plugin.py:22-55`; `.claude-plugin/marketplace.json`). The Python Stop hook is a no-op shim (`mlflow/claude_code/hooks.py`).
- The runtime is the plugin's `Stop` hook (`libs/typescript/integrations/claude-code/hooks/hooks.json:3`), which calls `processTranscript(transcript_path, session_id)` (`src/hooks/stop.ts:15`).
- The plugin reads only `MLFLOW_CLAUDE_TRACING_ENABLED`, `MLFLOW_TRACKING_URI`, `MLFLOW_EXPERIMENT_ID`, `MLFLOW_EXPERIMENT_NAME`, `MLFLOW_TRACE_LOCATION`, `MLFLOW_WORKSPACE`, and `MLFLOW_ENABLE_ASYNC_TRACE_LOGGING` (`src/config.ts:6-19`). The TS core's `MLflowTracingConfig` has no default-tags or default-metadata option (`libs/typescript/core/src/core/config.ts:72-140`).
- The metadata it writes is fixed (`src/tracing.ts:584-617`):
  - `mlflow.trace.session`: the Claude Code session id. If the hook has none, it falls back to a timestamp, `claude-YYYYMMDDTHHMMSS` (`tracing.ts:549-551`).
  - `mlflow.trace.user`, `mlflow.trace.working_directory` (= `process.cwd()`), `mlflow.trace.permission_mode`, `mlflow.claude_code_version`, and cost.
  - `requestPreview` is set to the last user prompt (`tracing.ts:578`).
- **The plugin has no environment variable or hook setting for arbitrary tags or metadata.**
- On the Python side, `MLFLOW_RUN_CONTEXT` (JSON tags) does not reach traces either. `resolve_env_metadata` keeps only `TRACE_RESOLVE_TAGS_ALLOWLIST` keys (`mlflow/tracing/utils/environment.py:35`; `mlflow/utils/mlflow_tags.py:138-151`).
- `update_current_trace(tags=..., metadata=..., client_request_id=...)` (`mlflow/tracing/fluent.py:1506`) needs an active span in the calling Python process (`fluent.py:1617-1625`). It cannot reach traces produced by the TS plugin or by an external agent.
- After the trace exists, tags can still be added with `MlflowClient.set_trace_tag` or `mlflow traces set-tag` (`mlflow/cli/traces.py:476-490`). Metadata "cannot be updated once the trace is logged" (`fluent.py:1523-1524`).

### Claude Code over OTel (native telemetry → MLflow `/v1/traces`)

First-party docs (https://code.claude.com/docs/en/monitoring-usage):

- Traces are a beta, off by default. They need `CLAUDE_CODE_ENABLE_TELEMETRY=1`, `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`, and `OTEL_TRACES_EXPORTER`. Traces use the common OTLP endpoint and header variables, including `OTEL_EXPORTER_OTLP_TRACES_HEADERS`, which is "merged with `OTEL_EXPORTER_OTLP_HEADERS`".
- "Every span carries the standard attributes plus a `span.type` attribute matching its name." `session.id` is one of the standard attributes.
- `OTEL_RESOURCE_ATTRIBUTES`: "Claude Code attaches these values as attributes on every metric datapoint and event record, in addition to sending them in the OTLP resource block." The docs do not explicitly say this applies to trace spans.
- `--session-id`: "Use a specific session ID for the conversation (must be a valid UUID)" (https://code.claude.com/docs/en/cli-reference).
- `--settings` accepts inline JSON per invocation (same page).

Observed **[verified]** (Claude Code 2.1.296, `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`, local capture server):

- With `OTEL_RESOURCE_ATTRIBUTES=pairing.key=task-42`, the **trace** export's resource block contained `pairing.key: task-42` (together with `service.name: claude-code`, `service.version`, `os.type`, `os.version`, `host.arch`). Every span also carried `pairing.key` as a span attribute.
- `--session-id 3f1b2c4d-…` appeared as `session.id` on every span.
- `OTEL_EXPORTER_OTLP_HEADERS=x-mlflow-run-id=run-abc` was sent as an HTTP header.
- Reusing the same `--session-id` in a second invocation failed: `Error: Session ID … is already in use.`

MLflow ingestion (source):

- `/v1/traces` passes each `ResourceSpans.resource` into `Span.from_otel_proto(..., resource=resource)` (`mlflow/server/otel_api.py:208-213`; `mlflow/entities/span.py:578-587`).
- `SqlAlchemyStore.log_spans` writes **every resource attribute as a trace tag**, skipping keys that start with `telemetry.sdk.` or `mlflow.` (`sqlalchemy_store.py:6285-6316`; added in #23829, 2026-06-11). Tested by `test_log_spans_persists_resource_attributes_as_tags` (`tests/store/tracking/sqlalchemy_store/test_sqlalchemy_store_traces.py:4057-4118`). Resource attributes become **tags only, never metadata**.
- Span attribute `session.id` (or `gen_ai.conversation.id`) becomes the trace `session_id`, which is exposed as `mlflow.trace.session` (`sqlalchemy_store.py:5759-5767`).
- Root-span attributes named `mlflow.traceTag.<k>` become trace tags (`sqlalchemy_store.py:5814-5835`, `6320-6326`; prefix at `mlflow/tracing/constant.py:157`).
- Header `x-mlflow-run-id` links all completed traces in the request to that run (`otel_api.py:111`, `268-274`; #23664).
- The `service.name` allowlist (`claude-code`, `codex_cli_rs`, `codex_vscode`, `gemini-cli`, `qwen-code`; `otel_api.py:62-73`) only controls copying `service.name` onto the root span attributes and telemetry. Resource tags are stored either way.

End-to-end replay **[verified]** (captured payloads POSTed to MLflow 3.16.1 `/v1/traces`):

- `tag.\`pairing.key\` = 'task-42'` matched the Claude Code trace and the Codex traces.
- The Claude Code trace carried the tags `pairing.key`, `service.name`, `service.version`, `os.*`, and `host.arch`.
- `run_id = '<run>'` matched the Claude Code trace sent with `x-mlflow-run-id`.
- `metadata.\`mlflow.trace.session\` = '3f1b2c4d-…'` matched it.
- `tag.\`pairing.key\` = 'task-42' AND tag.\`service.name\` = 'claude-code'` returned exactly that one trace.

### Codex (OTLP)

Source: `openai/codex@c3d3b142d1` (main, 2026-10-10). Docs: https://learn.chatgpt.com/docs/config-file/config-advanced and https://learn.chatgpt.com/docs/config-file/config-reference.

- The documented keys are `otel.environment` and `otel.trace_exporter.<id>.{endpoint,headers,protocol,tls.*}`. `headers` is "Static headers included with OTEL trace exporter requests".
- Per-run overrides: "Use `-c` / `--config` when you need to override an arbitrary key … Keys can use dot notation."
- Source-only key (not in the public config reference): `otel.span_attributes`, "Attributes to add to every exported trace span" (`codex-rs/config/src/types.rs:649-650`). It is applied by `SpanAttributesProcessor.on_start` (`codex-rs/otel/src/provider.rs:419-433`). It was added in openai/codex#21556 (2026-05-07) and is documented in `codex-rs/otel/README.md` ("Configured span attributes … are applied to exported trace spans").
- The trace resource is built with `Resource::builder()` (`codex-rs/otel/src/provider.rs:363-372`). In `opentelemetry_sdk` 0.31.0 (the version Codex pins in `codex-rs/Cargo.toml:424`), that builder includes `EnvResourceDetector`, which reads `OTEL_RESOURCE_ATTRIBUTES` (`opentelemetry-sdk/src/resource/mod.rs:57-70` and `resource/env.rs:9,23` at tag `v0.31.0`). Codex adds `service.name`, `service.version`, and `env` on top.

Observed **[verified]** (codex-cli 0.162.0, `codex exec`, `OTEL_RESOURCE_ATTRIBUTES=pairing.key=task-42`, `-c otel.trace_exporter={otlp-http={…,headers={"x-mlflow-run-id"="run-abc"}}}`, `-c otel.span_attributes={"task.id"="task-42-span"}`):

- The resource carried `pairing.key: task-42`, `service.name: codex_exec`, `env: dev`, and `service.version: 0.162.0`. Every captured span (1171 of 1171) carried `task.id`, and the header was sent.
- After replay into MLflow, `tag.\`pairing.key\` = 'task-42'` and `span.attributes.\`task.id\` LIKE '%task-42-span%'` each matched the 8 Codex traces in the replayed batch.
- One `codex exec` run produced **25 distinct OTel trace ids**, with roots such as `auth`, `fs.read_file`, `thread/start`, `session_loop`, `codex.exec`, and others. The model-sampling spans (`codex.sampling`, carrying `conversation.id`) were in a different trace from the `codex.exec` root.
- Codex labels its conversation `conversation.id` (`codex-rs/core/src/session/turn.rs:1455-1459`). MLflow derives `session_id` only from `session.id` or `gen_ai.conversation.id` (`sqlalchemy_store.py:5759-5767`). So these Codex traces got no `mlflow.trace.session`, which was observed after replay.
- `service.name` for `codex exec` is `codex_exec`, which is not in MLflow's allowlist (`otel_api.py:62-73`). It was still stored as a resource tag.

## 3. Does any existing field already identify a "task"?

- **`mlflow.trace.session`** (`constant.py:25`) identifies a conversation, not a task.
  - Hook-based Claude Code uses the Claude Code session id (`tracing.ts:586`). OTel Claude Code uses the `session.id` span attribute (`sqlalchemy_store.py:5759-5767`). Evaluate copies `source_data.session_id` only for static rows (`trace_utils.py:1021-1048`).
  - It is filterable and indexed (`sqlalchemy_store.py:10544-10557`; promoted column, `mlflow/store/tracking/utils/trace_analytics.py:36-40`).
  - Claude Code lets the caller choose it (`--session-id`), but it must be a UUID and a second run cannot reuse it **[verified]**.
- **Request preview**: exists on every trace (the plugin sets it at `tracing.ts:578`; `mlflow/tracing/utils/truncation.py:21-22`), but it is not searchable **[verified]**. `mlflow.traceInputs` metadata is searchable but truncated to 250 chars (`constant.py:190`). Claude Code OTel prompt text is `<REDACTED>` unless `OTEL_LOG_USER_PROMPTS=1` (`mlflow/tracing/otel/translation/claude_code.py:15-16`).
- **`mlflow.eval.requestId`**: with `predict_fn` it is a random uuid4, so it identifies one prediction, not a record (`harness.py:824`).
- **`mlflow.test.case_id`**: the pytest parametrize id, applied to all traces of the `evaluate` call in that test (`harness.py:635-659`).
- **`mlflow.trace.working_directory`** (hook plugin) and **`client_request_id`** are filterable, but nothing sets them to a task id today.
- **No first-class task, record, or pairing field exists** in `TraceMetadataKey` or `TraceTagKey` (`mlflow/tracing/constant.py:5-53`).
