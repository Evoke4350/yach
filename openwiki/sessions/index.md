# Files

- [Context measurement and compaction](context-and-compaction.md) - How YACH tracks how full the context window is and shrinks long conversations with the API's on-demand compaction, automatically between turns or via /compact, including cost accounting and failure handling.
- [Session storage and resume](session-storage.md) - How YACH saves every conversation as an append-only JSONL file per project, recovers from crashes and dangling tool calls, lists sessions without touching them, and resumes with the right system prompt and thinking settings.
