# 03 MCP Tool Contracts

All tool inputs and outputs are validated with Zod. Copy these schemas into the tool implementation files.

## Tool Search & Discoverability

Since early 2026, Claude Code uses **Tool Search** (lazy loading) for MCP tools by default, reducing context usage by ~95% compared to loading all tool schemas upfront. Claude Code uses semantic matching on tool names and descriptions to decide which tools to load for a given task.

**Implication for Optima:** Our tool names and `.describe()` strings in the Zod schemas are not just documentation — they are the discovery surface. Claude Code decides whether to load `optima_get_context` based on whether its description semantically matches the user's request. Write descriptions that clearly state what the tool does using keywords a developer would naturally use.

**Tool naming convention:** All tools are prefixed with `optima_` to avoid collisions with other MCP servers. The descriptions below are tuned for Tool Search discoverability.

## Tool Descriptions for Tool Search

These descriptions are critical for Claude Code's Tool Search system. They determine whether Claude Code finds and invokes Optima's tools. Descriptions must say WHEN to use the tool, not WHAT it does.

| Tool | Description |
|---|---|
| `optima_get_context` | `Use before modifying any files in a directory, when encountering an error you haven't seen before, when starting work in a new area of the codebase, or when you need to understand what entities and patterns exist in a directory. Pass search_query with error text or keywords to find relevant gotchas and rules across the entire project, not just the current directory. Always call this before writing code.` |
| `optima_memorize` | `Use after fixing any error, after discovering a rule the codebase follows, after noticing a recurring pattern, when the developer states a preference, or after completing a non-trivial task (to record the outcome and key learnings). Call immediately — do not batch or defer.` |
| `optima_reindex` | `Use after major refactoring, after adding or removing significant dependencies, after restructuring directories, or when optima_get_context returns stale results that don't match the current file state.` |
| `optima_dismiss_warning` | `Use to dismiss a security finding that is a false positive, test fixture, or already-rotated secret. Pass the finding_id from the security_warnings array in get_context output.` |
| `optima_forget` | `Use to remove a stale, incorrect, or outdated memory (gotcha, rule, or task outcome). Pass the numeric id from the gotchas, architectural_rules, or task_insights arrays in get_context output. Triggers CLAUDE.md regeneration.` |

**CSO Principles (from Superpowers writing-skills):**
1. Descriptions = triggering conditions only, NOT workflow summaries
2. Start with "Use when..." or "Use before/after..."
3. Include specific symptoms and situations that signal the tool applies
4. Never summarize the tool's internal process
5. If the description tells the agent enough to skip calling the tool, the description is wrong

## MCP Server Bootstrap (`src/index.ts` and `src/server.ts`)

Copy these files verbatim. They form the entry point and tool registration layer.

### `src/index.ts` — Entry point

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { registerTools } from "./server.js";

const server = new Server(
  { name: "optima", version: "0.1.0" },
  { capabilities: { tools: {} } },
);

registerTools(server);

const transport = new StdioServerTransport();

// Graceful shutdown: close DB connection, flush writes
async function shutdown(): Promise<void> {
  try {
    // Import lazily — connection may not exist if no tool was ever called
    const { closeDatabase } = await import("./db/connection.js");
    closeDatabase();
  } catch { /* DB was never opened — fine */ }
  process.exit(0);
}

process.on("SIGINT", shutdown);
process.on("SIGTERM", shutdown);
process.on("SIGHUP", shutdown);

// Prevent unhandled rejections from crashing the server
process.on("unhandledRejection", (reason) => {
  console.error("[optima] Unhandled rejection:", reason);
  // Don't exit — the MCP server should stay alive for future tool calls
});

await server.connect(transport);
```

### `src/server.ts` — Tool registration

```typescript
import type { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { GetContextInputSchema, GetContextOutputSchema } from "./schemas.js";
import { MemorizeInputSchema, MemorizeOutputSchema } from "./schemas.js";
import { ReindexInputSchema, ReindexOutputSchema } from "./schemas.js";
import { handleGetContext } from "./tools/get-context.js";
import { handleMemorize } from "./tools/memorize.js";
import { handleReindex } from "./tools/reindex.js";
import { OptimaError } from "./types.js";

export function registerTools(server: Server): void {
  server.tool(
    "optima_get_context",
    "Get project context, code structure, known errors and fixes, architectural rules, and recent changes for a directory. Lazily re-indexes changed files. Call at the start of any coding task.",
    GetContextInputSchema.shape,
    async (params) => {
      try {
        const result = await handleGetContext(params);
        return { content: [{ type: "text", text: JSON.stringify(result) }] };
      } catch (error) {
        return wrapError(error);
      }
    },
  );

  server.tool(
    "optima_memorize",
    "Store knowledge learned during this session: error fixes, architectural decisions, code patterns, developer preferences, or task outcomes. Call after fixing bugs, making design decisions, establishing conventions, or completing any non-trivial task.",
    MemorizeInputSchema.shape,
    async (params) => {
      try {
        const result = await handleMemorize(params);
        return { content: [{ type: "text", text: JSON.stringify(result) }] };
      } catch (error) {
        return wrapError(error);
      }
    },
  );

  server.tool(
    "optima_reindex",
    "Force a full project re-index. Use after git clone, branch switch, large merge, or when the index seems stale. Rarely needed — normal indexing is automatic.",
    ReindexInputSchema.shape,
    async (params) => {
      try {
        const result = await handleReindex(params);
        return { content: [{ type: "text", text: JSON.stringify(result) }] };
      } catch (error) {
        return wrapError(error);
      }
    },
  );
}

/** Wraps OptimaError (or unknown errors) into MCP error responses. */
function wrapError(error: unknown): { content: Array<{ type: "text"; text: string }>; isError: true } {
  if (error instanceof OptimaError) {
    return {
      content: [{ type: "text", text: `[${error.code}] ${error.message}` }],
      isError: true,
    };
  }
  const message = error instanceof Error ? error.message : String(error);
  return {
    content: [{ type: "text", text: `[UNKNOWN_ERROR] ${message}` }],
    isError: true,
  };
}
```

### `src/db/connection.ts` — Database lifecycle

```typescript
import Database from "better-sqlite3";
import { mkdirSync, writeFileSync, existsSync } from "node:fs";
import { join } from "node:path";
import { runMigrations } from "./migrations.js";
import { OptimaError } from "../types.js";

let db: Database.Database | null = null;
let projectRoot: string | null = null;

/** Returns the project root path. Resolves from cwd on first call. */
export function getProjectRoot(): string {
  if (!projectRoot) {
    projectRoot = process.cwd();
  }
  return projectRoot;
}

/** Lazily initializes the database. First call creates .optima/ and runs migrations. */
export function getDatabase(): Database.Database {
  if (db) return db;

  const root = getProjectRoot();
  const dbDir = join(root, ".optima");
  const dbPath = join(dbDir, "optima.db");

  // Step 1: Create directory
  mkdirSync(dbDir, { recursive: true });

  // Step 2: Open connection
  try {
    db = new Database(dbPath);
  } catch (error: unknown) {
    // Corruption detection
    if (error instanceof Error && /SQLITE_CORRUPT|SQLITE_NOTADB/.test(error.message)) {
      const { unlinkSync } = require("node:fs");
      try { unlinkSync(dbPath); } catch {}
      try { unlinkSync(dbPath + "-wal"); } catch {}
      try { unlinkSync(dbPath + "-shm"); } catch {}
      db = new Database(dbPath);
    } else {
      throw new OptimaError("DB_ERROR", `Failed to open database: ${String(error)}`, error);
    }
  }

  // Step 3-4: Enable WAL and foreign keys
  db.pragma("journal_mode = WAL");
  db.pragma("foreign_keys = ON");

  // Step 5: Run migrations
  try {
    runMigrations(db);
  } catch (error) {
    throw new OptimaError("SCHEMA_MIGRATION_FAILED", String(error), error);
  }

  // Step 6: Write inner .gitignore
  const innerGitignore = join(dbDir, ".gitignore");
  if (!existsSync(innerGitignore)) {
    writeFileSync(innerGitignore, "*\n");
  }

  return db;
}

/** Close the database connection (called during shutdown). */
export function closeDatabase(): void {
  if (db) {
    db.close();
    db = null;
  }
}
```

---

## Zod Schemas

```tsx
// File: src/schemas.ts — Copy these schemas into this file.

import { z } from "zod";

// ── optima_get_context ────────────────────────────────────────

export const GetContextInputSchema = z.object({
  path: z.string().min(1).describe("Directory or file path to get context for"),
  task_type: z.enum(["bug_fix", "feature", "refactor", "test", "review"]).optional()
    .describe("Optional hint about the type of task being performed"),
  // **Status: Reserved / Currently unused.** The field is accepted and validated but not yet used by the handler. It is intended as a future hint for context filtering (e.g., return gotchas more relevant to bug_fix vs. refactor tasks). For MVP, omit it from tool calls — it has no effect on the response.
  search_query: z.string().max(200).optional()
    .describe("Free-text search across gotchas and rules — enables cross-directory discovery. Pass the error message when debugging, or keywords like 'authentication timeout' to find relevant gotchas outside the current directory."),
});

export const GetContextOutputSchema = z.object({
  project: z.object({
    name: z.string(),
    purpose: z.string().nullable(),
    linter_detected: z.array(z.string()).nullable(), // Parsed from JSON string in DB. null if none detected.
    tech_stack: z.array(z.string()),
    build_command: z.string().nullable(),
    test_command: z.string().nullable(),
    lint_command: z.string().nullable(),
  }),
  directory_context: z.object({
    path: z.string(),
    description: z.string(),
    key_files: z.array(z.string()),
    entities: z.array(z.object({
      name: z.string(),
      kind: z.enum(["function", "class", "interface", "type", "export", "variable"]),
      file: z.string(),
      line: z.number(),
      signature: z.string().nullable(),
      exported: z.boolean(),
      description: z.string().nullable(),
    })),
  }),
  gotchas: z.array(z.object({
    id: z.number(),
    error_text: z.string(),
    resolution: z.string(),
    root_cause: z.string().nullable(),
    dependency_version: z.string().nullable(),
    possibly_outdated: z.boolean(),  // computed: true if dependency_version differs from current project version
    files: z.array(z.string()),
    directory: z.string().nullable(),
    hit_count: z.number(),
  })),
  architectural_rules: z.array(z.object({
    id: z.number(),
    type: z.enum(["architectural_rule", "pattern", "preference"]),
    content: z.string(),
    rationale: z.string().nullable(),
    directory: z.string().nullable(),
  })),
  security_warnings: z.array(z.object({
    id: z.number(),
    file: z.string(),
    line: z.number(),
    pattern_name: z.string().describe("Type of secret detected: aws_key, private_key, generic_api_key, jwt, connection_string"),
    severity: z.enum(["critical", "high", "medium"]),
    snippet: z.string().nullable().describe("Redacted preview — NEVER the actual secret value"),
    dismissed: z.boolean(),
  })).describe("Potential secrets or credentials detected in source files during indexing"),
  task_insights: z.array(z.object({
    id: z.number(),
    task: z.string(),
    outcome: z.enum(["success", "partial", "abandoned"]),
    learnings: z.string().nullable(),
    files: z.array(z.string()),
    directory: z.string().nullable(),
    duration_hint: z.string().nullable(),
    created_at: z.string(),
  })).describe("Past task outcomes relevant to the current directory — what worked, what didn't, how long it took"),
  recent_changes: z.array(z.string()),
  dependency_context: z.object({
    key_dependencies: z.array(z.object({
      name: z.string().describe("Dependency name, e.g. 'drizzle-orm'"),
      version: z.string().describe("Version string from package.json, e.g. '^0.39.0'"),
      detected_from: z.string().describe("Source file, e.g. 'package.json'"),
    })),
  }).describe("Dependency versions for key project dependencies (matching tech stack detection dictionary)"),
  // **How `dependency_context.key_dependencies` is populated:** `project-analyzer.ts` parses `package.json` (both `dependencies` and `devDependencies`) and matches against an internal `DEPENDENCY_MAP` dictionary. Only known, significant dependencies are included — not every package. Examples of detected dependencies: `typescript`, `react`, `next` (→ next.js), `drizzle-orm` (→ drizzle), `vitest`, `@modelcontextprotocol/sdk` (→ mcp). The `detected_from` field is always `"package.json"` for MVP. The version string is the raw semver range from `package.json` (e.g., `"^0.39.0"`), not the resolved version.
  // Verification fields (for Iron Law 3: Evidence Before Claims)
  files_reindexed: z.number(),        // how many files had changed mtimes and were re-indexed
  files_removed: z.number(),          // how many deleted files were cleaned from the index
  reindex_duration_ms: z.number(),    // how long re-indexing took
  is_cold_start: z.boolean(),         // true if this was the first-ever call (full project init)
});

// ── optima_memorize ───────────────────────────────────────────

export const MemorizeInputSchema = z.object({
  type: z.enum(["error_fix", "architectural_rule", "pattern", "preference", "task_outcome"])
    .describe("The kind of knowledge being stored"),
  error: z.string().optional()
    .describe("The original error message or test failure (for error_fix)"),
  resolution: z.string().optional()
    .describe("What fixed the error (for error_fix)"),
  rootCause: z.string().max(500).optional()
    .describe("Why the error occurred — deeper explanation beyond the resolution (for error_fix, optional)"),
  dependencyVersion: z.string().max(100).optional()
    .describe("Dependency name@version active when the error occurred, e.g. 'drizzle-orm@^0.38.0' (for error_fix, optional)"),
  rule: z.string().optional()
    .describe("The rule or decision (for architectural_rule)"),
  rationale: z.string().optional()
    .describe("Why this decision was made (for architectural_rule)"),
  pattern: z.string().optional()
    .describe("The pattern description (for pattern)"),
  example: z.string().optional()
    .describe("Code example demonstrating the pattern (for pattern)"),
  preference: z.string().optional()
    .describe("Developer preference description (for preference)"),
  task: z.string().optional()
    .describe("Task description — what was attempted (for task_outcome)"),
  outcome: z.enum(["success", "partial", "abandoned"]).optional()
    .describe("How the task ended (for task_outcome)"),
  learnings: z.string().max(1000).optional()
    .describe("Key insight from the task — what worked, what didn't, what was surprising (for task_outcome)"),
  duration_hint: z.string().max(50).optional()
    .describe("Rough time estimate, e.g. '~45min' — helps future task estimation (for task_outcome)"),
  // **`duration_hint` format:** Free text, human-readable. No machine parsing is performed on this field — it is stored and displayed as-is. Convention: `"~45min"`, `"~2hr"`, `"~30min"`. The tilde prefix (`~`) signals approximation. Future sessions use this field to calibrate effort estimates. If the actual duration was very short (<5 minutes), omit the field rather than logging `"~0min"`.
  files: z.array(z.string()).optional()
    .describe("Files involved"),
  directory: z.string().optional()
    .describe("Directory scope"),
  tags: z.array(z.string()).optional()
    .describe("Freeform tags for retrieval"),
  // **Tags (`tags?: string[]`):** Accepted and stored as a JSON array in all memory types (gotchas, rules, task outcomes). Currently orphaned — there is no query-by-tags capability and tags are not surfaced in `optima_get_context` output or CLAUDE.md generation. Reserved for Phase 2 filtering. For MVP, omit `tags` from all tool calls.
}).refine(
  (data) => {
    if (data.type === "error_fix") return !!data.error && !!data.resolution;
    if (data.type === "architectural_rule") return !!data.rule;
    if (data.type === "pattern") return !!data.pattern;
    if (data.type === "preference") return !!data.preference;
    if (data.type === "task_outcome") return !!data.task && !!data.outcome;
    return false;
  },
  { message: "Required fields missing for the specified type" },
);

export const MemorizeOutputSchema = z.object({
  stored: z.boolean(),
  memory_id: z.string().uuid(),
  total_memories: z.number(),
  // Verification fields (for Iron Law 3: Evidence Before Claims)
  type: z.enum(["error_fix", "architectural_rule", "pattern", "preference", "task_outcome"]),  // echo back the type for confirmation
  claude_md_regenerated: z.boolean(),  // true if CLAUDE.md was rewritten, false if content hash matched
  claude_md_instruction_count: z.number(),  // current total instructions in CLAUDE.md (budget tracking)
  feedback_rules_written: z.boolean(),  // true if optima-feedback.md was created/updated
  duplicate_detected: z.boolean(),  // true if a gotcha with the same normalized hash already existed
  duplicate_of: z.number().nullable(),  // if stored=false, the ID of the existing record that matched. null otherwise
  hit_count_updated: z.boolean(),  // true if an existing gotcha's hit_count was incremented (dedup case)
});

// ── optima_reindex ────────────────────────────────────────────

export const ReindexInputSchema = z.object({
  path: z.string().optional()
    .describe("Directory to reindex (default: project root)"),
  reason: z.string().optional()
    .describe("Why the reindex was requested (logged)"),
});

export const ReindexOutputSchema = z.object({
  files_indexed: z.number(),
  entities_found: z.number(),
  duration_ms: z.number(),
  // Verification fields (for Iron Law 3: Evidence Before Claims)
  total_files_scanned: z.number(),
  total_entities_found: z.number(),
  project_analysis_updated: z.boolean(),
  claude_md_regenerated: z.boolean(),
  feedback_rules_written: z.boolean(),
});

// ── optima_dismiss_warning ────────────────────────────────────

export const DismissWarningInputSchema = z.object({
  finding_id: z.number(),
  reason: z.string().max(500).optional(),
});

export const DismissWarningOutputSchema = z.object({
  dismissed: z.boolean(),
  finding_id: z.number(),
  file: z.string(),
  line: z.number(),
  pattern_name: z.string(),
  was_already_dismissed: z.boolean(),
  remaining_warnings: z.number(),
});

// ── optima_forget ─────────────────────────────────────────────

export const ForgetInputSchema = z.object({
  memory_id: z.number(),
  reason: z.string().max(500).optional(),
});

export const ForgetOutputSchema = z.object({
  deleted: z.boolean(),
  deleted_from: z.enum(["gotchas", "rules", "task_outcomes"]),
  memory_id: z.number(),
  total_memories: z.number(),
  claude_md_regenerated: z.boolean(),
  claude_md_instruction_count: z.number(),
});
```

---

## Tool Behavior Specifications

### optima_get_context

**Internal steps (in order):**

1. Resolve `path` relative to project root using `path.resolve(projectRoot, inputPath)`. **Security: validate the resolved absolute path starts with `projectRoot`** — if not (e.g., `../../.ssh/`), throw `OptimaError("PATH_NOT_FOUND", "Path escapes project root")`. Then normalize to forward slashes. If the validated path does not exist on disk, throw `OptimaError("PATH_NOT_FOUND")`.
2. Check if project metadata exists in `project_meta` table. If not, run full project analysis first (detect tech stack from `package.json`, `tsconfig.json`, `pyproject.toml`, etc.). During project analysis, also parse `dependencies` and `devDependencies` from `package.json` into `key_dependencies` — filter to dependencies matching the tech stack detection dictionary (`DEPENDENCY_MAP` in Doc 02). Store as JSON array of `{name, version, detected_from}` objects in `project_meta.key_dependencies`.
3. Query `file_index` for all files under the requested path.
4. For each file, check `fs.stat(file).mtimeMs` against the stored `mtime_ms`.
5. Files with newer mtimes: re-read content, recompute hash, re-extract entities via Tree-sitter, update `file_index` and `entities` tables.
5a. **Secret scanning (during step 5):** After reading file content but before entity extraction, run `SECRET_PATTERNS` regex pass over the content. For each match: store a `security_findings` row with the file_id, line number, pattern name, and a **redacted** snippet (first 4 + `****` + last 4 chars of the matched value — NEVER the full secret). If a finding already exists for the same file_id + line + pattern_name, skip (dedup). Note: during lazy re-indexing, only changed files are re-scanned — unchanged files retain their existing findings (including dismissed state). During `optima_reindex`, file_index rows are deleted and re-created (cascade deletes findings), so dismissed state resets for all files.
6. Files with matching mtimes: skip (index is fresh).
7. Files in `file_index` that no longer exist on disk: delete from `file_index` (cascade deletes entities).
8. Query `gotchas` table filtered by `directory` matching the requested path (see **Gotcha Retrieval Strategy** below). If `search_query` is provided, ALSO run FTS5 search on `gotchas_fts` (see **FTS5 Search Merge Strategy** below) and merge results. Deduplicate by gotcha `id`. Increment `hit_count` for all returned gotchas. Update `updated_at` to current timestamp. For each returned gotcha with non-null `dependency_version`: parse the dependency name from the stored string (format: `name@version`), look up the current version in `project_meta.key_dependencies`. If the versions differ or the dependency is no longer present, set `possibly_outdated: true` on the output. If `dependency_version` is null, set `possibly_outdated: false`.
9. Query `rules` table filtered by `directory` matching the requested path (see **Directory Scoping Precedence** below). If `search_query` is provided, ALSO run FTS5 search on `rules_fts` and merge results. Deduplicate by rule `id`. Most-specific-directory-first ordering still applies; FTS results without directory match are appended after directory-matched results.
9a. Query `task_outcomes` table filtered by `directory` matching the requested path (same hierarchical prefix match as gotchas/rules). Return outcomes with non-null `learnings`, sorted by `created_at` DESC, capped at 5 entries.
9b. Query `security_findings` table for all non-dismissed findings whose `file_id` references a file under the requested path. Join with `file_index` to get the file path. Sort by severity (critical > high > medium), then by `found_at` DESC. Cap at 10 entries.
10. Collect `recent_changes` — file paths re-indexed in steps 5-7, sorted by mtime descending, capped at 20 entries. Empty on cold start.
11. Assemble and return `GetContextOutput`. For `directory_context.description`, use **pure string concatenation** (no LLM inference):
    - If a `README.md` exists in the requested directory: use its first non-empty, non-heading line (truncated to 200 chars).
    - Else if `projectPurpose` is non-null: `"${dirName} — part of ${projectPurpose}"`.
    - Else: `"${dirName} directory"`.
    - Example: `"auth — part of REST API for managing user accounts"` or `"utils directory"`.

**Gotcha Retrieval Strategy:**

Gotchas are matched to the current context using TWO mechanisms:

1. **Directory matching:** Return gotchas where `directory` matches or is a parent of the requested path, OR `directory IS NULL` (project-wide gotchas). Example: requesting path `src/api/auth/` returns gotchas scoped to `src/api/auth/`, `src/api/`, `src/`, and project-wide gotchas.
2. **File matching:** Return gotchas where any entry in the `files` JSON array matches a file under the requested path.

Results are deduplicated by gotcha `id` and sorted by `hit_count` descending. Capped at 10 entries returned per call (matching the CLAUDE.md instruction budget). All matching gotchas get their `hit_count` incremented, even if they exceed the cap (tracking is separate from display).

**FTS5 Search Merge Strategy:**

When `search_query` is provided in the input, run FTS5 full-text search in ADDITION to the structural directory/file matching above. This enables cross-directory gotcha and rule discovery — a gotcha stored against `src/api/auth/` can be found from `src/middleware/` if the search terms match.

1. Run directory/file matching (existing logic above) → produces `directoryResults` set.
2. Run `searchGotchasFts(db, search_query, 10)` → produces `ftsResults` set (gotcha IDs ranked by FTS5 relevance).
3. Merge: start with `directoryResults` (these have structural relevance). Append `ftsResults` that aren't already in `directoryResults` (these have textual relevance but may be from unrelated directories).
4. Deduplicate by gotcha `id`. Final sort: directory-matched results first (sorted by `hit_count` DESC), then FTS-only results (sorted by FTS5 rank).
5. Cap at 10 total entries.
6. Same merge logic applies to rules via `searchRulesFts()`.

**FTS5 error handling:** If the `search_query` contains FTS5 syntax errors (e.g., unbalanced quotes), catch the SQLite error and fall back to directory-only matching. Do not throw — a bad search query should degrade gracefully, not fail the entire tool call. Log: `"FTS5 query error for '${search_query}': ${error.message}. Falling back to directory-only matching."`

**Directory Scoping Precedence:**

When querying `rules` and `gotchas` by directory, apply hierarchical matching — a rule scoped to `src/api/` applies to `src/api/auth/login.ts`. The matching logic:

1. Return all entries where `directory IS NULL` (project-wide).
2. Return all entries where `directory` is an exact prefix of the requested path (after forward-slash normalization). **Path boundary check:** use `requestedPath === directory || requestedPath.startsWith(directory + "/")` to prevent `src/api` from matching `src/api-v2/`. Ensure both paths use forward slashes and have no trailing slash before comparison.
3. Sort results: most specific directory first (longest `directory` path), then by `created_at` descending within the same scope.
4. When the same topic has rules at multiple scopes, the most specific scope takes precedence in CLAUDE.md generation (e.g., a rule at `src/api/auth/` overrides a rule at `src/api/` if they cover the same subject).

**Tree-sitter entity extraction (step 5):**

Parse TypeScript files using `tree-sitter-typescript`. Extract:

- Top-level function declarations → kind: `"function"`
- Class declarations → kind: `"class"`
- Interface declarations → kind: `"interface"`
- Type alias declarations → kind: `"type"`
- Named exports → kind: `"export"`
- Top-level const/let declarations → kind: `"variable"`

For each entity, capture: name, line range, signature (for functions: parameter list and return type), whether it’s exported, and JSDoc description if present. **JSDoc extraction:** implemented via `extractJsDocDescription()` in the verbatim code below — checks if the preceding named sibling is a `/** ... */` comment, strips decorators, and returns the first paragraph before any `@param`/`@returns` tags. Returns `null` if no JSDoc present. Getter/setter members inside a class do not inherit the class JSDoc; their `description` is always `null`.

**Entity extraction edge cases:**

- **Async functions:** Extract as kind `"function"`. The `signature` should include `async` prefix.
- **Arrow functions assigned to const/let:** Extract as kind `"variable"`. If the arrow function has a name (via the variable), capture it. Signature is the parameter list.
- **Default exports:** If `export default function foo()`, extract as kind `"function"` with `exported: true`. If `export default class Bar`, extract as kind `"class"` with `exported: true`. Anonymous default exports (`export default () => {}`) are skipped — no meaningful name to capture.
- **Re-exports:** `export { foo } from ‘./bar’` — extract as kind `"export"` with name `foo`. Do NOT follow the import to resolve the original definition.
- **Function overloads:** Extract only the implementation signature (not the overload declarations). One entity per overloaded function.
- **Getter/setter pairs:** Extract each as kind `"function"` with name `get_propName` / `set_propName`. Two entities per pair.
- **`declare` statements (ambient declarations):** Skip entirely. These describe external types and are not project entities.
- **Namespace declarations:** Skip. Namespaces are a legacy pattern; their contents are typically re-exported.
- **Generic types:** Extract normally. The `signature` field should include the generic parameter list (e.g., `<T extends Base>`).
- **Union/intersection types:** Extract as kind `"type"`. No special handling needed — Tree-sitter gives us the full declaration.
- **Enum declarations:** Extract as kind `"type"`. Enums are conceptually type definitions.

**Tree-sitter entity extraction implementation (`src/indexer/entity-extractor.ts`):**

Copy this module verbatim. It uses the `tree-sitter` and `tree-sitter-typescript` packages with the Node API (not WASM).

```typescript
import Parser from "tree-sitter";
import TypeScript from "tree-sitter-typescript";
import type { EntitySummary } from "../types.js";

const parser = new Parser();
parser.setLanguage(TypeScript.typescript);

// Node types that indicate ambient/declaration context — skip these
const AMBIENT_PARENTS = new Set(["ambient_declaration", "declare_statement"]);

/**
 * Extracts the first paragraph of a JSDoc comment immediately preceding a node.
 * Returns null if the preceding sibling is not a `/** ... *\/` comment.
 */
function extractJsDocDescription(node: Parser.SyntaxNode, source: string): string | null {
  const prev = node.previousNamedSibling;
  if (!prev || prev.type !== "comment") return null;
  const comment = source.slice(prev.startIndex, prev.endIndex);
  if (!comment.startsWith("/**")) return null;
  const body = comment
    .slice(3, -2)                                // strip /** and */
    .split("\n")
    .map(line => line.replace(/^\s*\*\s?/, ""))  // strip leading * from each line
    .join(" ")
    .trim();
  // Return first paragraph only — stop before any @param/@returns/@tags
  const firstParagraph = body.split(/\s@/)[0].trim();
  return firstParagraph || null;
}

export function extractEntities(source: string, filePath: string): EntitySummary[] {
  const tree = parser.parse(source);
  const root = tree.rootNode;
  const entities: EntitySummary[] = [];

  for (let i = 0; i < root.childCount; i++) {
    let node = root.child(i)!;
    let exported = false;

    // Skip ambient declarations (declare module, declare global, etc.)
    if (AMBIENT_PARENTS.has(node.type)) continue;

    // Capture description from leading JSDoc before any unwrapping.
    // The JSDoc comment is a sibling of the export_statement (or declaration itself).
    const description = extractJsDocDescription(node, source);

    // Unwrap export_statement to get the inner declaration
    if (node.type === "export_statement") {
      exported = true;
      const decl = node.childForFieldName("declaration");
      const clause = node.childForFieldName("source")
        ? null  // re-export: export { x } from './y'
        : node.children.find((c) => c.type === "export_clause");

      // Named re-exports: export { foo, bar } from './baz'
      if (node.childForFieldName("source") && clause) {
        // Skip re-exports with source — they reference external modules
        continue;
      }

      // Local re-exports: export { foo, bar }
      if (clause) {
        for (const spec of clause.children) {
          if (spec.type === "export_specifier") {
            const name = spec.childForFieldName("name")?.text
              ?? spec.childForFieldName("alias")?.text;
            if (name) {
              entities.push({
                name, kind: "export", file: filePath,
                line: spec.startPosition.row + 1,
                signature: null, exported: true,
              });
            }
          }
        }
        continue;
      }

      // Default export without declaration: export default expr
      if (!decl) {
        // Anonymous default export — skip (no meaningful name)
        continue;
      }
      node = decl;
    }

    switch (node.type) {
      case "function_declaration":
      case "generator_function_declaration": {
        const name = node.childForFieldName("name")?.text;
        if (!name) break;
        const params = node.childForFieldName("parameters")?.text ?? "()";
        const retType = node.childForFieldName("return_type")?.text ?? "";
        const isAsync = source.slice(node.startIndex, node.startIndex + 6) === "async ";
        entities.push({
          name, kind: "function", file: filePath,
          line: node.startPosition.row + 1,
          signature: `${isAsync ? "async " : ""}${name}${params}${retType}`,
          exported, description: description ?? null,
        });
        break;
      }

      case "class_declaration": {
        const name = node.childForFieldName("name")?.text;
        if (!name) break;
        entities.push({
          name, kind: "class", file: filePath,
          line: node.startPosition.row + 1,
          signature: null, exported, description: description ?? null,
        });
        // Extract getters/setters from class body
        const body = node.childForFieldName("body");
        if (body) {
          for (let j = 0; j < body.childCount; j++) {
            const member = body.child(j)!;
            if (member.type === "method_definition") {
              const kind = member.children.find(
                (c) => c.type === "get" || c.type === "set"
              );
              if (kind) {
                const propName = member.childForFieldName("name")?.text;
                if (propName) {
                  const params = member.childForFieldName("parameters")?.text ?? "";
                  const retType = member.childForFieldName("return_type")?.text ?? "";
                  entities.push({
                    name: `${kind.type}_${propName}`,
                    kind: "function", file: filePath,
                    line: member.startPosition.row + 1,
                    signature: `${kind.type} ${propName}${params}${retType}`,
                    exported, description: null,  // getters/setters don't get JSDoc from outer scope
                  });
                }
              }
            }
          }
        }
        break;
      }

      case "interface_declaration": {
        const name = node.childForFieldName("name")?.text;
        if (!name) break;
        const typeParams = node.children.find(
          (c) => c.type === "type_parameters"
        )?.text ?? "";
        entities.push({
          name, kind: "interface", file: filePath,
          line: node.startPosition.row + 1,
          signature: typeParams ? `${name}${typeParams}` : null,
          exported, description: description ?? null,
        });
        break;
      }

      case "type_alias_declaration": {
        const name = node.childForFieldName("name")?.text;
        if (!name) break;
        const typeParams = node.children.find(
          (c) => c.type === "type_parameters"
        )?.text ?? "";
        entities.push({
          name, kind: "type", file: filePath,
          line: node.startPosition.row + 1,
          signature: typeParams ? `${name}${typeParams}` : null,
          exported, description: description ?? null,
        });
        break;
      }

      case "enum_declaration": {
        const name = node.childForFieldName("name")?.text;
        if (!name) break;
        entities.push({
          name, kind: "type", file: filePath,
          line: node.startPosition.row + 1,
          signature: null, exported, description: description ?? null,
        });
        break;
      }

      case "lexical_declaration":
      case "variable_declaration": {
        for (const declarator of node.children) {
          if (declarator.type !== "variable_declarator") continue;
          const name = declarator.childForFieldName("name")?.text;
          if (!name) continue;
          const value = declarator.childForFieldName("value");
          // Arrow functions get a signature
          let signature: string | null = null;
          if (value?.type === "arrow_function") {
            const params = value.childForFieldName("parameters")?.text ?? "()";
            const retType = value.childForFieldName("return_type")?.text ?? "";
            signature = `${name}${params}${retType}`;
          }
          entities.push({
            name, kind: "variable", file: filePath,
            line: declarator.startPosition.row + 1,
            signature, exported, description: description ?? null,
          });
        }
        break;
      }

      // Skip: namespace_declaration, module_declaration, function_signature (overload)
      default:
        break;
    }
  }

  return entities;
}
```

**Function overload handling:** The `function_signature` node type (overload declarations without body) is NOT in the switch — only `function_declaration` (which has a body) is extracted. This ensures one entity per overloaded function.

**Error recovery:** If `parser.parse(source)` returns a tree with `tree.rootNode.hasError`, the tree is still usable — Tree-sitter does error recovery. Extract what we can. Only return empty entities if the file is completely unparseable (e.g., binary content that got past the binary check).

Files that fail catastrophically should be indexed in `file_index` (path, mtime, hash) but produce zero entities. Log a warning via `console.warn`, do not throw.

---

**File walker implementation (`src/indexer/file-indexer.ts`):**

```typescript
import { readdir, stat, lstat, readFile } from "node:fs/promises";
import { join } from "node:path";
import ignore from "ignore";
import { createHash } from "node:crypto";
import type { FileState } from "../types.js";

// ── Secret Scanning Patterns ─────────────────────────────────
// Run against file content during indexing (step 5a). Results stored
// in security_findings table. NEVER store the matched value — only
// the pattern name, line number, and a redacted snippet.
const SECRET_PATTERNS = [
  { name: "aws_key", pattern: /AKIA[0-9A-Z]{16}/g, severity: "critical" as const },
  { name: "private_key", pattern: /-----BEGIN (?:RSA |EC )?PRIVATE KEY-----/g, severity: "critical" as const },
  { name: "generic_api_key", pattern: /(?:api[_-]?key|apikey)\s*[:=]\s*['"][a-zA-Z0-9_\-]{20,}['"]/gi, severity: "high" as const },
  { name: "jwt", pattern: /eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+/g, severity: "high" as const },
  { name: "connection_string", pattern: /(postgres|mysql|mongodb|redis):\/\/[^\s"']+/gi, severity: "critical" as const },
  { name: "github_token", pattern: /ghp_[a-zA-Z0-9]{36}/g, severity: "critical" as const },
  { name: "generic_secret", pattern: /(?:secret|password|passwd|token)\s*[:=]\s*['"][a-zA-Z0-9_\-]{8,}['"]/gi, severity: "medium" as const },
];

/** Redact a matched secret value for safe storage. NEVER store the raw value. */
function redactSecret(value: string): string {
  if (value.length <= 8) return "****";
  return value.slice(0, 4) + "****" + value.slice(-4);
}

/** Scan file content for secret patterns. Returns findings with redacted snippets. */
export function scanForSecrets(content: string, filePath: string): Array<{
  line: number;
  patternName: string;
  severity: "critical" | "high" | "medium";
  snippet: string;
}> {
  const findings: Array<{ line: number; patternName: string; severity: "critical" | "high" | "medium"; snippet: string }> = [];
  const lines = content.split("\n");

  for (const { name, pattern, severity } of SECRET_PATTERNS) {
    // Reset regex lastIndex for each file
    pattern.lastIndex = 0;
    let match: RegExpExecArray | null;
    while ((match = pattern.exec(content)) !== null) {
      // Find line number from character offset
      const beforeMatch = content.slice(0, match.index);
      const lineNum = beforeMatch.split("\n").length;
      findings.push({
        line: lineNum,
        patternName: name,
        severity,
        snippet: `${name} detected: ${redactSecret(match[0])}`,
      });
    }
  }

  return findings;
}

// Binary file extensions — skip these entirely
const BINARY_EXTENSIONS = new Set([
  ".png", ".jpg", ".jpeg", ".gif", ".ico", ".webp", ".bmp", ".svg",
  ".woff", ".woff2", ".ttf", ".eot", ".otf",
  ".mp3", ".mp4", ".wav", ".ogg", ".webm", ".avi", ".mov",
  ".zip", ".tar", ".gz", ".bz2", ".7z", ".rar",
  ".pdf", ".exe", ".dll", ".so", ".dylib", ".bin",
  ".db", ".sqlite", ".sqlite3",
]);

// Hardcoded exclusions — always skip regardless of .gitignore
const ALWAYS_EXCLUDE = [
  "node_modules", ".git", ".optima", "dist", "build", "out",
  ".env",
  ".env.local", ".env.development", ".env.production", ".env.staging",
  ".env.development.local", ".env.production.local",
  // NOTE: .env.example and .env.template are NOT excluded — they are
  // typically committed and safe to index (they contain placeholder values).
  "**/secrets",           // Matches any secrets/ directory at any depth (per Doc 00 security-sensitive paths)
  ".claude/settings.local.json",
  "CLAUDE.local.md",
];

const CLAUDE_INTERNAL_DIRS = [
  ".claude/session-env", ".claude/projects", ".claude/file-history",
  ".claude/paste-cache", ".claude/image-cache",
];
```

**Traversal order:** Depth-first, alphabetical within each directory. This ensures deterministic ordering across runs and platforms. Implementation:

```typescript
export async function walkProject(
  rootPath: string,
  gitignorePath?: string,
): Promise<string[]> {
  // Build ignore filter
  const ig = ignore();
  ALWAYS_EXCLUDE.forEach((p) => ig.add(p));
  if (gitignorePath) {
    try {
      const content = await readFile(gitignorePath, "utf-8");
      ig.add(content);
    } catch { /* no .gitignore — fine */ }
  }

  const results: string[] = [];

  async function walk(dir: string, relativeDir: string): Promise<void> {
    let entries;
    try {
      entries = await readdir(dir, { withFileTypes: true });
    } catch {
      return; // Permission denied or deleted — skip
    }

    // Sort alphabetically for deterministic order
    entries.sort((a, b) => a.name.localeCompare(b.name));

    for (const entry of entries) {
      const relativePath = relativeDir
        ? `${relativeDir}/${entry.name}`
        : entry.name;

      // Check gitignore + hardcoded exclusions
      if (ig.ignores(relativePath)) continue;

      // Check Claude Code internal dirs
      if (CLAUDE_INTERNAL_DIRS.some((d) => relativePath.startsWith(d))) continue;

      const fullPath = join(dir, entry.name);

      if (entry.isDirectory()) {
        await walk(fullPath, relativePath);
      } else if (entry.isFile()) {
        // Check symlink
        try {
          const lstats = await lstat(fullPath);
          if (lstats.isSymbolicLink()) continue; // Skip symlinks
        } catch { continue; }

        // Check binary extension
        const ext = entry.name.substring(entry.name.lastIndexOf(".")).toLowerCase();
        if (BINARY_EXTENSIONS.has(ext)) continue;

        results.push(relativePath.replace(/\\/g, "/")); // Forward-slash normalize
      }
    }
  }

  await walk(rootPath, "");
  return results;
}
```

**Binary detection fallback:** If extension is unknown, read first 512 bytes and check for null bytes:

```typescript
export async function isBinaryFile(filePath: string): Promise<boolean> {
  try {
    const fd = await readFile(filePath, { flag: "r" });
    const sample = fd.subarray(0, 512);
    return sample.includes(0x00);
  } catch { return true; } // Can't read → treat as binary
}
```

**Content hashing:**

```typescript
export function hashContent(content: string): string {
  return createHash("sha256").update(content, "utf-8").digest("hex");
}

// Empty file hash (constant — SHA-256 of empty string)
export const EMPTY_FILE_HASH =
  "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855";
```

**Performance budget:** <500ms for incremental re-index (<20 changed files). <2s for cold start full index of a typical project (<500 files).

---

**Gitignore integration notes:**

The `ignore` npm package (^7.0.0) implements the full `.gitignore` specification including negation patterns (`!important.log`), directory-only patterns (`logs/`), and nested `.gitignore` files. Optima reads only the root `.gitignore` in MVP. The `ig.ignores(relativePath)` call takes a forward-slash relative path from the project root.

Optima DOES index `.claude/settings.json`, `.claude/agents/`, `.claude/commands/`, `.claude/skills/`, and `.claude/rules/` because these are project configuration files. But it never indexes Claude Code's internal operational state.

---

### optima_memorize

**Internal steps:**

1. Validate input with `MemorizeInputSchema`.
2. Based on `type`:
    - `error_fix`: **First, sanitize the raw error text** — scrub sensitive data (API keys, JWTs, database connection strings, bearer tokens, long hash strings) using the `sanitizeError()` function below. Store the sanitized version as `error_text` in the database. Then normalize the sanitized string (strip paths, line numbers, timestamps) and compute `error_hash` as SHA-256 of the normalized string. Check `gotchas` table for existing entry with same `error_hash`. If found: update `resolution`, `root_cause`, `dependency_version`, `files`, `updatedAt`. If not found: insert new row with all fields (see MemorizeInput → DB Column Mapping in Doc 02).
    - `architectural_rule`: Insert into `rules` table with `type = "architectural_rule"`.
    - `pattern`: Insert into `rules` table with `type = "pattern"`.
    - `preference`: Insert into `rules` table with `type = "preference"`.
    - `task_outcome`: Insert into `task_outcomes` table. Store `task`, `outcome`, `learnings`, `duration_hint`, `files`, `directory`, `tags`.
3. Generate a random UUID v4 for `memory_id`. This is a correlation ID for the caller to reference this memorize operation — it is NOT the database row's `id`. The UUID is generated fresh per call and is not stored in the database. It serves as a receipt: "your memorize call succeeded, here's a reference."
4. Count total rows across `gotchas` + `rules` + `task_outcomes` tables for `total_memories`.
5. **Trigger CLAUDE.md regeneration** after EVERY successful memorize call, regardless of type. Rationale: all memory types (error_fix, architectural_rule, pattern, preference, task_outcome) contribute to CLAUDE.md sections. The generator uses content hashing (step 5 of regeneration logic in Doc 04) to skip the actual file write if nothing changed, so frequent regeneration is cheap. Do NOT selectively regenerate only for project-wide rules — this causes stale CLAUDE.md when gotchas or patterns accumulate.
6. Return `MemorizeOutput`.

**Error normalization for dedup:**

```tsx
// File: src/memory/error-normalizer.ts

function normalizeError(error: string): string {
  return error
    // Strip Unix paths (/foo/bar.ts)
    .replace(/\/[\w\-./]+\.(ts|js|tsx|jsx)/g, "<file>")
    // Strip Windows paths (C:\foo\bar.ts or C:/foo/bar.ts)
    .replace(/[A-Z]:[\\\/][\w\-.\\/]+\.(ts|js|tsx|jsx)/gi, "<file>")
    // Strip line:column references
    .replace(/:\d+:\d+/g, ":<line>")
    // Strip ISO timestamps
    .replace(/\d{4}-\d{2}-\d{2}T[\d:.]+Z?/g, "<time>")
    // Strip memory addresses
    .replace(/0x[0-9a-f]+/gi, "<addr>")
    // Strip UUIDs
    .replace(/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi, "<uuid>")
    // Strip secrets, API keys, and tokens
    .replace(/bearer\s+[\w\-.]+/gi, "<token>")
    .replace(/[a-zA-Z0-9_-]{40,}/g, "<secret>")
    .trim()
    .toLowerCase();
}
```

**Error sanitization function (runs BEFORE normalization, BEFORE storage):**

```tsx
// File: src/memory/error-normalizer.ts

function sanitizeError(error: string): string {
  return error
    // Strip Bearer tokens
    .replace(/bearer\s+[\w\-.]+/gi, "bearer <REDACTED>")
    // Strip AWS access key IDs (AKIA prefix, 20 chars — too short for 40+ catch-all)
    .replace(/AKIA[0-9A-Z]{16}/g, "<AWS_KEY_REDACTED>")
    // Strip GitHub tokens (ghp_ prefix)
    .replace(/ghp_[a-zA-Z0-9]{36}/g, "<GITHUB_TOKEN_REDACTED>")
    // Strip API keys (common prefixes: sk-, pk-, api_, key_)
    .replace(/(?:sk|pk|api|key)[-_][a-zA-Z0-9]{20,}/g, "<API_KEY_REDACTED>")
    // Strip private key PEM headers
    .replace(/-----BEGIN (?:RSA |EC )?PRIVATE KEY-----/g, "<PRIVATE_KEY_REDACTED>")
    // Strip database connection strings
    .replace(/(postgres|mysql|mongodb|redis):\/\/[^\s"']+/gi, "<DB_URL_REDACTED>")
    // Strip JWTs (three base64 segments separated by dots)
    .replace(/eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+/g, "<JWT_REDACTED>")
    // Strip password/secret/token assignments (values 8+ chars, catches short secrets)
    .replace(/(?:secret|password|passwd|token)\s*[:=]\s*['"]([^'"]{8,})['"]/gi, (_, p1) => `<SECRET_REDACTED>`)
    // Strip long hex/base64 strings (likely tokens, >=40 chars)
    .replace(/[a-zA-Z0-9_-]{40,}/g, "<SECRET_REDACTED>");
}
```

**Important:** Sanitization runs on the raw error text before storage in the `gotchas.error_text` column. This prevents accidental exposure of secrets in `CLAUDE.md` generated output. The sanitized text is what gets stored; the original unsanitized error is never persisted.

**Important:** The normalization intentionally preserves the error *type* and *message structure*. Two errors like "cannot find module 'foo'" and "cannot find module 'bar'" will normalize to the same hash — this is correct behavior. They represent the same *class* of error. The raw `error_text` (stored verbatim) preserves the specifics. **Security warning:** The raw `error_text` must also have sensitive secrets (like API keys, JWTs, and long hash strings) redacted before storage in the database to prevent accidental token leaks in `CLAUDE.md`. Apply a scrubbing pass against the raw error before creating the `gotchas` record.

---

### optima_reindex

**Internal steps:**

1. Resolve `path` (default: project root). **Validate path stays within project root** (same security check as `optima_get_context` step 1).
2. Delete all entries in `file_index` (and cascaded `entities`) for files under the path.
3. Run full file walk + Tree-sitter extraction for all files under the path.
4. Update `project_meta.lastFullIndex` to current timestamp.
5. Re-run project analysis (tech stack, commands, purpose) to pick up changes from branch switch or major restructure.
6. **Trigger CLAUDE.md regeneration** (project context changed substantially after reindex — `project_overview` section may need updating, entity counts changed, etc.). The generator's content hash check prevents unnecessary file writes.
7. **Regenerate `.claude/rules/optima-feedback.md`** (in case it was deleted or the project structure changed).
8. Log the reindex event with `reason` if provided.
9. Return `ReindexOutput`.

---

## Verification Requirements

Inspired by Superpowers' verification-before-completion pattern: **evidence before claims, always.**

### optima_memorize Output Verification

After every `optima_memorize` call, the tool output MUST include verification fields:

```typescript
interface MemorizeVerificationOutput {
  memory_id: string;              // UUID v4 receipt — proof the record was stored
  type: MemorizeType;             // echo back the type for confirmation
  claude_md_regenerated: boolean; // true if CLAUDE.md was rewritten, false if content hash matched
  claude_md_instruction_count: number; // current total instructions in CLAUDE.md (budget tracking)
  feedback_rules_written: boolean;    // true if optima-feedback.md was created/updated
  duplicate_detected: boolean;        // true if a gotcha with the same normalized hash already existed
}
```

Claude Code MUST report these fields to the developer. Minimum acceptable report:

```
✅ Stored error fix (ID: abc-123). CLAUDE.md regenerated (27/35 instructions).
```

Or if duplicate:

```
ℹ️ Similar gotcha already exists (ID: existing-456). Hit count incremented. CLAUDE.md unchanged.
```

### optima_get_context Output Verification

If `optima_get_context` performed re-indexing (files had changed mtimes), the output MUST include:

```typescript
interface GetContextVerificationFields {
  files_reindexed: number;    // how many files had changed mtimes and were re-indexed
  files_removed: number;      // how many deleted files were cleaned from the index
  reindex_duration_ms: number; // how long re-indexing took
}
```

Claude Code SHOULD mention re-indexing if it happened:

```
🔄 Optima re-indexed 3 changed files in src/auth/ (took 240ms). Context is fresh.
```

### optima_reindex Output Verification

```typescript
interface ReindexVerificationOutput {
  total_files_scanned: number;
  total_entities_found: number;
  project_analysis_updated: boolean;
  claude_md_regenerated: boolean;
  feedback_rules_written: boolean;
  duration_ms: number;
}
```

Claude Code MUST report:

```
🔄 Full re-index complete: 142 files, 87 entities (2.3s). CLAUDE.md regenerated.
```

### The Rule

```
NO COMPLETION CLAIMS WITHOUT TOOL RESPONSE EVIDENCE.
```

If Claude Code says "I've updated Optima" or "gotcha stored" without showing the `memory_id` or `claude_md_regenerated` field, it is making an unverified claim. This violates Iron Law 3.

---

## Performance Budgets

### Per-Phase Timing Budgets for optima_get_context

| Phase | Operation | Budget | Notes |
|---|---|---|---|
| 1 | Path resolution + traversal validation | 5 ms | Synchronous path.resolve + startsWith check |
| 2 | Project meta check | 10 ms | Single SQLite query |
| 3 | File index query (all files under path) | 50 ms | Indexed query on file_index.path |
| 4-6 | Mtime comparison + re-indexing | 100 ms per changed file | fs.stat is async; Tree-sitter parsing ~50ms per file |
| 7 | Deleted file cleanup | 20 ms | Batch DELETE with cascade |
| 8 | Gotcha retrieval (hierarchical match) | 30 ms | Indexed query + hit_count update |
| 9 | Rules retrieval (directory scoping) | 20 ms | Indexed query with prefix match |
| 10 | Recent changes collection | 5 ms | In-memory sort of already-collected data |
| 11 | Output assembly | 10 ms | String concatenation, no LLM |
| **Total (no re-indexing)** | | **< 150 ms** | Typical case when index is fresh |
| **Total (10 files re-indexed)** | | **< 1.5 s** | Worst realistic case |
| **Total (full cold start, 500 files)** | | **< 30 s** | First-ever call on a medium project |

### optima_memorize Budget

| Operation | Budget |
|---|---|
| Input validation (Zod) | 2 ms |
| Error sanitization + normalization (error_fix only) | 10 ms |
| SQLite insert/upsert | 20 ms |
| CLAUDE.md regeneration (content hash check first) | 200 ms |
| **Total** | **< 250 ms** |

### optima_reindex Budget

| Operation | Budget |
|---|---|
| Full file walk (500 files) | 5 s |
| Tree-sitter entity extraction (500 files) | 25 s |
| Project analysis | 500 ms |
| CLAUDE.md regeneration | 200 ms |
| Feedback rules regeneration | 50 ms |
| **Total** | **< 35 s** |

### Enforcement

Add timing assertions to integration tests:

```typescript
it('optima_get_context completes within 150ms when index is fresh', async () => {
  // Pre-populate index, then call get_context
  const start = performance.now();
  await handleGetContext({ path: 'src/' });
  expect(performance.now() - start).toBeLessThan(150);
});

it('optima_memorize completes within 250ms', async () => {
  const start = performance.now();
  await handleMemorize({ type: 'error_fix', error: 'test', resolution: 'test', files: [], directory: 'src/' });
  expect(performance.now() - start).toBeLessThan(250);
});
```

---

## Error Taxonomy

Every error thrown by Optima must use `OptimaError` with one of these codes:

| Code | When | Severity |
| --- | --- | --- |
| `INDEX_FAILED` | File walking or mtime check fails | Recoverable |
| `PARSE_FAILED` | Tree-sitter cannot parse a file | Recoverable (skip file) |
| `DB_ERROR` | SQLite operation fails | Critical |
| `INVALID_INPUT` | Zod validation fails on tool input; also used by `optima_forget` when `memory_id` is not found in any table | Recoverable |
| `PATH_NOT_FOUND` | Requested path does not exist | Recoverable |
| `GENERATION_FAILED` | [CLAUDE.md](http://CLAUDE.md) or rules file generation fails | Recoverable |
| `HASH_COLLISION` | Two different errors produce same normalized hash | Warn and store both |
| `SCHEMA_MIGRATION_FAILED` | Database schema upgrade fails | Critical |

**Error severity behavior:**
- **Recoverable:** Return an MCP error response with `isError: true` and a human-readable message containing the error code. The MCP server process stays alive.
- **Critical:** Return an MCP error response. If `DB_ERROR` or `SCHEMA_MIGRATION_FAILED`, the server may be in an unusable state — subsequent calls will likely fail until the database issue is resolved (e.g., delete and recreate).
- **Warn (HASH_COLLISION only):** Store both entries, return success, but include a `warnings` field in internal logs.

---

## File System Edge Cases

- **Symlinks:** Skip. `fs.lstat` is used instead of `fs.stat` to detect symlinks without following them. If `isSymbolicLink()`, skip the file silently.
- **Permission errors:** If `fs.stat` or `fs.readFile` throws `EACCES` or `EPERM`, skip the file, log a warning. Do not throw `INDEX_FAILED` for a single inaccessible file.
- **File deleted between stat and read:** If `fs.readFile` throws `ENOENT` after a successful `fs.stat`, skip the file. The next index cycle will clean up the stale `file_index` entry.
- **Empty directories:** Ignored. Only files are indexed.
- **Binary files:** Detect by checking if the file extension is in a known set of non-text extensions (`.png`, `.jpg`, `.gif`, `.ico`, `.woff`, `.woff2`, `.ttf`, `.eot`, `.mp3`, `.mp4`, `.zip`, `.tar`, `.gz`, `.pdf`, `.exe`, `.dll`, `.so`, `.dylib`). Skip binary files — do not attempt to read or hash them. If an unknown extension is encountered, read the first 512 bytes and check for null bytes (`\x00`) — if found, treat as binary and skip.
- **Large files:** Files exceeding 1MB are indexed in `file_index` (path, mtime, size, hash) but skipped for Tree-sitter entity extraction. The hash is computed, but parsing multi-MB files is too slow for the performance budget.
- **Empty files:** Indexed in `file_index` with a fixed hash (SHA-256 of empty string). Zero entities extracted.

---

## Testing Infrastructure

### DatabaseProvider Interface

For unit testing, tool handlers should accept a `DatabaseProvider` interface rather than calling `getDatabase()` directly. This allows injecting an in-memory SQLite instance for tests.

```typescript
export interface DatabaseProvider {
  getDatabase(): Database;
}

// Production implementation
export class FileDatabaseProvider implements DatabaseProvider {
  private db: Database | null = null;
  
  getDatabase(): Database {
    if (!this.db) {
      this.db = openDatabase(getProjectRoot());
    }
    return this.db;
  }
}

// Test implementation
export class InMemoryDatabaseProvider implements DatabaseProvider {
  private db: Database;
  
  constructor() {
    this.db = new Database(':memory:');
    runMigrations(this.db);
  }
  
  getDatabase(): Database {
    return this.db;
  }
}
```

### FileSystemProvider Interface

For unit testing file operations (walker, entity extraction), use a provider that can be mocked:

```typescript
export interface FileSystemProvider {
  readFile(path: string): Promise<string>;
  stat(path: string): Promise<{ mtimeMs: number; isSymbolicLink(): boolean }>;
  readdir(path: string): Promise<string[]>;
  exists(path: string): Promise<boolean>;
}
```

Production implementation uses `node:fs/promises`. Test implementation uses an in-memory file map.

### Usage in Tool Handlers

```typescript
// Tool handler accepts providers
export async function handleGetContext(
  params: GetContextInput,
  db: DatabaseProvider = new FileDatabaseProvider(),
  fs: FileSystemProvider = new NodeFileSystemProvider(),
): Promise<GetContextOutput> {
  // ... implementation uses db.getDatabase() and fs.readFile() etc.
}
```

This pattern enables:
- Unit tests with in-memory SQLite (fast, no disk I/O)
- Unit tests with mock file systems (deterministic, no real project needed)
- Integration tests with real providers (end-to-end verification)