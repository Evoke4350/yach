---
type: concept
title: Session storage and resume
description: How YACH saves every conversation as an append-only JSONL file per project, recovers from crashes and dangling tool calls, lists sessions without touching them, and resumes with the right system prompt and thinking settings.
tags: [sessions, persistence, jsonl, resume, crash-recovery]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T01:26:12.508Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-10T01:26:12.508Z" }
---

# Session storage and resume

Every conversation is a `Session`: an in-memory list of API messages plus a
JSONL file that mirrors it. `Agent.messages` is just `session.messages`, and
`Agent.add_message` writes through to disk. No separate save step exists.

## Where and how it's stored

Files live in
`$XDG_DATA_HOME/yach/sessions/<project>/<id>.jsonl`, defaulting to
`~/.local/share/yach/sessions/...`. `<project>` is the project root path with
every non-alphanumeric character replaced by `-`, so each project has its own
list. Session IDs are `YYYYMMDD-HHMMSS-<4 hex>`.

Each file contains:

1. **A metadata line** written with the first entry: `yach_session` (format
   version 1), `id`, `root`, the **system prompt** the session was started
   with, and `created`. Files are created lazily, so a session with no messages
   leaves nothing on disk.
2. **One line per message**, exactly as sent to the API. Assistant messages are
   the response's blocks in wire form, including thinking blocks.
3. **Compaction markers**, `{"yach_compaction": {"at": ..., "replaced_messages": N}}`,
   each followed by the summary message
   ([Context measurement and compaction](context-and-compaction.md)).

Saving is a single append because the history is append-only: YACH never edits
or removes a message once written, except that compaction swaps the in-memory
list while the file keeps the full transcript. Replaying identical bytes is what
keeps preserved thinking and the prompt cache valid on resume.

Session files hold everything said in the conversation, including tool output,
so they should be treated as sensitive.

## Loading and crash recovery

`Session.load(id)`:

- Reads the system prompt from the metadata line.
- Parses lines until the first invalid one. A marker line clears the messages
  collected so far, so the result starts from the latest compaction summary.
- Records the first plain-string user message as the session's `title`, which
  survives compaction because it is read from the whole file.
- **Truncates a partial last line**, left when YACH was killed mid-write,
  together with anything after it, so new appends don't get glued onto broken
  JSON.
- Calls `repair()`.

`repair()` handles the other crash shape: the last message is an assistant
`tool_use` with no result. It appends an error `tool_result` for each pending
call ("Not run: the turn ended before this tool call ran.") so the history can
be sent again. Besides `load`, it runs at the start of every `Agent.send`,
after a refusal, and before a `!` shell message is added. A tool call can never
be followed by a plain user message, which the API would reject with a 400 on
every later request.

### Read-only loading for the picker

`pick_session()` lists the 20 most recently modified sessions with age,
message count and title, using `Session.load(..., readonly=True)`. Read-only
loading skips both truncation and repair, so browsing never modifies a file,
including one another running YACH is writing to. Only the session the user
chooses is then loaded normally.

## Resume flows

| Entry point | Behavior |
|---|---|
| `yach -c` | Load the most recently modified session; exit if none |
| `yach -r <id>` | Load that session; exit if its file doesn't exist |
| `yach -r` / `/resume` | Picker; cancel exits (CLI) or does nothing (REPL) |
| `/clear` | Start a fresh `Session`; the old one stays on disk |

`Agent.resume(session)` then:

- **Chooses the system prompt.** It reuses the saved prompt if it equals a
  freshly built one with the `Date:` line ignored, which preserves the cache
  and thinking blocks across days. Otherwise it uses the current prompt, so
  the model's description of its environment matches reality, for example
  when the sandbox was on when the session started and is off now.
- **Marks the session as resumed.** `thinking_config()` then adds
  `block_binding.prefix_mismatch_behavior: "drop_block"` (with the
  `thinking-binding-controls-2026-08-01` beta). The API drops replayed
  thinking blocks that no longer match their prefix, for example after a
  system prompt or tool-definition change, instead of failing the request.
- **Resets context measurement**, and unless quiet, prints the session ID,
  message count and the last few lines of the model's last reply.

Not carried over: the cost counters, "always allow" answers, and the approval
mode. Undo history is dropped as well: both `/clear` and a resume empty the
`/undo` stack ([Undo and checkpoints](../safety/undo.md)).

On exit, YACH prints `resume with: yach -r <id>` if the session was saved.

## Related

- [Agent loop](../architecture/agent-loop.md)
- [Context measurement and compaction](context-and-compaction.md)
- [Terminal interface and CLI](../interface/repl-and-cli.md)
- [Undo and checkpoints](../safety/undo.md)
