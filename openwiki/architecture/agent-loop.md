---
type: architecture
title: Agent loop
description: How Agent.send drives YACH's streaming tool-use loop against the Claude API, handles stop reasons, keeps history append-only, and accounts cost and context per turn.
tags: [agent-loop, streaming, tool-use, prompt-caching, claude-api]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T01:26:12.508Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-10T01:26:12.508Z" }
---

# Agent loop

The agent loop lives on the `Agent` class in `yach.py`. `Agent.send(user_text)`
runs one user turn to completion: it calls the model, executes any tools the
model asks for, feeds the results back, and repeats until the model stops
asking for tools. `Agent.stream_turn()` makes and renders exactly one model
call. `Agent.run_tool()` executes one tool call; see
[Tool surface](../tools/tool-surface.md) for the tools themselves.

## One turn, step by step

`send` (around `yach.py` L1232-L1303):

1. **Repair, checkpoint, then maybe compact.** It first calls
   `session.repair()` so a previous refused or interrupted turn can't leave an
   unanswered `tool_use` behind, and starts a fresh `Checkpoint` for the turn,
   which `run_tool` fills in as files are written
   ([Undo and checkpoints](../safety/undo.md)). Then, if the last measured context size has reached the
   `--compact-at` threshold, it compacts the conversation *before* adding the
   new message. Compaction never happens mid tool loop
   ([Context measurement and compaction](../sessions/context-and-compaction.md)).
2. **Append the user message** through `add_message`, which also persists it
   to the session file ([Session storage](../sessions/session-storage.md)).
3. **Loop:** call `stream_turn()`, append the assistant reply, account cost
   and context, then decide what to do from `stop_reason`.

### Stop-reason handling

| Outcome | What `send` does |
|---|---|
| `pause_turn` | Re-sends immediately (the reply is already appended). |
| `refusal` | Prints a notice, answers any `tool_use` blocks in the refused reply via `session.repair()` without running them, and ends the turn. |
| No `tool_use` blocks | Ends the turn; on `max_tokens` it tells the user to reply "continue". |
| `max_tokens` with `tool_use` | Never runs the tools: a truncated input can parse as a plausible partial object. Every call gets an error `tool_result` asking the model to split the work, and the loop continues. |
| Otherwise | Runs each tool in order via `run_tool`, then appends **all** results in a single user message so parallel tool use keeps working. |

Ctrl-C while tools are running stops the loop. Results already produced are
kept, every remaining `tool_use` gets an "Interrupted by the user." error
result, and those are appended before returning, so the history stays valid.

### Malformed tool JSON

Tool inputs are streamed eagerly (each tool definition sets
`eager_input_streaming: true`), so the server no longer validates them. If the
SDK can't parse the streamed JSON at all it raises `ValueError` from the stream;
`send` re-issues the turn, giving up after more than two consecutive failures.
Inputs that parse but are wrong are caught later by `validate_input` in
`run_tool` and returned to the model as an `INVALID_INPUT` error result.

## The model call

`stream_turn` (around L1080-L1143) sends one streaming request to
`client.beta.messages.stream` with:

- `model`, `max_tokens=64000`, the session's `system` prompt and the full
  `TOOLS` list;
- `thinking` from `thinking_config()`: adaptive thinking with
  `display: "summarized"`, plus `block_binding.prefix_mismatch_behavior:
  "drop_block"` (and its beta header) only for resumed sessions;
- `output_config.effort` from `--effort`;
- top-level `cache_control: {"type": "ephemeral"}`, so the cache breakpoint
  moves forward each turn and every request reads the previous prefix from
  cache;
- `fallbacks="default"` with the `server-side-fallback-2026-07-01` beta, so a
  safety-classifier refusal is retried server-side on a fallback model;
- the compaction beta when the history starts with a compaction summary
  (an assistant-role first message).

While streaming it prints summarized thinking in dim text and response text
normally, both passed through `clean()`. For tool calls it shows a spinner line
that names the tool, the target (`path`, `command`, `pattern` or `question`
from the partial input snapshot) and how many characters of input have
arrived, then replaces it with `● <tool>` when the block completes. A
"waiting for Claude" spinner covers the gap before the first event.

## Append-only history in wire form

Each assistant reply is stored as `[b.to_dict() for b in response.content]`,
exactly the blocks the API returned, including thinking and fallback blocks.
YACH never edits earlier messages. That keeps three things valid:

- **Prompt caching:** the request prefix is byte-identical turn to turn.
- **Preserved thinking:** replayed thinking blocks are only accepted against
  the exact history they were produced under.
- **Resume:** a saved session replays the same bytes it originally sent.

## Cost and context per turn

After each reply `send` calls:

- `add_usage(usage)`: adds top-level token counts plus any `compaction`
  entries from `usage.iterations` (which the API excludes from the top-level
  counts), prices them from the `PRICES` table, and accumulates the session
  total. Fallback turns are priced at the primary model's rate, an
  approximation. Responses without `iterations` (for example the reviewer's
  non-beta responses) are handled.
- `measure(usage)`: records the context size as input + output + cache
  creation + cache read tokens from the last non-compaction iteration.

It then prints a status line with the turn cost, session cost, cached tokens
and `ctx used/window (pct%)`. The context figure turns yellow at 60% and red at
85%, where a warning says the conversation will be compacted before the next
message.

## Related

- [Architecture overview](overview.md)
- [Tool surface](../tools/tool-surface.md)
- [Undo and checkpoints](../safety/undo.md)
- [Context measurement and compaction](../sessions/context-and-compaction.md)
- [Session storage and resume](../sessions/session-storage.md)
