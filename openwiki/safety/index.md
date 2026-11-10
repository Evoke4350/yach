# Files

- [Approvals, auto mode and dcg](approvals-and-dcg.md) - How YACH decides whether a tool call may run, covering ask, auto and yolo modes, the reviewer model, destructive_command_guard checks that override every mode, approval for reads outside the project, and fail-closed defaults.
- [Docker sandbox and command execution](sandbox.md) - How YACH runs the model's shell commands, on the host or in a hardened throwaway Docker container, including timeouts, process-group cleanup, the read-only .git mount, custom images, and what the sandbox does not protect.
- [Undo and checkpoints](undo.md) - How /undo restores the files a turn wrote or edited through write_file and edit_file, using an in-memory Checkpoint per turn; what it asks about, what it cannot reverse (shell commands), and how the model learns of the undo.
