# apiari-codex-sdk

Rust SDK wrapping the OpenAI Codex CLI via JSONL stdout streaming.

## Quick Reference

```bash
cargo test -p apiari-codex-sdk
cargo test -p apiari-codex-sdk -- --ignored  # Integration tests (requires live `codex` CLI)
```

## Swarm Worker Rules

1. **You are working in a git worktree.** Always create a new branch (`swarm/*`), never commit directly to `main`.
2. **Only modify files within this repo (`codex-sdk/`).** Do not touch other repos in the workspace (e.g., `hive/`, `common/`, `swarm/`).
3. **When done, create a PR:**
   ```bash
   gh pr create --repo ApiariTools/apiari-codex-sdk --title "..." --body "..."
   ```
4. **Do not run `cargo install` or modify system state.** No global installs, no modifying dotfiles, no system-level changes.
5. **Plan+execute in one go without pausing.**

## Git Workflow

- You are working in a swarm worktree on a `swarm/*` branch. Stay on this branch.
- NEVER push to or merge into `main` directly.
- NEVER run `git push origin main` or `git checkout main`.
- When done, push your branch and open a PR. Swarm will handle merging.

## Architecture

```
src/
  lib.rs          # Module declarations + re-exports
  client.rs       # CodexClient (factory) + Execution (read-only handle)
  options.rs      # ExecOptions, ResumeOptions, SandboxMode, ApprovalPolicy
  transport.rs    # ReadOnlyTransport (spawn, recv-only, interrupt, kill)
  types.rs        # Event, Item variants, Usage, supporting types
  error.rs        # SdkError enum + Result alias
tests/
  integration.rs  # Live CLI tests (#[ignore] by default)
```

## Protocol

Spawns: `codex exec --json [opts...] <prompt>`

**Unidirectional**: stdin is `/dev/null`. The SDK only reads JSONL events from stdout.
No `send_message()` or `send_tool_result()` — codex handles tool execution internally.

### Event Types (codex -> stdout)

- `thread.started` — thread ID assigned
- `turn.started` / `turn.completed` / `turn.failed` — turn lifecycle
- `item.started` / `item.updated` / `item.completed` — item lifecycle
- `token_count` — token usage statistics
- `error` — execution error

### Item Types

- `agent_message` — model text response
- `reasoning` — thinking/reasoning text
- `command_execution` — shell command with output
- `file_change` — file modifications
- `mcp_tool_call` — MCP tool invocation
- `web_search` — web search query
- `todo_list` — task list
- `error` — item-level error

## Design Rules

- **Wrap CLI, not API.** This SDK spawns the `codex` binary. It does NOT call the OpenAI API directly.
- **Forward-compatible parsing.** Unknown event/item types deserialize as `Unknown` variant. Fields use `#[serde(default)]` liberally.
- **Async throughout.** All I/O uses tokio. Transport runs a background task to drain stderr.
- **No apiari-common dependency.** This crate is standalone.

## Error Handling

`SdkError` variants:
- `ProcessSpawn` — codex binary not found or failed to start
- `ProcessDied { exit_code, stderr }` — subprocess exited unexpectedly
- `InvalidJson` — NDJSON parse failure
- `ProtocolError` — unexpected protocol state
- `Timeout` — operation timed out
- `Io` — underlying I/O error
- `NotRunning` — execution already finished
