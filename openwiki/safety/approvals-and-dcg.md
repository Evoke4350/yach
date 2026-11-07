---
type: safety
title: Approvals, auto mode and dcg
description: How YACH decides whether a tool call may run, covering ask, auto and yolo modes, the reviewer model, destructive_command_guard checks that override every mode, approval for reads outside the project, and fail-closed defaults.
tags: [permissions, approvals, auto-mode, dcg, security]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T00:49:17.948Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-07T00:49:17.948Z" }
---

# Approvals, auto mode and dcg

Every tool call passes through `Permissions.allow(name, args, review)` before
its handler runs (called from `Agent.run_tool`). It returns
`(allowed, note)`; on denial, the note is appended to the error result the
model receives, so the model can adjust instead of retrying blindly.

## Decision order

`allow` checks, in this order:

1. **Free read-only tools.** `read_file`, `grep`, `glob` and `ask_user` run
   with no approval, *unless* `read_file` or `grep` targets a path outside the
   project root. Those go through approval like a write, because reading
   `~/.ssh`, `~/.aws` or another project's `.env` is a common exfiltration
   step. (`glob` can't leave the project; see [Tool surface](../tools/tool-surface.md).)
2. **Preview.** The action is shown before any decision:
   - shell commands in yellow;
   - a unified diff for edits and overwrites;
   - for a brand-new file, a line count and the first 12 lines (up to 200 in
     `ask` mode, where you're approving by hand);
   - an "outside the project" line for out-of-root reads.
3. **dcg** (shell commands only, when the guard is on). Anything dcg flags
   needs an explicit `y` at a red prompt, and nothing else can approve it:
   not `yolo`, not an "always" answer, not the reviewer. Declining sends dcg's
   reason and rule back to the model.
4. **`yolo` mode or "always".** Approve.
5. **`auto` mode.** Ask the reviewer model. `allow` runs immediately and
   prints the reason; `ask` falls through to the prompt with the reason shown.
6. **Prompt.** `[y]es / [n]o / [a]lways this session`. "Always" adds the tool
   name to the session's `always` set, so later calls to that tool skip the
   reviewer and the prompt, though not dcg. EOF (no terminal) means no.

The mode can be changed mid-session with `/approval`; "always" answers are not
saved with the session.

## The reviewer (auto mode)

`Agent.review` gathers context and calls `review_action`, which sends one
request to `REVIEW_MODEL` (`claude-sonnet-5-5`) with:

- **`REVIEW_SYSTEM`**: allow routine, in-scope work (inspecting, tests and
  builds, edits inside the project, installing declared dependencies, local
  git that doesn't discard work); ask for anything that:
  - deletes or discards data, or pushes, publishes or deploys;
  - touches secrets, `.env` files or SSH keys;
  - downloads and runs code, uses `sudo`, or changes system config;
  - edits CI config, git hooks or files outside the project;
  - goes beyond the request, or is obfuscated.

  "When in doubt, answer ask." The action text is declared to be data, not
  instructions.
- **The user's last three plain-string messages**, each cut to 2,000
  characters. Tool output, the agent's own text, and `!` shell output (stored
  as block lists) are excluded on purpose, so injected content in a file or
  webpage can't argue for its own approval.
- **The action** from `describe_action`: the command, the path plus up to
  4,000 characters of contents, both sides of an edit (marked "every
  occurrence" for `replace_all`), or the out-of-project read target.

The request uses `effort: low` and structured output constrained to
`{"decision": "allow"|"ask", "reason": ...}`. Its cost is added to the session
total at the reviewer model's prices.

### Fail-closed behavior

- A reviewer `stop_reason` other than `end_turn` (refusal, `max_tokens`)
  returns `ask`.
- API errors and unparseable output return `ask` with "reviewer unavailable".
- `dcg_check` treats a missing binary, a timeout (10 s) or invalid JSON as a
  flag, so the user is asked rather than the command running unchecked.

## dcg integration

`dcg_check` runs `dcg test --format json <command>` from the project root, so
project-level dcg allowlists and config apply. A `decision` of `allow` passes;
anything else returns dcg's reason with its `rule_id` and severity, which is
shown to the user and, on denial, to the model. `main()` turns the guard on
only when `dcg` is on `PATH` and `--no-dcg` wasn't given.

## Diffs

`print_diff` renders a unified diff with two lines of context and a 200-line
cap. Headers are bold, hunk markers cyan, additions green and removals red.
Every line goes through `clean()`. The edit preview applies the same
replacement count as `edit_file` (all occurrences for `replace_all`), so the
diff approved is the change written.

## Related

- [Tool surface](../tools/tool-surface.md)
- [Docker sandbox and command execution](sandbox.md)
- [Terminal interface and CLI](../interface/repl-and-cli.md)
