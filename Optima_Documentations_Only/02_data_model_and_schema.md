# 02 Data Model and Schema

All data lives in ONE SQLite database: `.optima/optima.db` at the project root.

**SQLite driver:** `better-sqlite3` (not `bun:sqlite`) for runtime portability — works on both Bun and Node.js. See resolved Q6 in Product Specification.

## Database Initialization & Lifecycle

**When:** Database is created on the first `optima_get_context` call, as part of the cold start bootstrap sequence (see `01_PRODUCT_SPEC_MVP.md` "Cold Start Bootstrap Sequence"). The MCP server process starts and listens on stdio immediately — database creation is deferred until first tool invocation.

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
  linterDetected: text("linter_detected"), // JSON array of strings, e.g. '["eslint","prettier"]'. Detected config files: eslint.config.*, .eslintrc.*, .prettierrc*, biome.json, ruff.toml, pyproject.toml [tool.ruff], .editorconfig. null if none detected.
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
}, (table) => ({
  fileIdx: index("idx_entities_file").on(table.fileId),
  nameIdx: index("idx_entities_name").on(table.name),
}));

// ── Gotcha Ledger ─────────────────────────────────────────────

export const gotchas = sqliteTable("gotchas", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  errorText: text("error_text").notNull(),
  errorHash: text("error_hash").notNull(),
  resolution: text("resolution").notNull(),
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
```

---

## TypeScript Interfaces

Copy into `src/types.ts` verbatim.

```tsx
// ── Tool Input/Output Types ─────────────────────────────────────

export interface GetContextInput {
  path: string;
  task_type?: "bug_fix" | "feature" | "refactor" | "test" | "review";
}

export interface GetContextOutput {
  project: ProjectSummary;
  directory_context: DirectoryContext;
  gotchas: GotchaEntry[];
  architectural_rules: RuleEntry[];
  recent_changes: string[];
}

export type MemorizeInput =
  | { type: "error_fix"; error: string; resolution: string; files?: string[]; directory?: string; tags?: string[] }
  | { type: "architectural_rule"; rule: string; rationale?: string; files?: string[]; directory?: string; tags?: string[] }
  | { type: "pattern"; pattern: string; example?: string; files?: string[]; directory?: string; tags?: string[] }
  | { type: "preference"; preference: string; files?: string[]; directory?: string; tags?: string[] };

export interface MemorizeOutput {
  stored: boolean;
  memory_id: string;
  total_memories: number;
}

export interface ReindexInput {
  path?: string;
  reason?: string;
}

export interface ReindexOutput {
  files_indexed: number;
  entities_found: number;
  duration_ms: number;
}

// ── Internal Types ────────────────────────────────────────────

export interface ProjectSummary {
  name: string;
  purpose: string | null;
  tech_stack: string[];
  linter_detected: string | null;
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
}

export interface GotchaEntry {
  id: number;
  error_text: string;
  resolution: string;
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

## Retention Policy

- **File index:** No limit. Rows deleted when files are deleted from disk (detected during re-index).
- **Entities:** Cascade-deleted with their parent file_index row.
- **Gotchas:** Keep all. `hit_count` tracks usefulness. `updated_at` is refreshed each time `hit_count` increments, making it usable as a "last hit" proxy for the necessity test filter during [CLAUDE.md](http://CLAUDE.md) generation (MVP: exclude gotchas where `hit_count = 0` AND `created_at` is older than 30 days).
- **Rules:** Keep all. Manual pruning via future tooling.
- **Generation log:** Keep last 50 entries per file path. Prune on database initialization.

## Schema Migration Strategy

Optima does NOT use Drizzle Kit's migration system at runtime. Migrations are hand-written SQL scripts executed programmatically by `src/db/migrations.ts`. This keeps the runtime dependency minimal and avoids requiring `drizzle-kit` as a production dependency.

**Migration file convention:**
```
src/db/migrations/
├── 001_initial.ts        # Creates all 7 tables
├── 002_phase2_pruning.ts # Adds hit_count + pinned to rules (future)
└── index.ts              # Exports ordered migration list
```

**Each migration file exports:**
```typescript
export const migration = {
  version: 1,
  description: "Initial schema — 7 tables",
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

## Phase 2 Schema Migration

Phase 2 automated pruning (see Product Spec section 10.4) requires two new columns:

- `rules.hit_count` (INTEGER, default 0) — tracks how often a rule was referenced in `optima_get_context` output
- `rules.pinned` (INTEGER/BOOLEAN, default false) — developer override to prevent automatic pruning

These are additive columns with defaults, so the migration is non-breaking. Add via Drizzle migration when Phase 2 begins.