# Claude Code Operations Guide

A practical reference for context management, session workflows, and effective use of Claude Code as an agentic development tool.

---

## How Claude Code Works

Claude Code is a terminal-native agent that reads your codebase, executes shell commands, modifies files, and manages git workflows — all running with your user-level filesystem permissions. It's not an autocomplete plugin. It plans, acts, and iterates autonomously within the boundaries you set.

Effective use comes down to five systems: **context management**, **session state**, **workflow orchestration**, **permissions**, and **interactive controls**. This guide covers each one.

---

## 1. Context Management

The context window is a finite token budget shared between system prompts, loaded files, conversation history, tool schemas, and your instructions. As a session progresses, accumulated content fills this budget. When it runs low, instruction adherence degrades — the model doesn't selectively forget; compliance drops across *all* directives simultaneously.

Context hygiene is the single most important operational skill.

### The `/context` Command

Run `/context` to visualize current token consumption as a colored grid. It breaks down what's consuming your budget: system prompts, active rules, loaded skills, files in context, and conversation history. Use it as a diagnostic dashboard — check it periodically, especially when you notice quality dropping.

### CLAUDE.md: The Persistent Intelligence Layer

`CLAUDE.md` is a configuration file automatically ingested at the start of every session. It's your initialization protocol — not documentation, but a set of persistent instructions that shape how the model reasons, writes code, and makes decisions.

Run `/init` to generate a baseline `CLAUDE.md`. Set `CLAUDE_CODE_NEW_INIT=1` for an interactive flow that also walks through skills, hooks, and memory files.

**What to include:**

| Content Type | Include? | Rationale |
| :--- | :--- | :--- |
| Custom build/deploy scripts | **Yes** | The model can't infer proprietary commands |
| Business context and risk profile | **Yes** | Calibrates the model's judgment and error-handling posture |
| Explicit prohibitions (frozen dirs, no new deps) | **Yes** | The model defaults to action — boundaries must be explicit |
| Branch naming, PR conventions | **Yes** | Ensures smooth team integration |
| Standard language syntax | **No** | The model already knows this |
| Rules the linter enforces | **No** | The model self-corrects from linter output |
| Detailed API documentation | **No** | Provide a URL; instruct the model to fetch it when needed |

Keep `CLAUDE.md` under 150 distinct instructions. Beyond that, you're adding noise that weakens everything else. For more on writing effective CLAUDE.md files, see the companion *Context Engineering Guide*.

### Progressive Disclosure with Rules

Don't front-load everything into the root `CLAUDE.md`. The `.claude/rules/` directory holds modular instruction files loaded alongside `CLAUDE.md` automatically. Each supports frontmatter globs for path-scoped loading:

```yaml
---
paths: ["src/api/**/*.ts"]
---
Validate all inputs with Zod schemas.
Return errors using ApiError.fromZod().
Rate limiting is handled at the gateway — do not implement it per-route.
```

Backend database rules won't pollute the context when the model is refactoring a React component, and vice versa.

---

## 2. Session State Management

When `/context` reveals high token usage or you notice reasoning quality dropping, you have two distinct operations available.

### `/compact` — Surgical Compression

Synthesizes conversation history into a dense summary, retaining key decisions while discarding verbose mechanics (error stacks, debugging tangents, repeated file reads). You can guide it with a focus argument:

```
/compact retain the database schema changes and current test failures
```

**When to use:** Mid-task when context is bloated but the direction is still correct. A good rule of thumb is to compact proactively around 50–80% context utilization — by the time you notice quality degradation, you've already lost fidelity.

### `/clear` — Hard Reset

Purges the entire session. Use it when transitioning between unrelated tasks, or when the model has gone down the wrong path and corrections aren't working. Dragging broken iterations forward is a common anti-pattern — sometimes the fastest fix is a clean start.

**Tip:** Use `/rename` before clearing to archive the session, then `/resume` to return to it later if needed.

### Preserving State Across Sessions

Before compacting or clearing:

1. Ask the model to summarize progress into a structured file (e.g., `progress.md`).
2. Run `/compact` or `/clear`.
3. Start fresh — the model re-reads `CLAUDE.md` automatically, and you can reference the progress file with `@progress.md`.

### Named Sessions

For multi-session tasks, name your sessions with the `-n` flag (`claude -n auth-refactor`) or `/rename` mid-session. Resume later with `claude --resume auth-refactor`. For parallel workstreams across multiple sessions or branches, see the *Parallel Sessions & Multi-Instance Workflows* section below.

---

## 3. Parallel Sessions & Multi-Instance Workflows

When a single session isn't sufficient — whether you're working on multiple features in parallel, isolating risky work, or running concurrent agents — Claude Code provides several mechanisms for spawning and managing parallel contexts.

### Programmatic Parallel Sessions (CLI)

The `-p` flag starts an isolated session with a single prompt, returning structured output:

```bash
# Start an isolated session and return JSON output
claude -p "analyze the error in auth.ts" --output-format json
```

This captures the session ID and result:

```json
{
  "session_id": "abc123def456",
  "output": "The error occurs because...",
  "status": "completed"
}
```

For spawning multiple async analysis tasks in parallel, use shell backgrounding:

```bash
# Spawn three isolated analysis tasks concurrently
claude -p "audit src/api/*.ts for SQL injection vectors" &
claude -p "check for unhandled promise rejections in src/handlers/*.ts" &
claude -p "list all TODO comments in src/core/*.ts" &
wait
```

Each session gets its own context window, memory, and conversation state. They don't block each other — useful for running time-consuming analysis tasks while keeping your primary session active.

### Resuming Sessions Programmatically

Capture and resume sessions by ID:

```bash
# Run a task and save the session ID
SESSION_ID=$(claude -p "identify performance bottlenecks" --output-format json | jq -r '.session_id')

# Follow up in the same session later
claude --resume "$SESSION_ID" "now optimize the three bottlenecks you found"
```

This pattern is valuable for multi-step workflows where each phase needs results from the previous one — debug phase, then optimization phase, then verification phase — without losing continuity.

### Desktop App Parallel Sessions

In the Claude desktop app, use the **New Session** button in the sidebar to spawn independent contexts. Each gets its own conversation history and can be named for tracking:

- Click the **+** button to create a new session
- Click the session name to rename it (e.g., "auth-refactor", "database-upgrade")
- Switch between sessions using the sidebar tabs
- Sessions persist across app restarts — you can return to them anytime

### Git Worktrees for Branch-Isolated Work

For large refactors or risky experiments that span multiple files, combine parallel sessions with Git worktrees:

```bash
# Create an isolated worktree for the refactor
git worktree add .worktrees/auth-refactor origin/main
cd .worktrees/auth-refactor

# Start a parallel session here
claude -n auth-refactor

# The session is isolated to this branch/directory.
# Changes don't affect the main working tree.
```

When done, clean up:

```bash
cd ../..
git worktree remove .worktrees/auth-refactor
```

Worktrees prevent file-level conflicts — each session modifies a separate filesystem tree. This is essential when running agents or subagents that might overwrite each other's work.

### Concurrency Considerations

**Session Isolation:** Each session is completely isolated. They don't share context, conversation history, or variable state. A variable set in one session is invisible in another.

**File-Level Conflicts:** The developer is responsible for preventing file overwrites. If two parallel sessions modify the same file, the last write wins — Git doesn't help here. Solutions:

- **Use worktrees** (recommended for large changes)
- **Partition by file:** Session A modifies auth.ts, Session B modifies middleware.ts
- **Partition by directory:** Frontend work in one session, backend work in another
- **Use a coordination file:** Track which session owns which work in a `progress.md` or similar

**MCP and Database Concurrency:** MCP servers and external databases handle concurrent access via their own mechanisms. SQLite uses WAL (Write-Ahead Logging) mode to support multiple readers and a single writer. Cloud databases (Postgres, MongoDB) handle multi-client concurrency natively. No special coordination is needed unless your MCP server is stateful.

**Named Sessions for Tracking:** Use the `-n` flag to name sessions, making it easy to identify and resume your parallel workstreams:

```bash
# Start three parallel features
claude -n feature-auth &
claude -n feature-payments &
claude -n feature-reporting &

# Later, resume a specific one
claude --resume feature-auth
```

The session name acts as a human-readable label in your terminal history and `~/.claude/history.jsonl`.

### Best Practices for Parallel Work

- **One primary session per task.** Avoid switching the same session between unrelated tasks — use `/clear` and start fresh instead.
- **Use worktrees for risky changes.** If two sessions might collide on the same files, isolate them on separate branches.
- **Coordinate via files.** Maintain a `PROGRESS.md` or status file that parallel sessions read and update, creating a lightweight handoff mechanism.
- **Clean up named sessions.** After a session completes, delete it to avoid clutter: `rm -rf ~/.claude/projects/<session-dir>/<session-id>.jsonl`
- **Reserve subagents for heavy lifting.** For compute-intensive analysis or batch operations, use the `/batch` skill with isolated subagents rather than spawning raw parallel CLI sessions.

---

## 4. Workflow Orchestration

### The Explore → Plan → Implement Loop

Letting the agent jump straight into file modifications from a vague prompt wastes tokens and causes regressions. Instead:

1. **Explore:** Toggle to Plan mode (`Shift+Tab` to cycle permission modes, or `/plan`). The agent operates read-only, mapping the codebase and analyzing dependencies.
2. **Plan:** Have the agent outline its approach. For complex tasks, export the strategy to a `SPEC.md` file.
3. **Implement:** Run `/clear` to reclaim context, then start a fresh session focused on executing the plan.

### Skills and Custom Commands

Skills are encapsulated workflows (prompts, bash commands, variables) stored in `.claude/skills/`. They remain dormant — consuming zero tokens — until invoked. Claude Code ships with several bundled skills:

| Skill | Purpose |
| :--- | :--- |
| `/batch` | Spawn parallel agents using Git worktrees for large-scale changes |
| `/simplify` | Analyze and simplify complex code |
| `/loop` | Iterate on a task with repeated validation |
| `/debug` | Systematic debugging workflow |
| `/claude-api` | Build applications with the Claude API and Anthropic SDK |

Create your own in `.claude/skills/` — they follow the open Agent Skills standard and can include frontmatter for auto-invocation triggers.

### MCP (Model Context Protocol)

MCP connects Claude Code to external data servers — Jira, Slack, Notion, databases, and more. Configure servers with `/mcp` or in `.claude/.mcp.json`.

Since early 2026, Claude Code uses **Tool Search** (lazy loading) for MCP tools by default, reducing context usage by roughly 95% compared to loading all tool schemas upfront.

**Practical tip:** When a standard CLI exists for the service (like `gh` for GitHub or `aws` for AWS), using it via bash passthrough is often simpler and carries less schema overhead than an MCP server.

### Subagents

For complex or risky work, delegate to subagents rather than running everything in one session. Subagents get isolated context windows and can be constrained to specific tools:

```yaml
# .claude/agents/upgrade-checker.yml
name: upgrade-checker
description: Check for outdated dependencies and report upgrade paths.
tools: [Read, Bash]  # No write access
```

Use `/batch` with Git worktrees to run multiple agents concurrently on different parts of the codebase.

---

## 5. Permission Modes

Claude Code's security boundaries control what the agent can do autonomously. Cycle through modes with `Shift+Tab` (or `Alt+M`).

| Mode | Autonomy | Description |
| :--- | :--- | :--- |
| **Plan** | Read-only | Strict analysis. Blocked from modifying files or executing shell commands. Best for initial exploration. |
| **Default** | Verified | The baseline. Prompts for manual authorization on the first invocation of each tool or command. |
| **Accept Edits** | Semi-autonomous | Auto-approves direct file writes and benign commands (`mkdir`, `mv`). Still prompts for destructive bash operations. |
| **Auto** | Probabilistic | Uses a secondary classifier to dynamically approve safe executions based on user intent. Review recent denials with `/permissions`. |
| **Bypass** | Fully autonomous | Skips nearly all prompts. Use only in ephemeral, isolated environments (Docker, VMs, CI). Requires `--dangerously-skip-permissions` or `--allow-dangerously-skip-permissions`. |

**Key principle:** Deny rules always win, regardless of scope. A managed deny cannot be overridden by any lower-priority allow. Permission rules can be managed interactively with `/permissions`.

---

## 6. Interactive Controls

### Essential Keyboard Shortcuts

| Shortcut | Action | Notes |
| :--- | :--- | :--- |
| `Esc` + `Esc` | Rewind / restore | Opens checkpoint menu to restore code, conversation, or both to a previous point |
| `Shift+Tab` | Cycle permission modes | Rotates through `default` → `acceptEdits` → `plan` → (others if enabled) |
| `Ctrl+C` | Cancel / interrupt | Immediately stops current generation |
| `Ctrl+B` | Background task | Moves running bash commands or agents to background. Tmux users: press twice |
| `Ctrl+R` | Reverse history search | Interactive search through previous prompts |
| `Ctrl+O` | Toggle transcript viewer | Shows detailed tool usage and execution logs |
| `Ctrl+T` | Toggle task list | Show/hide internal task list during multi-step operations |
| `Ctrl+G` | Open in external editor | Edit your prompt in your default text editor |
| `Option+P` / `Alt+P` | Switch model | Change the underlying model mid-session |
| `Option+T` / `Alt+T` | Toggle extended thinking | Enable deeper reasoning for complex tasks |
| `Option+O` / `Alt+O` | Toggle fast mode | Speed-optimized API settings for rapid iteration |
| `Option+Enter` | Multiline input | Insert a line break without sending |

**macOS users:** Option/Alt shortcuts require configuring your terminal's Option key as Meta (iTerm2: Profiles → Keys → set Option key to "Esc+").

### Quick Input Prefixes

| Prefix | Purpose | Example |
| :--- | :--- | :--- |
| `/` | Commands and skills | `/compact`, `/plan fix the auth bug` |
| `@` | File path injection | `@src/lib/auth.ts` — injects the file directly into context |
| `!` | Shell passthrough | `!npm test` — executes directly, output added to context |
| Hold `Space` | Voice input | Push-to-talk dictation (requires `/voice` setup) |

### Rewind and Checkpointing

Claude Code checkpoints before every file edit. When something goes wrong, press `Esc` twice (or run `/rewind`) to open the recovery menu:

- **Restore code and conversation** — erases the mistake entirely
- **Restore code only** — reverts file changes but keeps the transcript (useful for preserving diagnostic insights)
- **Summarize from here** — condenses the timeline from the selected point forward

---

## 7. Model Selection and Cost Management

### Choosing the Right Model

Use `Option+P` / `Alt+P` or `/model` to switch models mid-session. Match the model to the task:

| Tier | Best For |
| :--- | :--- |
| **Opus 4.6** | Complex reasoning, architectural planning, subtle bug diagnosis, large refactors |
| **Sonnet 4.6** | General development work — the default for most tasks |
| **Haiku 4.5** | Fast exploration, isolated subagent tasks, quick lookups |

### Effort Levels

Use `/effort` to control how deeply the model reasons. Options: `low`, `medium`, `high`, `max` (Opus 4.6 only). Lower effort is faster and cheaper; higher effort catches more edge cases. The `max` setting applies to the current session only. Effort can also be set via the `--effort` CLI flag.

### Fast Mode

Toggle with `Option+O` / `Alt+O` or `/fast`. Fast mode runs the same model with speed-optimized API settings — ideal for interactive rapid iteration and live debugging. Turn it off when cost efficiency matters, since enabling fast mode mid-session re-bills the entire prior context at fast mode rates.

### Tracking Costs

Run `/cost` to see token usage and spend for the current session. For organizational tracking, the Admin API provides usage analytics across teams (`GET /v1/organizations/usage_report/claude_code`).

---

## 8. Additional Features

### Side Questions

Use `/btw` for quick tangential questions that don't pollute your main conversation thread.

### Diff Viewer

Run `/diff` to open an interactive viewer showing uncommitted changes and per-turn diffs. Use arrow keys to switch between the current git diff and individual Claude turns.

### Plugins

Plugins extend Claude Code with community and official capabilities. Manage them with `/plugin`:

```bash
claude plugin install code-review@claude-plugins-official
```

### Hooks

Hooks are event-driven shell commands or LLM prompts that run at lifecycle points: `PreToolUse`, `PostToolUse`, `TaskCompleted`, and others. Configured in `settings.json` under the `hooks` key.

A practical use: write a `PreToolUse` hook that truncates large log files before the agent reads them, saving thousands of tokens.

**Security note:** `PreToolUse` hooks run before the permission prompt and can deny tool calls, but they cannot bypass deny rules. Deny rules remain authoritative.

### Remote Control

Run `/remote-control` (or `claude remote-control --name "My Project"`) to make a terminal session controllable from `claude.ai/code` — useful for accessing Claude Code from a browser when you're away from your terminal.

### Sandbox Mode

Available on Linux and macOS, sandbox mode isolates bash command execution. Even if a prompt injection tricks the model into attempting a malicious command, the sandbox constrains the blast radius. Toggle with `/sandbox`.

---

## 9. Best Practices

**Compact proactively, not reactively.** Don't wait until the model starts forgetting rules. Check `/context` and compact when you cross 50–80% utilization.

**Reset when stuck.** If two correction attempts haven't fixed the model's direction, `/clear` and start fresh. Clean context with good instructions outperforms polluted context with patches.

**Use plan mode for exploration.** Toggle to plan mode before letting the agent analyze a new area of the codebase. This prevents accidental modifications while you're still forming your approach.

**Inject files with `@`, not conversation.** Using `@path/to/file.ts` injects file contents directly and efficiently, bypassing the agent's slower autonomous file search.

**Isolate risky work in subagents.** Dependency upgrades, destructive tests, and deep analysis belong in subagents with constrained permissions — not in your primary session.

**Let tooling handle style.** Don't write instructions for formatting rules that Prettier or ESLint enforce. The model reads linter output and self-corrects. Save your instruction budget for rules that require judgment.

**Review diffs before committing.** Run `/diff` after the agent finishes. Trust but verify — even code that compiles and passes tests can contain subtle logic errors.

---

## Further Reading

- [Claude Code Documentation](https://code.claude.com/docs/en/overview) — Official docs, always the most current source
- [CLI Reference](https://code.claude.com/docs/en/cli-reference) — Complete flag and command reference
- [Interactive Mode](https://code.claude.com/docs/en/interactive-mode) — All keyboard shortcuts and input modes
- [Permission Modes](https://code.claude.com/docs/en/permission-modes) — Detailed permission system docs
- [Skills](https://code.claude.com/docs/en/skills) — Creating and using custom skills
- [Hooks Reference](https://code.claude.com/docs/en/hooks) — Event-driven automation

For guidance on writing effective `CLAUDE.md` files, structuring rules, and managing long sessions with context optimization, see the companion *Context Engineering for AI-Assisted Development* guide. For details on local state management, directory structure, and security hardening, see the *Claude Code State Guide*.
