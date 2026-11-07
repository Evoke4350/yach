# YACH: Yet Another Coding Harness

A hyper-minimal coding agent for your terminal. One Python file, one dependency
(the `anthropic` SDK), your own API key.

Everyone has a coding agent now. This one is small enough to read in an
afternoon and change in an evening.

```
$ yach
YACH · claude-opus-5-5 · effort high · approval auto · no sandbox · dcg on · ~/code/app

> the date parser drops timezones, fix it and add a test
● grep
  parse_date
● read_file
  src/dates.py
● edit_file
  src/dates.py
@@ -41,3 +41,3 @@
-    return datetime.strptime(s, FMT)
+    return datetime.fromisoformat(s)
  ✓ auto-approved Focused edit to the file the user asked to fix.
● bash
$ uv run pytest -q tests/test_dates.py
  ✓ auto-approved Runs the project's tests.
  4 passed in 0.21s
Fixed: `parse_date` now keeps the offset, and there's a regression test.
[$0.0412 turn · $0.0412 session · 18k cached · ctx 21k/1M (2%)]
```

## Features

- **Streaming agent loop.** Text, summarized thinking, and tool calls stream
  live. Spinners cover the quiet gaps: waiting on the model, a file being
  written, a command running.
- **Seven tools:** `bash`, `read_file`, `write_file`, `edit_file`, `grep`
  (ripgrep), `glob`, and `ask_user`, so the model can stop and ask you a
  question with suggested answers.
- **Auto mode for approvals.** A reviewer model (Claude Sonnet 5.5) checks each
  shell command and file write. Routine work in scope runs; anything risky or
  off-task comes to you with a reason. The reviewer only sees your own
  messages, never tool output, so a malicious file can't argue for its own
  approval. If the reviewer fails, you get asked.
- **[DCG](https://github.com/Dicklesworthstone/destructive_command_guard) built
  in.** If `dcg` is on your PATH, every shell command goes through it first.
  A dcg hit always needs an explicit yes, even in `yolo` mode. If you say no,
  dcg's reason and safer alternative go back to the model.
- **Optional Docker sandbox.** `--sandbox` runs the model's commands in a
  throwaway container that sees only the project: your UID, no capabilities,
  read-only root filesystem, no sudo. YACH and your API key stay on the host,
  so nothing the model runs can read the key. `--sandbox-network none` turns
  off the network for commands.
- **`!` shell passthrough.** `!git status` runs in your shell and shares the
  output with the model, like Claude Code.
- **Sessions.** Every conversation is saved as you go. `-c` continues the
  latest, `-r` picks from a list. Replies are stored exactly as the API
  returned them, so a resumed conversation keeps the model's earlier thinking.
- **Context display and compaction.** Every turn shows how full the context
  window is. Past 150k tokens (configurable), the conversation is summarized
  server-side before your next message. `/compact` does it on demand.
- **Prompt caching and a cost meter.** History is append-only and the system
  prompt never changes mid-session, so each request reads the previous one
  from cache. Every turn shows its cost and the session total.
- **Project instructions.** `AGENTS.md` or `CLAUDE.md` in the project root is
  added to the system prompt.

## Documentation

How YACH works inside, page by page, with every claim linked to the lines of
code it describes. **Browse it at [yach.stayfresh.dev](https://yach.stayfresh.dev)**,
an interactive graph with a reader, or read the Markdown here:

- **[Wiki quickstart](openwiki/quickstart.md)**: start here; it routes you to
  the right page for a task.
- Pages: [architecture](openwiki/architecture/overview.md) ·
  [agent loop](openwiki/architecture/agent-loop.md) ·
  [tools](openwiki/tools/tool-surface.md) ·
  [approvals & dcg](openwiki/safety/approvals-and-dcg.md) ·
  [sandbox](openwiki/safety/sandbox.md) ·
  [sessions](openwiki/sessions/session-storage.md) ·
  [context & compaction](openwiki/sessions/context-and-compaction.md) ·
  [terminal & CLI](openwiki/interface/repl-and-cli.md)
- The site's source is the static export in
  [`docs/openwiki-visualizer/`](docs/openwiki-visualizer/), deployed to
  Cloudflare Pages.

The wiki is generated with [OpenWiki](https://www.npmjs.com/package/openwiki).

## Install

Needs Python 3.12+ and [uv](https://docs.astral.sh/uv/).
[ripgrep](https://github.com/BurntSushi/ripgrep) is needed for the `grep` tool;
[dcg](https://github.com/Dicklesworthstone/destructive_command_guard) and Docker
are optional.

```sh
git clone https://github.com/Evoke4350/yach && cd yach
uv tool install .            # puts `yach` on your PATH
export ANTHROPIC_API_KEY=sk-ant-...
```

Or run it without installing: `uv run --project /path/to/yach yach`.

## Usage

```sh
cd ~/code/some-project
yach                                  # interactive
yach "add a --verbose flag"           # one prompt, then exit
yach -c                               # continue the last conversation here
yach -r                               # pick a conversation to resume
yach --sandbox --sandbox-network none # commands in a container, no network
```

| Flag | Default | |
|---|---|---|
| `--model` | `claude-opus-5-5` | Any Claude model ID |
| `--effort` | `high` | `low` `medium` `high` `xhigh` `max` |
| `--approval` | `auto` | `ask`: confirm every command/write · `auto`: reviewer decides, asks when unsure · `yolo`: approve everything (dcg still applies) |
| `--yolo` | | Same as `--approval yolo` |
| `--sandbox` | off | Run shell commands in a Docker container |
| `--sandbox-network` | `bridge` | `none` turns off networking in the sandbox |
| `--no-dcg` | | Skip the dcg check |
| `--compact-at` | `150000` | Auto-compact threshold in tokens; `0` turns it off |
| `-c`, `--continue` | | Continue the most recent conversation in this project |
| `-r`, `--resume [ID]` | | Resume a conversation by ID, or pick from a list |

In the prompt:

| Command | |
|---|---|
| `!<command>` | Run a shell command yourself; the output is shared with the model |
| `/context` | How full the context window is |
| `/compact` | Summarize the conversation now |
| `/cost` | Token and cost breakdown |
| `/approval [mode]` | Show or switch the approval mode |
| `/resume` | Switch to a saved conversation |
| `/clear` | Start a new conversation (the old one stays saved) |
| `/exit` | Quit (Ctrl-C or Ctrl-D at the prompt also work) |

Ctrl-C during a turn stops the turn and keeps the conversation.

## Safety model

Layers, from outermost in:

1. **dcg**: fixed rules for destructive commands (`git reset --hard`, `rm -rf`,
   force pushes, …). It can't be talked out of a decision, and only an explicit
   yes from you overrides it.
2. **Approval mode**: in `auto`, the reviewer model judges intent and risk; in
   `ask`, you judge everything.
3. **Project confinement**: file writes outside the project root are refused,
   whatever the mode. Reading or searching outside it (`~/.ssh`, `~/.aws`,
   another project's `.env`) goes through approval like a write.
4. **Sandbox** (opt-in): commands can only touch the project directory, and
   never see your environment or credentials. The project's `.git` is mounted
   read-only, so nothing in the sandbox can plant hooks or config that git
   would later run on your host. That also means commits happen outside the
   sandbox.

Commands that time out or that you stop with Ctrl-C are killed along with
their child processes, on the host and in the sandbox.

The sandbox protects your machine, not your project: the model can still change
anything inside the project folder (including build scripts you'll run later),
and push or deploy if networking is on. Use git, and read diffs.

To give the sandbox your own toolchain, put a Dockerfile at `.yach/Dockerfile`
in your project.

## Where things live

- Sessions: `~/.local/share/yach/sessions/<project-path>/<id>.jsonl` (honors
  `XDG_DATA_HOME`). They contain everything said in the conversation, including
  tool output, so treat them as sensitive.
- Everything else: `yach.py`. Search it for `TOOLS`, `REVIEW_SYSTEM`,
  `class Sandbox`, `class Session`, and `Agent.send`.

## Status

Early and small on purpose. It's been built and tested piece by piece, but not
run hard on real projects yet. Expect rough edges, and read a diff before you
trust it.
