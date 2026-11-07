---
type: concept
title: Context measurement and compaction
description: How YACH tracks how full the context window is and shrinks long conversations with the API's on-demand compaction, automatically between turns or via /compact, including cost accounting and failure handling.
tags: [context-window, compaction, tokens, claude-api]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T00:49:17.948Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-07T00:49:17.948Z" }
---

# Context measurement and compaction

YACH keeps an up-to-date estimate of how many tokens the conversation occupies.
When that crosses a threshold, it replaces the history with a summary written
by the API (on-demand compaction, beta `compact-2026-09-04`).

## Measuring context

- **After every reply**, `Agent.measure(usage)` sets `context_tokens` to input
  + output + cache-creation + cache-read tokens. If `usage.iterations` is
  present, it uses the last *non-compaction* entry. A compaction iteration
  reports the summarizer's own work, not the size of the conversation, so it
  must never be used for this.
- **The context window** comes from `client.models.retrieve(model).max_input_tokens`,
  fetched once and cached on the agent. If that call fails or returns nothing,
  YACH assumes 1,000,000.
- **On demand**, `count_context()` asks the free token-counting endpoint for
  the current history, with the same system prompt, tools and thinking
  settings as a real request and the compaction beta, so a summary block can be
  counted. It returns `0` for an empty history and `None` on API errors.
  `/context` uses it when nothing has been measured yet (fresh start, just
  resumed, or just compacted).

`context_tokens` is reset to `None` whenever the session changes (`/clear`,
resume), because the old measurement no longer applies.

## When compaction happens

- **Automatically:** at the start of `Agent.send`, before the new user message
  is appended, if `compact_at` is non-zero and `context_tokens >= compact_at`
  (the `--compact-at` flag, default 150,000). It runs only between user turns,
  never inside a tool loop. The whole history is summarized and nothing older
  is replayed, which sidesteps the rules for splitting tool calls and keeping
  older thinking blocks valid.
- **Manually:** `/compact`.

The status line after each reply warns at 85% of the window that compaction
will run before the next message. A single very long turn can still grow past
the threshold before the next check, because nothing compacts mid-turn.

## How compaction works

`Agent.compact()`:

1. Does nothing if there's nothing to summarize: an empty history, or one that
   is already just a summary.
2. Sends `client.beta.messages.create` with `compaction={"type": "summarize"}`,
   the same model, system prompt, tools and thinking settings as the
   conversation, `max_tokens=16000`, and the compaction beta. Matching the
   system prompt and tools keeps thinking produced after the summary valid.
3. First tries with top-level `cache_control` so the history is read from the
   prompt cache. If the API rejects that with a 400, it retries once without
   it.
4. Adds the cost with `add_usage`. Compaction tokens appear only under
   `usage.iterations` (top-level counts are zero), and `add_usage` includes
   them.
5. Checks for `stop_reason == "compaction"` and a `compaction` block with
   non-empty content. If either is missing, or the API call fails, it prints a
   message and **keeps the full conversation**.
6. On success, calls `Session.compact()` with an assistant message containing
   the compaction block in wire form. That block becomes the first message of
   the history, and the next user message follows it.
7. Re-measures with `count_context()` and prints `compacted N messages into a
   summary → X tokens ($cost)`.

### After compaction

While the history starts with an assistant-role summary, `stream_turn` adds
the compaction beta to every request so the API can read the block. On disk,
`Session.compact` writes a `{"yach_compaction": ...}` marker line followed by
the summary, keeping the full transcript. `Session.load` discards everything
before the latest marker, so a resumed session starts from the summary
([Session storage and resume](session-storage.md)).

## Related

- [Agent loop](../architecture/agent-loop.md)
- [Session storage and resume](session-storage.md)
