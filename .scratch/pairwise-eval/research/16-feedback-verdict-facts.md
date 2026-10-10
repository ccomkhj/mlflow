# What would break if a Verdict were stored as Feedback? (ticket 16)

Research for the pairwise-evaluation map. Vocabulary follows `.scratch/pairwise-eval/GLOSSARY.md`.

Snapshot: MLflow master `7a010ed01`. Paths are repo-relative; frontend paths are relative to `mlflow/server/js/src/`.

The shape under study (from the ticket):

- an ordinary `Feedback` on the Pair's canonical-first trace (the **anchor**);
- a fixed name, the Judge's rationale in `rationale`;
- `source` = `LLM_JUDGE` with the Judge's id;
- `run_id` = the Comparison run;
- `value` = the outcome, either a string (`"tie"` or the winning trace id) or a dict;
- `metadata` = the other trace id, the Presentation order, the compared Component, both versions and the Rubric URI.

Method: I read the code. I also ran the SQLAlchemy store against a scratch SQLite database, using the worktree's source with no edits. Claims marked **(verified)** come from that run (section 9). Everything else is from reading, and claims about other database backends are not verified.

## Answer

- **Nothing on the server breaks, and server-side numbers ignore a Verdict's value.**
  - `assessment_aggregate` maps `"tie"`, a trace id and a dict all to `None` (`mlflow/store/tracking/utils/trace_analytics.py:193-207`).
  - So AVG/MIN/MAX/PERCENTILE in the ASSESSMENTS view skip the Verdict (**verified**).
  - Every *count* includes it, though:
    - `assessment_count` grouped by name and value returns the raw JSON text (`'"tie"'`, `'{"winner": "tie"}'`), one group per distinct winning trace id (**verified**);
    - the ungrouped count and the opt-in daily rollup add Verdicts to the experiment-wide total (`mlflow/store/db/trace_rollups.py:474-505`).
  - Metrics filters accept only `=` on `assessment.name` / `assessment.type` (`mlflow/utils/search_utils.py:2414-2419`), so no query can exclude Verdicts.
  - Eval-run metrics never see a Verdict: `compute_aggregated_metrics` reads only the current `evaluate()` call's scorer outputs (`mlflow/genai/scorers/aggregation.py:42-47`; `mlflow/genai/evaluation/harness.py:999-1010`).
  - Online scoring neither reads nor deletes it (`mlflow/genai/scorers/online/trace_processor.py:154-224`; `online/session_processor.py:211-247`).
  - `feedback.<name>` filters work on valid rows: `= 'tie'`, `IS NOT NULL`, and `LIKE` on a dict's JSON text (**verified**). Numeric comparators never match.
- **The Verdict is visible in the UI.** I read this code; I didn't run it.
  - **Trace pane:** the Verdict shows as a feedback group. A string shows as text and a dict as truncated JSON. Its metadata, including the partner id, is never displayed (`mlflow/server/js/src/shared/web-shared/model-trace-explorer/assessments-pane/FeedbackItemContent.tsx:35-53, 139`).
  - **Experiment traces table:** each Verdict name becomes a column. A dict cell shows `[object Object]` (`…/traces-v4/components/TraceAssessmentCell.tsx:168`).
  - **Overview quality charts:** each Verdict name gets a value-distribution chart labelled with raw JSON, but no average (`…/experiment-overview/hooks/useAssessmentChartsSectionData.ts:46-126`).
  - **Run Evaluations tab:** it hides assessments from other runs by `sourceRunId` (`…/genai-traces-table/utils/TraceUtils.ts:109-146`).
- **A later Judge can see earlier Verdicts.** The MCP `get_trace` / `search_traces` tools and the agentic-judge `search_traces` tool return the anchor's assessments, including value and rationale (`mlflow/cli/traces.py:375-427`; `mlflow/genai/judges/tools/search_traces.py:33-65`).
- **Grouping by run is client-side.**
  - `run_id` exists only as metadata `mlflow.assessment.sourceRunId`. The entity copies it into the indexed `assessments.run_id` column (`mlflow/entities/assessment.py:114-120`; **verified**).
  - No API filters on that column. There's no `search_assessments`, and `attribute.run_id` is a trace→run link (`mlflow/store/tracking/sqlalchemy_store.py:4213-4236`).
  - `search_traces` does return every assessment, valid and invalid, inside each `TraceInfo` (`sqlalchemy_store.py:10288-10342`).
  - The cheapest standalone read is one paged `search_traces(filter_string="feedback.<name> IS NOT NULL", include_spans=False)` plus a client filter on name, `valid` and `run_id`.
  - Inside `get_next_pair`, the Pairing-key `search_traces` it already runs returns each anchor's Verdicts, so "has this Pair been judged?" costs no extra call.
- **Precedents don't keep anything out of metrics.**
  - **Session-level assessments** sit on the session's first trace with metadata `mlflow.trace.session` (`mlflow/genai/evaluation/session_utils.py:86-96, 139-143`). They are aggregated and counted like any other Feedback. The key only does two things: it widens `feedback.<name>` filters to every trace in the session (`sqlalchemy_store.py:10442-10464`), and it hides the assessment from trace-level UI (`ModelTraceExplorer.utils.tsx:1613-1622`).
  - **`scorerTraceId`** is a metadata link the UI renders as "View trace" (`FeedbackItemContent.tsx:50, 192-218`). What it hides is the *referenced* judge trace, via tag `mlflow.trace.sourceScorer`; the Feedback carrying the link is counted normally.
  - So no reusable "exclude me" convention exists. A Verdict must avoid `mlflow.trace.session`.
- **Deleting the partner trace leaves an orphan, and deleting the run deletes everything.**
  - Deleting the anchor cascades (`mlflow/store/tracking/dbmodels/models.py:1187-1189`).
  - Deleting the partner leaves the Verdict and its dangling metadata untouched (**verified**).
  - Nothing breaks, because no code dereferences ids held in metadata. The UI only links `scorerTraceId`, and a missing linked trace shows "Failed to load trace" (`…/traces-v4/components/TracesV4TraceDrawer.tsx:118-122`).
  - `batch_get_trace_infos` silently omits the deleted id (**verified**).
  - **Deleting the Comparison run hard-deletes every Verdict that carries its `sourceRunId`, invalid rows included, and `restore_run` doesn't bring them back** (`sqlalchemy_store.py:964-975, 1307-1317`; **verified**).
- **Corrections fit, but only by convention.**
  - `overrides` marks the old row `valid=False` on the same trace only (`sqlalchemy_store.py:5144-5160`), and both Verdicts sit on the anchor.
  - Invalid rows drop out of filters and metrics but still come back in `TraceInfo.assessments` (**verified**).
  - `override_feedback` doesn't copy metadata (`mlflow/tracing/assessment.py:465-481`).
  - Deleting the correction revalidates the old row (`sqlalchemy_store.py:5378-5381`).
  - `update_assessment` can rewrite a Verdict in place (`:5206-5343`).
  - The REST update mask passes `valid=` to `update_assessment` (`mlflow/server/handlers.py:4979-4980`). The store has no such parameter and raises `TypeError` (**verified** with a direct store call; the REST path itself is inferred).
- **Limits and reserved names.**
  - `name` and `source_id` are `String(250)`, and `run_id` is `String(32)`.
  - `value`, `rationale` and the metadata blob are `Text`, with no MLflow length check (`models.py:1193-1268`). On MySQL, `TEXT` caps each at 65,535 bytes.
  - Metadata is `map<string,string>`.
  - No name is reserved or rejected. Some names and keys carry behavior: `expected_facts`/`expected_response`, the `/` split in eval aggregation, and the metadata keys `mlflow.assessment.sourceRunId`, `mlflow.trace.session` and `mlflow.assessment.onlineScoringSessionId` (`mlflow/tracing/constant.py:170-187`).

## 1. How the proposed fields map onto a Feedback

- **There is no `run_id` on the wire.**
  - The `Assessment` proto has no `run_id` field (`mlflow/protos/assessments.proto:97-148`).
  - `Feedback.__init__` takes no `run_id` (`mlflow/entities/assessment.py:242-255`), and neither does `mlflow.log_feedback` (`mlflow/tracing/assessment.py:252-261`).
  - The entity fills `run_id` from metadata key `mlflow.assessment.sourceRunId` (`AssessmentMetadataKey.SOURCE_RUN_ID`) in `__post_init__` (`mlflow/entities/assessment.py:114-120`; key at `mlflow/tracing/constant.py:170-172`).
  - The server rebuilds the entity from the proto, so this path also works over REST. The store then writes the `assessments.run_id` column (`mlflow/store/tracking/dbmodels/models.py:1241-1245, 1406`).
  - **(verified)** With that metadata key, the stored row's `run_id` equals the run id. If only the Python attribute `run_id` is set, it is lost in a proto round-trip.
- **`run_id` is indexed, but no reader uses it.**
  - The index is `index_assessments_run_id_created_timestamp` (`models.py:1278`).
  - No store, handler or search code filters on `SqlAssessments.run_id`; a grep finds no use.
  - The run-deletion code matches the metadata JSON instead (section 5).
- **Value:** `FeedbackValueType` accepts str, dict, list and scalars (`assessment.py:29-30`). It is stored as JSON text (`models.py:1369-1371`).
- **Metadata:** proto `map<string, string>` (`assessments.proto:138-140`). `to_proto` stringifies every value (`assessment.py:147-149`). The store writes it as one JSON text blob (`models.py:1392`).
- **Source:** `source_type` is `HUMAN`, `LLM_JUDGE` or `CODE`, and `source_id` is a free string of up to 250 characters (`models.py:1233-1240`).
- **Writable today without new code.** `mlflow.log_feedback` takes a `metadata` argument. So does the MCP tool `log_trace_feedback` (`mlflow/cli/traces.py:511-536`), whose `--metadata` takes a JSON string. The tool `json.loads` the value, so `"tie"` and `tr-…` stay strings and a JSON object becomes a dict (`cli/traces.py:577-582`).

## 2. Existing consumers

### 2.1 Eval-run metric aggregation (`mlflow/genai/scorers/aggregation.py`)

- **It only sees Feedback produced by the current `mlflow.genai.evaluate()` call.**
  - `compute_aggregated_metrics` iterates `eval_result.assessments` (`aggregation.py:42-47`).
  - Those are the scorer outputs of this call (`harness.py:999-1010`), plus expected-value Expectations, plus this call's multi-turn results (`harness.py:741, 887`).
  - It never reads `trace.info.assessments`. A Verdict already on an anchor trace is therefore not aggregated, even if that trace is later passed to `evaluate()`.
- **If a Verdict did reach it,** a string other than yes/no would be dropped with a one-time agent hint (`aggregation.py:77-94`). So would a dict. `CategoricalRating` only knows yes, no and unknown (`mlflow/genai/judges/utils/__init__.py:84-103`).
- **The eval result DataFrame is built from the run's traces.** `_get_assessment_values` keeps assessments with no `sourceRunId` or with this run's id, and drops other runs' (`mlflow/genai/utils/trace_utils.py:997-1007`). A Verdict carrying the Comparison's `sourceRunId` is therefore hidden from other eval runs' result frames. A Verdict without one would appear as a `<name>/value` column.

### 2.2 `query_trace_metrics`, ASSESSMENTS view, and `SqlAssessmentDailyRollup`

- **Two metrics exist.**
  - `assessment_count` is grouped by `assessment_name` and/or `assessment_value`.
  - `assessment_value` supports AVG/MIN/MAX/PERCENTILE, grouped by `assessment_name`.
  - Config: `mlflow/store/tracking/utils/sql_trace_metrics_utils.py:194-211`.
- **Only valid rows are read** (`sql_trace_metrics_utils.py:497-501`).
- **`assessment_value` reads the materialized `aggregate_value`** (`:420-425`). Null aggregates are filtered out (`:901`).
  - `aggregate_value` comes from `assessment_aggregate` (`mlflow/store/tracking/utils/trace_analytics.py:193-207`).
  - That maps bools and the strings yes/no to 1/0 and keeps finite numbers. Everything else is `None`.
  - **(verified)** `"tie"`, a trace id and a dict all give `(None, False)`, so a Verdict never enters AVG/MIN/MAX/PERCENTILE. AVG by name returned no row for it.
- **`assessment_count` counts every valid row, whatever its value.**
  - Grouped by `assessment_value`, it groups on the raw JSON text of `value` (`:474-477`).
  - **(verified)** It returned `'"tie"'` for the string and `'{"winner": "tie"}'` for the dict, so every distinct winning trace id is its own group.
  - **(verified)** With no dimension, the count includes Verdicts: 2 valid rows gave `COUNT: 2`.
- **Filters on this view:**
  - `assessment.name` and `assessment.type`, plus trace status, metadata and tag (`:640-650, 534-631`);
  - only the `=` comparator (`mlflow/utils/search_utils.py:2414-2419`);
  - nothing on assessment `run_id`, metadata or source.

  So a chart can select one name, but no query can exclude one name.
- **Daily rollups:**
  - They are opt-in: `MLFLOW_SQL_TRACE_ROLLUPS_ENABLED` defaults to `False` (`mlflow/environment_variables.py:1711-1712`).
  - `_aggregate_assessment` stores one GLOBAL row per experiment and day: the count of all valid assessments, plus sum/min/max of `aggregate_value`. There is no breakdown by name (`mlflow/store/db/trace_rollups.py:136-148, 474-505`).
  - Rollups serve only requests with no filters and no dimensions (`mlflow/store/tracking/utils/sql_trace_rollups.py:281-311, 404-410`).
  - So Verdicts inflate the rolled-up experiment-wide `assessment_count`, and add nothing to `assessment_value`.

### 2.3 `feedback.<name>` filter in `search_traces`

- **The grammar** is `feedback.<name>` with `=, !=, <, <=, >, >=, LIKE, ILIKE, RLIKE, IS NULL, IS NOT NULL` (`search_utils.py:1745-1755, 1971-1986`).
- **The SQL matches only valid rows of that type and name** (`mlflow/store/tracking/sqlalchemy_store.py:10679-10685`).
  - `=` / `!=` compare the raw text or its JSON-quoted form (`search_utils.py:2042-2053`).
  - `<`, `>` and the other numeric comparators require `is_numeric_value` (`sqlalchemy_store.py:10715-10721`).
  - LIKE-family comparators run on the JSON text (`search_utils.py:2079-2080`).
- **(verified) On SQLite:**
  - `feedback.<name> = 'tie'` matched the anchor;
  - `= '<overridden value>'` matched nothing;
  - `IS NOT NULL` matched the anchor;
  - `feedback.<dict-name> LIKE '%"winner": "tie"%'` matched the anchor.

  The LIKE pattern depends on `json.dumps` spacing.
- **Session expansion trap.**
  - Any matching assessment whose metadata contains `"mlflow.trace.session":` widens the match to every trace in that trace's session (`sqlalchemy_store.py:10442-10464`, used at `:10692-10694, 10723-10725`).
  - A Verdict that carried that key would make `feedback.<verdict-name>` filters return session siblings.
- **There's no filter by assessment run, source or metadata.**
  - `attribute.run_id` / the `run_id=` argument is a trace-level link: trace metadata `mlflow.sourceRun` or an `entity_associations` row (`search_utils.py:1806-1808`; `sqlalchemy_store.py:4213-4236`).
  - **(verified)** `attribute.run_id = '<comparison run>'` returned no traces, because the anchors were never linked to the run.

### 2.4 Trace UI

This section comes from reading the code; the UI was not run.

- **The trace drawer's assessment pane.**
  - Which copy runs:
    - OSS always uses the v2 explorer (`shared/web-shared/model-trace-explorer/shouldEnableRedesignedTraceExplorer.ts:5-7`).
    - Its feedback section imports the v1 `FeedbackGroup` (`…/v2/assessments-pane/AssessmentsPaneFeedbackSection.tsx:13`).
    - So the live renderer is v1 `assessments-pane/FeedbackItemContent.tsx`.
  - Values:
    - the value goes through `JSON.stringify` into `AssessmentDisplayValue` (`FeedbackItemContent.tsx:139`);
    - a plain string such as `"tie"` or `tr-…` shows as plain text;
    - a dict shows as pretty-printed JSON in a one-line, truncated tag, with the full JSON in a tooltip (`assessments-pane/AssessmentDisplayValue.tsx:105-120, 217-237`).
  - Grouping and invalid rows:
    - Feedback is grouped by name, then by `JSON.stringify(value)`, and invalid rows are skipped (`AssessmentsPaneFeedbackSection.tsx:31-62`; `FeedbackValueGroup.tsx:55-57`).
    - The "(N)" count in the header uses the unfiltered list, so it includes invalid rows (`AssessmentsPaneFeedbackSection.tsx:222`).
    - An overriding row gets an "(edited)" link to a history modal (`FeedbackItemContent.tsx:140-163`).
  - **Metadata is not shown generically.** The pane reads only four keys (`FeedbackItemContent.tsx:35, 41-53`):
    - `mlflow.trace.session`;
    - the chunk index;
    - `mlflow.assessment.scorerTraceId`;
    - `mlflow.assessment.judgeCost`.

    So the Verdict's other trace id, Presentation order, Component, versions and Rubric URI would be invisible in the pane.
- **The `scorerTraceId` link.**
  - When the key is present, "View trace" is always rendered, with no existence check (`FeedbackItemContent.tsx:50-51, 192-218`).
  - For a deleted trace, the traces-tab drawer shows `TraceLoadError` ("Failed to load trace") (`experiment-tracking/components/experiment-page/components/traces-v4/components/TracesV4TraceDrawer.tsx:118-122`).
  - A trace id under any other metadata key is not linked at all. So a dangling partner id breaks nothing in the UI; it is simply never shown.
- **Experiment traces table (V4, always on in OSS).**
  - A column is created for every valid, non-session, non-issue assessment name. Only `mlflow.notes` and `_issue_discovery_judge` are excluded (`…/traces-v4/utils/assessmentColumns.ts:25-33`).
  - Each cell shows the latest valid value as `value?.toString()` (`…/traces-v4/components/TraceAssessmentCell.tsx:168`). A string renders as text. A dict renders as `[object Object]`, because `JSON.parse` fails and the raw string is shown (`AssessmentDisplayValue.tsx:111-116`).
  - The column header has no aggregate.
- **Run Evaluations tab (`genai-traces-table`).**
  - Pages: `RunViewEvaluationsTab.tsx`, under the run page and the evaluation-runs page.
  - It drops assessments whose `sourceRunId` is set and differs from the viewed run, and it drops traces tagged `mlflow.trace.sourceScorer` (`shared/web-shared/genai-traces-table/utils/TraceUtils.ts:109-146`).
  - How Verdict values render (`genai-traces-table/utils/AggregationUtils.ts:127-176, 598-664`):
    - a non-yes/no string column becomes type `string`, and its header shows a bar per distinct value (with every winning trace id a separate value);
    - a dict maps to no typed value, so the column becomes `unknown`, with an empty header summary and an italic "null" cell (`TraceUtils.ts:254-256`).
  - Its server-side counts (`useAssessmentCountMetrics`) filter only by trace metadata `mlflow.sourceRun`, not by assessment `sourceRunId` (`…/traces-v3/hooks/useAssessmentCountMetrics.ts:38-61`).
- **Experiment Overview quality charts.**
  - The hook `experiment-tracking/pages/experiment-overview/hooks/useAssessmentChartsSectionData.ts:46-126` filters on `assessment.type = "feedback"` and runs two queries:
    - AVG of `assessment_value` by name over time;
    - COUNT of `assessment_count` by name and value.
  - Assessment names come from the COUNT result, excluding only `_issue_discovery_judge`.
  - So a Verdict name gets its own chart entry with a value distribution, labelled with the raw JSON text (e.g. `"tie"` with quotes; inferred from the backend's grouping column). Because `parseFloat` fails, it gets no average and no line chart (`:105-121`).
- **Session-level assessments in the UI.**
  - `isSessionLevelAssessment` means "has non-empty `metadata['mlflow.trace.session']`" (`shared/web-shared/model-trace-explorer/ModelTraceExplorer.utils.tsx:1613-1622`).
  - Such assessments are removed from the trace pane, the per-turn chat view and the V4 columns.
  - They are shown only in session views.
- **Deleting traces in the UI.** The delete modals show only a generic "cannot be undone" warning. Nothing checks for or warns about assessments on other traces that reference the deleted trace (`genai-traces-table/components/GenAiDeleteTraceModal.tsx:24-82`; `…/traces-v4/components/TracesV4DeleteModal.tsx:51`).
- **No name or source filtering.** No UI code hides assessments by an `mlflow.` name prefix or by `source_type`. `source_type` drives only icons, labels and editability (`assessments-pane/utils.tsx:28-37`; `AssessmentActionsOverflowMenu.tsx:19-21`).

### 2.5 Online and scheduled scoring

- **Per-trace online scoring never reads existing assessments.**
  - It selects traces by time window and the scorer's filter, AND-ed with `metadata.mlflow.sourceRun IS NULL` (`mlflow/genai/scorers/online/trace_processor.py:154-224`; `online/constants.py:15`).
  - It then logs new Feedback with `run_id=None` (`trace_processor.py:329-330`).
  - A Verdict on an anchor neither blocks nor triggers scoring.
- **Per-session online scoring deletes old assessments.** It deletes only assessments whose metadata `mlflow.assessment.onlineScoringSessionId` equals the session and whose name matches a newly logged one (`online/session_processor.py:211-247`). A Verdict without that key is untouched.
- **Scorers that read assessments off the trace:**
  - the `RetrievalSufficiency` scorer reads any assessment named `expected_facts` or `expected_response` (`mlflow/genai/scorers/builtin_scorers.py:683-688`);
  - judge alignment reads only `HUMAN`-sourced assessments whose name equals the judge's (`mlflow/genai/judges/optimizers/dspy_utils.py:430-440`).

  Neither touches an `LLM_JUDGE` Feedback with a distinct name.

### 2.6 Other readers that would see a Verdict

- **Visible to anyone reading the anchor trace, including a later Judge.**
  - The MCP tools `get_trace` and `search_traces` return `info.assessments` unless `--extract-fields` narrows them (`mlflow/cli/traces.py:375-427, 168-245`).
  - The agentic-judge tool `search_traces` hands each trace's Feedback, including value, rationale and `valid`, to an LLM judge (`mlflow/genai/judges/tools/search_traces.py:33-65, 260`).
  - So a Judge that reads the anchor through these tools sees earlier Verdicts on that Pair, with their outcomes and rationales.
- **Issue discovery** reads assessment metadata only to total its own triage run's judge cost (`mlflow/genai/discovery/pipeline.py:726-736`).
- **Auth.** Assessment permissions are one wildcard resource per experiment (`mlflow/server/auth/permissions.py:133, 173`). Validators exist for create, read, update and delete (`mlflow/server/auth/__init__.py:3934-4030, 4637`), and assessments are redacted from trace reads when denied (`:6660-6690`). A Verdict as Feedback inherits all of this, and can't be permissioned apart from other assessments.

## 3. Grouping by run: listing every Verdict of one Comparison

- **There is no assessment search API.**
  - No `search_assessments` or list-by-run endpoint exists. A grep of `abstract_store.py`, `tracking/client.py`, `handlers.py` and `service.proto` finds none.
  - The `run_id` column is indexed but unread (section 1).
- **`search_traces` results do carry assessments.**
  - `_build_trace_infos_from_rows` loads every assessment of each returned trace, valid and invalid, of every name, into `TraceInfo.assessments` (`sqlalchemy_store.py:10288-10342`).
  - `TraceInfo.to_proto` serializes them (`mlflow/entities/trace_info.py:123`).
  - With `include_spans=False`, the client builds `Trace(info, TraceData(spans=[]))` without span loads (`mlflow/tracing/client.py:383-405`).
- **Cheapest standalone read:**
  1. one paged `search_traces(locations=[exp], filter_string="feedback.\`<name>\` IS NOT NULL", include_spans=False)`;
  2. on the client, keep assessments with that name, `valid`, and `run_id` (or `metadata["mlflow.assessment.sourceRunId"]`) equal to the Comparison run.

  Notes:
  - Pages default to 100 traces and max out at 50,000 (`mlflow/store/tracking/__init__.py:21-23`).
  - Child rows load in batches of 500 traces (`sqlalchemy_store.py:10287, 10296-10319`).
  - The filter selects anchors with any valid Verdict of that name, across all Comparisons in the experiment. Only the name narrows it server-side, and names may be up to 250 characters (`models.py:1193`).
- **Cheaper still inside `get_next_pair`.** It already runs `search_traces` over the Pairing-key candidates (tickets 08, 13), and those `TraceInfo`s carry each anchor's Verdicts. So "has this Pair been judged?" and Win-rate inputs need no extra call: they are a client-side scan of assessments already returned.
- **Linking anchors to the run is the only server-side run filter.**
  - `link_traces_to_run` (100 traces per call: `mlflow/tracking/client.py:1158`; `sqlalchemy_store.py:5413-5430`) would make `search_traces(run_id=…)` return the linked anchors.
  - That is a trace-level link, so the linked traces would also appear in that run's trace views.

## 4. Precedents: session-level assessments and `scorerTraceId`

- **Session-level assessments:**
  - They are anchored on the chronologically first trace (`mlflow/genai/evaluation/session_utils.py:86-96, 126`).
  - They are tagged by metadata `mlflow.trace.session = <session id>` (`:139-143, 158`).
  - They are **not** kept out of metrics. `evaluate()` aggregates them under their scorer name with everything else (`harness.py:741, 768`). The ASSESSMENTS view and rollups count them like any other row (section 2.2).
  - What the metadata key does change: `feedback.<name>` filters expand to session siblings (section 2.3), and the UI treats them separately (section 2.4).
- **`mlflow.assessment.scorerTraceId`:**
  - The evaluation harness writes it on a Feedback to name the judge's own trace (`harness.py:960-967`).
  - The judge trace itself gets tag `mlflow.trace.sourceScorer` (`harness.py:968-973`).
  - No Python code reads `scorerTraceId` (only `constant.py:179` and `harness.py:965` mention it).
  - The Feedback carrying it is an ordinary score and is aggregated normally. The convention hides the **referenced trace** (section 2.4), not the Feedback.
- **There is no general "exclude from metrics" convention.**
  - Metrics filters select by name or type with `=` only.
  - Python-side, the only metadata keys with behavior are:

    | Key | Behavior |
    |---|---|
    | `mlflow.assessment.sourceRunId` | sets `run_id`; purged with the run; hidden from other runs' views |
    | `mlflow.trace.session` | session expansion and UI treatment |
    | `mlflow.assessment.onlineScoringSessionId` | online-scoring cleanup |

## 5. Deleting traces and runs

- **Deleting the anchor** cascades to its assessments through `trace_id` FK `ondelete="CASCADE"` (`models.py:1187-1189`). SQLite enforces this because MLflow turns on `PRAGMA foreign_keys` (`mlflow/store/db/utils.py:156`).
- **Deleting the partner leaves the Verdict on the anchor unchanged.**
  - `_delete_traces` only deletes trace rows, rollup partitions and review-queue items (`sqlalchemy_store.py:4789-4889`).
  - **(verified)** After deleting the partner, all Verdict rows remained on the anchor.
  - No Python code dereferences trace ids held in assessment metadata. Every backend read stays well-formed, because nothing follows the dangling id.
  - Looking up the partner directly behaves as follows:
    - `get_trace_info` raises `RESOURCE_DOES_NOT_EXIST` (`sqlalchemy_store.py:4136-4141`);
    - **(verified)** `batch_get_trace_infos` silently omits it.
- **Deleting the Comparison run hard-deletes every Verdict that carries its `sourceRunId`.**
  - `_mark_run_deleted` deletes assessments whose metadata JSON contains `"mlflow.assessment.sourceRunId": "<run id>"` (`sqlalchemy_store.py:964-975`).
  - It is called by `delete_run` (`:1314-1317`) and by `delete_experiment` for every run (`:928-936`).
  - `restore_run` only flips the run's lifecycle (`:1307-1312`).
  - **(verified)** After `delete_run`, all three Verdict rows were gone, including the invalidated one. After `restore_run`, they were still gone.
  - This contradicts ticket 08's "if the run is deleted, its Verdicts survive". A different metadata key would avoid the purge, but it also leaves the `run_id` column null.

## 6. Corrections: `overrides` and `valid`

- **How `overrides` works.**
  - `create_assessment` with `overrides=<id>` sets `valid=False` on that id, but only if it is on the **same trace**; otherwise it raises `RESOURCE_DOES_NOT_EXIST` (`sqlalchemy_store.py:5144-5160`).
  - It doesn't check the name, the assessment type, or whether the target is already invalid.
  - Both Verdicts of one Pair sit on the same anchor (the canonical-first trace), so a correction can always override its predecessor.
- **`mlflow.override_feedback`:**
  - copies only the old name and `span_id`;
  - takes new value, rationale, source and metadata, but does not copy the old metadata (`mlflow/tracing/assessment.py:403-481`);
  - defaults `source` to `CODE` when none is passed (`mlflow/entities/assessment.py:257-259`).

  So a correction has to restate every metadata key, `sourceRunId` included.
- **Invalid rows:**
  - They are excluded from `feedback.<name>` filters, from metrics and from rollups (sections 2.2–2.3).
  - They are still returned in `TraceInfo.assessments` (**verified**), so client readers must check `valid`.
- **Invalidation is reversible, and rows are not immutable.**
  - Deleting the overriding row restores the old row to `valid=True` (`sqlalchemy_store.py:5378-5381`).
  - `update_assessment` rewrites name, value, rationale and metadata in place (metadata is merged). Only source and `span_id` are immutable (`:5206-5343`).
  - So "append-only" would be a client convention, not something the store enforces.
- **Setting `valid` directly fails.**
  - The REST `UpdateAssessment` handler forwards an update-mask path `valid` as a `valid=` kwarg (`mlflow/server/handlers.py:4979-4980`).
  - `SqlAlchemyStore.update_assessment` has no such parameter (`sqlalchemy_store.py:5206-5215`; `abstract_store.py:658-667`).
  - **(verified)** Called directly, the store raises `TypeError: … unexpected keyword argument 'valid'`. I didn't run the REST call; reading the handler, it reaches the same call. Invalidation only works through `overrides`.

## 7. Limits and reserved names

- **Column sizes** (`models.py:1180-1268`; created in `mlflow/store/db_migrations/versions/770bee3ae1dd_add_assessments_table.py:20-44`):

  | Column | Type |
  |---|---|
  | `name` | `String(250)` |
  | `source_id` | `String(250)` |
  | `run_id` | `String(32)` |
  | `overrides` | `String(50)` |
  | `span_id` | `String(50)` |
  | `value`, `rationale`, `assessment_metadata`, `error` | `Text` |

- **No MLflow-side checks.**
  - I found no length check on assessment name, value, rationale or metadata in the entity, handler or store; a grep for assessment validators finds only the search-filter value check.
  - No later migration widens the `Text` columns.
  - On MySQL, `TEXT` holds 65,535 bytes. That is a MySQL property, not MLflow code, and it applies to the whole metadata blob.
  - On PostgreSQL and SQLite, `Text` has no practical limit. MSSQL was not checked.
- **Metadata values are strings on the wire** (`map<string,string>`). The entity stringifies them in `to_proto` (`assessment.py:147-149`).
- **No assessment name is reserved or rejected.** Some names and prefixes do carry behavior:
  - the default name is `feedback` (`mlflow/entities/assessment.py:189`);
  - eval aggregation looks up a scorer's aggregations by the part after the first `/` (`aggregation.py:64`);
  - `expected_facts` and `expected_response` are read by `RetrievalSufficiency` (section 2.5);
  - issue assessments use the issue id as their name (`models.py:1338-1340`).
- **Metadata keys with behavior** (`mlflow/tracing/constant.py:170-187`, plus `mlflow.trace.session`) are listed in section 4. A Verdict should avoid them unless it wants that behavior.

## 8. Write and read cost notes

- **A Verdict write is one `create_assessment`.**
  - It locks the anchor's `trace_info` row to copy `experiment_id` and `trace_timestamp_ms` (`sqlalchemy_store.py:5084-5142`).
  - When rollups are on, it also enqueues an assessment rebuild.
- **Experiment scoping comes from the anchor trace.** `assessments.experiment_id` is copied from it (`:5139-5141`). Nothing checks that the partner trace is in the same experiment.

## 9. What was run

- **Setup:**
  - interpreter: `/opt/miniconda3/bin/python3` through `uv run --no-project`, importing the worktree's `mlflow/` unchanged, with `sqlparse==0.5.3` added in a scratch directory;
  - store: a scratch SQLite database;
  - data: two traces sharing a Pairing-key tag, and a Comparison run.
- **Verdicts written:**
  - `pairwise_verdict` with the partner id as value, later overridden to `"tie"`;
  - `pairwise_verdict_dict` = `{"winner": "tie"}`.

  Both carried `sourceRunId`, the other trace id and the Presentation order in metadata.
- **Results:**
  - the run id round-tripped into `run_id`;
  - the overridden row was `valid=False` but still listed;
  - the filters behaved as in section 2.3;
  - `assessment_count` behaved as in section 2.2, and AVG returned nothing;
  - deleting the partner left 3 rows;
  - `delete_run` removed all of them, and `restore_run` didn't bring them back;
  - `update_assessment(valid=False)` raised `TypeError`.
- **Not run:** MySQL, PostgreSQL or MSSQL; the REST server; the UI.

## Implications for ticket 17

This is the one section that goes beyond facts. For each ADR 0001 requirement, it says whether a Verdict-as-Feedback meets it natively, meets it only client-side, or puts it at risk.

**Met natively**

- **Judge identity via `AssessmentSource`.** `source_type=LLM_JUDGE` plus a free `source_id` of up to 250 characters (`models.py:1233-1240`).
- **Cascade when the anchor is deleted.** FK `ON DELETE CASCADE` (`models.py:1187-1189`).
- **Invalidation of the old Verdict.** `overrides` works because both Verdicts sit on the anchor (`sqlalchemy_store.py:5144-5160`).
- **Access control without an auth PR.** Assessment endpoints already have validators (`mlflow/server/auth/__init__.py:3934-4030`). Grants are wildcard per experiment only (`permissions.py:173`).

**Met client-side only**

- **Canonical order and Presentation order.** The anchor is the canonical-first trace; the other id and the order are untyped metadata strings. The server checks neither.
- **Both traces in one experiment.** The row's `experiment_id` comes from the anchor only (`sqlalchemy_store.py:5139-5141`). The partner is never checked.
- **"Has this Pair been judged?"** It is a scan of the `TraceInfo.assessments` that `get_next_pair` already fetches. There's no per-Pair index: the metadata is JSON text, and `run_id` is indexed but no API reads it.
- **Win rate.** Server metrics can't compute it. AVG ignores strings, COUNT-by-value groups raw JSON per winning trace id, and no run filter exists (section 2.2).
- **Skipping orphans.** A Verdict whose partner isn't among the live traces from the same search must be dropped by the client.

**At risk**

- **Cascade when the partner is deleted.** Not met: orphan rows persist (**verified**). They're harmless to existing code, but every reader must filter them.
- **Append-only.** Only a convention: `update_assessment` mutates in place, and deleting an override revalidates the old row (`sqlalchemy_store.py:5206-5343, 5378-5381`).
- **Ticket 08's "Verdicts survive run deletion".** Broken if the run id is stored as `mlflow.assessment.sourceRunId`: `delete_run` and `delete_experiment` hard-delete those Verdicts (`sqlalchemy_store.py:928-936, 964-975`; **verified**). A different metadata key avoids the purge, but leaves the `run_id` column null.
- **Blind judging and "not on the assessments list".** Verdicts appear in the trace pane, the traces table (a dict shows `[object Object]`), the Overview charts and run Evaluations tabs. They are also returned to later Judges through MCP `get_trace` / `search_traces` (sections 2.4, 2.6).
