# Agent Instructions

## Inference constraints
- Backend: Ollama on localhost:11434. Multiple models can run simultaneously (unlike llama.cpp).
- Context window: up to 32k but treat 24k as the safe working limit.
- When context exceeds 70%, write a TODO.md summarising remaining work before compacting.

## Work style
- Read files before modifying them — never assume structure.
- Verify changes: run the relevant test, linter, or compile check after every edit.
- For tasks longer than ~6 turns, write a TASK.md plan first and work through it.
- Prefer surgical edits over full rewrites. Use the edit tool, not write, for existing files.
- Destructive operations (delete, drop, rm -rf) require an explicit confirmation comment
  in your reasoning before executing.

## Code quality defaults
- Python: type hints, no bare `except`, f-strings over `.format()`
- TypeScript/JS: strict mode, explicit return types on exported functions
- Docker: always define HEALTHCHECK, pin base image tags
- Shell: `set -euo pipefail` at the top of every script

## When to stop
Only pause and ask if:
1. You need a credential or secret that isn't in the repo
2. A destructive action has no clear rollback
3. The task spec is genuinely ambiguous (not just complex)
