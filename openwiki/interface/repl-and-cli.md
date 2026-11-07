---
type: interface
title: Terminal interface and CLI
description: How YACH's main() parses flags, sets up the sandbox, dcg and resumed sessions, and runs the interactive prompt with slash commands, the ! shell passthrough, Ctrl-C handling, and safe terminal output.
tags: [cli, repl, terminal, slash-commands, shell-passthrough]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T00:49:17.948Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-07T00:49:17.948Z" }
---

# Terminal interface and CLI

Everything the user touches goes through `main()` at the bottom of `yach.py`.
It parses flags, prepares the session, sandbox and approval policy, then either
runs one prompt and exits or enters the interactive loop.

## Flags

| Flag | Default | Effect |
|---|---|---|
| `prompt...` | none | Run one prompt non-interactively, then exit |
| `--model` | `claude-opus-5-5` | Model for the main loop |
| `--effort` | `high` | `low` `medium` `high` `xhigh` `max`, sent as `output_config.effort` |
| `--approval` | `auto` | `ask`, `auto` or `yolo` ([Approvals](../safety/approvals-and-dcg.md)) |
| `--yolo` | | Shortcut for `--approval yolo` |
| `--sandbox` | off | Run shell commands in Docker ([Sandbox](../safety/sandbox.md)) |
| `--sandbox-network` | `bridge` | `none` disables the sandbox's network |
| `--no-dcg` | | Skip destructive_command_guard checks |
| `--compact-at` | `150000` | Auto-compact threshold in tokens; `0` disables it |
| `-c`, `--continue` | | Resume the most recent session for this project |
| `-r`, `--resume [ID]` | | Resume by ID, or pick from a list when no ID is given |

## Startup order

1. **Resume selection.** `-c` loads the newest session file, and exits with an
   error if there is none. `-r ID` loads that session, and exits if the file
   doesn't exist. Bare `-r` opens the picker, and exits if the user cancels.
2. **Sandbox.** `--sandbox` requires `docker` on `PATH` and starts the
   container before anything else runs. A failed `docker build` or `docker run`
   exits with a message.
3. **dcg detection.** The guard is on when `--no-dcg` isn't given and `dcg` is
   found on `PATH`. If it's missing, YACH prints a warning with the install
   link and continues without it.
4. **Agent.** An `Agent` is created with the chosen model, effort,
   `Permissions(mode, guard)` and compaction threshold.

With a positional prompt, YACH resumes quietly (if asked), calls `send` once
and returns. Otherwise it prints a banner showing the model, effort, approval
mode, sandbox and dcg status and the project root, then resumes the session
visibly, which prints the model's last reply.

## The prompt loop

Each line is stripped and dispatched:

- **`!<command>`**: run in the user's shell (below).
- **`/exit`, `/quit`**: leave the loop.
- **`/help`**: print the command list (`HELP`).
- **`/compact`**: summarize the conversation now.
- **`/context`**: show context used, the window size, a 30-cell bar, message
  count, the auto-compact setting and the number of compactions. If the size
  hasn't been measured yet, it counts tokens via the API first.
- **`/cost`**: show cumulative input, output, cache-write and cache-read
  tokens and the dollar total.
- **`/approval [mode]`**: show or switch the approval mode for the rest of the
  session.
- **`/clear`**: start a new session; the old one stays on disk.
- **`/resume`**: pick a saved session and switch to it.
- Anything else goes to `agent.send(text)`.

On exit YACH prints the session cost and, if the session was saved, the
`yach -r <id>` command to resume it.

### Errors and Ctrl-C

`send` is wrapped so API problems don't end the session. Rate limits, other
status errors and connection errors print a message and return to the prompt.
An authentication failure exits with a hint to set `ANTHROPIC_API_KEY`.

Ctrl-C has two meanings:

- **At the prompt** (and Ctrl-D), it quits.
- **During a turn**, it stops the turn and returns to the prompt. A partially
  streamed reply is discarded; the user message stays in history and the next
  message is simply appended after it.

## `!` shell passthrough

`run_user_shell` runs the command on the host with `shell=True` in the project
root. It never goes through the sandbox, approvals or dcg, because the user
typed it. stdin stays attached so interactive prompts work. Output is read in
raw 4 KB chunks rather than lines, so prompts without a trailing newline
appear, and is echoed live while also being captured. Ctrl-C lets the child
exit for up to 3 seconds, then kills it.

The captured output is passed through `clean()`, truncated, and added to the
conversation by `shell_context` as a user message saying which command ran and
its exit status (or that it was interrupted). The message content is a list of
blocks rather than a plain string on purpose: the auto-approval reviewer only
reads plain-string user messages as the user's intent, so pasted shell output
can't sway it. Before appending, the loop calls `session.repair()` so a dangling
tool call can't be followed directly by this message. The model sees the output
with the next request.

## Terminal output hygiene

- **Color** is on only when stdout is a TTY, `TERM` isn't `dumb` and
  `NO_COLOR` is unset; otherwise all color codes are empty strings.
- **`clean()`** strips CSI, OSC and two-byte escape sequences, then every
  remaining C0 control character except tab and newline. It is applied to
  model text, tool output, diffs, previews and shell output before display.
  Removing `\r` also flattens progress-bar redraws.
- **`prompt()`** wraps `input()` and marks color codes as non-printing
  (`\001`…`\002`) so readline measures the prompt width correctly.

## Spinner

`Spinner` draws a one-line indicator (braille frames, a label and elapsed
seconds) from a background thread every 100 ms. Each frame starts with
`\r\033[K`, so it overwrites itself in place. It does nothing when stdout isn't
a TTY. `stop()` joins the thread and erases the line, and callers must stop it
before printing anything else. A single global `SPINNER` is used for:

- waiting for the model
- streaming tool input
- the reviewer's check
- running shell commands

## Related

- [Architecture overview](../architecture/overview.md)
- [Approvals, auto mode and dcg](../safety/approvals-and-dcg.md)
- [Session storage and resume](../sessions/session-storage.md)
