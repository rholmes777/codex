# Architecture Overview

This document gives a newcomer a tour of the major parts of the Codex codebase and how they interact.

## Repository Layout

```
/             – top level project files
codex-cli/    – TypeScript/React based CLI
codex-rs/     – Rust implementation of the CLI
scripts/      – helper scripts
```

## TypeScript CLI

`codex-cli` contains the Node based command line tool. The executable entry point is [`src/cli.tsx`](../codex-cli/src/cli.tsx).  It uses `meow` to parse options such as history browsing, approval policies and output control. Lines below show part of the flag definitions:

The CLI defines many flags such as `--history` to browse sessions, `--login` for sign in, `--quiet` to run non-interactively, and `--writable-root` or `--approval-mode` to control how changes are applied. Options also exist to automatically approve edits, disable response storage, enable flex mode, set reasoning effort, and skip confirmations entirely.
【F:codex-cli/src/cli.tsx†L81-L114】

The CLI supports a non‑interactive `--quiet` mode, a `--history` browser and other convenience flags.

### Terminal UI

The interactive interface is implemented with React components using Ink. `TerminalChat` is the main container. On start it instantiates an `AgentLoop` which manages calls to the model and tool execution:

The `TerminalChat` component's `useEffect` creates a new `AgentLoop` for each session. It passes the selected model, provider, configuration, and callbacks that record rollouts, update loading state, and request command confirmation from the user.
【F:codex-cli/src/components/chat/terminal-chat.tsx†L206-L271】

`TerminalChatInput` handles slash commands. Pressing `/compact` condenses the conversation to free context. The handler calls `generateCompactSummary` and replaces the message history:

The `/compact` command triggers a `handleCompact` function that calls `generateCompactSummary` to summarize the conversation. The UI then displays the summary or an error message and resets the loading state.
【F:codex-cli/src/components/chat/terminal-chat.tsx†L158-L188】

Slash commands are defined in [`utils/slash-commands.ts`](../codex-cli/src/utils/slash-commands.ts) and include options like `/history`, `/sessions` and `/compact`.

`SLASH_COMMANDS` describes the available slash commands such as `/clear`, `/history`, `/sessions`, `/compact`, and `/diff`. These open various panels or perform actions like showing diffs or clearing conversation history.
【F:codex-cli/src/utils/slash-commands.ts†L1-L27】

### Interrupting Tasks

`TerminalChatInput` exposes an `interruptAgent` prop which triggers `agent.cancel()`. When invoked the UI prints a system message to indicate the interruption:

The `interruptAgent` callback calls `agent.cancel()` to stop execution, updates loading state, and appends a system message informing the user that execution was interrupted.
【F:codex-cli/src/components/chat/terminal-chat.tsx†L550-L575】

### Session History and Resuming

All conversation turns are periodically written to `.codex/sessions`. Saving is handled in `save-rollout.ts`:

`saveRolloutAsync` writes session data to `.codex/sessions`. It creates the directory if needed, builds a timestamped filename, loads the config, and stores the conversation items alongside metadata.
【F:codex-cli/src/utils/storage/save-rollout.ts†L9-L34】

Users can browse or resume previous sessions via the Sessions overlay:

The sessions overlay loads past conversations using `loadSessions()` and formats them into a list shown in a typeahead menu. Keyboard input toggles between viewing and resuming a selected session.
【F:codex-cli/src/components/sessions-overlay.tsx†L82-L107】

When a session is chosen the CLI loads the stored rollout and can resume the run:

The CLI presents a `SessionsOverlay` allowing users to view or resume a previous rollout. After selection, it either loads the JSON file for display or restarts the agent from that session.
【F:codex-cli/src/cli.tsx†L456-L483】

## AgentLoop

`AgentLoop` encapsulates interaction with the model. It exposes `cancel()` for soft interruption and `terminate()` to permanently end a loop. Key portions of the implementation illustrate the behaviour:

The `AgentLoop` class provides `cancel()` to abort ongoing requests and `terminate()` to stop the loop entirely by calling `cancel()` after a hard abort.
【F:codex-cli/src/utils/agent/agent-loop.ts†L174-L205】【F:codex-cli/src/utils/agent/agent-loop.ts†L246-L257】

When `run()` is called the loop constructs input, handles any aborted tool calls and manages the transcript. It differentiates between server‑side and local memory modes:

When `run()` begins it increments a generation counter, resets cancellation state, prepares an abort controller and gathers any pending aborted tool outputs before building the list of messages to send to the model.
【F:codex-cli/src/utils/agent/agent-loop.ts†L556-L612】

## Git Integration

Codex checks if it is running inside a Git repository and can show diffs of pending changes. This is handled by utilities:

`checkInGit` detects whether the CLI is inside a Git repository by running `git rev-parse --is-inside-work-tree`.
【F:codex-cli/src/utils/check-in-git.ts†L1-L31】

`getGitDiff` returns the current diff by running Git commands for both tracked and untracked files.
【F:codex-cli/src/utils/get-diff.ts†L24-L40】

## Rust Implementation

The `codex-rs` directory houses a Rust version of the CLI. It follows a similar architecture with tasks processed by a core engine. The TUI in `tui/` communicates with the core via async channels. Interrupts are forwarded to the core so that long running tasks can be aborted. The Rust documentation in [`codex-rs/docs/protocol_v1.md`](../codex-rs/docs/protocol_v1.md) describes entities such as the model, codex engine and sessions.

## Memory Handling

Codex can compact its conversation history using `/compact` to preserve memory while keeping a summary. The summarization logic lives in `utils/compact-summary.ts`:

The `generateCompactSummary` function compiles recent user and assistant messages into a single text block and sends it to the model to obtain a short summary, returning the summary text for display.
【F:codex-cli/src/utils/compact-summary.ts†L1-L69】

## Summary

* **CLI entrypoint** – `src/cli.tsx` parses options and launches the React UI.
* **TerminalChat / AgentLoop** – main UI and execution loop; supports cancellations and auto approvals.
* **Slash Commands** – `/history`, `/sessions`, `/compact`, `/diff`, etc.
* **Sessions** – conversation rolls are saved to `.codex/sessions` allowing resume or inspection.
* **Git Integration** – utilities detect repositories and compute diffs.
* **Memory Compaction** – `/compact` condenses prior messages via a summarizing request.
* **Rust CLI** – parallel implementation using async channels with similar concepts.

