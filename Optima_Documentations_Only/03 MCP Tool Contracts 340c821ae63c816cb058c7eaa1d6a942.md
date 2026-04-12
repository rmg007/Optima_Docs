# 03 MCP Tool Contracts

All tool inputs and outputs are validated with Zod. Copy these schemas into the tool implementation files.

## Tool Search & Discoverability

Since early 2026, Claude Code uses **Tool Search** (lazy loading) for MCP tools by default, reducing context usage by ~95% compared to loading all tool schemas upfront. Claude Code uses semantic matching on tool names and descriptions to decide which tools to load for a given task.

**Implication for Optima:** Our tool names and `.describe()` strings in the Zod schemas are not just documentation — they are the discovery surface. Claude Code decides whether to load `optima_get_context` based on whether its description semantically matches the user's request. Write descriptions that clearly state what the tool does using keywords a developer would naturally use.

**Tool naming convention:** All tools are prefixed with `optima_` to avoid collisions with other MCP servers. The descriptions below are tuned for Tool Search discoverability.

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

1. Resolve `path` relative to project root. If path does not exist, throw `OptimaError("PATH_NOT_FOUND")`.
2. Check if project metadata exists in `project_meta` table. If not, run full project analysis first (detect tech stack from `package.json`, `tsconfig.json`, `pyproject.toml`, etc.).
3. Query `file_index` for all files under the requested path.
4. For each file, check `fs.stat(file).mtimeMs` against the stored `mtime_ms`.
5. Files with newer mtimes: re-read content, recompute hash, re-extract entities via Tree-sitter, update `file_index` and `entities` tables.
6. Files with matching mtimes: skip (index is fresh).
7. Files in `file_index` that no longer exist on disk: delete from `file_index` (cascade deletes entities).
8. Query `gotchas` table filtered by `directory` matching the requested path. Increment `hit_count` for returned gotchas.
9. Query `rules` table filtered by `directory` matching the requested path (or `directory IS NULL` for project-wide rules).
10. Assemble and return `GetContextOutput`.

**Tree-sitter entity extraction (step 5):**

Parse TypeScript files using `tree-sitter-typescript`. Extract:

- Top-level function declarations → kind: `"function"`
- Class declarations → kind: `"class"`
- Interface declarations → kind: `"interface"`
- Type alias declarations → kind: `"type"`
- Named exports → kind: `"export"`
- Top-level const/let declarations → kind: `"variable"`

For each entity, capture: name, line range, signature (for functions: parameter list and return type), whether it’s exported, and JSDoc description if present.

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
    - `error_fix`: Normalize the error string (strip absolute paths, line numbers, timestamps). Compute `error_hash` as SHA-256 of the normalized string. Check `gotchas` table for existing entry with same `error_hash`. If found: update `resolution`, `files`, `updatedAt`. If not found: insert new row.
    - `architectural_rule`: Insert into `rules` table with `type = "architectural_rule"`.
    - `pattern`: Insert into `rules` table with `type = "pattern"`.
    - `preference`: Insert into `rules` table with `type = "preference"`.
3. Generate a UUID for the `memory_id`.
4. Count total rows across `gotchas` + `rules` tables for `total_memories`.
5. If the new entry is an `architectural_rule` and `directory` is null (project-wide), trigger [CLAUDE.md](http://CLAUDE.md) regeneration.
6. Return `MemorizeOutput`.

**Error normalization for dedup:**

```tsx
function normalizeError(error: string): string {
  return error
    .replace(/\/[\w\-./]+\.(ts|js|tsx|jsx)/g, "<file>")
    .replace(/:\d+:\d+/g, ":<line>")
    .replace(/\d{4}-\d{2}-\d{2}T[\d:.]+Z?/g, "<time>")
    .replace(/0x[0-9a-f]+/gi, "<addr>")
    .trim()
    .toLowerCase();
}
```

---

### optima_reindex

**Internal steps:**

1. Resolve `path` (default: project root).
2. Delete all entries in `file_index` (and cascaded `entities`) for files under the path.
3. Run full file walk + Tree-sitter extraction for all files under the path.
4. Update `project_meta.lastFullIndex` to current timestamp.
5. Log the reindex event with `reason` if provided.
6. Return `ReindexOutput`.

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