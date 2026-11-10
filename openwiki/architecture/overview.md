---
type: architecture
title: Architecture overview
description: YACH is a single-file Python coding agent on the Claude API; this page maps yach.py's sections, its main classes, global state, and how one user message flows through the model, approvals, and tools.
tags: [architecture, overview, single-file, python]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T01:26:12.508Z
sources:
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-10T01:26:12.508Z" }
---

# Architecture overview

YACH ("Yet Another Coding Harness") is a terminal coding agent written as one
Python module, `yach.py`, with one runtime dependency: the `anthropic` SDK
(`pyproject.toml`). The package exposes a single console script, `yach`, that
calls `yach:main`. Python 3.12+ is required.

## Layout of `yach.py`

The file is divided into sections by `# ----- <name>` banner comments, in
dependency order:

| Section | Contents |
|---|---|
| config | Model IDs (`DEFAULT_MODEL` Opus 5.5, `REVIEW_MODEL` Sonnet 5.5), beta flags, `PRICES`, `ROOT`, terminal colors, `clean()`, `prompt()`, `Spinner` |
| tools | `TOOLS` schemas, `validate_input`, `resolve`, `truncate`, the sandbox, and the tool handlers (`HANDLERS`) |
| sandbox | `Sandbox` (Docker) and `run_bash` |
| permissions | Reviewer prompt and schema, `review_action`, `dcg_check`, `Permissions`, `print_diff` |
| undo | `Checkpoint`: what one turn wrote, for `/undo` |
| agent | `build_system_prompt` and `sandbox_note` |
| sessions | `Session` persistence, the session picker |
| agent (class) | `Agent`: the model loop, compaction, cost and context accounting |
| repl | `HELP`, `!` shell passthrough, `main()` |

## Main objects

- **`Agent`**: owns the API client, the current `Session`, the system prompt,
  cost and token counters, and context measurements. `send()` runs a user turn;
  see [Agent loop](agent-loop.md).
- **`Session`**: the conversation's message list and its JSONL file on disk;
  see [Session storage and resume](../sessions/session-storage.md).
- **`Permissions`**: decides whether a tool call may run (ask, auto or yolo
  mode, plus dcg); see [Approvals, auto mode and dcg](../safety/approvals-and-dcg.md).
- **`Sandbox`**: an optional Docker container that runs shell commands; see
  [Docker sandbox and command execution](../safety/sandbox.md).
- **`Checkpoint`**: the files one turn wrote or edited and their earlier
  contents, kept in memory so `/undo` can restore them; see
  [Undo and checkpoints](../safety/undo.md).

## Global state

Two module-level values shape everything:

- **`ROOT`** is the current working directory, resolved once at import. It is
  the project root: file writes are confined to it, the sandbox mounts it,
  sessions are stored per `ROOT`, and the system prompt names it.
- **`SANDBOX`** is `None` unless `--sandbox` was passed, in which case `main()`
  assigns a running `Sandbox`. `run_bash`, `resolve` and `sandbox_note` all
  check it.

## How a message flows

1. `main()` reads a line at the `>` prompt. Lines starting with `!` or `/` are
   handled locally ([Terminal interface and CLI](../interface/repl-and-cli.md));
   anything else goes to `agent.send(text)`.
2. `send` appends the message to the session and calls `stream_turn`, which
   streams one request to the Claude API with the system prompt, all tools,
   and the full history.
3. For each `tool_use` in the reply, `run_tool` validates the input, asks
   `Permissions.allow` (which may consult dcg, the reviewer model, or the user),
   and then calls the handler in `HANDLERS`
   ([Tool surface](../tools/tool-surface.md)). File writes are recorded in the
   turn's `Checkpoint` first.
4. The results go back as one user message and the loop repeats until the
   model replies without tool calls.

## The system prompt

`build_system_prompt()` builds the prompt once per session from the project
root, platform, a sandbox description when the sandbox is on, today's date, a
short set of working rules, and the contents of `AGENTS.md` and `CLAUDE.md`
from the project root if they exist. It must stay byte-stable for the whole
session because it is the front of the cached prompt prefix. Sessions save the
prompt they were started with and reuse it on resume when only the date has
changed.

## Related

- [Agent loop](agent-loop.md)
- [Tool surface](../tools/tool-surface.md)
- [Approvals, auto mode and dcg](../safety/approvals-and-dcg.md)
- [Undo and checkpoints](../safety/undo.md)
- [Terminal interface and CLI](../interface/repl-and-cli.md)
