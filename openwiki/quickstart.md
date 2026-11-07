---
type: guide
title: Quickstart
description: What YACH is, how to run it, and which wiki page to read for common changes such as adding a tool, changing approval policy, touching sessions or compaction, or debugging the sandbox.
tags: [quickstart, overview, onboarding]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T00:49:17.948Z
sources:
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-07T00:49:17.948Z" }
---

# Quickstart

YACH is a terminal coding agent for the Claude API. It is one Python file,
`yach.py`, whose only dependency is the `anthropic` SDK. It streams a tool-use
loop with seven tools and checks every shell command and file write with an
approval policy. Behind the policy sit a reviewer model, destructive_command_guard
and an optional Docker sandbox. Conversations are saved per project and can be
resumed, and long ones are compacted automatically.

## Run it

```sh
uv tool install .                 # from a clone; installs the `yach` command
export ANTHROPIC_API_KEY=sk-ant-...
cd ~/code/some-project && yach    # the current directory is the project root
```

`yach "prompt"` runs once and exits; `yach -c` continues the last conversation
in this project. All flags and in-session commands are on
[Terminal interface and CLI](interface/repl-and-cli.md).

## Where to look

| If you want to… | Read |
|---|---|
| Understand the file layout and how a message flows | [Architecture overview](architecture/overview.md) |
| Change how turns, stop reasons, caching or cost work | [Agent loop](architecture/agent-loop.md) |
| Add or change a tool, or tighten path handling | [Tool surface](tools/tool-surface.md) |
| Change what gets approved, the reviewer prompt, or dcg behavior | [Approvals, auto mode and dcg](safety/approvals-and-dcg.md) |
| Debug sandboxed commands, timeouts or orphaned processes | [Docker sandbox and command execution](safety/sandbox.md) |
| Change how conversations are saved, repaired or resumed | [Session storage and resume](sessions/session-storage.md) |
| Change the context readout or compaction | [Context measurement and compaction](sessions/context-and-compaction.md) |
| Add a slash command, flag or change terminal output | [Terminal interface and CLI](interface/repl-and-cli.md) |

## Invariants to keep in mind

- **History is append-only.** Never edit or remove earlier messages. Prompt
  caching, preserved thinking and resume all depend on replaying identical
  bytes.
- **The system prompt and tool list must be stable within a session.**
  Changing either mid-session breaks the cache and invalidates replayed
  thinking.
- **Every `tool_use` needs a `tool_result`** before the next user message;
  `Session.repair()` enforces this.
- **Untrusted text goes through `clean()`** before it reaches the terminal.
- **Safety checks fail closed:** when the reviewer or dcg can't decide, the
  user is asked.

## Testing

The repository has no automated test suite. Changes have been verified with
ad-hoc scripts that stub the API client, and with a real Docker daemon for the
sandbox. When changing behavior, exercise the affected path directly, ideally
in a throwaway project directory with `XDG_DATA_HOME` pointed somewhere
disposable so real sessions aren't touched.
