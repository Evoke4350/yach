---
type: concept
title: Undo and checkpoints
description: How /undo restores the files a turn wrote or edited through write_file and edit_file, using an in-memory Checkpoint per turn; what it asks about, what it cannot reverse (shell commands), and how the model learns of the undo.
tags: [undo, checkpoints, safety, file-edits, rollback]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T01:26:12.508Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-10T01:26:12.508Z" }
---

# Undo and checkpoints

`/undo` puts back the files the most recent file-changing turn wrote or
edited, and deletes the files that turn created. It exists because auto mode
approves most edits without the user reading them closely; it is a cheap way
back when a turn goes wrong. It is not a replacement for git: it covers only
the two file tools and lives in memory.

## What is recorded

A `Checkpoint` (the `undo` section of `yach.py`, around L668-L732) holds what
one turn changed:

| Field | Meaning |
|---|---|
| `request` | The user message that started the turn, shown when undoing |
| `before` | Each written path mapped to its bytes before the turn; `None` means the file didn't exist |
| `written` | A SHA-256 digest of each file as YACH last left it, to detect later changes |
| `new_dirs` | The topmost directory each write had to create |
| `commands` | How many shell commands the turn ran |

`Agent.send` starts a new `Checkpoint` for every user message. `Agent.run_tool`
fills it in, after the call has been approved:

1. For `write_file` and `edit_file`, `Checkpoint.snapshot(path)` reads the
   file's current bytes and finds the topmost missing parent directory,
   *before* the handler runs.
2. If the handler succeeds, `record()` stores the snapshot. A failed or denied
   call records nothing.
3. The first recorded write adds the checkpoint to `Agent.checkpoints`, the
   undo stack. Turns that wrote no files never enter the stack.

`before` keeps the oldest version: if a turn writes the same file several
times, undo returns it to how it was when the turn began. Contents are handled
as bytes, so files that aren't valid text survive a round trip.

`bash` calls only increment `commands`. Nothing a shell command changes is
recorded.

## What `/undo` does

`Agent.undo` pops the newest checkpoint and calls `Checkpoint.undo`, which for
each file:

- skips it silently if it already matches its earlier contents;
- **asks first** if the file no longer matches what YACH last wrote, meaning
  the user or a shell command changed it afterwards (or deleted it). Declining,
  Ctrl-C or Ctrl-D leaves the file alone;
- otherwise deletes it if the turn created it, or writes the earlier bytes
  back, recreating parent directories if needed.

A file that was declined, or that failed with an `OSError`, stays in the
checkpoint, and the checkpoint goes back on the stack, so running `/undo`
again retries just those files. Directories the turn created are removed if
they are now empty.

Each `/undo` handles one turn; repeating it walks back through earlier turns
that changed files. When the turn ran shell commands, the output says how many
and that their effects were not undone.

## Telling the model

History is append-only ([Agent loop](../architecture/agent-loop.md)), so an
undo can't remove the turn from the conversation. Instead, when anything was
restored or deleted, `Agent.undo` appends a user message listing what was done,
stating that the files are back to their earlier state, and telling the model
to read a file again before editing it. If the turn ran shell commands, the
note adds that those were not undone.

The note is stored as a block list rather than a plain string, like `!` shell
output. The auto-mode reviewer only reads plain-string user messages, so the
note is never treated as a request
([Approvals, auto mode and dcg](../safety/approvals-and-dcg.md)).
`session.repair()` runs first so the note never follows an unanswered tool
call.

## Limits

- **Shell commands are not reversed.** Files changed by `bash` (formatters,
  code generators, `git`, `rm`) stay changed.
- **In memory only.** The stack is lost on exit, and is cleared by `/clear`
  and by resuming a session, since the checkpoints belong to the conversation
  being left.
- **File modes:** a file that was deleted after the turn and then restored is
  recreated with default permissions, so an executable bit is lost.
- **Empty-directory cleanup** can also remove an empty directory that
  something else created inside a directory the turn made.

## Related

- [Tool surface](../tools/tool-surface.md)
- [Agent loop](../architecture/agent-loop.md)
- [Terminal interface and CLI](../interface/repl-and-cli.md)
- [Approvals, auto mode and dcg](../safety/approvals-and-dcg.md)
