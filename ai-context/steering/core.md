# Project: Claw Code

## Version control note

The repository is a single git tree rooted at the workspace root (`.git/` at the top level). Unlike monorepos that nest independent git repos, all subdirectories — including `ai-context/`, `rust/`, `src/`, `tests/`, and `docs/` — are tracked by the same outer git repo. Upstream is `ultraworkers/claw-code` on GitHub.

## On-demand docs

Analysis documents and reference materials are in `ai-context/docs/`. These are NOT auto-loaded — only read them when the user explicitly asks or when relevant to the current task.

When generating markdown documents or analysis files, always place them in `ai-context/docs/`. Follow the existing numbering format for filenames, e.g. `001.xxxx.md` or `001.001.xxxx.md`. Before allocating a new series number, you MUST scan the target directory for existing numbered files to identify all series numbers already in use, then pick the next unused number. Never reuse an existing series number for a different topic.

## Q&A directory

`ai-context/qa/` is for user-initiated Q&A interactions with AI.

- The user creates prompt files with names like `001.xxxx.q.md`, `001.q.md`, or simply `001.md`.
- When the user asks you to respond to a prompt file:
  1. Read the prompt content, summarize a concise topic name in Chinese or English.
  2. Rename the prompt file to `NNN.TopicName.q.md`.
  3. Generate the response file as `NNN.TopicName.a.md`.
  - Example: user creates `001.md` or `001.q.md` → after responding, both files become `001.SessionBoot.q.md` and `001.SessionBoot.a.md`.
- These files are NOT auto-loaded — only read them when the user explicitly asks.

# Core Project Context

> This project is normally opened as a single workspace folder rooted at the repository root. There is no checked-in `.code-workspace` file, so all paths are repo-relative from the top-level directory and the AI sees the same path layout the filesystem has.
>
> Paths in this document are repository-relative (e.g. `rust/crates/runtime/src/lib.rs`).

## Project Overview

Claw Code is the public Rust implementation of the `claw` CLI agent harness — an autonomous coding-agent runtime that drives LLM-backed development loops. The canonical implementation lives in `rust/`, and the binary it produces is named `claw`.

The project's stated goal (see `ROADMAP.md`, `PHILOSOPHY.md`) is to be the most "clawable" coding harness: deterministic to start, machine-readable in state and failure modes, recoverable without a human watching the terminal, and event-first rather than log-first. Humans set direction via Discord/chat; claws (autonomous agents) coordinate, build, test, recover, and push.

The repo is build-from-source only (the `claw-code` crate on crates.io is a deprecated stub). The upstream binary published to crates.io as a separate package is `agent-code` (binary name `agent`). Claude subscription login is **not** a supported auth path — `claw` requires an API key (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `XAI_API_KEY`, `DASHSCOPE_API_KEY`) or an OAuth bearer token (`ANTHROPIC_AUTH_TOKEN`).

## Repository Layout

- **`rust/`** — Canonical Rust workspace and the `claw` CLI binary. Cargo workspace at `rust/Cargo.toml` (edition 2021, resolver 2). All real runtime/CLI code lives under `rust/crates/`.
- **`src/`** — Python porting/reference workspace. CLI entrypoint at `src/main.py`. Provides parity-audit harnesses, command/tool snapshots, and a mirrored runtime/session/setup model used to cross-check the Rust port. Modules include `runtime.py`, `query_engine.py`, `commands.py`, `tools.py`, `session_store.py`, `parity_audit.py`, `port_manifest.py`, `system_init.py`. Reference snapshots in `src/reference_data/` (`commands_snapshot.json`, `tools_snapshot.json`, `archive_surface_snapshot.json`, plus per-subsystem JSON under `src/reference_data/subsystems/`).
- **`tests/`** — Python validation surface. Single file `test_porting_workspace.py`. Run with `python3 -m unittest discover -s tests -v`.
- **`docs/`** — User-facing documentation. `MODEL_COMPATIBILITY.md` documents Kimi `is_error` exclusion, reasoning-model param stripping, GPT-5 `max_completion_tokens`, and Qwen/DashScope routing. `container.md` documents the Docker/Podman bind-mount workflow.
- **`ai-context/`** — AI context and steering. `ai-context/steering/core.md` (this file) and `ai-context/steering/coretemp.md` (template reference). `ai-context/docs/` and `ai-context/qa/` follow the conventions above.
- **`assets/`** — Branding assets (`claw-hero.jpeg`).
- **`.github/`** — GitHub config. `workflows/rust-ci.yml` runs `cargo fmt --all --check` and `cargo test -p rusty-claude-cli` on `ubuntu-latest`. `workflows/release.yml` handles release publishing. `FUNDING.yml` for sponsorship config.
- **`Containerfile`** — Reusable Rust build/test image (FROM `rust:bookworm`). Bind-mount the repo into `/workspace`; do not bake source into the image. Used by both Docker and Podman per `docs/container.md`.
- **`install.sh`** — Linux/macOS/WSL installer. Detects host OS, verifies the Rust toolchain, builds the `claw` binary, runs a post-install verification step. Flags: `--release`, `--no-verify`. Env overrides: `CLAW_BUILD_PROFILE`, `CLAW_SKIP_VERIFY`.
- **Top-level docs** — `README.md` (entry point + Windows setup), `USAGE.md` (task-oriented usage guide, slash commands, providers), `PARITY.md` (Rust-port 9-lane checkpoint + tool surface parity matrix), `ROADMAP.md` ("clawable" harness roadmap; very large file), `PHILOSOPHY.md` (project intent — humans direct, claws execute), `CLAUDE.md` (per-repo guidance for upstream Claude Code), `progress.txt` (Ralph iteration progress notes).
- **Top-level config** — `.claw.json` (user aliases), `.claude.json` (default permission mode `dontAsk`), `.gitignore`, `prd.json` (product requirements driving roadmap iterations).

## Rust Workspace Architecture

Cargo workspace at `rust/Cargo.toml` with members `crates/*`. Workspace lints forbid `unsafe_code` and warn at clippy `all` + `pedantic` levels, with `module_name_repetitions`, `missing_panics_doc`, and `missing_errors_doc` allowed. Workspace edition is 2021, license MIT, `publish = false`.

Nine crates, each with a single responsibility:

- **`api`** — Provider clients and HTTP plumbing. Anthropic Messages API client, OpenAI-compatible client (covers OpenAI, OpenRouter, Ollama, DashScope, Kimi), xAI client. SSE streaming, request/response types, prompt cache, proxy config (`build_http_client_with`, `ProxyConfig`). Model alias resolution (`resolve_model_alias`), provider detection (`detect_provider_kind`). Re-exports `telemetry` types.
- **`commands`** — Slash command registry and parsing. `slash_command_specs()`, `validate_slash_command_input()`, help rendering, JSON/text command rendering. Drives `/skills`, `/agents`, `/mcp`, `/plugin`, `/doctor`, `/subagent`, etc. (See `USAGE.md` for the full slash-command surface.)
- **`compat-harness`** — TS manifest extraction harness. `extract_manifest()` over `UpstreamPaths` to pull tool/prompt manifests from upstream TypeScript source for parity checks.
- **`mock-anthropic-service`** — Deterministic local Anthropic-compatible mock service. Backs the `mock_parity_harness` integration tests and the manual `cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0` workflow.
- **`plugins`** — Plugin metadata, `PluginManager`, install/enable/disable/uninstall flows, hook integration. Bundled plugins under `crates/plugins/bundled/` (`example-bundled/`, `sample-hooks/`).
- **`runtime`** — Largest crate. Owns sessions, config, permissions, prompt assembly, MCP, conversation loop, file ops, bash, hooks, sandbox, OAuth, lane events, task/team/cron registries, plugin lifecycle, policy engine, recovery recipes, stale-branch detection, worker boot. Public modules include `bash_validation`, `branch_lock`, `config_validate`, `green_contract`, `lsp_client`, `mcp_lifecycle_hardened`, `mcp_server`, `mcp_tool_bridge`, `permission_enforcer`, `plugin_lifecycle`, `recovery_recipes`, `sandbox`, `session_control`, `stale_base`, `stale_branch`, `summary_compression`, `task_packet`, `task_registry`, `team_cron_registry`, `worker_boot`. Re-exported public surface is documented in `crates/runtime/src/lib.rs`.
- **`rusty-claude-cli`** — The CLI binary `claw` (`[[bin]] name = "claw"`, `path = "src/main.rs"`). REPL loop (rustyline), one-shot prompt mode, init scaffolding, markdown rendering with syntax highlighting (`pulldown-cmark` + `syntect`), `crossterm` terminal control, tool-call display. Sub-modules: `init.rs`, `input.rs`, `render.rs`, `main.rs` (currently monolithic, ~13K LOC; see `rust/TUI-ENHANCEMENT-PLAN.md`).
- **`telemetry`** — Session tracing types: `SessionTracer`, `SessionTraceRecord`, `TelemetryEvent`, `TelemetrySink` (`JsonlTelemetrySink`, `MemoryTelemetrySink`), `AnalyticsEvent`, `AnthropicRequestProfile`, `ClientIdentity`. Constant `DEFAULT_ANTHROPIC_VERSION`.
- **`tools`** — All built-in tools and tool dispatch. `mvp_tool_specs()` exposes 40 tool specs; `execute_tool()` is the central dispatcher. Tools include Bash, ReadFile, WriteFile, EditFile, GlobSearch, GrepSearch, WebSearch, WebFetch, Agent, TodoWrite, NotebookEdit, Skill, ToolSearch, Task*, Team*, Cron*, LSP, MCP, ListMcpResources, ReadMcpResource, McpAuth, AskUserQuestion, RemoteTrigger, Sleep, SendUserMessage/Brief, Config, EnterPlanMode, ExitPlanMode, StructuredOutput, REPL, PowerShell, TestingPermission. PDF extraction in `pdf_extract.rs`, lane completion in `lane_completion.rs`.

## Project Dependencies

Crate dependency graph (read top-down; arrows point at the dependency):

```
rusty-claude-cli ──┬──> api
                   ├──> commands
                   ├──> compat-harness
                   ├──> runtime
                   ├──> plugins
                   └──> tools

tools ─────────────┬──> api
                   ├──> commands
                   ├──> plugins
                   └──> runtime

api ───────────────┬──> runtime
                   └──> telemetry

runtime ───────────┬──> plugins
                   └──> telemetry

mock-anthropic-service  (no internal deps; used only as dev-dependency by rusty-claude-cli)
plugins, commands, compat-harness, telemetry  (no internal deps)
```

External crate highlights:

- HTTP: `reqwest 0.12` (rustls-tls, blocking + async).
- Async runtime: `tokio 1` (selectively-featured per crate).
- Serde: `serde 1` + `serde_json 1` (workspace-pinned).
- TUI: `crossterm 0.28`, `rustyline 15`, `pulldown-cmark 0.13`, `syntect 5`.
- Misc: `regex`, `glob`, `walkdir`, `sha2`, `flate2`, `criterion` (api benches).

## Key Technical Details

- **Toolchain**: Rust stable, edition 2021. Workspace lints forbid `unsafe_code`. CI is `cargo fmt --all --check` + `cargo test -p rusty-claude-cli` on `ubuntu-latest` (`.github/workflows/rust-ci.yml`).
- **Verification triple**: `cargo fmt && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace`. Run from `rust/`. Container-based equivalent in `docs/container.md`.
- **Binary**: `claw` (Windows: `claw.exe`). Default model `claude-opus-4-6`. Default permission mode `danger-full-access` (the runtime's permissive default; user `.claude.json` may set `defaultMode: dontAsk`).
- **Permission modes**: `read-only`, `workspace-write`, `danger-full-access`. Enforcement in `runtime::permission_enforcer` (`PermissionEnforcer::check`, `check_file_write`, `check_bash`).
- **Model aliases** (built-in): `opus` → `claude-opus-4-6`, `sonnet` → `claude-sonnet-4-6`, `haiku` → `claude-haiku-4-5-20251213`, `grok`/`grok-3` → `grok-3`, `grok-mini`/`grok-3-mini` → `grok-3-mini`, `grok-2`. User-defined aliases live under the `aliases` key in any settings file.
- **Provider routing**: prefix-based — `claude*` → Anthropic, `grok*` → xAI, `qwen/*` or `qwen-*` → DashScope, `openai/*` or `gpt-*` → OpenAI-compatible. Otherwise, falls back by ambient env var (`ANTHROPIC_API_KEY` → `OPENAI_API_KEY` → `XAI_API_KEY`), defaulting to Anthropic.
- **Auth env vars**: `ANTHROPIC_API_KEY` (header `x-api-key`), `ANTHROPIC_AUTH_TOKEN` (header `Authorization: Bearer`), `OPENAI_API_KEY` (+ `OPENAI_BASE_URL`), `XAI_API_KEY` (+ `XAI_BASE_URL`), `DASHSCOPE_API_KEY` (+ `DASHSCOPE_BASE_URL`). API-key vs bearer-token is **not** interchangeable — wrong-slot use returns 401, and the runtime detects the `sk-ant-*`-in-Bearer case and appends a fix hint.
- **Proxy support**: standard `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` (case-insensitive), plus programmatic `ProxyConfig::proxy_url` as a unified override.
- **Config file resolution order** (later overrides earlier): `~/.claw.json` → `~/.config/claw/settings.json` → `<repo>/.claw.json` → `<repo>/.claw/settings.json` → `<repo>/.claw/settings.local.json`. Loaded by `runtime::ConfigLoader::discover()`.
- **Sessions and worker state**: REPL turns persist under `<repo>/.claw/sessions/` (per turn). `<repo>/.claw/worker-state.json` tracks the most recent worker (read by `claw state`). Both are gitignored. Resume via `claw --resume latest|<session-id>|<path>`.
- **MCP**: full lifecycle in `runtime::mcp_stdio` (largest single file in the runtime crate, ~110 KB), bridged to the tool surface via `runtime::mcp_tool_bridge::McpToolRegistry`. SDK/stdio/remote/managed-proxy/WebSocket transports are all configurable.
- **Mock parity harness**: deterministic Anthropic-compatible mock service in `crates/mock-anthropic-service`, exercised by the clean-environment harness in `crates/rusty-claude-cli/tests/mock_parity_harness.rs`. 10 scripted scenarios (`streaming_text`, `read_file_roundtrip`, `grep_chunk_assembly`, `write_file_allowed`, `write_file_denied`, `multi_tool_turn_roundtrip`, `bash_stdout_roundtrip`, `bash_permission_prompt_approved`, `bash_permission_prompt_denied`, `plugin_tool_roundtrip`). Run via `rust/scripts/run_mock_parity_harness.sh`. Scenario manifest in `rust/mock_parity_scenarios.json`. Behavioral diff runner: `rust/scripts/run_mock_parity_diff.py`.
- **Container workflow**: `Containerfile` at repo root provides a Debian/Rust dev image. Recommended invocation bind-mounts the repo into `/workspace` and points `CARGO_TARGET_DIR` at `/tmp/claw-target` to keep build artifacts off the host. Container-aware sandbox detection lives in `runtime::sandbox::detect_container_environment` (looks for `/.dockerenv`, `/run/.containerenv`, `/proc/1/cgroup` markers, env hints).
- **ACP / Zed**: `claw acp` is a discoverability-only status surface as of April 2026. `claw acp serve` is currently an alias to that status, not a real protocol entrypoint. Real ACP support remains tracked in `ROADMAP.md`.
- **Codex naming caveat**: in this repo "codex" never refers to OpenAI Codex. `oh-my-codex` (OmX) is the workflow layer, `.codex/` directories are legacy lookup paths scanned alongside `.claw/`, and `CODEX_HOME` is an optional override for user-level skills/commands. The CLI does not import or export OpenAI Codex sessions.
- **Python reference workspace** (`src/` + `tests/`): not the runtime surface. Used to keep generated guidance and parity audits aligned with the Rust implementation. `CLAUDE.md` calls out that `src/` and `tests/` should be updated together when behavior changes.

## Domain Terms

- **claw** — An autonomous coding agent driven by `claw-code`. Plural usage ("claws coordinate, build, test...") refers to multi-agent execution.
- **clawable** — A harness property: deterministic to start, machine-readable in state and failure modes, recoverable without human babysitting, event-first.
- **clawhip** — Sibling repo (`Yeachan-Heo/clawhip`); an event/notification router that watches git, tmux, GitHub issues/PRs, and agent lifecycle events, keeping monitoring out of the agent context window. Local artifacts in `.clawhip/` (gitignored).
- **lobster** — Project mascot/branding equivalent to "claw" in some places (see emoji 🦞 in `rust/README.md`).
- **OmX** — `oh-my-codex` workflow layer (planning keywords, execution modes, parallel multi-agent workflows). Local artifacts in `.omx/` (gitignored).
- **OmO** — `oh-my-openagent`; multi-agent coordination layer (planning, handoffs, disagreement resolution, verification loops).
- **MCP** — Model Context Protocol. Servers are modeled as `McpServerSpec` / `McpServer`; transports include stdio, SDK, remote, managed-proxy, and WebSocket.
- **Lane** — A workstream in the autonomous coding pipeline. Lane events (`runtime::lane_events`) are the canonical machine-readable status format with `EventProvenance`, `LaneEventMetadata`, and terminal-event deduplication.
- **Worker** — A single agent execution unit (`runtime::worker_boot::Worker`, tracked in a `WorkerRegistry`). Has lifecycle states, ready snapshots, failure classifications, and trust resolution.
- **Permission mode** — `read-only` / `workspace-write` / `danger-full-access`. Drives `PermissionEnforcer` and tool gating.
- **Skill / Agent / Plugin** — Three discovery surfaces. Skills and agents are content-defined extensions (lookup paths under `.claw/`, `.codex/`, and `CODEX_HOME`); plugins are managed by `PluginManager` with install/enable/disable/uninstall flows.
- **Session** — A persisted REPL conversation under `.claw/sessions/`. Resume by id or path; `latest` resolves to the most recent.
- **Stale branch / stale base** — Detection that a side branch has missed already-landed `main` fixes. Policy lives in `runtime::stale_branch::StaleBranchPolicy` (`AutoRebase`, `AutoMergeForward`, `WarnOnly`, `Block`).
- **Recovery recipe** — Structured recovery flow in `runtime::recovery_recipes` (`FailureScenario`, `RecoveryRecipe`, `RecoveryLedger`, `attempt_recovery`).
- **Parity harness** — The mock-LLM-driven scripted scenarios that validate Rust-port behavior against upstream Claude Code.
