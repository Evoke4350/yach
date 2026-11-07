---
type: safety
title: Docker sandbox and command execution
description: How YACH runs the model's shell commands, on the host or in a hardened throwaway Docker container, including timeouts, process-group cleanup, the read-only .git mount, custom images, and what the sandbox does not protect.
tags: [sandbox, docker, bash, process-management, security]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-07T00:49:17.948Z
sources:
  - id: openwiki-source-d3fe2c58a00acb98c6319084
    resource: repo://yach.py
generated: { by: "claude-code", at: "2026-11-07T00:49:17.948Z" }
---

# Docker sandbox and command execution

The model's `bash` tool is handled by `run_bash`. By default commands run on
the host. With `--sandbox`, `main()` creates a `Sandbox` and every command runs
inside it via `docker exec`. Only the model's commands are sandboxed: YACH
itself, its API key and the user's own `!` commands stay on the host.

## Running a command

`run_bash(command, timeout=120)`:

1. **Clamps the timeout** to 1-1800 seconds. The model chooses it, and `0`
   would disable coreutils' `timeout` inside the container.
2. **Starts the process.**
   - *Host:* `shell=True` in the project root, in a **new session**
     (`start_new_session=True`), with `PLAIN_ENV` (`TERM=dumb`, `NO_COLOR=1`,
     `PAGER=cat`, `GIT_PAGER=cat`) layered over the user's environment, so
     tools print plain output and never block on a pager.
   - *Sandbox:* the argv from `Sandbox.exec_argv`.
   stdin is `/dev/null`, and stdout and stderr are merged.
3. **Waits** for the timeout, plus 15 s of slack in the sandbox, where the
   in-container `timeout` normally fires first.
4. **On timeout or Ctrl-C**, kills the whole command, not just its shell:
   `os.killpg` on the host process group, or `Sandbox.kill_run` in the
   container. It then kills the client process and collects whatever output
   arrived (giving up after 5 s if a detached grandchild still holds the pipe).
   Ctrl-C is re-raised to the agent loop; a timeout becomes a `ToolError` with
   the partial output.
5. **Returns** cleaned output, with `[exit code N]` appended for non-zero exits.
   In the sandbox, exit code 124 (coreutils' timeout) is reported as a
   timeout.

Because host commands run in their own session, the terminal's Ctrl-C reaches
YACH rather than the command. YACH then kills the group itself.

## The container

`Sandbox.__init__` starts one long-lived container per YACH process, named
`yach-<pid>-<random>` and running `sleep infinity` under `--init`, with:

| Setting | Why |
|---|---|
| `--user <uid>:<gid>` | Files it creates belong to you; it isn't root |
| `--cap-drop ALL`, `--security-opt no-new-privileges` | No capabilities, no setuid escalation |
| `--read-only` + `--tmpfs /tmp` | Root filesystem immutable; scratch space in `/tmp`, also `HOME` |
| `--pids-limit 1024` | Caps fork bombs |
| `--network bridge\|none` | From `--sandbox-network`; `none` cuts all network access |
| `-v ROOT:ROOT` | Only the project is mounted, at the same path as on the host |
| `-v ROOT/.git:ROOT/.git:ro` | `.git` is read-only (below) |

Nothing from the user's environment is passed in, so `ANTHROPIC_API_KEY` and
other secrets are not visible to sandboxed commands. The container is removed
with `docker rm -f` at exit (registered with `atexit`). `rm -f` waits for
removal, where a plain `kill` would return before `--rm` finishes.

### Read-only `.git`

Git runs hooks and config-driven commands (`core.fsmonitor`, `core.pager`, …)
on the host. If the sandbox could write `.git`, a command could plant code that
later runs outside the container with the user's full environment. So `.git` is
mounted read-only when it's a directory. As a result, the model can't commit
from inside the sandbox; `sandbox_note()` tells it so in the system prompt.

### Process groups inside the container

Killing the `docker exec` client doesn't stop the command inside the container.
`exec_argv` therefore runs each command as:

```
setsid -w bash -c 'echo $$ > /tmp/yach-run-<id>.pid; exec timeout -k 5 <t> bash -c "$1"' yach <command>
```

`setsid` gives the run its own process group, whose ID is written to a pid
file. `kill_run` sends `kill -9 -<pgid>` through `docker exec`. The container's
`sh` is dash, whose `kill` doesn't accept `--`. The in-container `timeout`
handles the normal case.

### Images

`build_image` uses `.yach/Dockerfile` from the project if it exists, otherwise a
built-in image: `python:3.14-slim` with git, ripgrep, curl, ca-certificates,
procps and uv. Images are tagged `yach-sandbox:<sha256 of Dockerfile>[:12]` and
built only if that tag doesn't exist yet, so editing the Dockerfile triggers a
rebuild and unchanged ones are reused.

## What the sandbox does not protect

- **The project itself.** It is mounted read-write, so the model can still
  change or delete project files, including build scripts the user will run
  later on the host.
- **Remote systems,** when the network is on: pushes, uploads and API calls
  still work. Approvals and dcg apply to sandboxed commands exactly as on the
  host ([Approvals, auto mode and dcg](approvals-and-dcg.md)).

With the sandbox on, `resolve()` also confines the host-side file tools
(`read_file`, `grep`) to the project, matching what the container can see
([Tool surface](../tools/tool-surface.md)).

## Related

- [Approvals, auto mode and dcg](approvals-and-dcg.md)
- [Tool surface](../tools/tool-surface.md)
