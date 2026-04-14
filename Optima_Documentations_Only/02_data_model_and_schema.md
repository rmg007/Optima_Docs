# 02 Data Model and Schema

All data lives in ONE SQLite database: `.optima/optima.db` at the project root.

**SQLite driver:** `better-sqlite3` (not `bun:sqlite`) for runtime portability — works on both Bun and Node.js. See resolved Q6 in Product Specification.

## Database Initialization & Lifecycle

**When:** Database is created on the first `optima_get_context` call, as part of the cold start bootstrap sequence (see `01_product_spec_mvp.md` "Cold Start Bootstrap Sequence"). The MCP server process starts and listens on stdio immediately — database creation is deferred until first tool invocation.

**Owner:** `src/db/connection.ts` exports a `getDatabase()` function that lazily initializes the connection. First call creates `.optima/` directory, opens/creates `optima.db`, and runs migrations. Subsequent calls return the cached connection. The tool handlers call `getDatabase()` — they never manage the connection directly.

**Initialization steps (idempotent):**
1. `fs.mkdirSync('.optima', { recursive: true })` — sync is acceptable here (one-time cold path, not hot path).
2. Open `better-sqlite3` connection with `{ fileMustExist: false }`.
3. Enable WAL mode: `PRAGMA journal_mode=WAL` — allows concurrent reads during writes, improves performance.
4. Enable foreign keys: `PRAGMA foreign_keys=ON`.
5. Check `schema_version` table. If missing or version < current, run migrations (see Schema Migration Strategy below).
6. Write `.optima/.gitignore` with content `*` (if not already present).

**Corruption recovery:** If the database file is corrupted (detected by `better-sqlite3` throwing `SqliteError` with code `SQLITE_CORRUPT` or `SQLITE_NOTADB` on open), delete `.optima/optima.db`, `.optima/optima.db-wal`, and `.optima/optima.db-shm`, then reinitialize from scratch. All data is rebuildable — the file index comes from disk, gotchas/rules are the only data loss, and that's acceptable for a local tool.

**Concurrency:** `better-sqlite3` is synchronous and single-threaded — only one write can happen at a time within the process. If Claude Code sends two tool calls concurrently, the MCP SDK's stdio transport serializes them (stdio is a single message stream). There is no concurrent write contention within a single Optima process. If two separate Claude Code sessions point at the same project, SQLite's WAL mode handles reader/writer concurrency correctly, but two writers will serialize via SQLite's file-level lock.

---

## Database Initialization & Pruning

### Automatic Record Pruning

On cold start (first database initialization), the connection layer prunes stale records before returning the database connection:

1. **`generation_log`:** Keeps only the last 50 entries **per distinct `file_path`**. Older entries are deleted.
   - This is per-file, not global — `CLAUDE.md` regeneration history and `.claude/rules/` regeneration history are tracked and pruned separately.

2. **`task_outcomes`:** Keeps only the last 50 entries **globally** (across all directories).
   - Single global cap, not per-directory.

**When pruning runs:** Once, during `initializeDatabase()`, before the connection is returned to the caller. Not during tool calls.

**CLAUDE.md impact:** The instruction budget algorithm (`claude-md.ts`) always queries from the unpruned live data. Recent gotchas and task insights are never silently lost — only records older than the 50-entry window are pruned.

**Observable in logs:**
```json
{ "component": "db", "msg": "pruned old records", "generation_log_rows_deleted": 12, "task_outcomes_rows_deleted": 0 }
```
This line appears at DEBUG level during every cold start. If it shows non-zero deletes, the database has grown beyond the retention window.

## Drizzle Schema Definitions

Copy these into `src/db/schema.ts` verbatim.

```tsx
import { sqliteTable, text, integer, index, uniqueIndex } from "drizzle-orm/sqlite-core";
import { sql } from "drizzle-orm";

// ── Project Metadata (singleton — one row) ────────────────────────────

export const projectMeta = sqliteTable("project_meta", {
  id: integer("id").primaryKey().default(1),
  name: text("name").notNull(),
  rootPath: text("root_path").notNull(),
  techStack: text("tech_stack").notNull().default("[]"),
  buildCommand: text("build_command"),
  testCommand: text("test_command"),
  lintCommand: text("lint_command"),
  projectPurpose: text("project_purpose"),
  linterDetected: text("linter_detected"), // JSON array of strings, e.g. '["eslint","prettier"]'. See AUTHORITATIVE linter config list below. null if none detected.
  keyDependencies: text("key_dependencies"), // JSON array of {name, version, detected_from} objects. Parsed from package.json dependencies matching DEPENDENCY_MAP. null if none detected.
  lastFullIndex: text("last_full_index"),
  updatedAt: text("updated_at").notNull().default(sql`(datetime('now'))`),
});

// ── File Index ────────────────────────────────────────────────

export const fileIndex = sqliteTable("file_index", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  path: text("path").notNull().unique(),
  mtimeMs: integer("mtime_ms").notNull(),
  sizeBytes: integer("size_bytes").notNull(),
  language: text("language"),
  hash: text("hash").notNull(),
  indexedAt: text("indexed_at").notNull().default(sql`(datetime('now'))`),
}, (table) => ({
  pathIdx: uniqueIndex("idx_file_path").on(table.path),
}));

// ── Code Entities ─────────────────────────────────────────────

export const entities = sqliteTable("entities", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  fileId: integer("file_id").notNull().references(() => fileIndex.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  kind: text("kind").notNull(),
  lineStart: integer("line_start").notNull(),
  lineEnd: integer("line_end").notNull(),
  signature: text("signature"),
  exported: integer("exported", { mode: "boolean" }).notNull().default(false),
  description: text("description"),
  createdBySession: text("created_by_session"),          // Session ID that first indexed this entity (NULL if from initial scan)
  lastModifiedContext: text("last_modified_context"),     // Brief context from the memorize call that last touched this entity's file (NULL if never modified via optima_memorize)
}, (table) => ({
  fileIdx: index("idx_entities_file").on(table.fileId),
  nameIdx: index("idx_entities_name").on(table.name),
}));

// ── Entity Provenance Tracking ────────────────────────────────
// When `optima_memorize` is called with an `error_fix` or `pattern` type
// that includes files in its `files` array, and those files contain entities
// in the `entities` table, update the `last_modified_context` field on those
// entities with a brief summary (e.g., "Fixed null reference in auth flow"
// truncated to 200 chars). This gives future `optima_get_context` calls
// richer context: instead of "function fetchUser exists at line 42", the
// context can include "function fetchUser exists at line 42 — last modified
// to fix null reference in auth flow."

// ── Field Surfacing Policy ───────────────────────────────────
// Several database columns are stored for future use but intentionally
// excluded from MVP tool output. The table below documents which fields
// are surfaced in TypeScript interfaces / Zod output schemas and which
// are stored only for Phase 2.
//
// | Field         | Table    | In DB | In TypeScript Interface | In Zod Output | Reason |
// |---------------|----------|-------|------------------------|---------------|--------|
// | description   | entities | Yes   | Yes (EntitySummary)    | Yes           | JSDoc content provides immediate value for entity context. |
// | tags          | gotchas  | Yes   | No                     | No            | Freeform tags for Phase 2 search. Not needed in MVP tool output. |
// | files         | rules    | Yes   | No                     | No            | Stored for provenance. Rules are scoped by `directory`, not individual files. |
// | tags          | rules    | Yes   | No                     | No            | Same as gotchas tags. Phase 2 search feature. |
// | dep_version   | gotchas  | Yes   | Yes (GotchaEntry)      | Yes           | Enables staleness detection when dependency versions change. |
// | key_deps      | proj_meta| Yes   | Yes (via dependency_context)| Yes      | Surfaces detected dependency versions for context + staleness checks. |
//
// COMPUTED FIELDS (not stored in DB — calculated at query time in tool handlers):
// | Field             | Interface        | Computed From | Notes |
// |-------------------|------------------|---------------|-------|
// | possibly_outdated | GotchaEntry      | dependency_version vs project_meta.key_dependencies | See Doc 03 step 8 |
//
// If you add a column to a table, decide whether it belongs in tool output.
// If not, add it to this table so the omission is documented, not accidental.

// ── Gotcha Ledger ─────────────────────────────────────────────

export const gotchas = sqliteTable("gotchas", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  errorText: text("error_text").notNull(),
  errorHash: text("error_hash").notNull(),
  resolution: text("resolution").notNull(),
  rootCause: text("root_cause"),  // Why the error occurred, not just what fixed it (optional)
  dependencyVersion: text("dependency_version"),  // e.g. "drizzle-orm@^0.38.0" — dep name@version when gotcha was recorded (optional)
  files: text("files").notNull().default("[]"),
  directory: text("directory"),
  tags: text("tags").notNull().default("[]"),
  hitCount: integer("hit_count").notNull().default(0),
  createdAt: text("created_at").notNull().default(sql`(datetime('now'))`),
  updatedAt: text("updated_at").notNull().default(sql`(datetime('now'))`),
}, (table) => ({
  hashIdx: index("idx_gotchas_hash").on(table.errorHash),
  dirIdx: index("idx_gotchas_directory").on(table.directory),
}));

// ── Rules and Patterns ────────────────────────────────────────

export const rules = sqliteTable("rules", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  type: text("type").notNull(),
  content: text("content").notNull(),
  rationale: text("rationale"),
  directory: text("directory"),
  files: text("files").notNull().default("[]"),
  tags: text("tags").notNull().default("[]"),
  createdAt: text("created_at").notNull().default(sql`(datetime('now'))`),
  updatedAt: text("updated_at").notNull().default(sql`(datetime('now'))`),
}, (table) => ({
  typeIdx: index("idx_rules_type").on(table.type),
  dirIdx: index("idx_rules_directory").on(table.directory),
}));

// ── Task Outcomes ─────────────────────────────────────────────

export const taskOutcomes = sqliteTable("task_outcomes", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  task: text("task").notNull(),                                     // "Add JWT auth to /api/users"
  outcome: text("outcome").notNull(),                               // "success" | "partial" | "abandoned"
  learnings: text("learnings"),                                     // "Had to use RS256 not HS256 because..."
  files: text("files").notNull().default("[]"),                     // JSON array of file paths involved
  directory: text("directory"),                                     // Primary directory scope
  durationHint: text("duration_hint"),                              // "~45min" — helps future estimation
  tags: text("tags").notNull().default("[]"),                       // JSON array of freeform tags
  createdAt: text("created_at").notNull().default(sql`(datetime('now'))`),
}, (table) => ({
  outcomeIdx: index("idx_task_outcomes_outcome").on(table.outcome),
  dirIdx: index("idx_task_outcomes_directory").on(table.directory),
}));

// Task outcomes capture proactive knowledge — "this approach worked/didn't work
// for this kind of task." This is fundamentally different from error fixes (reactive:
// "this broke, here's the fix") and rules (prescriptive: "always do X"). When
// Claude Code starts a similar task, Optima can surface insights like "a similar
// task took ~45min and required RS256 instead of HS256."

// ── Security Findings ─────────────────────────────────────────

export const securityFindings = sqliteTable("security_findings", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  fileId: integer("file_id").notNull().references(() => fileIndex.id, { onDelete: "cascade" }),
  line: integer("line").notNull(),
  patternName: text("pattern_name").notNull(),  // "aws_key" | "private_key" | "generic_api_key" | "jwt" | "connection_string" | "github_token" | "generic_secret"
  severity: text("severity").notNull().default("high"),  // "critical" | "high" | "medium"
  snippet: text("snippet"),                      // Surrounding context (NOT the secret itself — redacted). e.g. "AKIA****...****WXYZ on line 42"
  foundAt: text("found_at").notNull().default(sql`(datetime('now'))`),
  dismissed: integer("dismissed", { mode: "boolean" }).notNull().default(false),  // Developer can dismiss false positives
}, (table) => ({
  fileIdx: index("idx_security_findings_file").on(table.fileId),
  patternIdx: index("idx_security_findings_pattern").on(table.patternName),
}));

// CRITICAL SAFETY RULE: The `snippet` field must NEVER contain the actual secret value.
// Store only a redacted preview: first 4 chars + "****" + last 4 chars, or the
// pattern name and line number. The secret scanning step in the file indexer
// (Doc 03, step 5a) is responsible for redacting before storage.
// See also Doc 00 "DO NOT" list: "DO NOT store the actual secret value."
//
// UNIQUE CONSTRAINT (added by migration 003): security_findings has a UNIQUE index on
// (file_id, line, pattern_name). This enables INSERT OR IGNORE semantics — re-indexing
// the same file does not create duplicate findings.
//
// DISMISSED STATE PERSISTENCE:
// - On `optima_get_context` lazy re-index: only rows with dismissed=0 are deleted before
//   re-scanning; dismissed=1 rows are preserved regardless.
// - On `optima_reindex` (full): dismissed findings are saved to a temp list before the
//   cascade-delete of file_index rows, then restored after re-indexing completes. This
//   ensures dismissed state survives a full reindex.

// ── Generation Log ────────────────────────────────────────────

export const generationLog = sqliteTable("generation_log", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  filePath: text("file_path").notNull(),
  contentHash: text("content_hash").notNull(),
  trigger: text("trigger").notNull(),
  generatedAt: text("generated_at").notNull().default(sql`(datetime('now'))`),
});

// ── Schema Version ────────────────────────────────────────────

export const schemaVersion = sqliteTable("schema_version", {
  version: integer("version").primaryKey(),
  appliedAt: text("applied_at").notNull().default(sql`(datetime('now'))`),
});

// ── FTS5 Full-Text Search (virtual tables — not modeled in Drizzle) ──
//
// These virtual tables enable cross-directory text search on gotchas and rules.
// Created by migration 002_fts5.ts. Queried via raw SQL (db.prepare), not Drizzle.
//
// gotchas_fts: FTS5 content-sync table mirroring gotchas(error_text, resolution, root_cause)
// rules_fts:  FTS5 content-sync table mirroring rules(content, rationale)
//
// Sync is maintained by AFTER INSERT/UPDATE/DELETE triggers created in the migration.
// The triggers keep the FTS index in lock-step with the source tables automatically.
//
// Query example:
//   db.prepare(`
//     SELECT g.* FROM gotchas g
//     JOIN gotchas_fts ON gotchas_fts.rowid = g.id
//     WHERE gotchas_fts MATCH ?
//     ORDER BY rank
//     LIMIT 10
//   `).all(searchQuery);
```

---

### Root Cause Tracking (Inspired by Superpowers systematic-debugging)

Gotchas store `errorText` (what happened) and `resolution` (what fixed it). The optional `root_cause` field stores **why it happened** — the deeper explanation that helps future developers understand the error instead of just fixing it.

**Example without root_cause:**
- Error: `Cannot read property 'id' of undefined`
- Resolution: `Add null-check after fetchUser()`

**Example with root_cause:**
- Error: `Cannot read property 'id' of undefined`
- Resolution: `Add null-check after fetchUser()`
- Root cause: `Session cookies expire after 30 minutes of inactivity. When a user returns after idle timeout, fetchUser() returns null because the session token is invalid but doesn't throw — it returns undefined silently.`

The root cause transforms a gotcha from "patch this" to "understand this." When Claude Code encounters the same error, it can explain the WHY to the developer, not just apply the fix.

**When to populate:** Claude Code should populate `root_cause` when:
1. The developer explains WHY an error occurred (not just what to do about it)
2. Claude Code traces the error through the systematic-debugging 4-phase process and identifies the root cause
3. The error has a non-obvious cause (the resolution alone wouldn't help someone understand the problem)

`root_cause` is optional. Simple gotchas ("missing semicolon" → "add semicolon") don't need it.

---

## Authoritative Linter/Formatter Config Detection List

This is the **single source of truth** for which config files Optima checks when populating `projectMeta.linterDetected`. All docs reference this list. MVP supports TypeScript projects, but detects all linters for completeness (a TypeScript project may use `.editorconfig` or have Python tooling in a monorepo).

| Linter/Formatter | Config File Glob | Stored As |
| --- | --- | --- |
| ESLint | `eslint.config.*`, `.eslintrc.*`, `.eslintrc` | `"eslint"` |
| Prettier | `.prettierrc*`, `prettier.config.*` | `"prettier"` |
| Biome | `biome.json`, `biome.jsonc` | `"biome"` |
| EditorConfig | `.editorconfig` | `"editorconfig"` |
| Ruff (Python) | `ruff.toml`, `pyproject.toml` with `[tool.ruff]` | `"ruff"` |

**Detection algorithm (in `src/indexer/project-analyzer.ts`):**
1. For each config pattern above, check if matching file exists at project root (one level only, no recursive search).
2. Special case for `pyproject.toml`: read file content, check if `[tool.ruff]` section exists before adding `"ruff"`.
3. Collect matching linter names into a JSON array. Store as `'["eslint","prettier"]'` in `projectMeta.linterDetected`.
4. If no linter configs found, store `null`.

**Impact on CLAUDE.md generation:** When `linterDetected` is non-null, the necessity test in Doc 04 excludes formatting/style rules from the `architecture_rules` section (the linter already enforces them).

**JSON field serialization/deserialization:** Several Drizzle `text()` columns store JSON-encoded arrays or objects: `techStack`, `linterDetected`, `keyDependencies`, `files` (in gotchas, rules, and task_outcomes), and `tags`. These are stored as raw JSON strings in SQLite (e.g., `'["eslint","prettier"]'`). When reading from the database for tool output or internal use, **always `JSON.parse()` these fields** into their typed array form. When writing, **always `JSON.stringify()`** the array before storage. The TypeScript interfaces and Zod schemas define the *parsed* types (e.g., `string[]`, `string[] | null`), NOT the raw JSON string form.

```typescript
// Reading from DB → tool output
const row = db.prepare("SELECT * FROM project_meta WHERE id = 1").get();
const techStack: string[] = JSON.parse(row.tech_stack);                    // '["typescript","react"]' → ["typescript","react"]
const linterDetected: string[] | null = row.linter_detected
  ? JSON.parse(row.linter_detected)                                        // '["eslint"]' → ["eslint"]
  : null;                                                                   // null → null
const files: string[] = JSON.parse(row.files);                             // '["src/foo.ts"]' → ["src/foo.ts"]

// Writing to DB
db.prepare("UPDATE project_meta SET tech_stack = ? WHERE id = 1")
  .run(JSON.stringify(["typescript", "react"]));                            // ["typescript","react"] → '["typescript","react"]'
```

---

## Tech Stack Detection Dictionary

This is the **authoritative lookup table** for populating `projectMeta.techStack`. The project analyzer reads config files at the project root and maps detected signals to stack labels. Labels are lowercase strings stored in a JSON array.

**Detection sources (checked in order):**

### 1. `package.json` dependency mapping

Scan `dependencies` and `devDependencies` keys. Map dependency names to stack labels:

| Dependency Name (contains) | Stack Label | Category |
| --- | --- | --- |
| `typescript` | `"typescript"` | Language |
| `react` | `"react"` | Framework |
| `next` | `"next.js"` | Framework |
| `vue` | `"vue"` | Framework |
| `svelte` | `"svelte"` | Framework |
| `angular` (scoped `@angular/core`) | `"angular"` | Framework |
| `express` | `"express"` | Server |
| `fastify` | `"fastify"` | Server |
| `hono` | `"hono"` | Server |
| `drizzle-orm` | `"drizzle"` | ORM |
| `prisma` | `"prisma"` | ORM |
| `vitest` | `"vitest"` | Test |
| `jest` | `"jest"` | Test |
| `tailwindcss` | `"tailwind"` | CSS |
| `@modelcontextprotocol/sdk` | `"mcp"` | Protocol |

**Matching rule:** Use `dependencies[key].includes(name)` — e.g., `@next/font` matches `"next"`. Deduplicate labels.

### 2. Config file signals

| Config File | Stack Label |
| --- | --- |
| `tsconfig.json` exists | `"typescript"` (if not already from package.json) |
| `pyproject.toml` exists | `"python"` |
| `Cargo.toml` exists | `"rust"` |
| `go.mod` exists | `"go"` |
| `Gemfile` exists | `"ruby"` |
| `pom.xml` or `build.gradle` exists | `"java"` |

### 3. Runtime label

Always add `"bun"` if the project uses Optima (Optima runs on Bun).

**Output format:** Sorted alphabetically. Stored as JSON array string in DB: `'["bun","drizzle","mcp","typescript","vitest"]'`.

**Extensibility:** This dictionary is intentionally limited to the most common stacks. An agent implementing this should use exact string matching (not fuzzy). Unknown dependencies are ignored — better to have a short accurate list than a noisy one. Phase 2 can expand with more granular detection.

### Tech stack detection implementation

```typescript
import { readFile } from "node:fs/promises";
import { join } from "node:path";
import { existsSync } from "node:fs";

const DEPENDENCY_MAP: Record<string, string> = {
  typescript: "typescript",
  react: "react",
  next: "next.js",
  vue: "vue",
  svelte: "svelte",
  "@angular/core": "angular",
  express: "express",
  fastify: "fastify",
  hono: "hono",
  "drizzle-orm": "drizzle",
  prisma: "prisma",
  vitest: "vitest",
  jest: "jest",
  tailwindcss: "tailwind",
  "@modelcontextprotocol/sdk": "mcp",
};

const CONFIG_SIGNALS: Record<string, string> = {
  "tsconfig.json": "typescript",
  "pyproject.toml": "python",
  "Cargo.toml": "rust",
  "go.mod": "go",
  "Gemfile": "ruby",
  "pom.xml": "java",
  "build.gradle": "java",
};

export async function detectTechStack(projectRoot: string): Promise<string[]> {
  const labels = new Set<string>();

  // 1. Scan package.json dependencies
  try {
    const pkg = JSON.parse(await readFile(join(projectRoot, "package.json"), "utf-8"));
    const allDeps = { ...pkg.dependencies, ...pkg.devDependencies };
    for (const depName of Object.keys(allDeps)) {
      for (const [pattern, label] of Object.entries(DEPENDENCY_MAP)) {
        if (depName === pattern || depName.includes(pattern)) {
          labels.add(label);
        }
      }
    }
  } catch { /* no package.json */ }

  // 2. Check config file signals
  for (const [file, label] of Object.entries(CONFIG_SIGNALS)) {
    if (existsSync(join(projectRoot, file))) {
      labels.add(label);
    }
  }

  // 3. Always add "bun" (Optima runs on Bun)
  labels.add("bun");

  return [...labels].sort();
}
```

### Command detection implementation

```typescript
export interface DetectedCommands {
  build: string | null;
  test: string | null;
  lint: string | null;
}

export async function detectCommands(projectRoot: string): Promise<DetectedCommands> {
  const result: DetectedCommands = { build: null, test: null, lint: null };

  try {
    const pkg = JSON.parse(await readFile(join(projectRoot, "package.json"), "utf-8"));
    const scripts = pkg.scripts ?? {};

    if (scripts.build) result.build = "bun run build";
    if (scripts.test) result.test = "bun run test";
    if (scripts.lint) result.lint = "bun run lint";

    // Fallbacks: check for common patterns
    if (!result.test && scripts["test:run"]) result.test = "bun run test:run";
    if (!result.lint && scripts.check) result.lint = "bun run check";
  } catch { /* no package.json */ }

  return result;
}
```

---

## TypeScript Interfaces

Copy into `src/types.ts` verbatim.

```tsx
// ── Tool Input/Output Types ─────────────────────────────────────

export interface GetContextInput {
  path: string;
  task_type?: "bug_fix" | "feature" | "refactor" | "test" | "review";
  search_query?: string;
}

export interface SecurityFinding {
  id: number;
  file: string;
  line: number;
  pattern_name: string;
  severity: "critical" | "high" | "medium";
  snippet: string | null;  // Redacted preview — NEVER the actual secret
  dismissed: boolean;
}

export interface GetContextOutput {
  project: ProjectSummary;
  directory_context: DirectoryContext;
  gotchas: GotchaEntry[];
  architectural_rules: RuleEntry[];
  task_insights: TaskOutcomeEntry[];
  security_warnings: SecurityFinding[];
  recent_changes: string[];
  dependency_context: {
    key_dependencies: DependencyInfo[];
  };
}

export type MemorizeInput =
  | { type: "error_fix"; error: string; resolution: string; rootCause?: string; dependencyVersion?: string; files?: string[]; directory?: string; tags?: string[] }
  | { type: "architectural_rule"; rule: string; rationale?: string; files?: string[]; directory?: string; tags?: string[] }
  | { type: "pattern"; pattern: string; example?: string; files?: string[]; directory?: string; tags?: string[] }
  | { type: "preference"; preference: string; files?: string[]; directory?: string; tags?: string[] }
  | { type: "task_outcome"; task: string; outcome: "success" | "partial" | "abandoned"; learnings?: string; files?: string[]; directory?: string; duration_hint?: string; tags?: string[] };

**MemorizeInput → DB Column Mapping:**

| Type | Input Field | DB Table | DB Column | Notes |
|------|-------------|----------|-----------|-------|
| `error_fix` | `error` | `gotchas` | `error_text` | Sanitized via `sanitizeError()` before storage |
| `error_fix` | `resolution` | `gotchas` | `resolution` | Stored verbatim |
| `error_fix` | `rootCause` | `gotchas` | `root_cause` | Optional — stored verbatim |
| `error_fix` | `dependencyVersion` | `gotchas` | `dependency_version` | Optional — e.g. `"drizzle-orm@^0.38.0"` |
| `error_fix` | (computed) | `gotchas` | `error_hash` | SHA-256 of `normalizeError(sanitizeError(error))` |
| `architectural_rule` | `rule` | `rules` | `content` | — |
| `architectural_rule` | `rationale` | `rules` | `rationale` | Optional |
| `pattern` | `pattern` | `rules` | `content` | — |
| `pattern` | `example` | `rules` | `rationale` | Reuses `rationale` column — stores code example |
| `preference` | `preference` | `rules` | `content` | — |
| `task_outcome` | `task` | `task_outcomes` | `task` | — |
| `task_outcome` | `outcome` | `task_outcomes` | `outcome` | — |
| `task_outcome` | `learnings` | `task_outcomes` | `learnings` | Optional |
| `task_outcome` | `duration_hint` | `task_outcomes` | `duration_hint` | Optional |
| All types | `files` | respective | `files` | JSON stringified: `JSON.stringify(files ?? [])` |
| All types | `directory` | respective | `directory` | Stored as-is (nullable) |
| All types | `tags` | respective | `tags` | JSON stringified: `JSON.stringify(tags ?? [])` |

export interface TaskOutcomeEntry {
  id: number;
  task: string;
  outcome: "success" | "partial" | "abandoned";
  learnings: string | null;
  files: string[];
  directory: string | null;
  duration_hint: string | null;
  created_at: string;
}

export interface MemorizeOutput {
  stored: boolean;
  memory_id: string;
  total_memories: number;
  // Verification fields (Iron Law 3: Evidence Before Claims)
  type: "error_fix" | "architectural_rule" | "pattern" | "preference" | "task_outcome";
  duplicate_of: number | null;
  hit_count_updated: boolean;
  claude_md_regenerated: boolean;
  claude_md_instruction_count: number;
  feedback_rules_written: boolean;
  duplicate_detected: boolean;
}

export interface ReindexInput {
  path?: string;
  reason?: string;
}

export interface ReindexOutput {
  files_indexed: number;
  entities_found: number;
  duration_ms: number;
  // Verification fields (Iron Law 3: Evidence Before Claims)
  total_files_scanned: number;
  total_entities_found: number;
  project_analysis_updated: boolean;
  claude_md_regenerated: boolean;
  feedback_rules_written: boolean;
}

// ── Internal Types ────────────────────────────────────────────

export interface DependencyInfo {
  name: string;           // "drizzle-orm"
  version: string;        // "^0.39.0"
  detected_from: string;  // "package.json"
}

export interface ProjectSummary {
  name: string;
  purpose: string | null;
  tech_stack: string[];
  linter_detected: string[] | null; // Parsed from JSON string in DB. null if no linters detected. See authoritative list above.
  build_command: string | null;
  test_command: string | null;
  lint_command: string | null;
}

export interface DirectoryContext {
  path: string;
  description: string;
  key_files: string[];
  entities: EntitySummary[];
}

export interface EntitySummary {
  name: string;
  kind: "function" | "class" | "interface" | "type" | "export" | "variable";
  file: string;
  line: number;
  signature: string | null;
  exported: boolean;
  description: string | null;
}

export interface GotchaEntry {
  id: number;
  error_text: string;
  resolution: string;
  root_cause: string | null;
  dependency_version: string | null;  // "drizzle-orm@^0.38.0" — dep version when gotcha was recorded
  possibly_outdated: boolean;          // COMPUTED at query time (not a DB column) — true if stored dependency_version differs from current project version. See Doc 03 step 8.
  files: string[];
  directory: string | null;
  hit_count: number;
}

export interface RuleEntry {
  id: number;
  type: "architectural_rule" | "pattern" | "preference";
  content: string;
  rationale: string | null;
  directory: string | null;
}

export interface FileState {
  path: string;
  mtimeMs: number;
  sizeBytes: number;
  hash: string;
  language: string | null;
}

// ── Error Types ───────────────────────────────────────────────

export type OptimaErrorCode =
  | "INDEX_FAILED"
  | "PARSE_FAILED"
  | "DB_ERROR"
  | "INVALID_INPUT"
  | "PATH_NOT_FOUND"
  | "GENERATION_FAILED"
  | "HASH_COLLISION"
  | "SCHEMA_MIGRATION_FAILED";

export class OptimaError extends Error {
  constructor(
    public readonly code: OptimaErrorCode,
    message: string,
    public readonly cause?: unknown,
  ) {
    super(`[${code}] ${message}`);
    this.name = "OptimaError";
  }
}
```

---

## Internal Module Interfaces

These are the exported function signatures for the CRUD modules referenced by tool handlers. Copy into the respective files.

### `src/memory/gotcha-ledger.ts`

```typescript
import type { GotchaEntry } from "../types.js";
import type { Database } from "better-sqlite3";

export interface GotchaLedger {
  /** Find gotchas matching a directory path (hierarchical prefix match) OR file array overlap.
   *  Deduplicates by id, sorts by hit_count DESC, increments hit_count for all matches. */
  findByContext(db: Database, directoryPath: string, filePaths?: string[]): GotchaEntry[];

  /** Upsert a gotcha by error_hash. If hash exists: update resolution, files, updated_at.
   *  If not: insert new row. Returns the row id. */
  upsertByHash(db: Database, entry: {
    errorText: string;      // Already sanitized
    errorHash: string;      // SHA-256 of normalized error
    resolution: string;
    rootCause: string | null;  // Why the error occurred (optional, maps from MemorizeInput.rootCause)
    dependencyVersion: string | null;  // e.g. "drizzle-orm@^0.38.0"
    files: string[];
    directory: string | null;
    tags: string[];
  }): number;

  /** Get gotchas passing the necessity test for CLAUDE.md generation.
   *  Filters: hit_count >= 2. Sorts by hit_count DESC. Caps at limit. */
  getForGeneration(db: Database, limit: number): GotchaEntry[];
}
```

### `src/memory/rules-store.ts`

```typescript
import type { RuleEntry } from "../types.js";
import type { Database } from "better-sqlite3";

export interface RulesStore {
  /** Insert a new rule (architectural_rule, pattern, or preference). Returns the row id. */
  insert(db: Database, entry: {
    type: "architectural_rule" | "pattern" | "preference";
    content: string;
    rationale: string | null;
    directory: string | null;
    files: string[];
    tags: string[];
  }): number;

  /** Find rules matching a directory path (hierarchical prefix match).
   *  Returns project-wide (directory IS NULL) + prefix matches, sorted by specificity. */
  findByDirectory(db: Database, directoryPath: string): RuleEntry[];

  /** Get rules passing the necessity test for CLAUDE.md generation.
   *  Filters by type, excludes linter-duplicates if linterDetected is provided.
   *  Sorts by created_at DESC. Caps at limit per type. */
  getForGeneration(db: Database, type: "architectural_rule" | "pattern" | "preference", limit: number, linterDetected: string[] | null): RuleEntry[];
}
```

### `src/db/connection.ts`

```typescript
import type Database from "better-sqlite3";

/** Lazily initializes the database connection. First call creates .optima/ directory,
 *  opens/creates optima.db, runs migrations, prunes generation_log.
 *  Subsequent calls return the cached connection.
 *  Uses getProjectRoot() internally — callers do NOT pass the root path. */
export function getDatabase(): Database;
```

### `src/indexer/project-analyzer.ts`

```typescript
import type { Database } from "better-sqlite3";

export interface ProjectAnalysis {
  name: string;
  purpose: string | null;
  techStack: string[];
  buildCommand: string | null;
  testCommand: string | null;
  lintCommand: string | null;
  linterDetected: string[] | null;
  keyDependencies: DependencyInfo[] | null;  // Parsed from package.json deps matching DEPENDENCY_MAP
}

/** Analyzes the project root to detect tech stack, commands, purpose, linters, and key dependencies.
 *  Reads package.json, tsconfig.json, pyproject.toml, README.md, and linter configs.
 *  Parses dependency versions from package.json for entries matching DEPENDENCY_MAP.
 *  Upserts result into project_meta table. Returns the analysis. */
export function analyzeProject(db: Database, projectRoot: string): ProjectAnalysis;

/** findAnalysisRoot — Workspace-Parent CWD Helper
 *
 * When Claude Code is opened at a workspace parent directory (e.g. `C:/dev/workspace`)
 * rather than the project root (e.g. `C:/dev/workspace/my-app`), the CWD does not contain
 * a package.json and `analyzeProject` would return all-null project metadata.
 *
 * `findAnalysisRoot(requestedPath)` walks UP from the requested path to find the nearest
 * ancestor directory containing `package.json`, `pyproject.toml`, `Cargo.toml`, or `go.mod`.
 * If found, that directory is used as the `analysis_root` passed to `analyzeProject`.
 * If not found (e.g. the path is already at a filesystem root), falls back to `requestedPath`.
 *
 * Callers: `handleGetContext` (step 2) and `handleReindex` both call `findAnalysisRoot`
 * before calling `analyzeProject`. The resolved `analysis_root` is logged at DEBUG level. */
export function findAnalysisRoot(requestedPath: string): string;
```

### `src/generator/claude-md.ts`

```typescript
/** Regenerates CLAUDE.md with section markers. Reads existing content, replaces
 *  marker-bounded sections, appends new sections, handles malformed markers.
 *  Computes content hash — skips file write if unchanged. Returns true if file was written. */
export function regenerateClaudeMd(db: Database, projectRoot: string): boolean;
```

### `src/generator/feedback-rules.ts`

```typescript
/** Writes .claude/rules/optima-feedback.md with the inception payload.
 *  Creates .claude/rules/ directory if needed. Content is static for MVP.
 *  Skips write if file already exists with correct content. Returns true if file was written. */
export function generateFeedbackRules(projectRoot: string): boolean;
```

---

## Retention Policy

- **File index:** No limit. Rows deleted when files are deleted from disk (detected during re-index).
- **Entities:** Cascade-deleted with their parent file_index row.
- **Gotchas:** Keep all in database (never deleted — needed for future error matching). `hit_count` tracks usefulness; `updated_at` refreshes each time `hit_count` increments. For CLAUDE.md generation, include only gotchas with `hit_count >= 2` (resolved Q4 — single authoritative threshold, see Doc 04 necessity test).
- **Rules:** Keep all. Manual pruning via future tooling.
- **Security findings:** Cascade-deleted with their parent file_index row. During **lazy re-indexing** (`optima_get_context` steps 5-7), only files with changed mtimes are re-indexed — unchanged files keep their security findings including dismissed state. When a changed file is re-indexed, its old findings are cascade-deleted and re-scanned from scratch (dismissed state is lost for that file). During **full re-index** (`optima_reindex`), all file_index rows are deleted and re-created, so ALL dismissed states are reset. This is acceptable because: (a) `optima_reindex` is a rare escape hatch, not a routine operation, (b) re-scanning after reindex is correct behavior — the file may have changed, and (c) dismissed false positives are fast to re-dismiss.
- **Task outcomes:** Keep last 50 entries. Prune oldest entries beyond 50 during `getDatabase()` initialization. Task outcomes are proactive knowledge with diminishing relevance — old outcomes about abandoned approaches from 6 months ago are less useful than recent ones.
- **Generation log:** Keep last 50 entries per file path. Prune during `getDatabase()` initialization (runs once on cold start, not on every tool call). SQL: `DELETE FROM generation_log WHERE id NOT IN (SELECT id FROM generation_log WHERE file_path = ? ORDER BY generated_at DESC LIMIT 50)` for each distinct `file_path`.

### Binary File Extensions (Always Skipped)

The file walker skips files with these extensions regardless of `.gitignore` rules. They are never indexed, never entity-extracted, and never scanned for secrets:

**Images:** `.png` `.jpg` `.jpeg` `.gif` `.ico` `.webp` `.bmp` `.svg`  
**Fonts:** `.woff` `.woff2` `.ttf` `.eot` `.otf`  
**Media:** `.mp3` `.mp4` `.wav` `.ogg` `.webm` `.avi` `.mov`  
**Archives:** `.zip` `.tar` `.gz` `.bz2` `.7z` `.rar`  
**Documents:** `.pdf`  
**Compiled/native:** `.exe` `.dll` `.so` `.dylib` `.bin`  
**Databases:** `.db` `.sqlite` `.sqlite3`  

**Rationale:** Binary files produce meaningless entity extraction output and cannot contain human-readable secrets. Skipping them reduces indexing time and prevents garbage data in `file_index`.

**Note:** `.svg` is listed as binary here because Tree-sitter entity extraction is meaningless on XML/SVG markup. SVG files ARE human-readable but are skipped for entity purposes. They are also not secret-scanned.

## Schema Migration Strategy

Optima does NOT use Drizzle Kit's migration system at runtime. Migrations are hand-written SQL scripts executed programmatically by `src/db/migrations.ts`. This keeps the runtime dependency minimal and avoids requiring `drizzle-kit` as a production dependency.

**Migration file convention:**
```
src/db/migrations/
├── 001_initial.ts                    # Creates 9 tables (8 Drizzle-modeled + schema_version)
├── 002_fts5.ts                       # FTS5 virtual tables + sync triggers for gotchas and rules
├── 003_security_findings_unique.ts   # UNIQUE index on security_findings(file_id, line, pattern_name); deduplicates existing rows before adding constraint; enables INSERT OR IGNORE semantics
└── index.ts                          # Exports ordered migration list
```

**Each migration file exports:**
```typescript
export const migration = {
  version: 1,
  description: "Initial schema — 9 tables (project_meta, file_index, entities, gotchas, rules, security_findings, task_outcomes, generation_log, schema_version)",
  up(db: Database): void {
    // Raw SQL via db.exec()
  },
};
```

**Migration runner logic (in `src/db/migrations.ts`):**
1. Create `schema_version` table if it doesn't exist.
2. Query `SELECT MAX(version) FROM schema_version`.
3. For each migration with `version > current`: run `up(db)` inside a transaction, then insert into `schema_version`.
4. If any migration fails: rollback transaction, throw `OptimaError("SCHEMA_MIGRATION_FAILED")`.

**Drizzle Kit is a dev-only tool.** Use `drizzle-kit` during development to generate migration SQL from schema changes (`bunx drizzle-kit generate`), then paste the SQL into a numbered migration file. Drizzle ORM is used at runtime for typed queries — Drizzle Kit is not.

### `001_initial.ts` — Full CREATE TABLE SQL

This is the exact SQL that `src/db/migrations/001_initial.ts` must execute. Every column matches the Drizzle schema above 1:1. Copy verbatim.

```typescript
export const migration = {
  version: 1,
  description: "Initial schema — 9 tables (project_meta, file_index, entities, gotchas, rules, security_findings, task_outcomes, generation_log, schema_version)",
  up(db: Database): void {
    db.exec(`
      CREATE TABLE IF NOT EXISTS schema_version (
        version INTEGER PRIMARY KEY,
        applied_at TEXT NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS project_meta (
        id INTEGER PRIMARY KEY DEFAULT 1,
        name TEXT NOT NULL,
        root_path TEXT NOT NULL,
        tech_stack TEXT NOT NULL DEFAULT '[]',
        build_command TEXT,
        test_command TEXT,
        lint_command TEXT,
        project_purpose TEXT,
        linter_detected TEXT,
        key_dependencies TEXT,
        last_full_index TEXT,
        updated_at TEXT NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS file_index (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        path TEXT NOT NULL UNIQUE,
        mtime_ms INTEGER NOT NULL,
        size_bytes INTEGER NOT NULL,
        language TEXT,
        hash TEXT NOT NULL,
        indexed_at TEXT NOT NULL DEFAULT (datetime('now'))
      );
      CREATE UNIQUE INDEX IF NOT EXISTS idx_file_path ON file_index(path);

      CREATE TABLE IF NOT EXISTS entities (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        file_id INTEGER NOT NULL REFERENCES file_index(id) ON DELETE CASCADE,
        name TEXT NOT NULL,
        kind TEXT NOT NULL,
        line_start INTEGER NOT NULL,
        line_end INTEGER NOT NULL,
        signature TEXT,
        exported INTEGER NOT NULL DEFAULT 0,
        description TEXT,
        created_by_session TEXT,
        last_modified_context TEXT
      );
      CREATE INDEX IF NOT EXISTS idx_entities_file ON entities(file_id);
      CREATE INDEX IF NOT EXISTS idx_entities_name ON entities(name);

      CREATE TABLE IF NOT EXISTS gotchas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        error_text TEXT NOT NULL,
        error_hash TEXT NOT NULL,
        resolution TEXT NOT NULL,
        root_cause TEXT,
        dependency_version TEXT,
        files TEXT NOT NULL DEFAULT '[]',
        directory TEXT,
        tags TEXT NOT NULL DEFAULT '[]',
        hit_count INTEGER NOT NULL DEFAULT 0,
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        updated_at TEXT NOT NULL DEFAULT (datetime('now'))
      );
      CREATE INDEX IF NOT EXISTS idx_gotchas_hash ON gotchas(error_hash);
      CREATE INDEX IF NOT EXISTS idx_gotchas_directory ON gotchas(directory);

      CREATE TABLE IF NOT EXISTS rules (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        type TEXT NOT NULL,
        content TEXT NOT NULL,
        rationale TEXT,
        directory TEXT,
        files TEXT NOT NULL DEFAULT '[]',
        tags TEXT NOT NULL DEFAULT '[]',
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        updated_at TEXT NOT NULL DEFAULT (datetime('now'))
      );
      CREATE INDEX IF NOT EXISTS idx_rules_type ON rules(type);
      CREATE INDEX IF NOT EXISTS idx_rules_directory ON rules(directory);

      CREATE TABLE IF NOT EXISTS security_findings (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        file_id INTEGER NOT NULL REFERENCES file_index(id) ON DELETE CASCADE,
        line INTEGER NOT NULL,
        pattern_name TEXT NOT NULL,
        severity TEXT NOT NULL DEFAULT 'high',
        snippet TEXT,
        found_at TEXT NOT NULL DEFAULT (datetime('now')),
        dismissed INTEGER NOT NULL DEFAULT 0
      );
      CREATE INDEX IF NOT EXISTS idx_security_findings_file ON security_findings(file_id);
      CREATE INDEX IF NOT EXISTS idx_security_findings_pattern ON security_findings(pattern_name);

      CREATE TABLE IF NOT EXISTS task_outcomes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        task TEXT NOT NULL,
        outcome TEXT NOT NULL,
        learnings TEXT,
        files TEXT NOT NULL DEFAULT '[]',
        directory TEXT,
        duration_hint TEXT,
        tags TEXT NOT NULL DEFAULT '[]',
        created_at TEXT NOT NULL DEFAULT (datetime('now'))
      );
      CREATE INDEX IF NOT EXISTS idx_task_outcomes_outcome ON task_outcomes(outcome);
      CREATE INDEX IF NOT EXISTS idx_task_outcomes_directory ON task_outcomes(directory);

      CREATE TABLE IF NOT EXISTS generation_log (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        file_path TEXT NOT NULL,
        content_hash TEXT NOT NULL,
        trigger TEXT NOT NULL,
        generated_at TEXT NOT NULL DEFAULT (datetime('now'))
      );
    `);
  },
};
```

**Verification:** Compare every column in the Drizzle schema (above) against this CREATE TABLE SQL. They must be 1:1. Zero columns in Drizzle that aren't in the migration, and vice versa.

### `002_fts5.ts` — FTS5 Full-Text Search Virtual Tables

Creates FTS5 content-sync virtual tables for gotchas and rules, plus AFTER INSERT/UPDATE/DELETE triggers that keep the FTS index in sync with the source tables automatically. FTS5 is built into `better-sqlite3` — zero new dependencies.

```typescript
export const migration = {
  version: 2,
  description: "FTS5 virtual tables + sync triggers for gotchas and rules",
  up(db: Database): void {
    db.exec(`
      -- Gotchas FTS5 (content-sync: mirrors error_text, resolution, root_cause)
      CREATE VIRTUAL TABLE IF NOT EXISTS gotchas_fts USING fts5(
        error_text, resolution, root_cause,
        content=gotchas, content_rowid=id
      );

      -- Populate FTS from existing gotchas
      INSERT INTO gotchas_fts(gotchas_fts) VALUES('rebuild');

      -- Sync triggers: keep FTS in lock-step with gotchas table
      CREATE TRIGGER IF NOT EXISTS gotchas_ai AFTER INSERT ON gotchas BEGIN
        INSERT INTO gotchas_fts(rowid, error_text, resolution, root_cause)
        VALUES (new.id, new.error_text, new.resolution, new.root_cause);
      END;

      CREATE TRIGGER IF NOT EXISTS gotchas_ad AFTER DELETE ON gotchas BEGIN
        INSERT INTO gotchas_fts(gotchas_fts, rowid, error_text, resolution, root_cause)
        VALUES ('delete', old.id, old.error_text, old.resolution, old.root_cause);
      END;

      CREATE TRIGGER IF NOT EXISTS gotchas_au AFTER UPDATE ON gotchas BEGIN
        INSERT INTO gotchas_fts(gotchas_fts, rowid, error_text, resolution, root_cause)
        VALUES ('delete', old.id, old.error_text, old.resolution, old.root_cause);
        INSERT INTO gotchas_fts(rowid, error_text, resolution, root_cause)
        VALUES (new.id, new.error_text, new.resolution, new.root_cause);
      END;

      -- Rules FTS5 (content-sync: mirrors content, rationale)
      CREATE VIRTUAL TABLE IF NOT EXISTS rules_fts USING fts5(
        content, rationale,
        content=rules, content_rowid=id
      );

      -- Populate FTS from existing rules
      INSERT INTO rules_fts(rules_fts) VALUES('rebuild');

      -- Sync triggers: keep FTS in lock-step with rules table
      CREATE TRIGGER IF NOT EXISTS rules_ai AFTER INSERT ON rules BEGIN
        INSERT INTO rules_fts(rowid, content, rationale)
        VALUES (new.id, new.content, new.rationale);
      END;

      CREATE TRIGGER IF NOT EXISTS rules_ad AFTER DELETE ON rules BEGIN
        INSERT INTO rules_fts(rules_fts, rowid, content, rationale)
        VALUES ('delete', old.id, old.content, old.rationale);
      END;

      CREATE TRIGGER IF NOT EXISTS rules_au AFTER UPDATE ON rules BEGIN
        INSERT INTO rules_fts(rules_fts, rowid, content, rationale)
        VALUES ('delete', old.id, old.content, old.rationale);
        INSERT INTO rules_fts(rowid, content, rationale)
        VALUES (new.id, new.content, new.rationale);
      END;
    `);
  },
};
```

**Why content-sync (not external-content):** FTS5 `content=` tables are read-only mirrors — the triggers handle all writes. This means:
- Inserts/updates/deletes on `gotchas` and `rules` tables automatically update the FTS index.
- No manual FTS maintenance code needed in gotcha-ledger.ts or rules-store.ts.
- The `rebuild` command populates the FTS index from existing data on migration (handles databases created before FTS5 was added).

**Query pattern (used in optima_get_context step 8):**
```typescript
// FTS5 search on gotchas — returns matching gotcha IDs ranked by relevance
function searchGotchasFts(db: Database, query: string, limit: number): number[] {
  const rows = db.prepare(`
    SELECT g.id FROM gotchas g
    JOIN gotchas_fts ON gotchas_fts.rowid = g.id
    WHERE gotchas_fts MATCH ?
    ORDER BY rank
    LIMIT ?
  `).all(query, limit) as Array<{ id: number }>;
  return rows.map(r => r.id);
}

// FTS5 search on rules — returns matching rule IDs ranked by relevance
function searchRulesFts(db: Database, query: string, limit: number): number[] {
  const rows = db.prepare(`
    SELECT r.id FROM rules r
    JOIN rules_fts ON rules_fts.rowid = r.id
    WHERE rules_fts MATCH ?
    ORDER BY rank
    LIMIT ?
  `).all(query, limit) as Array<{ id: number }>;
  return rows.map(r => r.id);
}
```

**FTS5 query syntax notes:** The `MATCH` operator supports implicit AND (`auth timeout` matches rows containing both words), quoted phrases (`"auth timeout"` matches the exact phrase), prefix queries (`auth*`), and boolean operators (`auth OR timeout`). Optima passes the raw `search_query` string from the tool input directly to `MATCH` — FTS5 handles tokenization. If the query contains FTS5 syntax errors, catch the SQLite error and fall back to directory-only matching (do not throw).

### Migration 003: Security Findings Deduplication Index

**Version:** 3
**Applied:** At cold start, after migration 002

**Change:**
```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_security_findings_unique
  ON security_findings(file_id, line, pattern_name);
```

**Behavior:** Enables `INSERT OR IGNORE` semantics. When a file is re-indexed and the same secret pattern is found at the same file+line combination, the insert is silently ignored — no duplicate, no error. Dismissed findings (where `dismissed = true`) are preserved on re-index as long as the pattern is still detected at the same location.

**Rationale:** Without this index, re-indexing a file that contains a secret would create duplicate `security_findings` rows (one per re-index). The unique index makes re-indexing idempotent.

## Phase 2 Schema Migration

Phase 2 automated pruning (see Product Spec section 10.4) requires two new columns:

- `rules.hit_count` (INTEGER, default 0) — tracks how often a rule was referenced in `optima_get_context` output
- `rules.pinned` (INTEGER/BOOLEAN, default false) — developer override to prevent automatic pruning

These are additive columns with defaults, so the migration is non-breaking. Add via Drizzle migration when Phase 2 begins.