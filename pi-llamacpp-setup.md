# Pi Coding Agent — llama.cpp on 127.0.0.1:8081

---

## Model Selection

From your list, three models cover every scenario without redundancy:

| Role | Model | Size | Why |
|---|---|---|---|
| **Primary** | `qwen3-coder:30b` | 18 GB | Dedicated coding model, best tool-following in your list |
| **Reasoning / Architecture** | `qwen3.6:35b-a3b-q4_K_M` | 23 GB | Latest Qwen3.6 MoE, strongest broad reasoning |
| **Fast / Cheap** | `qwen2.5-coder:7b` | 4.7 GB | Quick reads, grep-style queries, throwaway tasks |

Skip `qwen3-coder-next` (51 GB — too heavy for smooth iteration), `gemma4` variants (not coding-optimised), and embed models.

> **llama.cpp model ID caveat:** llama.cpp serves one model at a time. The `id` field in `models.json` must match what `GET http://127.0.0.1:8081/v1/models` actually returns. Check it before writing the config:
> ```bash
> curl -s http://127.0.0.1:8081/v1/models | python3 -m json.tool
> ```
> Use the exact `id` string from that response in the config below. If the field is a GGUF filename like `qwen3-coder-30b-q4_K_M.gguf`, use that string.

---

## 1. `~/.pi/agent/models.json`

```json
{
  "providers": {
    "llama-cpp": {
      "baseUrl": "http://127.0.0.1:8081/v1",
      "api": "openai-completions",
      "apiKey": "none",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "qwen3-coder:30b",
          "name": "Qwen3-Coder 30B (primary)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 32768,
          "maxTokens": 8192,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        },
        {
          "id": "qwen3.6:35b-a3b-q4_K_M",
          "name": "Qwen3.6 35B MoE (reasoning)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 32768,
          "maxTokens": 8192,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        },
        {
          "id": "qwen2.5-coder:7b",
          "name": "Qwen2.5-Coder 7B (fast)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 16384,
          "maxTokens": 4096,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

**If the model ID from `/v1/models` is a filename** (e.g. `qwen3-coder-30b-q4_k_m.gguf`), replace the `id` values accordingly. The file reloads every time you open `/model` — no restart needed.

---

## 2. `~/.pi/agent/settings.json`

```json
{
  "defaultProvider": "llama-cpp",
  "defaultModel": "qwen3-coder:30b",
  "autoCompaction": true,
  "compactionThreshold": 0.80,
  "favorites": [
    "llama-cpp/qwen3-coder:30b",
    "llama-cpp/qwen3.6:35b-a3b-q4_K_M",
    "llama-cpp/qwen2.5-coder:7b",
    "anthropic/claude-sonnet-4-6"
  ]
}
```

`favorites` lets you cycle through all four with `Ctrl+P`. The Anthropic entry requires `ANTHROPIC_API_KEY` in your environment — set it in `~/.zshrc` or `~/.bashrc` for cloud fallback:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

---

## 3. `~/.pi/agent/AGENTS.md`

General-purpose development instructions with no project specifics:

```markdown
# Agent Instructions

## Inference constraints
- Backend: llama.cpp on localhost:8081. One model loaded at a time.
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
```

---

## 4. Community Packages — Install Order

These are the highest-signal packages from the current `pi-package` ecosystem, each actively maintained and directly useful for general development.

### 4a. Official pi-skills (from pi's own author)

```bash
pi install git:github.com/badlogic/pi-skills
```

Installs these on-demand skills, invoked with `/skill:name`:

| Skill | What it adds |
|---|---|
| `brave-search` | Web search from inside the agent (needs Brave API key) |
| `browser-tools` | Open URLs, take screenshots (needs Chrome + Node) |
| `gccli` | Google Calendar read/write from the agent |
| `gdcli` | Google Drive file access |
| `gmcli` | Gmail read/compose |
| `subagent` | Spawn a child pi instance for parallel sub-tasks |
| `vscode` | Open files in VS Code from the agent |
| `youtube-transcript` | Fetch YT transcripts as context |

Skills load on-demand — they don't bloat your base prompt. Call `brave-search` only when the agent needs external lookup; the rest of the time context is clean.

### 4b. LSP / Real-time Code Feedback

```bash
pi install npm:lsp-pi
```

Adds live LSP diagnostics after every file edit: syntax errors, type mismatches, unused imports — all surfaced as tool results before the next turn. Supports Python, TypeScript, Go, Rust, Kotlin, and more.

After installing, enable for your languages in `~/.pi/agent/settings.json`:

```json
{
  "lsp": {
    "python": true,
    "typescript": true,
    "go": false
  }
}
```

This is the single highest-leverage package for reducing iteration loops: the agent gets compiler feedback immediately without you triggering a test run.

### 4c. Subagent Delegation (nicopreme's package)

```bash
pi install npm:pi-subagents
```

Adds `/task` command for delegating sub-tasks to parallel pi instances with chain and parallel execution modes. The orchestrator pi stays in your terminal; workers run in tmux panes. Good for: "implement X in module A while also writing tests for module B."

### 4d. Autoresearch (autonomous optimization loop)

```bash
pi install https://github.com/davebcn87/pi-autoresearch
```

Adds `/autoresearch <goal>` — the autonomous iteration loop. Requires two scripts in the project root:

- `autoresearch.benchmark.sh` — prints a single numeric score to stdout
- `autoresearch.checks.sh` (optional) — correctness guard; if it exits non-zero, the run is logged as `checks_failed` and changes are reverted

The agent then runs: edit → benchmark → if score improved, commit; else revert → repeat. Never stops until interrupted with `/autoresearch:stop`.

### 4e. Heart of Gold Skills (cross-agent skill pack)

```bash
pi install npm:heart-of-gold
```

Cross-platform skill pack compatible with pi, Claude Code, and Codex CLI. Includes skills for TDD workflows, code review, architecture decisions, and debugging pipelines. Actively maintained, 16+ hours of runtime against real codebases.

### 4f. Long-running agent loops

```bash
pi install npm:pi-agent-loops
```

Adds iterative development loop commands — useful for "keep working on this until tests pass" style tasks without needing autoresearch's benchmark machinery.

---

## 5. Verify the full setup

```bash
# 1. Check llama.cpp is reachable and confirm the model ID
curl -s http://127.0.0.1:8081/v1/models | python3 -m json.tool

# 2. Check pi version
pi --version

# 3. Launch pi — should show llama-cpp as provider in the footer
pi

# Inside pi:
# /model           — verify models list loads correctly
# Ctrl+P           — cycle through favorites
# /skill:subagent  — verify pi-skills installed (after step 4a above)
```

---

## 6. Model switching workflow

**Normal coding work:** stay on `qwen3-coder:30b` (primary).

**Stuck on architecture, multi-file design, or security analysis:** `Ctrl+P` → `qwen3.6:35b-a3b-q4_K_M`. Broader reasoning, handles more conceptual weight.

**Quick file reads, grep-style questions, throwaway scripts:** `Ctrl+P` → `qwen2.5-coder:7b`. Faster, lower context cost.

**Task exceeds local model capability** (complex refactor, subtle bug you can't pin down): `Ctrl+P` → `claude-sonnet-4-6`. Context carries across the switch — no restart. Switch back to local when the hard part is resolved.

> Note: switching models in pi does **not** restart the llama.cpp server. Pi simply sends the next API call with the new model ID in the request body. If llama.cpp is serving a single model, it may ignore the `model` field and respond with whatever is loaded — which is fine as long as you know which model is actually running.

---

## 7. Quick reference

| Action | Key |
|---|---|
| Switch model | `Ctrl+L` (full list) or `Ctrl+P` (cycle favorites) |
| Force compaction | `/compact` |
| Navigate session tree | `/tree` |
| Load a skill | `/skill:brave-search` etc. |
| Start autoresearch loop | `/autoresearch <goal>` |
| Stop autoresearch | `/autoresearch:stop` |
| Continue last session | `pi -c` |
| Browse sessions | `pi -r` |
| One-shot headless | `pi -p "query" --no-session` |
| Update all packages | `pi update` |
| List installed packages | `pi list` |
