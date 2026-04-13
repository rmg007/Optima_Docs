# 03 MCP Tool Contracts

All tool inputs and outputs are validated with Zod. Copy these schemas into the tool implementation files.

## Tool Search & Discoverability

Since early 2026, Claude Code uses **Tool Search** (lazy loading) for MCP tools by default, reducing context usage by ~95% compared to loading all tool schemas upfront. Claude Code uses semantic matching on tool names and descriptions to decide which tools to load for a given task.

**Implication for Optima:** Our tool names and `.describe()` strings in the Zod schemas are not just documentation — they are the discovery surface. Claude Code decides whether to load `optima_get_context` based on whether its description semantically matches the user's request. Write descriptions that clearly state what the tool does using keywords a developer would naturally use.

**Tool naming convention:** All tools are prefixed with `optima_` to avoid collisions with other MCP servers. The descriptions below are tuned for Tool Search discoverability.

## Tool Descriptions for Tool Search

Claude Code uses Tool Search (semantic matching on tool names and descriptions) to decide which MCP tools to load. These are the exact `.describe()` strings to use when registering tools with the MCP SDK. Copy verbatim.

| Tool | Registration Name | Description String (copy exactly) |
| --- | --- | --- |
| `optima_get_context` | `"optima_get_context"` | `"Get project context, code structure, known errors and fixes, architectural rules, and recent changes for a directory. Lazily re-indexes changed files. Call at the start of any coding task."` |
| `optima_memorize` | `"optima_memorize"` | `"Store knowledge learned during this session: error fixes, architectural decisions, code patterns, or developer preferences. Call after fixing bugs, making design decisions, or establishing conventions."` |
| `optima_reindex` | `"optima_reindex"` | `"Force a full project re-index. Use after git clone, branch switch, large merge, or when the index seems stale. Rarely needed — normal indexing is automatic."` |

**Registration example (in `src/server.ts`):**
```tsx
server.tool(
  "optima_get_context",
  "Get project context, code structure, known errors and fixes, architectural rules, and recent changes for a directory. Lazily re-indexes changed files. Call at the start of any coding task.",
  GetContextInputSchema.shape,
  async (params) => { /* handler */ }
);
```

## Zod Schemas

```tsx
import { z } from "zod";

// ── optima_get_context ────────────────────────────────────────

export const GetContextInputSchema = z.object({
  path: z.string().min(1).describe("Directory or file path to get context for"),
  task_type: z.enum(["bug_fix", "feature", "refactor", "test", "review"]).optional()
    .describe("Optional hint about the type of task being performed"),
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
    })),
  }),
  gotchas: z.array(z.object({
    id: z.number(),
    error_text: z.string(),
    resolution: z.string(),
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
  recent_changes: z.array(z.string()),
});

// ── optima_memorize ───────────────────────────────────────────

export const MemorizeInputSchema = z.object({
  type: z.enum(["error_fix", "architectural_rule", "pattern", "preference"])
    .describe("The kind of knowledge being stored"),
  error: z.string().optional()
    .describe("The original error message or test failure (for error_fix)"),
  resolution: z.string().optional()
    .describe("What fixed the error (for error_fix)"),
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
  files: z.array(z.string()).optional()
    .describe("Files involved"),
  directory: z.string().optional()
    .describe("Directory scope"),
  tags: z.array(z.string()).optional()
    .describe("Freeform tags for retrieval"),
}).refine(
  (data) => {
    if (data.type === "error_fix") return !!data.error && !!data.resolution;
    if (data.type === "architectural_rule") return !!data.rule;
    if (data.type === "pattern") return !!data.pattern;
    if (data.type === "preference") return !!data.preference;
    return false;
  },
  { message: "Required fields missing for the specified type" },
);

export const MemorizeOutputSchema = z.object({
  stored: z.boolean(),
  memory_id: z.string().uuid(),
  total_memories: z.number(),
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
});
```

---

## Tool Behavior Specifications

### optima_get_context

**Internal steps (in order):**

1. Resolve `path` relative to project root using `path.resolve(projectRoot, inputPath)`. **Security: validate the resolved absolute path starts with `projectRoot`** — if not (e.g., `../../.ssh/`), throw `OptimaError("PATH_NOT_FOUND", "Path escapes project root")`. Then normalize to forward slashes. If the validated path does not exist on disk, throw `OptimaError("PATH_NOT_FOUND")`.
2. Check if project metadata exists in `project_meta` table. If not, run full project analysis first (detect tech stack from `package.json`, `tsconfig.json`, `pyproject.toml`, etc.).
3. Query `file_index` for all files under the requested path.
4. For each file, check `fs.stat(file).mtimeMs` against the stored `mtime_ms`.
5. Files with newer mtimes: re-read content, recompute hash, re-extract entities via Tree-sitter, update `file_index` and `entities` tables.
6. Files with matching mtimes: skip (index is fresh).
7. Files in `file_index` that no longer exist on disk: delete from `file_index` (cascade deletes entities).
8. Query `gotchas` table filtered by `directory` matching the requested path (see **Gotcha Retrieval Strategy** below). Increment `hit_count` for returned gotchas. Update `updated_at` to current timestamp.
9. Query `rules` table filtered by `directory` matching the requested path (see **Directory Scoping Precedence** below).
10. Collect `recent_changes` — file paths re-indexed in steps 5-7, sorted by mtime descending, capped at 20 entries. Empty on cold start.
11. Assemble and return `GetContextOutput`. (For `directory_context.description`, synthesize a concise summary of the directory's role, constructed by combining the project purpose with the directory's name, or parse a local `README.md` if present).

**Gotcha Retrieval Strategy:**

Gotchas are matched to the current context using TWO mechanisms:

1. **Directory matching:** Return gotchas where `directory` matches or is a parent of the requested path, OR `directory IS NULL` (project-wide gotchas). Example: requesting path `src/api/auth/` returns gotchas scoped to `src/api/auth/`, `src/api/`, `src/`, and project-wide gotchas.
2. **File matching:** Return gotchas where any entry in the `files` JSON array matches a file under the requested path.

Results are deduplicated by gotcha `id` and sorted by `hit_count` descending. Capped at 10 entries returned per call (matching the CLAUDE.md instruction budget). All matching gotchas get their `hit_count` incremented, even if they exceed the cap (tracking is separate from display).

**Directory Scoping Precedence:**

When querying `rules` and `gotchas` by directory, apply hierarchical matching — a rule scoped to `src/api/` applies to `src/api/auth/login.ts`. The matching logic:

1. Return all entries where `directory IS NULL` (project-wide).
2. Return all entries where `directory` is an exact prefix of the requested path (after forward-slash normalization).
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

For each entity, capture: name, line range, signature (for functions: parameter list and return type), whether it’s exported, and JSDoc description if present.

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

**Tree-sitter query patterns (TypeScript):**

Use the `tree-sitter-typescript` parser with these S-expression queries to extract entities. These are the minimal viable queries — adapt field names to exact tree-sitter-typescript grammar node types.

```scheme
;; Top-level function declarations
(function_declaration
  name: (identifier) @name
  parameters: (formal_parameters) @params
  return_type: (type_annotation)? @return_type) @func

;; Arrow functions assigned to const/let at top level
(lexical_declaration
  (variable_declarator
    name: (identifier) @name
    value: (arrow_function
      parameters: (formal_parameters) @params))) @arrow

;; Class declarations
(class_declaration
  name: (type_identifier) @name) @class

;; Interface declarations
(interface_declaration
  name: (type_identifier) @name) @interface

;; Type alias declarations
(type_alias_declaration
  name: (type_identifier) @name
  value: (_) @value) @type_alias

;; Enum declarations → extract as kind "type"
(enum_declaration
  name: (identifier) @name) @enum

;; Export statements (named exports)
(export_statement
  declaration: (_)? @decl
  (export_clause)? @clause) @export

;; Top-level variable declarations (const/let/var)
(program
  (lexical_declaration
    (variable_declarator
      name: (identifier) @name)) @variable)
```

**Implementation notes:**
- Run queries against `tree.rootNode` (top-level only — skip nested function declarations inside other functions).
- For `exported`: check if the parent node is an `export_statement`.
- For `signature`: concatenate parameters text and return type annotation text.
- For getters/setters: query `(method_definition kind: "get"|"set")` inside class bodies. Extract as kind `"function"` with name `get_propName` / `set_propName`.
- For function overloads: query all `function_signature` nodes but only emit the `function_declaration` (implementation). Skip signatures.
- For `declare` statements: filter out nodes where the parent is `ambient_declaration`. Skip entirely.

Files that fail to parse (invalid syntax) should be indexed in `file_index` but produce zero entities. Log a warning, do not throw.

**Gitignore and security exclusions:**

Before walking the file tree, read `.gitignore` (if it exists) and build a filter function using the `ignore` npm package (not hand-rolled regex — resolved Q5). Always exclude these paths regardless of `.gitignore` content:

- `node_modules/`
- `.git/`
- `.optima/`
- `dist/`, `build/`, `out/`
- `.claude/settings.local.json` (personal overrides — never index)
- `CLAUDE.local.md` (developer's personal instruction overrides — never index or modify)
- `.env`, `.env.*` (secrets)
- `**/.claude/session-env/` (Claude Code plaintext env snapshots)
- `**/.claude/projects/` (Claude Code session transcripts)
- `**/.claude/file-history/` (Claude Code edit snapshots)
- `**/.claude/paste-cache/`, `**/.claude/image-cache/` (Claude Code media buffers)
- Files matching patterns in project `.gitignore`

Note: Optima DOES index `.claude/settings.json`, `.claude/agents/`, `.claude/commands/`, `.claude/skills/`, and `.claude/rules/` because these are project configuration files that inform Optima's understanding. But it never indexes Claude Code's internal operational state.

**Performance budget:** <500ms for incremental re-index (<20 changed files). <2s for cold start full index of a typical project (<500 files).

---

### optima_memorize

**Internal steps:**

1. Validate input with `MemorizeInputSchema`.
2. Based on `type`:
    - `error_fix`: **First, sanitize the raw error text** — scrub sensitive data (API keys, JWTs, database connection strings, bearer tokens, long hash strings) using the `sanitizeError()` function below. Store the sanitized version as `error_text` in the database. Then normalize the sanitized string (strip paths, line numbers, timestamps) and compute `error_hash` as SHA-256 of the normalized string. Check `gotchas` table for existing entry with same `error_hash`. If found: update `resolution`, `files`, `updatedAt`. If not found: insert new row.
    - `architectural_rule`: Insert into `rules` table with `type = "architectural_rule"`.
    - `pattern`: Insert into `rules` table with `type = "pattern"`.
    - `preference`: Insert into `rules` table with `type = "preference"`.
3. Generate a random UUID v4 for `memory_id`. This is a correlation ID for the caller to reference this memorize operation — it is NOT the database row's `id`. The UUID is generated fresh per call and is not stored in the database. It serves as a receipt: "your memorize call succeeded, here's a reference."
4. Count total rows across `gotchas` + `rules` tables for `total_memories`.
5. **Trigger CLAUDE.md regeneration** after EVERY successful memorize call, regardless of type. Rationale: all memory types (error_fix, architectural_rule, pattern, preference) contribute to CLAUDE.md sections. The generator uses content hashing (step 5 of regeneration logic in Doc 04) to skip the actual file write if nothing changed, so frequent regeneration is cheap. Do NOT selectively regenerate only for project-wide rules — this causes stale CLAUDE.md when gotchas or patterns accumulate.
6. Return `MemorizeOutput`.

**Error normalization for dedup:**

```tsx
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
function sanitizeError(error: string): string {
  return error
    // Strip Bearer tokens
    .replace(/bearer\s+[\w\-.]+/gi, "bearer <REDACTED>")
    // Strip API keys (common prefixes: sk-, pk-, api_, key_)
    .replace(/(?:sk|pk|api|key)[-_][a-zA-Z0-9]{20,}/g, "<API_KEY_REDACTED>")
    // Strip database connection strings
    .replace(/(postgres|mysql|mongodb|redis):\/\/[^\s"']+/gi, "<DB_URL_REDACTED>")
    // Strip JWTs (three base64 segments separated by dots)
    .replace(/eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.[a-zA-Z0-9_-]+/g, "<JWT_REDACTED>")
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

## Error Taxonomy

Every error thrown by Optima must use `OptimaError` with one of these codes:

| Code | When | Severity |
| --- | --- | --- |
| `INDEX_FAILED` | File walking or mtime check fails | Recoverable |
| `PARSE_FAILED` | Tree-sitter cannot parse a file | Recoverable (skip file) |
| `DB_ERROR` | SQLite operation fails | Critical |
| `INVALID_INPUT` | Zod validation fails on tool input | Recoverable |
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