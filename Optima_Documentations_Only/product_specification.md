# Product Specification

**Version:** 0.1.0 (MVP) | **Date:** April 12, 2026 | **Status:** Planning Complete

---

## 1. Problem

Claude Code starts every session with amnesia. It doesn't remember the weird framework bug you fixed yesterday, the architectural decision you made last week, or the project conventions you've explained five times. Developers compensate by manually maintaining [CLAUDE.md](http://CLAUDE.md) files and re-explaining context — tedious work that compounds with project complexity.

## 2. Solution

Optima is an MCP server that sits alongside Claude Code. It automatically understands your project's structure, remembers solutions to errors, and maintains the intelligence files that Claude Code reads. Every session is smarter than the last — without the developer doing anything extra.

## 3. Philosophy

- **Invisible.** The developer talks to Claude Code as they normally would. Optima works behind the scenes.
- **Plug and play.** Configure once in `.mcp.json`. No setup wizard, no onboarding flow.
- **Portable intelligence.** The generated files (`CLAUDE.md`, `.claude/rules/`) work even if Optima is uninstalled.
- **The intelligence layer bootstraps its own feedback loop.** Optima writes the `.claude/rules/` files that instruct Claude Code to report outcomes back to Optima.
- **Compound quality.** Every interaction makes the next one better.
- **The necessity test.** Every instruction Optima generates into [CLAUDE.md](http://CLAUDE.md) must pass: "Would Claude Code make a concrete mistake without this line?" If the model can figure it out from the codebase or general knowledge, leave it out. Every unnecessary instruction dilutes the ones that matter. Gotchas that never re-trigger, rules the linter already enforces, and obvious conventions are noise.
- **Linter-aware generation.** If the project has ESLint, Prettier, Biome, or similar tooling configured, Optima does NOT generate formatting or style rules into [CLAUDE.md](http://CLAUDE.md). The model self-corrects from linter output. Instruction budget is reserved for rules that require judgment.

## 4. Architecture

```
Developer ↔ Claude Code ↔ Optima (MCP Server)
                              │
                              ├── Project Index (AST + file analysis)
                              ├── Gotcha Ledger (error → solution memory)
                              ├── SQLite Database (.optima/optima.db)
                              └── File Generator (CLAUDE.md, .claude/rules/)
```

### The Inception Pattern

MCP servers can't passively observe Claude Code conversations. They only activate when called. Optima solves this by generating the rules that instruct Claude Code to call it:

Optima generates `.claude/rules/optima-feedback.md` which tells Claude Code:

- Call `optima_get_context` at the start of any task
- Call `optima_memorize` after fixing errors or making architectural decisions

This creates a self-sustaining loop: Optima writes the instructions → Claude Code follows them → Optima learns → Optima improves the instructions.

### Hooks vs Rules (Phase 2 Enhancement)

The MVP inception pattern uses `.claude/rules/` files, which are advisory — the LLM decides whether to follow them. Claude Code also supports **hooks** configured in `.claude/settings.json` that fire deterministically at lifecycle points (`PreToolUse`, `PostToolUse`, `TaskCompleted`, etc.). Hooks execute shell commands automatically, guaranteed.

In Phase 2, Optima can generate hook configurations alongside rules to create a dual-layer feedback system: rules for LLM-directed tool calls ("call `optima_memorize` after fixing an error"), hooks for deterministic automation (auto-run linters after file edits, log session events to a file Optima reads on next context call). For MVP, rules alone are sufficient and simpler.

## 5. MVP Tool Surface

Three tools. No more.

**MCP Tool Search validation:** Since early 2026, Claude Code uses Tool Search (lazy loading) for MCP tools, reducing context usage by ~95%. This means Optima's 3 tools won't bloat Claude Code's context window — they're loaded on-demand only when semantically relevant. However, tool descriptions must be carefully written because Claude Code uses them for discoverability matching.

| Tool | Purpose | When Called |
| --- | --- | --- |
| `optima_get_context` | Returns project context with lazy re-indexing | Start of any task |
| `optima_memorize` | Stores error fixes, rules, patterns, preferences | After completing a task |
| `optima_reindex` | Forces full project re-index | Rarely — after clone/branch switch |

See **03 MCP Tool Contracts** for full Zod schemas and behavior specs.

## 6. Lazy Indexing Lifecycle

Indexing is NOT a tool Claude Code calls explicitly. It happens internally inside `optima_get_context`:

1. Claude Code calls `optima_get_context({path: "src/auth/"})`
2. Optima checks `fs.stat` mtime for every file against stored `mtime_ms`
3. Changed files are re-indexed (content hashed, entities re-extracted via Tree-sitter)
4. Unchanged files are skipped
5. Context is assembled from fresh index and returned

Performance: <500ms incremental, <2s cold start.

## 7. Generated Files

| File | Purpose |
| --- | --- |
| `CLAUDE.md` | Project memory with section ownership markers |
| `.claude/rules/optima-feedback.md` | The inception loop |
| `.gitignore` addition | Adds `.optima/` |

## 8. Tech Stack

| Component | Choice | Rationale |
| --- | --- | --- |
| Runtime | Bun | Fast cold starts, native SQLite |
| Language | TypeScript | MCP SDK ecosystem, strict mode |
| MCP SDK | @modelcontextprotocol/sdk | Official Anthropic SDK |
| Transport | Stdio | No ports, no network |
| Database | SQLite via better-sqlite3 | Zero config, single file, runtime-portable (resolved Q6) |
| ORM | Drizzle | Typed queries, SQLite-native |
| Code Analysis | Tree-sitter | Fast incremental AST parsing |
| Build | tsup | Fast TS bundler |
| Test | Vitest | Fast, TS-native |

## 9. MVP Scope

### Included

- MCP server with Stdio transport
- `optima_get_context` with lazy re-indexing
- `optima_memorize` unified input
- `optima_reindex` escape hatch
- Project analyzer (tech stack, commands)
- File indexer (walk, mtime, hash)
- Entity extractor (Tree-sitter for TypeScript)
- Gotcha Ledger with normalized dedup
- Rules store with directory scoping
- [CLAUDE.md](http://CLAUDE.md) generator with section markers
- Feedback rules generator
- SQLite with Drizzle
- TypeScript language support
- 80%+ test coverage

### Excluded (Phase 2+)

See Phase 2 section below.

## 10. Phase 2 — Intelligent Agent Generation & Optimization

**Prerequisite:** MVP dogfooded for 2+ weeks, Gotcha Ledger has 20+ entries.

### 10.1 Model-Aware Agent Generation

Optima generates `.claude/agents/` files with model recommendations in frontmatter:

- `deep-architect.md` → Opus 4.6 (complex design, architectural planning)
- `quick-fix.md` → Sonnet 4.6 (fast patches, general dev work)
- `fast-lookup.md` → Haiku 4.5 (quick lookups, isolated subagent tasks)
- Domain-specific agents (frontend, api, database) based on detected tech stack

**Agent memory integration:** Claude Code agents support a `memory` frontmatter field with three scopes: `project` (stored in `.claude/agent-memory/`, shared via git), `local` (gitignored), and `user` (global across projects). Optima-generated agents should use `memory: project` scope so accumulated agent knowledge persists and is shareable.

**Key rule:** Never generate agents Optima can't populate with real context.

### 10.2 Sub-Agent Work Assignments

Domain-specific agents get pre-loaded context scoped to their domain:

- Architecture rules filtered to that agent's directories
- Gotchas filtered to that domain
- Entity relationships within that domain
- Test patterns for that layer

### 10.2.1 Path-Scoped Rules Generation

The MVP generates one global `.claude/rules/optima-feedback.md` with `paths: "**/*"`. In Phase 2, Optima generates **directory-specific rules files** using Claude Code's frontmatter glob scoping:

- `.claude/rules/optima-api.md` with `paths: ["src/api/**/*.ts"]` — API-specific gotchas and patterns
- `.claude/rules/optima-db.md` with `paths: ["src/db/**/*.ts"]` — database-specific rules
- `.claude/rules/optima-frontend.md` with `paths: ["src/components/**/*.tsx"]` — frontend conventions

This follows Claude Code's progressive disclosure principle: backend rules don't pollute context when working on frontend code, and vice versa. Each rules file contains only the gotchas, patterns, and feedback instructions relevant to that directory scope.

### 10.3 Token Optimization

**Level 1:** Relevance-based context selection with scoring (path + task_type driven)

**Level 2:** Usage tracking — correlate what context was served vs. what was useful

**Level 3:** Configurable token budgets with allocation percentages

### 10.4 Automated Pruning

Rules and gotchas accumulate over time. Without pruning, Optima's [CLAUDE.md](http://CLAUDE.md) sections will bloat and degrade Claude Code's instruction adherence. Phase 2 adds automated staleness detection:

- **Gotchas** with `hit_count < 2`: already excluded from CLAUDE.md by the MVP necessity test (Q4). Phase 2 adds configurable thresholds and staleness-based pruning from the database itself (not just CLAUDE.md exclusion).
- **Architecture rules** with no references in 60+ days: flagged for review. Optima can surface these in `optima_get_context` output as "stale rules — consider removing."
- **Patterns** with only 1 recorded occurrence and no reuse in 30+ days: excluded from [CLAUDE.md](http://CLAUDE.md) generation.
- **Periodic sweep:** On each `optima_get_context` call, check for stale entries. No background process — piggyback on the lazy indexing model.
- The developer can override pruning by adding a `pinned: true` flag via `optima_memorize` for rules that should never expire.

## 11. Success Criteria

### Week One (MVP)

1. Claude Code calls `optima_get_context` at task start without prompting
2. Claude Code calls `optima_memorize` after fixes without prompting
3. Gotcha Ledger has 10+ real entries
4. [CLAUDE.md](http://CLAUDE.md) stays in sync automatically
5. Repeated errors resolve faster via historical fixes

### Month One

1. Gotcha Ledger is Claude Code's first reference for errors
2. Accumulated rules produce more consistent code
3. Developer hasn't manually edited [CLAUDE.md](http://CLAUDE.md) in 2 weeks

### Phase 2 (Month Two)

1. Generated sub-agents used for domain-specific tasks
2. Context relevance measurably improves
3. Token usage decreases with same quality
4. Agent model assignments match task complexity automatically

## 12. Claude Code Integration Boundaries

Optima must understand Claude Code's state architecture to operate safely alongside it.

**Settings hierarchy (highest to lowest priority):**

1. Managed settings (enterprise MDM) — immutable
2. CLI arguments (`--settings`)
3. `.claude/settings.local.json` — personal, gitignored
4. `.claude/settings.json` — team-shared, committed
5. `~/.claude/settings.json` — user global defaults

**Implication for Optima:** Optima writes `.claude/rules/optima-feedback.md` (safe — rules are additive). In Phase 2, if Optima generates hook configurations, they must go into `.claude/settings.json` (the committed, team-shared file) — never into `.claude/settings.local.json` (personal) or managed settings (enterprise). Optima must read the existing `settings.json`, merge its hooks into the existing `hooks` object, and write back — never overwrite the entire file.

**Enterprise graceful degradation:** Enterprise-managed Claude Code environments can set `disableAllHooks: true` or `allowManagedHooksOnly: true`, which silently blocks all non-managed hooks. If Optima's Phase 2 hook generation is deployed in such an environment, the hooks will be ignored with no error. Optima must not depend on hooks for core functionality — the rules-based inception pattern (MVP) must remain the primary feedback mechanism, with hooks as a best-effort enhancement. Optima should detect managed environments (check for managed settings presence) and skip hook generation if hooks are disabled.

**MCP server configuration placement:**

- Project-scoped: `.mcp.json` at project root (recommended for team projects)
- User-scoped: `~/.claude.json` (for personal, cross-project use)

**Paths Optima must never read, index, or access:**

- `~/.claude/session-env/` — plaintext environment variable snapshots
- `~/.claude/projects/` — full session transcript history (JSONL)
- `~/.claude/file-history/` — before/after edit snapshots
- `~/.claude/paste-cache/`, `~/.claude/image-cache/` — media buffers
- `CLAUDE.local.md` — developer's personal instruction overrides (gitignored, takes priority over [CLAUDE.md](http://CLAUDE.md)). Optima must never read, modify, or generate this file. It is the developer's personal space.
- `.claude/settings.local.json` — personal permission overrides (gitignored)
- Any `.env` file or `**/secrets/**` path in the project

**Paths Optima reads but does not modify:**

- `.claude/settings.json` — to understand existing hooks and permissions (Phase 2)
- `package.json`, `tsconfig.json`, `pyproject.toml` — for tech stack detection

## 13. What Optima is NOT

- **Not a code generator.** It generates config and intelligence files.
- **Not an agent framework.** No CrewAI, no orchestration.
- **Not a cloud service.** Everything runs locally.
- **Not an editor extension.** No VS Code dependency.
- **Not a linter.** It teaches Claude Code the rules, doesn't enforce them.

## 14. Open Questions — ALL 16 RESOLVED

**All 16 questions resolved on April 12, 2026. These are binding design decisions.**

| Q | Topic | Decision | Scope | Enforced In |
|---|-------|----------|-------|------------|
| Q1 | Auto-generate on cold start? | YES — first `optima_get_context` triggers full init. 2s cold start acceptable for plug-and-play. | MVP | 01 (Cold Start Sequence), 02 (DB Init), 03 (tool step 1) |
| Q2 | Existing CLAUDE.md handling? | Append below markers, never touch content above first marker or outside markers. | MVP | 04 (Regeneration Logic step 4) |
| Q3 | Error dedup via hash collision? | Normalize for hash, store sanitized original. On collision: store both entries, emit `HASH_COLLISION` warning. | MVP | 03 (normalizeError, sanitizeError, error taxonomy) |
| Q4 | hit_count threshold for CLAUDE.md? | `hit_count >= 2` = proven value. Gotchas with `hit_count < 2` excluded from CLAUDE.md generation. This is the single authoritative threshold — no secondary age-based filter. All gotchas remain in DB for matching regardless. | MVP | 04 (necessity test table, instruction budget enforcement) |
| Q5 | .gitignore parsing? | `ignore` npm package (^7.0.0). Never hand-rolled regex. | MVP | 00 (Tech Stack), 03 (gitignore exclusions) |
| Q6 | Bun vs Node SQLite? | `better-sqlite3` for runtime portability (works on both Bun and Node). If Bun is proven reliable, swap to `bun:sqlite` later. | MVP | 00 (Tech Stack, DO NOT), 02 (driver note) |
| Q7 | Domain detection? | Dependency + directory signals (both required). | Phase 2 | Product Spec only |
| Q8 | Agent tools? | Explicit allowlists per agent. | Phase 2 | Product Spec only |
| Q9 | Token budget? | chars / 4 approximation. | Phase 2 | Product Spec only |
| Q10 | JSON merge for hooks? | Deep-merge into `.claude/settings.json` hooks object. Preserve all non-Optima entries. Never overwrite full file. | Phase 2 | Product Spec section 12 |
| Q11 | Enterprise detection? | Defer. Rules-based inception (MVP) is primary. Hooks are best-effort Phase 2 enhancement. | Phase 2 | Product Spec section 12 |
| Q12 | DB init timing? | Lazy — deferred to first tool call. MCP server starts immediately on stdio, DB created on first `optima_get_context`. | MVP | 02 (DB Init & Lifecycle), 01 (Cold Start step 3) |
| Q13 | Path normalization? | Always forward slashes in SQLite and tool output. Normalize on ingestion via `path.replace(/\\\\/g, '/')`. Symlinks skipped via `fs.lstat`. | MVP | 00 (Ground Rule 7), 03 (tool step 1) |
| Q14 | Gotcha retrieval? | Hierarchical directory prefix match + file array match. Dedup by ID, sort by `hit_count` desc, cap at 10. | MVP | 03 (Gotcha Retrieval Strategy) |
| Q15 | linterDetected format? | JSON array of strings (e.g., `'["eslint","prettier"]'`). null if none detected. | MVP | 02 (field comment on projectMeta) |
| Q16 | Schema migration? | Hand-written SQL in `src/db/migrations/`, numbered files (`001_initial.ts`), programmatic runner with transactions. Drizzle Kit dev-only. | MVP | 02 (Schema Migration Strategy) |
