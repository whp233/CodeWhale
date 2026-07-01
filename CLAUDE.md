# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Source of Truth

Read `AGENTS.md` first. It contains branch discipline, build/test commands, stewardship rules, contribution guidelines, and the removed-machinery guardrails. This file adds the architectural context that `AGENTS.md` does not cover.

## Project

CodeWhale — a terminal coding agent (TUI + CLI) for LLM-powered coding. Formerly `deepseek-tui`. Rust (Edition 2024, MSRV 1.88), MIT, version **0.8.65**. `Hmbown/CodeWhale` on GitHub.

## Build & Test

```bash
# Build all
cargo build

# Release binaries (CLI + TUI)
cargo build --release -p codewhale-cli -p codewhale-tui

# Run the TUI
cargo run --bin codewhale

# Targeted test (crate + binary + optional filter)
cargo test -p codewhale-tui --bin codewhale-tui --locked <filter>

# Full workspace gate
cargo test --workspace

# Lint gate
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

Pre-push checklist: `cargo fmt`, targeted tests, then `cargo test --workspace` before claiming done.

**Known flaky tests** (pre-existing, not regressions):
- `config_command_allow_shell_*` — fails when `~/.codewhale/settings.toml` sets `default_mode = "yolo"`
- `run_verifiers_background_*` — flaky under full-suite parallelism, passes in isolation

## Architecture

The live runtime and most TUI/engine/tool code live in `crates/tui/src/`. Other workspace crates are being extracted incrementally and are not yet the sole source of truth for runtime behavior.

### Workspace Crates

| Crate | Role |
|-------|------|
| `tui` | Live end-user runtime — ratatui TUI, runtime API, task manager, tool execution loop, prompt assembly, LSP post-edit diagnostics |
| `cli` | CLI dispatcher facade — produces `codewhale` and `coc` (legacy shim) binaries |
| `core` | Agent loop, session/turn management, capacity flow guardrails |
| `app-server` | HTTP/SSE + JSON-RPC transport for headless agent workflows (library-only) |
| `config` | Config schema, loading, precedence model, profiles, env vars |
| `protocol` | Request/response framing and protocol types |
| `tools` | Tool invocation lifecycle, schema validation, scheduler parallelism |
| `mcp` | MCP server lifecycle and tool proxy compatibility |
| `hooks` | Lifecycle hook dispatch (stdout/jsonl/webhook) |
| `execpolicy` | Approval/sandbox policy engine |
| `agent` | Model/provider registry and fallback strategy (`RouteResolver` → endpoint + protocol + model + context limit + cost) |
| `secrets` | OS keyring storage (Keychain / Credential Manager / libsecret) with file fallback |
| `state` | SQLite thread/session persistence and recovery |
| `release` | Shared release discovery and version comparison |
| `whaleflow` | Typed WhaleFlow workflow IR and validation (Starlark on non-ohos targets) |

### Key Subsystems

**Prompt assembly** — `crates/tui/src/prompts/` contains `constitution.md` (compile-time embedded via `include_str!()`) and mode-specific files. The constitution uses a four-tier hierarchy (Constitution > Statutes > Regulations > Evidence) with XML-tagged behavioral statutes. Template placeholders like `{subagent_economics}` are substituted at assembly time — do not break them.

**Sub-agent surface** — single `agent` tool only. No swarm, no lifecycle tools, no runtime prompt injection. Seven types: `General`, `Explore`, `Plan`, `Review`, `Implementer`, `Verifier`, `Custom`. Tool restrictions were deprecated in v0.6.6 — types are advisory roles.

**LSP post-edit diagnostics** — wired into the engine's post-tool-execution path (`core/engine/lsp_hooks.rs`). After every `edit_file`/`apply_patch`/`write_file`, diagnostics are collected and flushed as a synthetic user message before the next API request.

**Side-git snapshots** — `/restore` uses git snapshots outside the user's `.git` under `~/.codewhale/snapshots/<project_hash>/<worktree_hash>/.git`.

**Platform sandboxing** — Seatbelt (macOS), Landlock + seccomp (Linux), Job Objects (Windows, not yet fully enforced). Not advertised as complete on non-macOS.

### Data Flow

1. User input → TUI → `core/engine.rs`
2. Engine sends to LLM via `llm_client.rs` (DeepSeek Chat Completions API)
3. Response streamed, tool calls extracted
4. Pre-hooks → approval (non-yolo) → tool execution → post-hooks
5. If file edit + LSP enabled: post-edit diagnostics collected
6. Results aggregated, sent back to LLM
7. Final response rendered in TUI

Crash recovery: pre-turn checkpoint to `~/.codewhale/sessions/checkpoints/latest.json`. Offline queue persists to `offline_queue.json`. Side-git snapshots for agent/yolo turns.

## Guardrails

- **Trust-boundary surfaces** (auth, sandbox, publishing, branding, global prompts) require maintainer sign-off before PRs targeting them
- **Never commit directly to `main`** — work on a feature branch, open a PR
- **Sub-agent surface is `agent` only** — do not reintroduce removed machinery (swarm, lifecycle, coherence, runtime injection)
- **TUI freeze** (v0.8.61 cap-20 cutover) is resolved — do not commit speculative `spawn_blocking` fixes
- **Version bumps and releases** require Hunter's explicit approval

## Quick Reference

| Need | File/Dir |
|------|----------|
| System prompt (constitution) | `crates/tui/src/prompts/constitution.md` |
| Tool implementations | `crates/tui/src/tools/` |
| Sub-agent logic | `crates/tools/subagent/` |
| Prompt assembly | `crates/tui/src/prompts.rs` |
| Engine loop | `crates/tui/src/core/engine.rs` |
| Architecture doc | `docs/ARCHITECTURE.md` |
| Agent stewardship | `docs/AGENT_ETHOS.md` |
| Config example | `config.example.toml` |
| Contributor credit map | `.github/AUTHOR_MAP` |
