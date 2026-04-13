# 00 Start Here

**Read this file first. Then read the other four documents in order.**

## North Star

An MCP server that indexes your project, remembers what errors you've hit and how you fixed them, and maintains a [CLAUDE.md](http://CLAUDE.md) that makes every Claude Code session smarter than the last.

## Reading Order

| Order | File | Purpose |
| --- | --- | --- |
| 1 | `00_start_here.md` | This file. Rules, tech stack, constraints. |
| 2 | `01_product_spec_mvp.md` | What Optima is, how it works, what to build, project structure. |
| 3 | `02_data_model_and_schema.md` | Drizzle schemas, TypeScript interfaces, SQLite tables. Copy verbatim. |
| 4 | `03_mcp_tool_contracts.md` | Zod input/output schemas for all 3 tools. Copy verbatim. |
| 5 | `04_inception_payload.md` | Exact file content Optima generates in host projects. Copy verbatim. |

## Tech Stack (Locked)

| Component | Choice | Command |
| --- | --- | --- |
| Runtime | Bun | `bun run`, `bun test`, `bunx` — never `npm`, `npx`, or `node` |
| Language | TypeScript | Strict mode. No `any`. |
| MCP SDK | `@modelcontextprotocol/sdk` | Stdio transport only. |
| Database | SQLite via `better-sqlite3` | Single file at `.optima/optima.db`. Uses better-sqlite3 for runtime portability (works on both Bun and Node). See resolved Q6. |
| ORM | Drizzle ORM (`drizzle-orm`  • `drizzle-kit`) | Typed queries. SQLite dialect. |
| AST Parsing | `tree-sitter`  • `tree-sitter-typescript` | For entity extraction. |
| Gitignore Parsing | `ignore` (npm, ^7.0.0) | Correct gitignore semantics. Not hand-rolled regex (resolved Q5). |
| Bundler | `tsup` | Single entry point bundle. |
| Test Runner | Vitest | `bun run test` via vitest. |

## Project Configuration Files

Copy these configuration files verbatim when bootstrapping the project.

### `package.json`

```json
{
  "name": "optima-mcp",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "bin": {
    "optima-mcp": "dist/index.js"
  },
  "scripts": {
    "build": "tsup",
    "dev": "bun run src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "tsc --noEmit",
    "db:generate": "drizzle-kit generate"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "better-sqlite3": "^11.7.0",
    "drizzle-orm": "^0.39.0",
    "ignore": "^7.0.0",
    "tree-sitter": "^0.22.0",
    "tree-sitter-typescript": "^0.23.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.0",
    "drizzle-kit": "^0.30.0",
    "tsup": "^8.3.0",
    "typescript": "^5.7.0",
    "vitest": "^3.0.0"
  }
}
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": true,
    "sourceMap": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "test"]
}
```

### `tsup.config.ts`

```typescript
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm"],
  target: "node20",
  outDir: "dist",
  clean: true,
  dts: true,
  sourcemap: true,
  banner: { js: "#!/usr/bin/env node" },
  external: ["better-sqlite3", "tree-sitter", "tree-sitter-typescript"],
});
```

### `vitest.config.ts`

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    include: ["test/**/*.test.ts"],
    coverage: {
      provider: "v8",
      include: ["src/**/*.ts"],
      exclude: ["src/index.ts"],
      thresholds: { lines: 80, functions: 80, branches: 70 },
    },
  },
});
```

### `drizzle.config.ts` (dev-only — NOT used at runtime)

```typescript
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dialect: "sqlite",
  dbCredentials: { url: ".optima/optima.db" },
});
```

### Project bootstrap commands

```bash
bun init -y
bun add @modelcontextprotocol/sdk better-sqlite3 drizzle-orm ignore tree-sitter tree-sitter-typescript zod
bun add -d @types/better-sqlite3 drizzle-kit tsup typescript vitest
```

---

## Canonical Module Map

Every module is referenced by these paths across the corpus. If a stale document references a different structure, trust this table.

| File Path | Purpose | Specified In |
|---|---|---|
| `src/index.ts` | Entry point — creates and starts MCP server | Doc 03 |
| `src/server.ts` | Tool registration and dispatch | Doc 03 |
| `src/schemas.ts` | Zod input/output schemas for all 3 tools | Doc 03 |
| `src/types.ts` | TypeScript interfaces, OptimaError class, error codes | Doc 02 |
| `src/tools/get-context.ts` | optima_get_context implementation (11-step behavior) | Doc 03 |
| `src/tools/memorize.ts` | optima_memorize implementation | Doc 03 |
| `src/tools/reindex.ts` | optima_reindex implementation | Doc 03 |
| `src/indexer/project-analyzer.ts` | Tech stack detection, command discovery, linter detection | Doc 02 |
| `src/indexer/file-indexer.ts` | File walking, mtime checking, hash computation | Doc 03 |
| `src/indexer/entity-extractor.ts` | Tree-sitter AST parsing → entities | Doc 03 |
| `src/memory/gotcha-ledger.ts` | Error → solution CRUD with hierarchical matching | Doc 03 |
| `src/memory/rules-store.ts` | Architectural rules CRUD with directory scoping | Doc 03 |
| `src/memory/error-normalizer.ts` | sanitizeError + normalizeError pipeline for dedup | Doc 03 |
| `src/generator/claude-md.ts` | CLAUDE.md generation with section markers | Doc 04 |
| `src/generator/feedback-rules.ts` | .claude/rules/optima-feedback.md generation | Doc 04 |
| `src/db/schema.ts` | Drizzle schema (7 tables) | Doc 02 |
| `src/db/migrations.ts` | Schema versioning and migration runner | Doc 02 |
| `src/db/migrations/001_initial.ts` | Initial migration — CREATE TABLE for all 7 tables | Doc 02 |
| `src/db/migrations/index.ts` | Exports ordered migration list | Doc 02 |
| `src/db/connection.ts` | Database lifecycle (lazy init, WAL, corruption recovery) | Doc 03 |
| `src/utils/hasher.ts` | SHA-256 content hashing | Doc 01 |
| `src/utils/paths.ts` | Path normalization, .gitignore matching | Doc 00 (Ground Rule 7) |
| `src/utils/errors.ts` | Error taxonomy (OptimaError) | Doc 03 |

---

## DO NOT IMPLEMENT — Superseded Content

If any pre-review-pass version of a section conflicts with the current docs, the current docs win. The following content areas were revised during the first review pass (Q12–Q16 additions). If you encounter older versions:

| Topic | Old Location/State | Current Canonical Location |
|---|---|---|
| DB initialization timing | Previously unspecified | Doc 02 "Database Initialization & Lifecycle" section |
| Path normalization | Previously unspecified | Doc 00 Ground Rule 7 + Doc 03 tool step 1 |
| Gotcha retrieval strategy | Previously unspecified | Doc 03 "Gotcha Retrieval Strategy" section |
| linterDetected format | Previously ambiguous | Doc 02 field comment on projectMeta table |
| Schema migration strategy | Previously unspecified | Doc 02 "Schema Migration Strategy" section |
| Cold start bootstrap | Previously unspecified | Doc 01 "Cold Start Bootstrap Sequence" (8 steps) |
| Entity extraction edge cases | Previously incomplete | Doc 03 "Entity Extraction Edge Cases" list (12 cases) |
| Malformed marker handling | Previously unspecified | Doc 04 "Malformed marker handling" section |
| Preferences section | Previously missing | Doc 04 "Section: preferences" template |
| CLAUDE.md regeneration triggers | Previously limited to project-wide rules only | Doc 03 step 5 + Doc 04 header (all memorize types trigger regeneration) |
| Error sanitization | Previously absent | Doc 03 sanitizeError() function (runs before normalization and storage) |
| Schemas file location | Previously implicit | Doc 03 server.ts imports from `./schemas.js` — create `src/schemas.ts` |

---

## Ground Rules

1. **No file exceeds 300 lines.** Split into separate modules if approaching the limit.
2. **Copy verbatim.** When a spec provides a fenced TypeScript, SQL, or Zod block, reproduce it byte-for-byte. Do not rename fields, reorder properties, or add fields.
3. **All cross-module inputs validated with Zod.** No exceptions.
4. **All file I/O must be async.** Never use synchronous `fs` methods in the hot path. **Exceptions (synchronous allowed):**
   - `better-sqlite3` database queries — synchronous by design, single-threaded (see Doc 02 Concurrency section).
   - `fs.mkdirSync('.optima', { recursive: true })` — one-time cold start path, not hot path (see Doc 02 DB Init step 1).
   - `fs.lstatSync()` for symlink detection — acceptable in file walker if async `fs.lstat()` adds complexity with no measurable benefit.
5. **Tests are mandatory.** Every module ships with tests. Minimum 80% coverage.
6. **Errors use a closed taxonomy.** Every thrown error must use one of the error codes defined in `03_MCP_TOOL_CONTRACTS.md`. Never throw bare `new Error('...')`.
7. **Paths are always forward-slash normalized.** All file paths stored in SQLite — including `file_index.path`, `gotchas.directory`, `rules.directory`, and all paths inside `files` JSON arrays in `gotchas` and `rules` tables — use forward slashes (`src/utils/foo.ts`), regardless of OS. Normalize on ingestion using `path.replace(/\\/g, '/')`. Tool output paths are also forward-slash normalized. Symlinks are NOT followed — if `fs.lstat` reports a symlink, skip the file silently. This avoids infinite loops and ensures deterministic indexing.

## Claude Code State Awareness

Optima operates alongside Claude Code. Understanding Claude Code's internal state architecture is critical to avoid conflicts and respect boundaries.

**Claude Code's local state lives in two places:**

- `~/.claude/` — Global user state (session transcripts, file-history, caches, global settings). NEVER index this.
- `<project>/.claude/` — Project-level config (settings.json, agents/, commands/, skills/, rules/). Optima WRITES to this directory.

**Settings hierarchy (highest to lowest priority):**

1. Managed settings (enterprise MDM) — cannot be overridden
2. CLI arguments
3. `.claude/settings.local.json` — personal, gitignored
4. `.claude/settings.json` — team-shared, committed
5. `~/.claude/settings.json` — user global defaults

**Security-sensitive paths Optima must NEVER index:**

- `~/.claude/session-env/` — contains plaintext environment variables (API keys, tokens)
- `~/.claude/projects/` — full session transcripts in JSONL
- `~/.claude/file-history/` — before/after file edit snapshots
- `CLAUDE.local.md` — developer's personal instruction overrides (gitignored, higher priority than [CLAUDE.md](http://CLAUDE.md)). Optima must never read, modify, or generate this file. It is the developer's personal space.
- `.claude/settings.local.json` — personal permission overrides (gitignored)
- Any `.env` file or `**/secrets/**` path

**Hooks (Phase 2 only):** Claude Code supports lifecycle hooks (`PreToolUse`, `PostToolUse`, `TaskCompleted`, etc.) configured in `.claude/settings.json`. These could strengthen the inception pattern by auto-triggering `optima_memorize` calls. However, hooks execute shell commands — they cannot directly invoke MCP tools via stdio. Phase 2 will explore a thin CLI bridge (`bunx optima-hook log-outcome ...`) that hooks can call.

**Agent memory (Phase 2 only):** Claude Code agents support a `memory` frontmatter field with scopes (`project`, `local`, `user`). Phase 2 generated agents should use `memory: project` for persistent cross-session knowledge stored in `.claude/agent-memory/<agent-name>/`.

## DO NOT

- **DO NOT implement a file watcher.** Optima uses lazy indexing via `fs.stat` mtime checks inside `optima_get_context`. No background processes, no `chokidar`, no `fs.watch`.
- **DO NOT generate Claude Code hooks in MVP.** Hooks (`.claude/settings.json` `PostToolUse`/`TaskCompleted` events) are a Phase 2 enhancement. MVP uses `.claude/rules/` only for the inception pattern.
- **DO NOT modify `.claude/settings.json` permissions or deny rules.** Optima never writes permission rules. Security boundaries are the developer's domain.
- **DO NOT index Claude Code's internal state.** Never read or index `~/.claude/`, `.claude/session-env/`, `.claude/projects/`, or `.claude/file-history/`.
- **DO NOT build a web UI, CLI interface, or REST API.** Optima is a headless MCP server. Stdio transport only.
- **DO NOT implement Phase 2 features.** No agent generation, no token optimization scoring, no adaptive prompts, no hook-based feedback. If it's not in `01_PRODUCT_SPEC_MVP.md` included list, don't build it.
- **DO NOT use `npm`, `npx`, or `node` commands.** Bun only.
- **DO NOT use `bun:sqlite` directly.** Use `better-sqlite3` for runtime portability (resolved Q6 — allows fallback to Node if Bun's MCP spawner has issues). If Bun is proven reliable, can swap to `bun:sqlite` as a future optimization.
- **DO NOT install `axios`, `node-fetch`, `express`, `react`, or `electron`.** This project has no HTTP client, no HTTP server, no frontend.
- **DO NOT create a separate database file for learning/memory.** Everything lives in one SQLite database: `.optima/optima.db`.
- **DO NOT guess at schema fields, Zod shapes, or error codes.** Every one is defined explicitly in the spec documents.

## When Stuck

If a spec is ambiguous or two documents seem to contradict each other:

1. Trust `02_DATA_MODEL_AND_SCHEMA.md` for data shapes.
2. Trust `03_MCP_TOOL_CONTRACTS.md` for tool behavior.
3. Trust `04_INCEPTION_PAYLOAD.md` for generated file content.
4. If still unclear, stop and ask. Do not improvise.