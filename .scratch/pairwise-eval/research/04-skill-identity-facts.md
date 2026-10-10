# What identity and version does an agent skill carry? (ticket 04)

Researched 2026-10-10. Sources are pinned as follows:

- MLflow master `7a010ed01`. Paths are repo-relative.
- Claude Code docs fetched 2026-10-10 from `code.claude.com/docs/en/*.md`. Local CLI is `2.1.296`.
- Codex docs fetched 2026-10-10 from `developers.openai.com`. Codex source is `openai/codex` main at `c3d3b142d10f4316b46e35aad7e5317e7e506cb7`. Local CLI is `codex-cli 0.162.0`.
- Agent Skills spec fetched from `agentskills.io` 2026-10-10. Its validator is `agentskills/agentskills` main at `69ef37e9424c0a7ea9dd2293b559e43ec8176379`.
- MLflow bundled skills come from `mlflow/skills` at the submodule pin `5f4dec1e810f40bbd5395d84c99debd175f3e5c8`.

The local evidence (plugin cache, transcripts, Codex rollouts) comes from this machine. It was read without changing anything.

## Answer

- **No skill format defines a skill-level version.**
  - The Agent Skills spec defines six frontmatter fields, and `version` isn't one of them. Its validator rejects any other top-level key. The spec's only `version` appears as an example key inside the free-form `metadata` map.
  - Claude Code accepts more fields, but none of them is a version. It silently ignores unknown fields.
  - Codex parses only `name`, `description` and `metadata.short-description`.
- **Version lives one level up, on the plugin.**
  - In Claude Code, the version comes from the plugin manifest `version` first, then the marketplace entry `version`. When neither is set, it is a 12-character git SHA, an archive digest, or `unknown`. This resolved version names the cache directory `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`.
  - Codex works the same way: the version comes from the `plugin.json` `version` and names `~/.codex/plugins/cache/<marketplace>/<plugin>/<version>/`. The fallback is `local`, and curated plugins use an 8-character git SHA.
  - Standalone skills have no version anywhere. That covers Claude Code personal and project skills, Codex `.agents/skills`, and MLflow-bundled skills once installed.
- **MLflow's bundled skills** are identified by frontmatter `name` (falling back to the directory name) and `description`.
  - Their version is implicit: it is the `mlflow/skills` submodule commit that the MLflow release pins.
  - The `mlflow/skills` plugin manifest says `1.0.0`, but the installer copies skill directories without stamping any version.
- **Claude Code telemetry that MLflow ingests carries the skill name only.**
  - The `claude_code.tool` span for a Skill call has a `skill_name` attribute, gated by `OTEL_LOG_TOOL_DETAILS`. It has no path, no plugin and no version.
  - Richer attribution exists only on OTel *log events* and *metrics*:
    - `skill_activated` has `skill.source`, `plugin.name` and `marketplace.name`.
    - `plugin_loaded` has `plugin.version`.
    - The cost and token counters carry `skill.name` and `plugin.name`.
  - MLflow has no logs or metrics endpoint, only `/v1/traces`.
  - Hook payloads carry the skill name (`tool_input.skill` or `command_name`) but no path or version.
- **Codex telemetry carries no skill identity on traces.**
  - `codex.skill_invocation` is a log-only event. It carries `skill.name`, `skill.scope`, `skill.plugin_id` (`<plugin>@<marketplace>`, with no version) and `skill.invocation_type`. Its source comment says it is "without skill contents, filesystem paths".
  - Trace-side tool events carry lengths only.
- **MLflow's transcript-based integrations do capture the skill's path and content as a side effect, but not in a structured form.**
  - Claude Code: the plugin's Stop hook puts the injected message `Base directory for this skill: <abs dir>\n\n<rendered body>` into the next LLM span's input messages.
  - Codex: the notify hook puts `<skill><name>…</name><path>…/SKILL.md</path>\n<full SKILL.md>\n</skill>` into the LLM span's input messages.
  - For marketplace plugin skills, that path contains the plugin version segment.
- **Deriving the version from the trace alone is possible only in part.**
  - A plugin version can be parsed from the captured path, but only for cached marketplace plugins.
  - A content hash can be computed from the captured text, with limits:
    - Claude Code sends a *rendered* body: frontmatter is stripped, and arguments, `${CLAUDE_*}` variables and `!` command output are substituted. Codex sends the raw file.
    - Supporting files are never included.
    - Claude Code may send the body only once per session.
  - With default OTel export, nothing beyond the skill name is available. Anything more needs either client-side stamping or Claude Code's detailed beta tracing content attributes, which this research did not verify.

## 1. Skill formats and the identity fields they define

### 1.1 Agent Skills open spec (agentskills.io)

- **Frontmatter fields.** `name` (required, at most 64 characters, lowercase a-z 0-9 and hyphens, must match the parent directory), `description` (required), `license`, `compatibility`, `metadata` and `allowed-tools`. There is no `version` field. Source: https://agentskills.io/specification ("Frontmatter" table).
- **`metadata` is a free-form map.** It is "a map from string keys to string values", where "Clients can use this to store additional properties not defined by the Agent Skills spec". The spec's example puts `version: "1.0"` inside `metadata`, which makes it a convention, not a defined field. Source: https://agentskills.io/specification ("`metadata` field").
- **The validator rejects a top-level `version`.** The reference validator `skills-ref` lists exactly those six keys in `ALLOWED_FIELDS` and errors with "Unexpected fields in frontmatter" on anything else. Source: https://github.com/agentskills/agentskills/blob/69ef37e9424c0a7ea9dd2293b559e43ec8176379/skills-ref/src/skills_ref/validator.py (lines 15-22 and 108-112).
- **Location is a client concern.** The spec "does not mandate where skill directories live". Its client guide has clients keep a `location`, the absolute path to `SKILL.md`, and derive the skill's base directory from it. Source: https://agentskills.io/client-implementation/adding-skills-support.

### 1.2 Claude Code skills

- **No version field.** The frontmatter fields are `name`, `description`, `when_to_use`, `argument-hint`, `arguments`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `disallowed-tools`, `model`, `effort`, `context`, `agent`, `background`, `hooks`, `paths`, `shell`, `metadata`, `license` and `compatibility`. The docs say "Claude Code ignores a field it doesn't recognize without reporting an error". Source: https://code.claude.com/docs/en/skills#frontmatter-reference.
- **`metadata` is for your own tooling.** It is a "Free-form YAML map for your own key-value data … read by your own tooling from `SKILL.md`. Claude Code doesn't act on its contents." Source: same table.
- **Uploads allow only the spec's six fields.** claude.ai uploads, the Skills API and `package_skill.py` accept only `name`, `description`, `license`, `compatibility`, `metadata` and `allowed-tools`, and fail hard on others. Source: https://code.claude.com/docs/en/skills#using-skill-frontmatter-outside-claude-code.
- **Install locations.** The locations are enterprise, personal (`~/.claude/skills/<name>/SKILL.md`), project (`.claude/skills/<name>/SKILL.md`), nested, `--add-dir`, plugin (`<plugin>/skills/<name>/SKILL.md`) and claude.ai-synced (`~/.claude/skills/synced/`). Source: https://code.claude.com/docs/en/skills#where-skills-live and #where-synced-skills-load.
- **Identity is the command name.** It is the frontmatter `name` or the directory name. Plugin skills are namespaced as `/<plugin-manifest-name>:<skill>`, nested clashes become `/<subdir>:<skill>`, and synced skills become `/anthropic-skills:<name>`. Source: https://code.claude.com/docs/en/skills#how-a-skill-gets-its-command-name. The namespace is the plugin's *manifest* name, not the marketplace entry name. Source: https://code.claude.com/docs/en/plugins/loading#entry-name-and-manifest-name.
- **Content can change within one session.**
  - Live change detection picks up `SKILL.md` edits under the personal, project and `--add-dir` skill directories without a restart. Source: https://code.claude.com/docs/en/skills#live-change-detection.
  - Synced skills are re-checked about every 10 minutes and updated in the running session. Source: https://code.claude.com/docs/en/skills#where-synced-skills-load.

### 1.3 Claude Code plugins and marketplaces

- **`plugin.json` `version`.** It is "A version string, not checked against semver. Setting it pins the plugin to that version until you change it." Source: https://code.claude.com/docs/en/plugins/manifest-reference#version.
- **Marketplace entries can also carry `version`.** When both set it, the manifest wins. Sources: https://code.claude.com/docs/en/plugins/marketplace-reference (entry `version`) and https://code.claude.com/docs/en/plugins/manifest-reference ("Metadata precedence").
- **How the resolved version is computed.** The order is: the manifest `version`, then the marketplace-entry `version`, then a value derived from the source type:

  | Source type | Derived version |
  | - | - |
  | `github`, `url`, `git-subdir` | 12-character commit SHA |
  | `archive` | 12-character SHA-256 |
  | Relative path inside a git-hosted marketplace | Commit SHA |
  | Local non-git directory | `unknown` |
  | `npm` | `unknown` |
  | `command` source | `<hash>` or `<manifest version>-<hash>` |

  Source: https://code.claude.com/docs/en/plugins/loading#how-claude-code-computes-the-version.
- **Plugin identity.** A plugin id is `<name>@<origin>`, where origin is a marketplace name, `inline` (`--plugin-dir`), `skills-dir` or `synced`. Source: https://code.claude.com/docs/en/plugins/loading#find-where-a-plugin-came-from.
- **Cache layout.**
  - `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/` holds one directory per installed version, and "`${CLAUDE_PLUGIN_ROOT}` points at this directory".
  - The `installed_plugins.json` and `known_marketplaces.json` files record installs.
  - Old versions get an `.orphaned_at` marker and are removed 14 days later.
  - Source: https://code.claude.com/docs/en/plugins/loading#find-plugins-on-disk and #cleanup-of-previous-versions.
- **Some plugins are not copied, so their version string doesn't track content.**
  - `--plugin-dir`, skills-directory plugins, and relative-path plugins in a local-path marketplace all load in place.
  - "you don't need to increase the version": edits take effect at the next session.
  - For a claude.ai-hosted marketplace, claude.ai's recorded version is used and the manifest `version` "isn't read".
  - Source: https://code.claude.com/docs/en/plugins/loading#in-place-and-copied-plugins and #versions-and-updates.
- **Name shadowing.** A `--plugin-dir` plugin with the same manifest name silently replaces an installed marketplace plugin, while `claude plugin list` still shows the marketplace row. Source: https://code.claude.com/docs/en/plugins/loading#name-conflicts.
- **Local observation: cache layout.** `~/.claude/plugins/cache/mattpocock/mattpocock-skills/` holds `1.2.3/` and `1.3.1/`. Inside `1.3.1/`:
  - `.claude-plugin/plugin.json` has `"version": "1.3.1"` and lists 27 skill paths.
  - The co-located `.claude-plugin/marketplace.json` entry has no `version`.
  - No `SKILL.md` has a version field. Example: `skills/engineering/research/SKILL.md` has `name` and `description` only.
  - Each skill directory also ships an `agents/openai.yaml`, the Codex metadata file (see 1.4).
- **Local observation: `installed_plugins.json`** (version 2). Each install records `scope`, `installPath`, `version`, `gitCommitSha`, `installedAt` and `lastUpdated`, and `projectPath` for project or local scope.
  - Versions are semver strings (for example `mattpocock-skills@mattpocock` → `1.3.1`, sha `f3fc5632f401…`) or a 12-character SHA when the plugin declares none (for example `code-review@claude-plugins-official` → `b8e53f1c05df`).
  - In one case, `vercel-plugin@vercel`, the recorded `version` (`0.44.0`) differs from the `installPath` directory segment (`b73bc95636c4`).
  - `claude plugin list --json` exposes `id`, `version`, `scope`, `enabled`, `installPath`, `installedAt` and `lastUpdated`. It adds `readFromFolder` for in-place plugins, for example `autopilot-forge`.
- **Local observation: SKILL.md content across versions.** Across `1.2.3` and `1.3.1`, 32 of 34 skills present in both have different `SKILL.md` bytes, and 2 are byte-identical. A plugin version bump therefore doesn't imply that a given skill changed. Several skills carry supporting files (for example `tdd/` has `mocking.md` and `tests.md`; `teach/` has four `*-FORMAT.md` files). A hash of `SKILL.md` alone doesn't cover those files.
- **MLflow's own Claude Code marketplace.** `.claude-plugin/marketplace.json` declares `mlflow-tracing` with an `npm` source `@mlflow/claude-code`.
  - The plugin manifest says `"version": "0.2.0"` (`libs/typescript/integrations/claude-code/.claude-plugin/plugin.json:3`). The npm package says `"version": "0.4.0"` (`libs/typescript/integrations/claude-code/package.json:3`).
  - Because the manifest wins, the cache directory would be `…/0.2.0/`. An author-maintained `version` can lag the shipped content.

### 1.4 OpenAI Codex skills and plugins

- **Docs: skill format.** A skill is a directory whose `SKILL.md` "must include `name` and `description`", plus optional `scripts/`, `references/`, `assets/` and `agents/openai.yaml`. The docs list no version field. Source: https://developers.openai.com/codex/skills.
- **`agents/openai.yaml`.** It holds `interface` (`display_name`, `short_description`, icons, `brand_color`, `default_prompt`), `policy.allow_implicit_invocation` and `dependencies.tools`, with no version. Source: same page, "Optional metadata".
- **Locations.** Codex reads `.agents/skills` from the working directory up to the repo root (scope `REPO`), `$HOME/.agents/skills` (`USER`), `/etc/codex/skills` (`ADMIN`) and bundled skills (`SYSTEM`). Skills can be disabled by `path` in `~/.codex/config.toml` `[[skills.config]]`. Source: same page.
- **Source: frontmatter parsing.** `SkillFrontmatter` deserializes only `name`, `description` and `metadata.short-description`. Source: https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/skills/src/parser.rs (lines 6-20).
- **Source: in-memory model.** `SkillMetadata` holds `name`, `description`, `short_description`, `interface`, `dependencies`, `policy`, `path_to_skills_md`, `scope`, `plugin_id` and `remote_plugin_id`, with no version. Source: `codex-rs/skills/src/model.rs` lines 8-20, same commit. The app-server protocol type `SkillMetadata` mirrors it (`codex-rs/app-server-protocol/schema/typescript/v2/SkillMetadata.ts`).
- **Plugins carry the version.**
  - `plugin.json` has root `name` and `version` (example `"version": "1.0.0"`), with `.codex-plugin/plugin.json` as a compatibility fallback.
  - Plugins install into `~/.codex/plugins/cache/$MARKETPLACE_NAME/$PLUGIN_NAME/$VERSION/`, and "For local plugins, `$VERSION` is `local`".
  - A `.claude-plugin/marketplace.json` is accepted as a legacy-compatible marketplace.
  - Source: https://developers.openai.com/plugins/build/plugins.
- **Source: how Codex picks the cache version.**
  - The install version is the manifest version, or `DEFAULT_PLUGIN_VERSION = "local"` (`codex-rs/core-plugins/src/store.rs:425-445`, `codex-rs/core-plugin-common/src/installed.rs:10`).
  - Curated-marketplace plugins use a git SHA truncated to 8 characters (`codex-rs/core-plugins/src/loader.rs:76` and `:504-510`).
  - The plugin id is `<plugin>@<marketplace>` (`codex-rs/core-plugin-common/src/plugin_id.rs:10-47`).
- **Local observation: Codex plugin cache.** `~/.codex/plugins/cache/openai-curated/sentry/11c74d6b/.codex-plugin/plugin.json` declares `"version": "0.1.2"`, but the directory segment is the curated SHA `11c74d6b`. `superpowers` is the same: `5.1.3` in the manifest, under `11c74d6b/`.

### 1.5 MLflow's own bundled skills

- **Source of the skills.** They come from the git submodule `mlflow/assistant/skills` → `https://github.com/mlflow/skills` (`.gitmodules`), pinned at commit `5f4dec1e810f…` (`git ls-tree HEAD mlflow/assistant/` shows mode `160000`). The submodule isn't checked out in this worktree or the main checkout.
- **Listing.** `list_bundled_skills()` builds a `BundledSkill(name, description, path)` from frontmatter `name`, falling back to the directory name, and `description`. It has no version field. Source: `mlflow/assistant/skill_installer.py:19-23`, `:30-59`. `mlflow skills list` and `mlflow skills view` print only name, path and description (`mlflow/cli/skills.py:8-13`).
- **Installing.** `install_skills(dest)` `copytree`s each skill directory into `dest/<dir-name>` and writes nothing else: no version file and no marker (`mlflow/assistant/skill_installer.py:62-91`).
- **Where installs go.**
  - Repo-relative `.claude/skills` (Claude Code), `.agents/skills` (Codex, OpenCode) (`mlflow/agent/agents.py:30-50`).
  - The Assistant's global or project locations: `~/.claude/skills`, `~/.codex/skills` (`mlflow/agent/setup/cli.py:114-115`; `mlflow/assistant/providers/claude_code.py:472-474`; `mlflow/assistant/providers/codex.py:162-163`).
  - These are personal or project skill locations, so no plugin version segment is present.
- **What `mlflow/skills` itself declares** (at `5f4dec1e`):
  - `.claude-plugin/plugin.json` has `"name": "mlflow"`, `"version": "1.0.0"`, 13 skill paths, a `UserPromptSubmit` hook and `"minClaudeCodeVersion": "2.0.0"`.
  - The 13 `SKILL.md` files use only `name`, `description`, `allowed-tools` and `disable-model-invocation`, with no `metadata` and no version.
  - `debug-agent/agents/openai.yaml` has an `interface` block only.
  - Source: `gh api repos/mlflow/skills/tarball/5f4dec1e810f40bbd5395d84c99debd175f3e5c8`.

### 1.6 Side note: Claude API skills have server-assigned versions

In the Messages API `container.skills` parameter, a skill is `{type, skill_id, version}`:

- Anthropic skills use date versions such as `20251013` or `latest`.
- Custom skills use `skver_…` ids or `latest`.

This is the only first-party surface found where a skill version is a managed identifier. It doesn't apply to Claude Code or Codex local skills. Source: https://platform.claude.com/docs/en/build-with-claude/skills-guide ("Using Skills" table).

## 2. Which identity information reaches telemetry

### 2.1 Claude Code OTel traces (the signal MLflow ingests)

- **Ingestion is traces-only.** MLflow's OTLP router mounts only `OTLP_TRACES_PATH = "/v1/traces"` (`mlflow/server/otel_api.py:104`, `mlflow/tracing/utils/otlp.py:20`). There is no logs or metrics endpoint.
- **Correction to baseline-facts.** The service-name allowlist is `claude-code`, `codex_cli_rs`, `codex_vscode`, `gemini-cli` and `qwen-code` (`mlflow/server/otel_api.py:62-73`), not only the two Codex names.
- **Span hierarchy.** It is `claude_code.interaction` → `claude_code.llm_request` / `claude_code.tool` (→ `.blocked_on_user`, `.execution`), plus `claude_code.hook` under detailed beta tracing. Source: https://code.claude.com/docs/en/monitoring-usage#span-hierarchy.
- **`claude_code.tool` span attributes.** They are `tool_name`, `tool_name_safe`, `duration_ms`, `result_tokens`, `tool_use_id`, `agent_id` and the like, plus `skill_name` ("Skill name for the Skill tool"), gated by `OTEL_LOG_TOOL_DETAILS`. The span attributes in the docs carry no skill path, source, plugin, marketplace or version. Source: https://code.claude.com/docs/en/monitoring-usage#span-attributes.
- **Content attributes need detailed beta tracing.**
  - Prompt and tool content (`new_context`, `tool_input`, `system_reminders`, …) is emitted "only when detailed beta tracing is active". That requires `ENABLE_BETA_TRACING_DETAILED=1` and `BETA_TRACING_ENDPOINT`, plus org allowlisting for interactive CLI sessions.
  - On `claude_code.llm_request`, `new_context` is "The new user messages and tool results sent with the request", gated by `OTEL_LOG_USER_PROMPTS`.
  - Inference, not verified: the injected skill message (see 2.4) would be part of that `new_context`.
  - Source: same page, "Content attributes under detailed beta tracing".
- **What MLflow keeps from these spans.**
  - `ClaudeCodeTranslator` maps `claude_code.tool` → TOOL.
  - For TOOL inputs it uses `tool_input`. If that is absent, it builds a JSON object from `full_command`, `file_path`, `skill_name` and `subagent_type` (`mlflow/tracing/otel/translation/claude_code.py:36`, `:111-127`).
  - For LLM inputs it uses `new_context` (`:117-118`).
  - Translation only *adds* `mlflow.spanInputs` and `mlflow.spanOutputs`. The original attributes, including a raw `skill_name`, stay on the stored span (`mlflow/tracing/otel/translation/__init__.py:60-98`).
  - Nothing in `mlflow/` reads or promotes `skill_name` to trace metadata or tags. A grep for `skill_name`, `skill.name` and `skill_activated` across `mlflow/` hits only the translator and the bundled-skills CLI.

### 2.2 Claude Code OTel events and metrics (exported, but not ingested by MLflow)

- **`claude_code.skill_activated` event.** Attributes:
  - `skill.name`: `"custom_skill"` for user-defined and third-party plugin skills unless `OTEL_LOG_TOOL_DETAILS=1`.
  - `invocation_trigger`: `user-slash`, `claude-proactive` or `nested-skill`.
  - `skill.source`: for example `bundled`, `userSettings`, `projectSettings` or `plugin`.
  - `skill.kind`.
  - `plugin.name` and `marketplace.name`: only with `OTEL_LOG_TOOL_DETAILS=1` or an official marketplace.
  - It has no version and no path. Source: https://code.claude.com/docs/en/monitoring-usage#skill-activated-event.
- **`claude_code.plugin_loaded` event.** It fires once per enabled plugin at session start. It includes `plugin.version` ("version from the plugin manifest. Included only when the name is not redacted and the manifest declares a version"), `plugin.scope`, `enabled_via`, `plugin_id_hash` and `skill_path_count`. Source: #plugin-loaded-event.
- **`claude_code.plugin_installed` event.** It carries `plugin.version` ("when declared in the marketplace entry"). Source: #plugin-installed-event.
- **Cost and token counters, and `api_request` events.** Each API request is attributed with `skill.name` ("Skill active for the request") and `plugin.name`; `marketplace.name` is official-only. Third-party names are redacted to `"third-party"` unless `OTEL_LOG_TOOL_DETAILS=1`. Source: #cost-counter.
- **`tool_result` and `tool_decision` events.** They carry `tool_parameters` with `skill_name` for the Skill tool, and `tool_input` truncated to about 4K, both gated by `OTEL_LOG_TOOL_DETAILS`. Source: #tool-result-event.
- **Correlation keys.** Events and hooks share `prompt.id` / `prompt_id`. Spans and events share `tool_use_id`. Source: #tool-result-event and https://code.claude.com/docs/en/hooks#common-input-fields.

### 2.3 Claude Code hook payloads

- **Common fields.** They are `session_id`, `prompt_id`, `transcript_path`, `cwd`, `scratchpad_dir`, `permission_mode`, `effort` and `hook_event_name`, plus `agent_id` and `agent_type` inside subagents. None identifies a skill version. Source: https://code.claude.com/docs/en/hooks#common-input-fields.
- **`PreToolUse` / `PostToolUse`.**
  - They receive `tool_name`, `tool_input` and `tool_use_id`, and `PostToolUse` adds `tool_response` and `duration_ms`. Source: #pretooluse-input and #posttooluse-input.
  - The docs' per-tool `tool_input` schemas don't include the Skill tool.
  - Observed in a local transcript (Claude Code 2.1.295): the Skill `tool_use.input` is `{"skill": "mattpocock-skills:grilling"}`. The transcript's `toolUseResult` for it is `{"success": true, "commandName": "mattpocock-skills:grilling"}`. That suggests, without verifying it, that a Skill hook payload carries the namespaced skill name only.
- **`UserPromptExpansion`.** This covers the `/skill` path, which bypasses `PreToolUse`. It carries `expansion_type`, `command_name`, `command_args`, `command_source` (for example `"plugin"`) and `prompt`. It has no path or version. Source: #userpromptexpansion.
- **Hooks can read the disk themselves.** A hook gets `transcript_path`, so it can read the transcript, and it can read the plugin cache on disk. That would be client-side stamping, not something the payload provides.

### 2.4 Claude Code transcripts (what MLflow's Claude Code plugin reads)

- **Three entries per invocation.** Each Skill invocation writes three transcript entries (observed in `~/.claude/projects/-Users-huijokim-personal-mlflow/57d4232c-….jsonl`, lines 30-32):
  1. The assistant `tool_use` entry `{"name": "Skill", "input": {"skill": "<plugin>:<skill>"}}`.
  2. A user `tool_result` entry `"Launching skill: <name>"` with `toolUseResult: {success, commandName}`.
  3. A user entry with `isMeta: true` and `sourceToolUseID: <tool_use_id>`, whose text is `Base directory for this skill: <absolute skill dir>\n\n<rendered SKILL.md body>`.
- **Byte comparison.** For `grilling` (no substitutions), the injected body equals the `SKILL.md` text after the closing `---`, minus the frontmatter and its leading blank line. Checked with `cmp` against `~/.claude/plugins/cache/mattpocock/mattpocock-skills/1.3.1/skills/productivity/grilling/SKILL.md`.
- **The base-directory line appears for every skill source.** A tally over transcripts since 2026-09-01:
  - Plugin cache paths with the version segment: `…/cache/mattpocock/mattpocock-skills/1.2.3/…` (168 invocations) and `…/1.3.1/…` (9).
  - Personal skills: `~/.claude/skills/<name>` (no version).
  - Project skills: `<repo>/.claude/skills/<name>` (no version).
  - claude.ai-synced skills: `~/.claude/skills/synced/<uuid>_<uuid>/xlsx`. The meaning of those two UUID segments is undocumented.
- **Docs: how the body is rendered.**
  - "the rendered `SKILL.md` content enters the conversation as a single message". Source: https://code.claude.com/docs/en/skills#skill-content-lifecycle.
  - Rendering substitutes `$ARGUMENTS`, `$N`, named args, `${CLAUDE_SESSION_ID}`, `${CLAUDE_EFFORT}`, `${CLAUDE_SKILL_DIR}`, `${CLAUDE_PROJECT_DIR}`, `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}`. It appends `ARGUMENTS: <value>` when no placeholder consumes the arguments. Source: #available-string-substitutions.
  - It also replaces each `` !`cmd` `` with the command's output. Source: #inject-dynamic-context.
- **Docs: the body may not repeat.**
  - "When Claude re-invokes a skill whose rendered content is identical to the copy already in context, Claude Code adds a short note … rather than a second copy".
  - After compaction, only "the first 5,000 tokens of each" recent skill are re-attached, within a 25,000-token budget.
  - Source: #skill-content-lifecycle.

### 2.5 What MLflow keeps for Claude Code, by integration path

- **Stop-hook plugin (`@mlflow/claude-code`, transcript-based).**
  - Each tool call becomes a TOOL span named `tool_<name>`, with `inputs = tool_use.input` (for Skill, `{"skill": "<plugin>:<skill>"}`) and attributes `tool_name` and `tool_id`. Its outputs are `{result: "Launching skill: …"}` (`libs/typescript/integrations/claude-code/src/tracing.ts:476-486`).
  - LLM span inputs are `messages` collected back to the previous assistant text entry, and every entry with `message.role` and `content` is included (`src/tracing.ts:132-171`, `:427-435`). The `isMeta` skill message is a user message with content, so the `Base directory for this skill: …` line and the body land in the next LLM span's `inputs.messages`. This is inferred from the code and not checked on a stored trace.
  - The user-prompt detector deliberately skips that injection (`src/transcript.ts:109-121`; test fixture `tests/transcript.test.ts:135-185`).
  - Trace metadata adds only session, user, working directory, permission mode, `mlflow.claude_code_version` and cost (`src/tracing.ts:584-617`, `src/_internal.ts:17-19`). There is no plugin or skill key.
- **Agent SDK live tracer (`tracedClaudeAgent` / `liveTracing.ts`).** `onSystemInit` reads only `session_id`, `model`, `permissionMode` and `claude_code_version` (`src/liveTracing.ts:62-70`, `:135-140`). The SDK init message also carries `skills: string[]` and `plugins: {name, path}[]`, and the path contains the version directory (https://code.claude.com/docs/en/agent-sdk/typescript#sdksystemmessage). MLflow doesn't record either. Tool spans match the Stop-hook path (`src/liveTracing.ts:181-188`).
- **Python Agent SDK tracing.** It writes TOOL spans with `inputs = tool_block.input` and `tool_name`/`tool_id` attributes, and nothing skill-specific (`mlflow/claude_code/tracing.py:312-321`).
- **OTel path.** See 2.1: the `skill_name` attribute only, plus content under detailed beta tracing.

### 2.6 Codex telemetry

- **Docs.** Codex OTel exports log events (`codex.conversation_starts`, `codex.api_request`, `codex.sse_event`, `codex.user_prompt`, `codex.tool_decision`, `codex.tool_result`, …), configured with `[otel] exporter`. There is a separate `otel.trace_exporter` for traces and `otel.metrics_exporter` for metrics. The metrics catalog includes `skill.injected` (fields `status`, `skill`) and `thread.skills.*` histograms. Sources: https://developers.openai.com/codex/config-advanced#observability-and-telemetry and https://developers.openai.com/codex/config-reference (`otel.*` keys).
- **Source: skill invocation event.** `SessionTelemetry::skill_invocation` emits `codex.skill_invocation` with `skill.name`, `skill.scope`, `skill.plugin_id`, `skill.invocation_type` (explicit or implicit), `turn.id`, `conversation.id`, `model` and `app.version`.
  - Its module comment says it "Exports skill-use metadata without skill contents, filesystem paths, or tool payloads".
  - Source: `codex-rs/otel/src/skill_invocation.rs` lines 1-2, 16-22 and 41-59, at `c3d3b142`.
  - It goes to target `codex_otel.log_only`, which the trace exporter never exports as an event. `trace_export_filter` passes non-span events only for `codex_otel.trace_safe` (`codex-rs/otel/src/targets.rs:1-11`; `codex-rs/otel/src/provider.rs:341-349`).
- **Source: where the skill event fires.**
  - Explicit invocations are emitted for injected skills (`codex-rs/core/src/skills.rs:87-100`).
  - Implicit invocations are detected from shell commands that read a skill's `SKILL.md` or run its scripts (`codex-rs/core/src/skills.rs:140-185`).
  - `skill.plugin_id` is `<plugin>@<marketplace>`, with no version (see 1.4).
- **Source: the tool-result event on traces.** The trace-side `codex.tool_result` fields are `tool_name`, `call_id`, `duration_ms`, `success`, `arguments_length`, `output_length`, `output_line_count`, `tool_origin` and `mcp_tool`. Arguments and output text go to logs only (`codex-rs/otel/src/tool_result.rs:54-80`). No skill-named span was found: the skill-loading spans carry only counts and are mostly `level = "trace"` (`codex-rs/ext/skills/src/host_prompt.rs:64-68`, `selection.rs:13-20`).
- **Source: how skill content reaches the model.**
  - The selected skill is injected as a user-role contextual fragment whose `body()` is `\n<name>{name}</name>\n<path>{path}</path>{resource_access}\n{contents}\n`, wrapped in `<skill>…</skill>` (`codex-rs/ext/skills/src/fragments.rs:62-110`).
  - `contents` is the whole `SKILL.md` text read from `path_to_skills_md`, including frontmatter (`ext/skills/src/host_prompt.rs:76-95`; `ext/skills/src/host_outcome.rs:117-127`). It is truncated only for agent-plugin skills over the main-prompt limit.
- **Local observation: rollout file.** `~/.codex/sessions/2026/07/14/rollout-…-019f61ea-8710-….jsonl` contains a `response_item` user message `<skill>\n<name>teach</name>\n<path>/Users/huijokim/.agents/skills/teach/SKILL.md</path>\n---\nname: teach\n…`, so the full file and frontmatter are recorded in the rollout.

### 2.7 What MLflow keeps for Codex

- **OTLP path.** MLflow accepts Codex service names on `/v1/traces` (`mlflow/server/otel_api.py:62-73`), but has no Codex translator (baseline-facts). Per 2.6, Codex traces carry no skill name, path or content.
- **Notify-hook plugin (`@mlflow/codex`, rollout-based).**
  - `reconstructMessages` maps every `response_item` message with role `user`, `assistant` or `developer` into the LLM span's `inputs.messages` without filtering (`libs/typescript/integrations/codex/src/tracing.ts:173-215`, `:265`). The `<skill>` fragment, with its absolute path and the full `SKILL.md`, therefore lands in LLM span inputs. This is inferred from the code and not checked on a stored trace.
  - Only user-prompt detection skips text starting with `<` (`src/transcript.ts:95-96`).
  - A grep for `skill` in `libs/typescript/integrations/codex/src/` returns nothing, so there is no structured skill field.

## 3. Could a skill's version be derived from the trace alone?

What a stored trace can contain today:

| Integration path | Skill name | Path (and plugin version) | Content |
| - | - | - | - |
| Claude Code OTel (default) | `skill_name` on the TOOL span if `OTEL_LOG_TOOL_DETAILS=1` | none | none |
| Claude Code OTel + detailed beta tracing + `OTEL_LOG_USER_PROMPTS` | same | probably inside LLM `new_context` (unverified) | probably the rendered body (unverified) |
| Claude Code Stop-hook plugin | TOOL span input `skill` (`<plugin>:<skill>`) | `Base directory for this skill:` line in the next LLM span's input messages | rendered body (no frontmatter) in the same message |
| Claude Agent SDK (TS live or Python) | TOOL span input | as the Stop-hook row, if the injected user message is streamed; not verified | as the Stop-hook row; not verified |
| Codex OTel | none on traces (log event only) | none | none |
| Codex notify-hook plugin | inside `<name>` of the fragment | `<path>` of the fragment | full `SKILL.md` including frontmatter |

Facts that bound derivation:

- **Parsing the version from the path.**
  - It works when the skill comes from a copied marketplace plugin. In Claude Code the version is the segment after `cache/<marketplace>/<plugin>/` (plugins/loading#find-plugins-on-disk). In Codex it is the segment after `plugins/cache/<marketplace>/<plugin>/` (plugins/build/plugins).
  - That segment may be an author string, a 12-character SHA (Claude Code), an 8-character SHA (Codex curated), `unknown` (Claude Code npm or local sources without `version`), or `local` (Codex).
  - It carries no version for personal, project, nested, synced or MLflow-installed skills, or for in-place plugins (`--plugin-dir`, local-path marketplaces), whose content can change without the string changing (§1.3).
  - The path is machine-specific: it embeds the home directory.
- **Hashing the content.**
  - Codex records the raw file, so a hash of the captured text equals a hash of `SKILL.md`.
  - Claude Code records the rendered body without frontmatter. The hash equals a hash of the file body only when the skill has no argument placeholders, `${CLAUDE_*}` variables or `` !`cmd` `` blocks. Frontmatter edits (description, `allowed-tools`, `model`, …) don't change it.
  - Neither covers supporting files such as `references/` or `scripts/`.
  - In Claude Code an identical re-invocation adds only a note, so the content appears at most once per session. After compaction it is cut to 5,000 tokens (§2.4).
  - Both hashes depend on MLflow storing full LLM input messages, which is how the transcript-based integrations work today. It is not a field MLflow designates for this.
- **Versions in telemetry today.**
  - Claude Code's `plugin_loaded.plugin.version` is exported, but as a log event that MLflow doesn't ingest. It is per session, not per invocation, and is withheld when the plugin name is redacted.
  - Codex emits no version anywhere in skill telemetry.
- **The client already has the data.** `installed_plugins.json` records `version`, `gitCommitSha` and `installPath`, and Codex has the plugin manifest on disk. Neither tool attaches these to a skill invocation in any signal MLflow currently receives.

## Not verified

- The exact `PreToolUse`/`PostToolUse` payload for the Skill tool, which isn't in the documented per-tool schemas. One way to check: a hook that dumps stdin during a model-initiated Skill call.
- Whether Claude Code's detailed-beta `new_context` actually contains the injected skill message.
- Whether the Claude Agent SDK message stream delivers the `isMeta` skill message to `liveTracing.ts`, and whether a stored MLflow trace actually contains the skill body in LLM span inputs. That last point is inferred from code for both the Claude Code and Codex plugins.
- The meaning of the `<uuid>_<uuid>` directory for claude.ai-synced skills.
