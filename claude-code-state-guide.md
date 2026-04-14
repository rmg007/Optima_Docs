# Claude Code: Local State, Session Data, and Caching Architecture

## How This Guide Relates to Context Engineering

This is a companion to the *Context Engineering for AI-Assisted Development* guide. That document covers how to write effective `CLAUDE.md` files — what to tell the model. This one covers where Claude Code stores its state and how to secure it.

Claude Code is a CLI tool that runs with your user-level filesystem permissions. It reads your codebase, executes shell commands, and persists session data to disk. Understanding where that data lives — and what's in it — is essential for any team that cares about secrets hygiene, disk usage, or reproducible environments.

---

## 1. The Two `.claude` Directories

Claude Code maintains state in two locations. Confusing them is the most common source of misconfiguration.

| Location | Scope | Version-controlled? | Purpose |
|---|---|---|---|
| **`your-project/.claude/`** | Team / project | Yes (selectively) | Shared rules, permissions, commands, skills, agents, MCP config |
| **`~/.claude/`** | Personal / global | Never | Session history, auto-memory, global settings, caches |

The project-level directory holds team configuration — commit it. The global directory holds personal state — never commit it.

### Project Directory (`your-project/.claude/`)

```
your-project/
├── CLAUDE.md                  # Team instructions (committed)
├── CLAUDE.local.md            # Personal overrides (gitignored)
└── .claude/
    ├── settings.json          # Permissions & config (committed)
    ├── settings.local.json    # Personal permission overrides (gitignored)
    ├── .mcp.json              # MCP server configurations
    ├── rules/                 # Modular instruction files
    ├── commands/              # Custom slash commands
    ├── skills/                # Auto-invoked workflows
    ├── agents/                # Specialized subagent personas
    └── hooks/                 # Event-driven automation scripts
```

### Global Directory (`~/.claude/`)

```
~/.claude/
├── CLAUDE.md                  # Global instructions (all projects)
├── settings.json              # Global permissions
├── commands/                  # Personal slash commands
├── projects/                  # Full conversation histories (large)
├── history.jsonl              # CLI command history
├── file-history/              # Before/after snapshots of edits
├── session-env/               # Terminal environment snapshots
├── stats-cache.json           # Aggregated usage statistics
├── paste-cache/               # Buffered text inputs
├── image-cache/               # Buffered image inputs
├── backups/                   # Configuration rollback snapshots (5 deep)
├── debug/                     # Diagnostic logs (dormant by default)
└── todos/                     # Task tracking for long-running work
```

### Project Isolation

Session data is isolated by project using a filesystem-based path-encoding scheme. The project root's absolute path is transformed into a sanitized directory name (e.g., `/Users/bill/My Project` becomes `-Users-bill-My-Project`). This prevents conversations, tool actions, and file edits from bleeding across workspaces.

---

## 2. Session Transcripts and Resumption

Every action within a session is recorded as an append-only event in `~/.claude/projects/<project-path>/<session-id>.jsonl`. When you resume a session (via `/resume` or `--continue`), the system reads this file sequentially from first line to last, reconstructing full awareness of all prior context and tool calls.

Because transcripts are standard JSONL, you can audit them directly with standard tools (`grep`, `jq`, `awk`) without going through the agent's interface.

### File Deletion Consequences

| Path | What You Lose |
|---|---|
| `projects/<id>/<session>.jsonl` | Permanent loss of that session — cannot resume, continue, or fork it |
| `history.jsonl` | Up-arrow prompt recall in the CLI |
| `file-history/` | The `/rewind` checkpoint restore function for past edits |
| `stats-cache.json` | Historical totals from `/cost` and `/stats` (regenerates automatically) |
| `session-env/` | Nothing immediate — regenerates on next session start |
| `debug/`, `todos/`, `paste-cache/` | No user-facing impact; safe for routine cleanup |

### Configuration Files

Two files in `~/.claude/` serve different purposes:

- **`.claude.json`** manages ephemeral state: active session IDs, cache metrics, and OAuth tokens. Never commit this.
- **`settings.json`** defines intentional policies: tool permissions, model selection, cleanup intervals. Safe to version and share.

---

## 3. Session Management

### Naming and Resuming Sessions

For multi-session tasks, name your sessions for easy retrieval:

```bash
claude -n auth-refactor          # Name on startup
/rename auth-refactor            # Rename mid-session
claude --resume auth-refactor    # Resume later
```

This is especially useful for parallel workstreams — separate named sessions for different features prevent cross-contamination.

### Context Window Optimization

The context window is finite. As it fills, costs increase and instruction adherence degrades.

- **Compaction (`/compact`):** Synthesizes conversation history into a dense summary while preserving your current line of work. You can steer it (e.g., `/compact Focus on the auth module`). Compact proactively at 50–80% context utilization — by the time you notice the model "forgetting" rules, fidelity has already dropped.

- **Hard reset (`/clear`):** Complete purge. Use it when transitioning between unrelated tasks, or when the model has gone down a wrong path and two correction attempts haven't fixed it. A clean context with good instructions outperforms a polluted context with patches.

### Preserving State Across Resets

Before compacting or clearing:

1. Ask the model to summarize progress into a structured file (e.g., `progress.md`).
2. Run `/compact` or `/clear`.
3. Start fresh — the model re-reads `CLAUDE.md` automatically, and you can reference the progress file.

For multi-session parallel work, see the *Parallel Sessions & Multi-Instance Workflows* section in the operations guide.

---

## 4. The Checkpoint System

To protect against destructive edits across multiple files, Claude Code takes snapshots before modifying anything.

**How it works:**

1. A user command or agent action signals a file modification.
2. The system creates a new snapshot record for the active session.
3. Pre-edit file contents are copied to `~/.claude/file-history/`, indexed by content hash.
4. The agent executes the write operations.
5. If something goes wrong, you can roll back.

**Rollback options:**

| Action | What It Does |
|---|---|
| `Esc` + `Esc` | Immediate undo of the most recent round of edits |
| `/rewind` (or `/checkpoint`) | Interactive picker for multi-step rollbacks — choose "Restore Code and Conversation" or "Restore Conversation Only" |
| `/diff` | Preview the differences between current state and the saved snapshot before reverting |

Once code changes are committed to Git, the `file-history/` snapshots are redundant and can be safely deleted.

---

## 5. Telemetry and Cost Tracking

LLM API costs add up, especially when switching between model tiers (Opus for complex reasoning, Sonnet for general work, Haiku for fast exploration). Claude Code tracks usage through two complementary mechanisms:

- **`history.jsonl`** is the source of truth — an append-only ledger recording every prompt, tool call, token count, and timestamp.
- **`stats-cache.json`** is a precomputed summary that powers `/cost` and `/stats`. It exists so the system doesn't have to parse the full ledger on every invocation.

The cache can be deleted safely — it regenerates from `history.jsonl` on next access. If `/cost` shows unexpected numbers, deleting `stats-cache.json` is a reasonable first step.

---

## 6. Media Buffers

Large inputs — pasted logs, images, lengthy documents — are stored out-of-band rather than inline in the session transcript. Claude Code hashes the content, writes it to `paste-cache/` or `image-cache/`, and embeds a lightweight reference in the transcript.

A background sweep removes files older than the `cleanupPeriodDays` threshold. Manually clearing these directories is safe — you'll just need to re-upload old content to reference it in a new session.

---

## 7. Security Considerations

### Environment Variable Capture

The `session-env/` directory stores a snapshot of your terminal's environment variables at session start. If your terminal has unencrypted secrets loaded — API keys, AWS credentials, OAuth tokens — those values are written to disk in plaintext and persist after the session ends.

**Realistic risks:**

- **Prompt injection via untrusted code.** A malicious repository or compromised dependency could include instructions that trick the agent into reading `session-env/` and exfiltrating contents.
- **Local filesystem access.** Any process running under your user account can read `~/.claude/session-env/` — it's a predictable path.
- **Accidental persistence.** Temporary tokens and one-off credentials sit on disk until the cleanup sweep runs.

### Plaintext Transcripts

Session transcripts and file-history backups are not encrypted at rest. If the agent modifies a file containing credentials (e.g., `.env`), those credentials are saved in plaintext within `file-history/`.

### Hardening Recommendations

Ordered from easiest to most thorough:

**1. Reduce the retention window:**
```json
{
  "cleanupPeriodDays": 7
}
```

**2. Deny access to sensitive file patterns** (in project or managed `settings.json`):
```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/*.key)",
      "Read(**/secrets/**)",
      "Bash(curl:*)",
      "Bash(wget:*)"
    ]
  }
}
```

**3. Disable hooks in untrusted environments:** Set `disableAllHooks: true` in managed settings.

**4. Use a secrets manager** instead of `.env` files. Tools like HashiCorp Vault, AWS Secrets Manager, or `direnv` with encrypted backends keep secrets out of your shell environment entirely. This is the most effective mitigation.

**5. Use sandbox mode** for CI/CD and non-interactive use (available on Linux and macOS):
```json
{
  "sandbox": {
    "enabled": true
  }
}
```

---

## 8. Storage Growth and Cleanup

The system lacks automatic garbage collection beyond the `cleanupPeriodDays` sweep for media caches. A heavily-used repository can generate gigabytes of data within weeks.

### Pruning Reference

| Target | Strategy | Rationale |
|---|---|---|
| `session-env/` | Delete contents regularly | Removes plaintext secrets; regenerates on next session |
| `paste-cache/`, `image-cache/` | Delete contents regularly | Re-upload needed for old content; high storage impact |
| `file-history/` | Delete after commits are confirmed | Redundant once changes are in Git |
| `projects/**/*.jsonl` | Delete sessions older than 14 days | Retains recent history while discarding stale context |
| `stats-cache.json` | Delete if `/cost` looks wrong | Regenerates automatically from `history.jsonl` |
| `debug/` | Delete logs older than 7 days | Rarely useful after immediate debugging |

### Cleanup Script

```bash
#!/bin/bash
# Periodic Claude Code state cleanup
rm -rf ~/.claude/session-env/*
rm -rf ~/.claude/paste-cache/*
rm -rf ~/.claude/image-cache/*
rm -f ~/.claude/stats-cache.json
echo "Cleared session-env, media caches, and stats cache."
```

---

## 9. Modern Features

### The `rules/` Directory

Instead of putting everything in `CLAUDE.md`, use `.claude/rules/` for modular instruction files. Every markdown file in this directory is loaded alongside `CLAUDE.md`. Rules files support frontmatter globs for path-scoped loading:

```yaml
---
paths: ["src/api/**/*.ts"]
---
Validate all inputs with Zod schemas.
Return errors using ApiError.fromZod().
```

The model only absorbs these rules when working on matching files, keeping context lean.

### Agent Memory

Agents can maintain persistent knowledge using the `memory` frontmatter field, with three scopes:

| Scope | Storage Location | Shared? |
|---|---|---|
| `project` | `.claude/agent-memory/<agent-name>/` | Yes (via Git) |
| `local` | `.claude/agent-memory-local/<agent-name>/` | No (gitignored) |
| `user` | `~/.claude/` | No (personal, all projects) |

### Hooks

Hooks are event-driven commands that run at specific lifecycle points: `PreToolUse`, `PostToolUse`, `StopFailure`, `TeammateIdle`, `TaskCompleted`, and others. They're configured in `settings.json` under the `hooks` key.

`PreToolUse` hooks run *before* the permission prompt and can deny tool calls — but they cannot bypass deny rules. A hook that returns "allow" still gets checked against permissions. Deny rules remain authoritative.

### Sandbox Mode

Available on Linux and macOS, sandbox mode isolates bash command execution. Even if a prompt injection tricks the model into attempting a malicious command, the sandbox constrains the blast radius.

---

## 10. Settings Hierarchy

Settings merge from lowest to highest priority:

| Priority | Source | Use Case |
|---|---|---|
| 1 (highest) | Managed settings (`managed-settings.json`) | Enterprise policies — cannot be overridden |
| 2 | CLI arguments (`--settings`) | Temporary overrides, CI/CD scripts |
| 3 | Local project (`.claude/settings.local.json`) | Personal project overrides (gitignored) |
| 4 | Shared project (`.claude/settings.json`) | Team-wide rules (committed) |
| 5 (lowest) | User global (`~/.claude/settings.json`) | Personal defaults across all projects |

Deny rules always win, regardless of scope. Array settings are concatenated and deduplicated.

### Version Control Strategy

| Path | Action | Why |
|---|---|---|
| `CLAUDE.md` | **Commit** | Core team instructions |
| `.claude/settings.json` | **Commit** | Shared permissions and hooks |
| `.claude/rules/`, `commands/`, `skills/`, `agents/` | **Commit** | Team-shared capabilities |
| `.claude/.mcp.json` | **Commit carefully** | Review for embedded secrets |
| `CLAUDE.local.md` | **Gitignore** | Personal overrides |
| `.claude/settings.local.json` | **Gitignore** | Personal permission overrides |
| `~/.claude/` (everything) | **Gitignore** | Session data, secrets, caches |

---

## 11. Enterprise Deployment

Managed settings (`managed-settings.json`) provide centralized control that individual developers cannot override. They can be delivered via file (`/etc/claude-code/managed-settings.json`), drop-in fragments (`/etc/claude-code/managed-settings.d/`), MDM profiles, or fetched remotely at startup.

### Starter Enterprise Policy

```json
{
  "cleanupPeriodDays": 7,
  "disableAllHooks": true,
  "env": {
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "deny": [
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/*.key)",
      "Read(**/*.pem)",
      "Read(**/secrets/**)",
      "Read(**/.ssh/**)",
      "Bash(sudo:*)",
      "Bash(su:*)",
      "Bash(curl:*)",
      "Bash(wget:*)",
      "Bash(ssh:*)",
      "Bash(nc:*)",
      "Bash(rm -rf /*)"
    ]
  }
}
```

### Managed-Only Settings

These are only effective in managed configuration — placing them in user or project settings has no effect:

| Setting | Purpose |
|---|---|
| `allowManagedPermissionRulesOnly` | Ignores permission rules from non-managed sources |
| `allowManagedHooksOnly` | Only managed hooks can run |
| `strictKnownMarketplaces` | Controls allowed plugin sources |
| `disableBypassPermissionsMode` | Prevents `--dangerously-skip-permissions` |

---

## 12. Diagnostics

The `~/.claude/debug/` directory is dormant by default. Activate it with:

```bash
claude --debug "api,hooks"
# or
claude --debug-file /path/to/debug.log
```

Or use `/debug` during an interactive session. Output includes API call details, hook execution traces, and tool call metadata.
