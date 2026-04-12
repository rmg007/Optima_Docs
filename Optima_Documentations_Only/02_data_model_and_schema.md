# 02 Data Model and Schema

All data lives in ONE SQLite database: `.opticode/opticode.db` at the project root.

**SQLite driver:** `better-sqlite3` for runtime portability — works on both Bun and Node.js.

## Database Initialization & Lifecycle

**When:** Database is created when the `ContextBuilder` triggers the lazy indexing layer during the first interaction with the agent loop.

**Owner:** `src/db/connection.ts` exports a `getDatabase()` function that lazily initializes the connection. First call creates `.opticode/` directory, opens/creates `opticode.db`, and runs migrations. Subsequent calls return the cached connection.

**Initialization steps (idempotent):**
1. `fs.mkdirSync('.opticode', { recursive: true })` — sync is acceptable here (one-time cold path, not hot path).
2. Open `better-sqlite3` connection with `{ fileMustExist: false }`.
3. Enable WAL mode: `PRAGMA journal_mode=WAL` — allows concurrent reads during writes, improves performance.
4. Enable foreign keys: `PRAGMA foreign_keys=ON`.
5. Check `schema_version` table. If missing or version < current, run migrations (see Schema Migration Strategy below).
6. Write `.opticode/.gitignore` with content `*` (if not already present).

**Corruption recovery:** If the database file is corrupted (detected by `better-sqlite3` throwing `SqliteError` with code `SQLITE_CORRUPT` or `SQLITE_NOTADB` on open), delete `.opticode/opticode.db`, `.opticode/opticode.db-wal`, and `.opticode/opticode.db-shm`, then reinitialize from scratch.

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
  linterDetected: text("linter_detected"), // JSON array of strings
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

Copy into `src/types.ts` (or relevant directory) verbatim.

```tsx
// ── WebSocket Protocol Types ────────────────────────────────────

export interface UserMessage {
  type: "user_message";
  content: string;
}

export interface AssistantChunk {
  type: "assistant_chunk";
  content: string;
}

export interface ToolCallMessage {
  type: "tool_call";
  id: string;
  name: string;
  input: any;
}

export interface ToolResultMessage {
  type: "tool_result";
  id: string;
  result: ToolResult;
}

export interface PermissionRequest {
  type: "permission_request";
  id: string;
  tool: string;
  input: any;
}

export interface PermissionResponse {
  type: "permission_response";
  id: string;
  approved: boolean;
}

// ── Agent Loop Types ────────────────────────────────────────────

export enum ModelTier {
  Haiku = "claude-3-haiku",
  Sonnet = "claude-3-5-sonnet",
  Opus = "claude-3-opus",
}

export interface RouterDecision {
  model: ModelTier;
  reasoning: string;
}

export interface SessionState {
  sessionId: string;
  messages: any[]; // Anthropic MessageParams
  memory: {
    touchedFiles: Set<string>;
    gotchasLearned: number;
  };
}

// ── Tool Executor Types ─────────────────────────────────────────

export interface ToolDefinition {
  name: string;
  description: string;
  input_schema: any; // JSON Schema
}

export type ToolResult = 
  | ReadFileResult
  | WriteFileResult
  | EditFileResult
  | BashResult
  | GrepResult
  | GlobResult
  | ListFilesResult;

export interface ReadFileResult { type: "read_file"; content: string; }
export interface WriteFileResult { type: "write_file"; success: boolean; path: string; }
export interface EditFileResult { type: "edit_file"; success: boolean; path: string; }
export interface BashResult { type: "bash"; stdout: string; stderr: string; exitCode: number; }
export interface GrepResult { type: "grep"; matches: string[]; }
export interface GlobResult { type: "glob"; files: string[]; }
export interface ListFilesResult { type: "list_files"; entries: string[]; }

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

export type OptiCodeErrorCode =
  | "INDEX_FAILED"
  | "PARSE_FAILED"
  | "DB_ERROR"
  | "INVALID_INPUT"
  | "PATH_NOT_FOUND"
  | "GENERATION_FAILED"
  | "HASH_COLLISION"
  | "SCHEMA_MIGRATION_FAILED";

export class OptiCodeError extends Error {
  constructor(
    public readonly code: OptiCodeErrorCode,
    message: string,
    public readonly cause?: unknown,
  ) {
    super(`[${code}] ${message}`);
    this.name = "OptiCodeError";
  }
}
```

---

## Retention Policy

- **File index:** No limit. Rows deleted when files are deleted from disk.
- **Entities:** Cascade-deleted with their parent file_index row.
- **Gotchas:** Keep all. `hit_count` tracks usefulness. `updated_at` is refreshed each time `hit_count` increments.
- **Rules:** Keep all. Manual pruning via future tooling.
- **Generation log:** Keep last 50 entries per file path. Prune on database initialization.

## Schema Migration Strategy

OptiCode does NOT use Drizzle Kit's migration system at runtime. Migrations are hand-written SQL scripts executed programmatically by `src/db/migrations.ts`.

**Migration file convention:**
```
src/db/migrations/
├── 001_initial.ts        # Creates all 7 tables
├── 002_phase2_pruning.ts # Adds hit_count + pinned to rules (future)
└── index.ts              # Exports ordered migration list
```

**Migration runner logic (in `src/db/migrations.ts`):**
1. Create `schema_version` table if it doesn't exist.
2. Query `SELECT MAX(version) FROM schema_version`.
3. For each migration with `version > current`: run `up(db)` inside a transaction, then insert into `schema_version`.
4. If any migration fails: rollback transaction, throw `OptiCodeError("SCHEMA_MIGRATION_FAILED")`.