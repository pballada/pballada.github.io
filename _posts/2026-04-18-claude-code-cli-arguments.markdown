---
layout: post
title: "Claude Code CLI Arguments You Probably Aren’t Using Yet"
date: 2026-04-18 09:30:00 +0000
categories: [AI, CLI, DeveloperTools]
header_image: /assets/images/claude-code-cli-arguments.jpg
tags: [claude-code, cli, productivity, tooling, ai]
---

Claude Code has grown a lot beyond the usual `claude`, `claude -p`, and `claude -c` workflows. If you only ever use the obvious flags, you are leaving a surprising amount of power on the table.

This post focuses on the **newer, lesser-known, or easily overlooked Claude Code CLI arguments** that are actually useful in day-to-day work. I am intentionally skipping the basics and focusing on the ones that tend to unlock better automation, safer scripting, cleaner session management, and more advanced tooling setups.

> **Note:** Claude Code's official documentation explicitly states that `claude --help` does **not** list every available flag, so the docs are the better source of truth for discovery.

---

## Why These Flags Matter

A lot of Claude Code usage falls into one of these buckets:

- **Interactive coding sessions**
- **Headless or scripted runs**
- **CI/CD and automation**
- **Multi-repo or multi-directory work**
- **MCP, plugins, and advanced integrations**

The flags below are especially useful when you move beyond casual terminal use and start treating Claude Code as part of your real tooling stack.

---

## 1. `--exclude-dynamic-system-prompt-sections`

This is one of the most interesting recent flags because it is not flashy, but it matters a lot for **prompt caching and reproducibility**.

### What it does
It moves machine-specific context, such as:

- working directory
- environment info
- memory paths
- git status

out of the system prompt and into the first user message.

### Why it matters
If you run the same scripted Claude task across:

- different machines
- multiple developers
- CI workers
- ephemeral containers

this can improve **prompt-cache reuse** and reduce needless variation.

### Example
```bash
claude -p --exclude-dynamic-system-prompt-sections "summarize the changes in this repo"
```

### Best use case
Use it for **repeatable print-mode automation** where consistency and cache efficiency matter more than convenience.

---

## 2. `--replay-user-messages`

This one is easy to miss, and it is especially useful if you are building wrappers around Claude Code.

### What it does
When using streamed JSON input and output, it re-emits user messages from stdin back to stdout as acknowledgments.

### Why it matters
This is handy when you are:

- building an event-driven wrapper around Claude Code
- integrating Claude Code with another process
- writing tooling that needs to correlate requests and responses cleanly

### Example
```bash
claude -p \
  --input-format stream-json \
  --output-format stream-json \
  --replay-user-messages
```

### Best use case
Use it in **SDK-style orchestration** or when piping structured events through multiple processes.

---

## 3. `--strict-mcp-config`

If you use MCP seriously, this flag deserves a place in your muscle memory.

### What it does
It tells Claude Code to use **only** the MCP servers provided via `--mcp-config`, ignoring other MCP configuration sources.

### Why it matters
Without this, local or inherited MCP config can affect behavior in subtle ways. That is annoying in automation and potentially risky in shared environments.

### Example
```bash
claude --strict-mcp-config --mcp-config ./mcp.json
```

### Best use case
Use it for:

- CI jobs
- demos
- reproducible environments
- security-conscious setups

If you want deterministic MCP behavior, this is the flag.

---

## 4. `--bare`

This one is great when you want Claude Code to start fast and act more like a minimal execution engine.

### What it does
It skips auto-discovery of:

- hooks
- skills
- plugins
- MCP servers
- auto memory
- `CLAUDE.md`

### Why it matters
Sometimes you do **not** want the full local environment loaded. In scripts, that extra discovery can add latency and unpredictability.

### Example
```bash
claude --bare -p "list the risky parts of this diff"
```

### Best use case
Perfect for:

- automation
- benchmark experiments
- hermetic-ish scripting
- debugging whether local configuration is influencing results

---

## 5. `--fallback-model`

This is one of the most practical headless flags for reliability.

### What it does
In print mode, Claude Code can automatically fall back to another model if the default model is overloaded.

### Example
```bash
claude -p --fallback-model sonnet "generate a changelog from these commits"
```

### Why it matters
If you run Claude Code in scripts, CI, or internal tools, transient model availability issues are painful. A fallback model can turn a failed run into a degraded-but-successful one.

### Best use case
Use it in:

- CI pipelines
- cron jobs
- internal developer tooling
- any automation where “some answer” is better than “job failed”

---

## 6. `--max-budget-usd`

A small flag with very real operational value.

### What it does
Sets a hard spending cap for API calls in print mode.

### Example
```bash
claude -p --max-budget-usd 2.50 "review this repository for likely dead code"
```

### Why it matters
This is a strong safety rail for:

- experiments
- batch jobs
- junior team members trying automation
- any workflow where token costs could unexpectedly balloon

### Best use case
If you are turning Claude Code into infrastructure, cost guardrails stop being optional.

---

## 7. `--fork-session`

This is a quiet but elegant session-management flag.

### What it does
When resuming with `--resume` or `--continue`, it creates a **new session ID** instead of continuing to append to the original.

### Example
```bash
claude --resume auth-refactor --fork-session
```

### Why it matters
Sometimes you want the context of an existing conversation without mutating its history. This gives you a branch-like workflow.

### Best use case
Great for:

- trying alternate approaches
- preserving a “golden” session
- debugging a task from a known checkpoint

---

## 8. `--from-pr`

This one is a strong signal that Claude Code is becoming more PR-native.

### What it does
Resumes sessions linked to a specific GitHub pull request, using either a PR number or URL.

### Example
```bash
claude --from-pr 123
```

### Why it matters
If your workflow already lives in GitHub, this reduces friction between coding sessions and review context.

### Best use case
Useful when:

- reviewing a PR over multiple sittings
- resuming an AI-assisted review flow
- handing off work tied to a specific pull request

---

## 9. `--remote`, `--teleport`, and `--remote-control`

These are not exactly hidden, but many developers still have not built them into their workflow.

### Why they are interesting
Together, these flags point to a bigger shift: Claude Code is no longer just a local terminal assistant.

- `--remote` creates a web session on Claude.ai with the provided task.
- `--teleport` resumes a web session locally.
- `--remote-control` enables a session you can also control from Claude.ai or the Claude app.

### Example
```bash
claude --remote "Investigate why this build fails on CI"
```

### Best use case
These are powerful if you move between:

- desktop and browser
- laptop and remote machine
- solo work and collaborative debugging

I would call this set **underused rather than unknown**.

---

## 10. `--channels` and `--dangerously-load-development-channels`

These feel like early glimpses of where Claude Code is headed.

### What they do
- `--channels` lets Claude listen for MCP server channel notifications in a session.
- `--dangerously-load-development-channels` allows non-approved channels for local development.

### Why they matter
This shifts Claude from a purely request-response tool into something more event-aware.

### Best use case
These are especially relevant if you are experimenting with:

- alerts
- webhooks
- event-driven development workflows
- custom integrations during local development

Because this area is still evolving, I would treat these as advanced and somewhat exploratory.

---

## 11. `--tmux` with `--worktree`

This pair is fascinating if you like isolation and parallelism.

### What it does
`--worktree` starts Claude in an isolated git worktree. `--tmux` creates a tmux session for that worktree.

### Example
```bash
claude -w feature-auth --tmux
```

### Why it matters
This is a serious quality-of-life improvement for developers juggling:

- multiple branches
- parallel investigations
- larger refactors
- agent-team workflows

### Best use case
If you often think, “I wish this experiment lived in its own branch and pane,” this is the answer.

---

## 12. `--allowedTools`, `--disallowedTools`, and `--tools`

These are easy to underestimate.

### Why they matter
These flags let you shape Claude Code’s tool surface in three distinct ways:

- `--allowedTools`: allow some tools without prompting
- `--disallowedTools`: completely remove specific tools
- `--tools`: restrict the built-in tool set

### Example
```bash
claude --tools "Bash,Read" --disallowedTools "Edit"
```

### Best use case
Excellent for:

- demos
- safer sandboxed sessions
- read-only audits
- controlling automation behavior in scripts

This is also one of the better ways to make sessions more predictable.

---

## 13. `--include-hook-events` and `--include-partial-messages`

These are very much “power user” flags.

### What they do
They enrich streamed output with:

- hook lifecycle events
- partial model messages

### Why they matter
If you are building observability or custom tooling around Claude Code, these flags make the output stream dramatically more informative.

### Example
```bash
claude -p \
  --output-format stream-json \
  --include-hook-events \
  --include-partial-messages \
  "analyze this repository"
```

### Best use case
Use them when building:

- dashboards
- wrappers
- custom UIs
- debugging tools for Claude-driven workflows

---

## 14. `--agent` and `--agents`

These are powerful, and still not widely discussed.

### What they do
- `--agent` selects an agent for the session.
- `--agents` defines custom subagents dynamically via JSON.

### Why they matter
This opens the door to more specialized, role-based workflows without baking everything into static config.

### Example
```bash
claude --agents '{
  "reviewer": {
    "description": "Reviews risky changes",
    "prompt": "Focus on bugs, regressions, and edge cases"
  }
}'
```

### Best use case
Great for:

- temporary role specialization
- experiments with subagent patterns
- session-local customization without changing shared config

---

## 15. A Small Flag That Tells a Bigger Story: `--enable-auto-mode`

This one is interesting because it is now effectively a historical footnote.

### Current status
The docs note that `--enable-auto-mode` was **removed in v2.1.111**, because auto mode is now included in the Shift+Tab cycle by default. The new recommendation is:

```bash
claude --permission-mode auto
```

### Why mention a removed flag?
Because it tells us something important: Claude Code is evolving quickly, and some blog posts or gists go stale fast.

If you are writing automation or internal docs, check the **current official reference** rather than trusting an old snippet.

---

## My Shortlist: The Most Useful “Unknown” Flags

If I had to pick the flags most developers should learn next, it would be these:

1. `--bare`
2. `--strict-mcp-config`
3. `--exclude-dynamic-system-prompt-sections`
4. `--fallback-model`
5. `--max-budget-usd`
6. `--fork-session`
7. `--replay-user-messages`

That group covers the biggest practical gains in:

- reproducibility
- reliability
- cost control
- safe automation
- advanced orchestration

---

## Final Thoughts

Claude Code CLI has quietly become much richer than many developers realize. The most useful advanced flags are not necessarily the flashy ones. They are the ones that make the tool:

- more reproducible
- more scriptable
- safer in automation
- easier to integrate into real engineering workflows

If you only try three after reading this, I would start with `--bare`, `--strict-mcp-config`, and `--exclude-dynamic-system-prompt-sections`.

Those three alone change how “serious” Claude Code feels in a production-grade developer setup.
