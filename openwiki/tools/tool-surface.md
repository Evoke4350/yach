---
type: reference
title: Tool surface
description: The seven tools YACH gives the model (bash, read_file, write_file, edit_file, grep, glob, ask_user), how their inputs are validated, how paths are confined to the project, how output is bounded, and how to add a tool.
tags: [tools, tool-use, validation, path-confinement, security]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T00:49:17.948Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-07T00:49:17.948Z" }
---

# Tool surface

YACH declares its tools in `TOOLS` and maps each name to a Python function in
`HANDLERS`. `Agent.run_tool` connects them: validate, approve, run, bound the
output, and return a `tool_result` ([Agent loop](../architecture/agent-loop.md)).

## The tools

| Tool | Handler | Approval | Behavior |
|---|---|---|---|
| `bash` | `run_bash` | Yes | Runs a command in the project root (or the sandbox); each call is a fresh shell, so `cd` and env vars don't persist. Optional `timeout`, default 120 s, clamped to 1-1800. See [Docker sandbox and command execution](../safety/sandbox.md). |
| `read_file` | `read_file` | Only outside the project | Numbered lines (`%6d\t`), `offset` (1-based) and `limit` (default 2,000); lines cut at 2,000 chars; says how many lines remain. |
| `write_file` | `write_file` | Yes | Creates parent directories, writes full contents; refuses paths outside the project. |
| `edit_file` | `edit_file` | Yes | Exact string replace; fails if `old_str` is missing, or appears more than once without `replace_all`. Refuses paths outside the project. |
| `grep` | `grep` | Only outside the project | ripgrep with line numbers, no headings, 300-column cap, optional `--glob` filter. |
| `glob` | `glob_files` | No | Relative patterns only; up to 500 files inside the project, newest first, skipping `.git`. |
| `ask_user` | `ask_user` | No | Asks the user a question with optional numbered suggestions. |

Approval rules are in [Approvals, auto mode and dcg](../safety/approvals-and-dcg.md).

## Input validation

Every tool definition sets `eager_input_streaming: true`, so large inputs such
as file contents stream as they're generated. The trade-off is that the server
no longer validates them. `validate_input` does it locally before anything
runs. It checks that:

- the tool exists and the input is an object;
- required fields are present and there are no unknown fields;
- each value's JSON type matches the schema, rejecting booleans where an
  integer is expected (`bool` is a subclass of `int` in Python);
- arrays contain only items of the declared type.

A failure becomes an `INVALID_INPUT` error result that includes the received
input, so the model can correct itself. Truncated inputs from a `max_tokens`
stop are caught earlier, in `send`.

## Path confinement

`resolve(path)` expands `~`, joins the path to `ROOT` and resolves it,
following symlinks and `..`:

- **Writes** (`write_file`, `edit_file`) outside the project root raise a
  `ToolError`, in every approval mode.
- **With the sandbox on**, any path outside the root is refused, because the
  container can't see it either.
- **Without the sandbox**, reads outside the root are allowed but need
  approval; `outside_root()` flags them for `Permissions`.

`glob` is confined differently: it rejects empty, absolute and `..` patterns,
and drops any match whose resolved path leaves the root (for example through a
symlink).

## Hardening

- **grep option injection:** the model's pattern is passed as `-e <pattern>`
  and options are ended with `--` before the path, so a pattern such as
  `--pre=bash`, which would make ripgrep run a command on every file, is only
  ever searched for as text.
- **Crashes:** `run_tool` catches every handler exception except Ctrl-C and
  returns it as an error result instead of crashing the session (for example
  a NUL byte in a path). `ToolError` and `OSError` messages are passed through
  as-is; other exceptions are prefixed with their type name.

## Output

- **Bounded:** results over 30,000 characters (`MAX_TOOL_OUTPUT`) keep the
  first and last 15,000 with a note of how much was cut, so the model sees both
  the start and the end of long output.
- **Cleaned:** bash output has terminal escape sequences and control characters
  removed before it reaches the model.
- **Preview:** the user sees the target path (or pattern) in bold before the
  call, and the first three lines of output afterwards. `bash` shows its command
  in the approval preview instead, and `ask_user` output isn't echoed because
  the user just typed it.

## `ask_user`

It prints the question and numbered options, all passed through `clean()`, and
reads an answer. A number picks that option; anything else is returned as
typed; an empty answer tells the model to use its best judgment. When stdin
isn't a terminal (one-shot runs with piped input), it returns an error telling
the model no user is available. Ctrl-C at the question cancels the turn.

## Adding a tool

1. Add a `_tool(name, description, properties, required)` entry to `TOOLS`.
   Only `string`, `integer`, `boolean` and `array` property types are
   supported by `validate_input` (`JSON_TYPES`).
2. Write the handler and register it in `HANDLERS`. Raise `ToolError` for
   expected failures.
3. Decide on approval. Add the name to `NO_APPROVAL` only for side-effect-free
   tools, and extend `outside_root` / `describe_action` / `Permissions.preview`
   if it touches paths.
4. Adding a tool changes the request prefix, so the first request after an
   upgrade misses the prompt cache, and resumed sessions drop older thinking
   blocks.

## Related

- [Approvals, auto mode and dcg](../safety/approvals-and-dcg.md)
- [Docker sandbox and command execution](../safety/sandbox.md)
- [Agent loop](../architecture/agent-loop.md)
