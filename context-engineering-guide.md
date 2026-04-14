# Context Engineering for AI-Assisted Development

## A Practical Guide to CLAUDE.md and Persistent Configuration

---

## Why Context Engineering Matters

Large language models are stateless. Every session starts from scratch — the model has no memory of your codebase, your conventions, or the decisions your team made last quarter. Without guidance, it will guess at your stack, hallucinate dependencies, and produce code that technically works but doesn't belong in your project.

**Context engineering** is the practice of giving the model the right information at the right time so it behaves like a teammate, not a stranger.

The highest-leverage tool for this is `CLAUDE.md` — a configuration file the AI assistant automatically reads at the start of every session. It's not documentation. It's an initialization protocol: a set of persistent instructions that shape how the model reasons, writes code, and makes decisions within your project.

The difference is stark:

- **Without it:** The model defaults to generic patterns, invents plausible-sounding commands, and drifts from your architecture.
- **With it:** The model operates within your constraints from the first message, following your build system, respecting your boundaries, and writing code that matches what's already there.

---

## Core Principle: Less Is More

Context windows are large but not unlimited, and instruction adherence degrades as volume increases. This isn't selective — the model doesn't just forget the newest rules. Compliance drops across *all* directives simultaneously. Research on instruction-following at scale confirms this: as directive count increases, fidelity degrades uniformly across the entire set, not just at the margins.

**The practical ceiling is roughly 100–150 distinct instructions.** Beyond that, you're adding noise that weakens everything else. Keep in mind that the AI tooling itself consumes part of your budget for system prompts, tool schemas, and execution logic — your instructions aren't the only thing in the context window.

Before adding any instruction, apply the necessity test:

> *"Would the model make a concrete mistake without this line?"*

If the model can figure it out from the codebase or from general knowledge, leave it out. Every unnecessary instruction dilutes the ones that matter. Granular file-by-file descriptions, basic syntax reminders for mainstream frameworks, and self-evident practices ("write clean, maintainable code") are the most common offenders.

---

## The Configuration Hierarchy

CLAUDE.md files exist at multiple levels, forming a layered system similar to CSS specificity. When instructions conflict, more specific scopes override more general ones:

| Scope | Location | Versioned? | Purpose |
|---|---|---|---|
| **Enterprise** | OS registry or `/etc/claude-code/` | Managed via MDM | Security policies, compliance, disabled features |
| **Project (shared)** | `./CLAUDE.md` | Yes (committed) | Team-wide standards, build commands, architecture rules |
| **Project (local)** | `./CLAUDE.local.md` | No (git-ignored) | Machine-specific paths, personal overrides |
| **Directory-scoped** | `./src/api/CLAUDE.md` | Yes (committed) | Rules specific to a subdirectory or service |
| **User (global)** | `~/.claude/CLAUDE.md` | No (local profile) | Personal style preferences, universal defaults |

**Conflict resolution works as follows:** arrays (like allowed commands or permitted paths) are concatenated and deduplicated. Objects are deep-merged, with specific keys overriding broader ones. Security parameters are the exception — explicit deny rules always win, regardless of where they appear. A local settings file cannot override a deny command established at the enterprise level.

The key insight: **put shared rules in the project file and commit it.** Your CLAUDE.md should be reviewed in PRs just like any other piece of project infrastructure.

---

## Structuring Your CLAUDE.md: What, Why, How

A good configuration file answers three questions.

### What — The Technology Stack

Tell the model exactly what it's working with. This prevents it from guessing (often wrongly) about your dependencies and runtime. In large monorepos, this section also serves as a topological boundary — it tells the model which services exist and how they relate, preventing unauthorized cross-boundary modifications.

```markdown
## Stack
- TypeScript monorepo managed with Turborepo
- Runtime: Node 20 (NOT Bun — do not suggest migration)
- Backend: Fastify with Zod validation
- Frontend: Next.js 14 (App Router), Tailwind CSS
- Database: PostgreSQL 16 via Drizzle ORM
- Testing: Vitest (unit), Playwright (e2e)
- Auth service is only accessible via the API gateway — do not import from it directly.
```

### Why — The Business Context

A sentence or two about what the project does. This is the most frequently neglected section, but it has the highest leverage. Telling the model it's working on a payment system biases it toward rigorous error handling, defensive coding, and pessimistic type safety. Telling it the code is an internal analytics dashboard lets it prioritize speed over exhaustive validation. The "why" sets the model's risk appetite.

```markdown
## Purpose
B2B invoicing platform handling financial transactions for ~2,000 SMBs.
Data accuracy and auditability are non-negotiable.
```

### How — The Operational Commands

Give the model copy-pasteable commands so it can build, test, and verify its own work autonomously. Without these, the model edits a file and then asks *you* how to test it. With them, it can edit, run the test suite, read the stack trace, and self-correct — all without returning to you for guidance.

```markdown
## Commands
- Install: `pnpm install`
- Dev server: `pnpm dev` (runs on port 3000)
- Test (all): `pnpm test`
- Test (single file): `pnpm test -- path/to/file.test.ts`
- Lint: `pnpm lint`
- Type check: `pnpm typecheck`
- Build: `pnpm build`
- DB migrations: `pnpm db:migrate`
```

---

## Writing Effective Rules

### Be Specific and Actionable

Vague rules get vague compliance. Compare:

| ❌ Vague | ✅ Specific |
|---|---|
| "Write clean code" | "Functions must not exceed 40 lines. Extract helpers when they do." |
| "Handle errors properly" | "Wrap all database calls in try/catch. Log the error with `logger.error()`, then throw a typed `AppError`." |
| "Follow our patterns" | "API routes go in `src/api/[resource]/route.ts`. Each exports GET, POST, etc. as named functions." |

### Encode Process, Not Just Rules

The most valuable instructions force the model to *think before it acts* — mimicking senior engineering habits:

```markdown
## Working Principles
- State your assumptions before writing code. Stop and ask if requirements are ambiguous.
- Do not add abstractions for single-use logic. Write it inline first; refactor only when a pattern repeats.
- Do not modify working code adjacent to your change unless the task explicitly requires it.
- After every multi-file change, run `pnpm typecheck` and fix any errors before moving on.
```

### Mandate Verifiable Execution

Vague goals lead to vague results. When the model finishes a task, it should *prove* the work is correct — not just declare it done. Structure your rules so every phase ends with a deterministic check:

```markdown
## Verification
- After modifying any API route, run `pnpm test -- src/api` and confirm zero failures.
- After any multi-file refactor, run `pnpm typecheck` before proceeding.
- Before declaring a task complete, run `pnpm lint` and fix all warnings.
- Clean up after yourself: remove orphaned variables, redundant imports, and temporary logging.
```

### Define Boundaries Clearly

```markdown
## Out of Scope
- Never modify files in `src/legacy/`. That module is frozen pending a separate rewrite.
- Do not install new dependencies without asking first.
- Do not change the database schema. If a migration seems necessary, stop and describe what you need.
```

---

## Managing Long Sessions

Extended sessions consume the context window with conversation history. When this happens, instruction adherence drops — not because the model forgets your rules, but because they're competing with thousands of tokens of dialogue.

### Proactive Compaction

When a session runs long (or when you notice quality dropping):

1. Ask the model to summarize its current progress into a structured document (e.g., `progress.md`).
2. Clear the session with `/clear` or `/compact`.
3. Start fresh — the model re-reads CLAUDE.md automatically, and you can reference the progress file.

This acts as a garbage collector for your context window: you get a clean, high-fidelity workspace instead of one degraded by old debugging tangents and irrelevant file reads. A good rule of thumb is to compact proactively around 50% context utilization, before quality visibly degrades.

For detailed guidance on managing context window utilization and choosing between `/compact` and `/clear`, see the *Claude Code Operations Guide*.

### Progressive Disclosure

Don't put everything in the root CLAUDE.md. Use modular files for domain-specific rules that only apply to certain parts of the codebase:

**`.claude/rules/api-routes.md`** (with frontmatter):
```yaml
---
paths: ["src/api/**/*.ts"]
---
All API routes must validate input with Zod schemas.
Return standardized error responses using `ApiError.fromZod()`.
Rate limiting is handled at the gateway — do not implement it per-route.
```

The model only absorbs these rules when it's actively working on files matching the glob pattern. Backend database rules don't pollute the context when the model is refactoring a React component, and vice versa. This keeps the base context lean and domain rules precise.

---

## Advanced Patterns

### The Working Context File

For complex, multi-session projects, maintain a living document that tracks state across sessions:

**`CLAUDE-context.md`:**
```markdown
## Current Sprint Goal
Migrate user authentication from session-based to JWT.

## Completed
- [x] JWT utility functions (src/lib/jwt.ts)
- [x] Login endpoint updated
- [x] Middleware rewritten

## In Progress
- [ ] Refresh token rotation (src/api/auth/refresh.ts)

## Known Issues
- Legacy session cleanup job in `src/jobs/cleanup.ts` still references old session table.
  Do not modify — scheduled for removal in Sprint 14.

## Decisions Made
- Chose RS256 over HS256 for key rotation support. See ADR-047.
- Refresh tokens stored in DB, not in cookies, per security review.
```

This gives every new session full situational awareness without bloating CLAUDE.md itself.

### The Memory Bank Pattern

For large or legacy codebases where a single context file is insufficient, use a structured set of linked documents — a "memory bank" that acts as long-term memory for a stateless assistant:

| File | Purpose | Update Cadence |
|---|---|---|
| `CLAUDE-context.md` | Current sprint goals, active tasks, uncommitted work | Every session |
| `CLAUDE-patterns.md` | Established code conventions, preferred paradigms, architectural boundaries | When patterns change |
| `CLAUDE-decisions.md` | Architecture Decision Records — *why* choices were made, what was rejected | When decisions are made |
| `CLAUDE-troubleshooting.md` | Proven solutions to recurring failures and environment-specific bugs | As issues arise |

The key discipline is the **read-then-plan-then-act** workflow: instruct the model to read the memory bank first, outline a proposal in a scratch file, and only write production code after you approve the plan. This prevents the model from charging ahead based on stale or partial understanding.

### Offloading Style to Tooling

Don't waste instructions on formatting rules the linter already enforces. If Prettier handles semicolons, quotes, and indentation, you don't need to tell the model about them — it'll see the linter output and self-correct.

Use lifecycle hooks to run formatters automatically after edits:

```markdown
## After Making Changes
Always run `pnpm lint:fix` after editing files. Do not manually adjust formatting.
```

This frees your instruction budget for rules that require *judgment*, not syntax.

### Subagents and Task Isolation

For complex automation, avoid running everything in a single session. Use subagents to isolate risky or specialized work:

```yaml
# .claude/agents/upgrade-checker.yml
name: upgrade-checker
system_prompt: "Check for outdated dependencies and report upgrade paths. Do not modify any files."
isolation: worktree  # Gets its own copy of the repo
tools: [read, bash]  # No write access
```

The subagent gets a cloned working tree, so it can run destructive tests, attempt dependency upgrades, or perform deep analysis without touching your primary workspace. Your main session stays clean.

For multi-session and parallel workflows, see the *Claude Code Operations Guide* section on *Parallel Sessions & Multi-Instance Workflows*.

The general principle is a **command → agent → skill** pipeline: a slash command captures intent (`/check-upgrades`), routes to a specialized subagent with constrained permissions, which invokes specific skills to do the work. This is safer and more maintainable than embedding complex shell scripts directly in your CLAUDE.md.

### Controlling Reasoning Depth

For tasks that require deeper analysis — debugging subtle concurrency issues, untangling complex type errors, planning large refactors — you can escalate the model's reasoning budget with the `ultrathink` keyword anywhere in your prompt. This forces extended computation for that turn.

For cost-sensitive environments, go the other direction: set `MAX_THINKING_TOKENS` to cap reasoning budget, or disable extended thinking entirely with `CLAUDE_CODE_DISABLE_THINKING=1`. Simple file renames don't need the same cognitive budget as architectural redesigns.

---

## Enterprise Considerations

When deploying AI assistants across an organization, use the managed configuration layer (`managed-settings.json` distributed via MDM) to enforce policies that individual developers cannot override.

### Security

- **Shell execution control:** Set `disableSkillShellExecution: true` to prevent user-sourced skills from executing arbitrary commands. The model sees a policy violation message instead of command output, neutralizing code execution risks from untrusted plugins or prompt injection.
- **Network policy:** Route traffic through corporate proxies via `HTTPS_PROXY` and `ANTHROPIC_BASE_URL`. For environments requiring strict endpoint verification, enforce mutual TLS with `CLAUDE_CODE_CLIENT_CERT` and `CLAUDE_CODE_CLIENT_KEY`.

### Cost Control

- Restrict access to expanded context windows with `CLAUDE_CODE_DISABLE_1M_CONTEXT=1`.
- Cap reasoning budgets globally with `MAX_THINKING_TOKENS`.
- Disable non-essential background traffic (version checks, telemetry) with `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`.

### Governance

These managed settings are immutable from the developer's perspective — they override everything else in the hierarchy. When launched with `forceRemoteSettingsRefresh`, the system blocks startup until managed settings are fetched, failing closed if the network is unavailable.

---

## Common Mistakes

**Restating what the linter enforces.** If ESLint already catches it, don't burn an instruction on it. The model reads linter output and self-corrects.

**Writing aspirational rules.** "Follow SOLID principles" is a platitude, not an instruction. If you mean "no class should exceed 200 lines," say that instead.

**Over-documenting the obvious.** If you're using React 18, you don't need to tell the model what JSX is. Reserve your budget for things it can't infer from the codebase.

**Front-loading everything into the root file.** Use progressive disclosure. Backend rules belong in a backend-scoped file, not in the root CLAUDE.md where they dilute frontend instructions.

**Neglecting the "why."** Two projects can have identical stacks but wildly different risk profiles. A fintech app and a hackathon prototype both use Next.js — but the model should treat them very differently. State the business context.

**Never pruning.** Rules accumulate. Quarterly, review your CLAUDE.md and remove anything the model consistently gets right without being told. Dead rules are noise.

---

## Quick-Start Checklist

When setting up CLAUDE.md for a new project:

1. **Start small.** Five to ten critical rules are better than fifty aspirational ones.
2. **Include build/test commands.** If the model can verify its own work, quality improves dramatically.
3. **Name your stack explicitly.** Don't let the model guess your framework, ORM, or runtime.
4. **State the business context.** One sentence about what the project does and what matters most (speed? correctness? security?).
5. **Set boundaries.** Tell it what *not* to touch. Frozen modules, manual-only migrations, off-limits directories.
6. **Commit the file.** Treat it as shared team infrastructure, not personal config.
7. **Iterate.** When the model makes a recurring mistake, add a rule. When a rule never triggers, remove it.

---

## Example: Minimal CLAUDE.md

```markdown
# Project: Acme Invoicing

B2B invoicing platform. TypeScript monorepo (Turborepo).
Data accuracy and auditability are the top priorities.

## Stack
- Node 20, pnpm
- Backend: Fastify + Drizzle ORM + PostgreSQL 16
- Frontend: Next.js 14 (App Router) + Tailwind
- Tests: Vitest (unit), Playwright (e2e)
- Auth service accessed only via API gateway — never import directly.

## Commands
- `pnpm install` — install deps
- `pnpm dev` — start dev server (port 3000)
- `pnpm test` — run all tests
- `pnpm typecheck` — type check
- `pnpm lint:fix` — lint and auto-fix

## Rules
- Run `pnpm typecheck` after any multi-file change.
- API routes live in `src/api/[resource]/route.ts`.
- All DB queries go through Drizzle. No raw SQL.
- Do not modify anything in `src/legacy/`.
- Do not add dependencies without asking.
- State assumptions before writing code. Ask if requirements are unclear.
- Clean up orphaned variables and imports before marking a task done.
```

Forty-seven lines. Roughly twenty instructions. Enough to keep the model aligned on architecture, workflow, and boundaries — without drowning it in noise.

---

## Further Reading

- [Claude Code Documentation](https://code.claude.com/docs/en/overview) — Official docs, always the most current source
- [Claude Code Operations Guide](./claude-code-operations-guide.md) — Session management, context optimization, parallel workflows, and operational best practices
- [Claude Code State Guide](./claude-code-state-guide.md) — Local state architecture, security hardening, directory structure, and enterprise deployment
