# Optima Build Order & Verification Runbook

This is the primary agent runbook for building **Optima** -- a headless stdio-based MCP server for Claude Code that indexes projects, remembers error fixes, and maintains CLAUDE.md files.

## Build Principles

- **Build in dependency order** -- do not start step N+1 until step N passes all verification criteria.
- **Copy verbatim** -- when a spec provides a fenced code block, reproduce it byte-for-byte.
- All cross-module inputs validated with Zod.
- No file exceeds 300 lines.
- Tests are mandatory at every step.

---

## Two-Stage Review Protocol

Inspired by Superpowers' subagent-driven-development pattern. After each build step, run TWO reviews before proceeding:

### Stage 1: Spec Compliance Review

**Question:** Did you build what the spec says, nothing more, nothing less?

For each build step:
1. Open the referenced specification document (Doc 02, Doc 03, or Doc 04)
2. Read the specification for this component line by line
3. Compare actual implementation against spec
4. Check for:
   - **Missing requirements** — spec says X, implementation doesn't have X
   - **Extra features** — implementation has Y, spec doesn't mention Y
   - **Misinterpretations** — spec says X, implementation does something similar but different

**CRITICAL:** Do not trust the implementer's claims. Read the actual code. The implementer may report "all requirements met" while missing a requirement they didn't notice.

### Stage 2: Code Quality Review

**Question:** Is what you built well-written?

Only run this AFTER spec compliance passes. Check for:
- `bun run build` succeeds with zero errors
- All tests pass (`bun run test`)
- No lint warnings
- Every file under 300 lines
- TypeScript strict mode compliance
- Error handling uses `OptimaError` class, not generic throws
- All cross-module boundaries validated with Zod

### Review Gate Template

Add this after every build step's "Definition of Done":

```markdown
**Review Gate:**
- [ ] **Spec Compliance:** Implementation matches referenced doc section exactly (no missing, no extra, no misinterpreted requirements)
- [ ] **Code Quality:** Build passes, tests pass, lint clean, files <300 lines, strict TypeScript, Zod validation at boundaries
- [ ] **Both stages passed:** Proceed to next step
```

---

## Build Steps

### Step 1: Types & Error Taxonomy

**Files to create:** `src/types.ts`, `src/utils/errors.ts`

**Dependencies:** None

**Estimated time:** 1 hour

**Build Prompt:**

> Create `src/types.ts` by transcribing all TypeScript interfaces from Doc 03 (Zod schemas section) and Doc 02 (TypeScript interfaces section). Include: `GetContextInput`, `GetContextOutput`, `MemorizeInput`, `MemorizeOutput`, `ReindexInput`, `ReindexOutput`, `OptimaError`, and all supporting types. Create `src/utils/errors.ts` with the `OptimaError` class and all error codes from the error taxonomy in Doc 03. No implementation logic -- types and error class only.

**Verification:**

- `bun run build` succeeds
- All types exported
- `OptimaError` has `code`/`message` fields

**Definition of Done:**

- [ ] All Zod input/output schemas from Doc 03 have corresponding TypeScript types
- [ ] `OptimaError` class matches Doc 03 definition exactly
- [ ] All error codes from Doc 03 error taxonomy are present
- [ ] File builds cleanly

---

### Step 2: Database Schema & Connection

**Files to create:** `src/db/schema.ts`, `src/db/connection.ts`, `src/db/migrations.ts`, `src/db/migrations/001_initial.ts`, `src/db/migrations/002_fts5.ts`, `src/db/migrations/index.ts`

**Dependencies:** Step 1 (types)

**Estimated time:** 2.5 hours

**Build Prompt:**

> Create the database layer. Copy the Drizzle schema from Doc 02 verbatim into `src/db/schema.ts` -- all 8 Drizzle-modeled tables (project_meta, file_index, entities, gotchas, rules, security_findings, task_outcomes, generation_log) plus the schema_version infrastructure table. Note: FTS5 virtual tables (gotchas_fts, rules_fts) are NOT in Drizzle -- they are created by migration 002. Create `src/db/connection.ts` with lazy initialization, WAL mode, foreign keys, corruption recovery -- copy the implementation from Doc 03. Create `src/db/migrations.ts` with the migration runner from Doc 02 (version table, transaction-wrapped migrations). Create `src/db/migrations/001_initial.ts` with CREATE TABLE statements for all 9 tables. Create `src/db/migrations/002_fts5.ts` with FTS5 virtual tables and sync triggers for gotchas and rules -- copy from Doc 02 verbatim. Create `src/db/migrations/index.ts` exporting the ordered migration list.

**Verification:**

- Database creates at `.optima/optima.db` on first `getDatabase()` call
- WAL mode enabled
- Foreign keys enabled
- All 9 tables exist after migration 001 (including schema_version)
- FTS5 virtual tables (gotchas_fts, rules_fts) exist after migration 002
- FTS5 sync triggers fire on gotcha/rule insert/update/delete
- `.optima/.gitignore` created
- Corruption recovery works

**Definition of Done:**

- [ ] All 8 Drizzle tables + schema_version match Doc 02 schema exactly
- [ ] FTS5 virtual tables created by migration 002 with sync triggers
- [ ] `getDatabase()` is lazy
- [ ] WAL mode and foreign keys enabled
- [ ] Migration runner works (001 then 002 in order)
- [ ] Corruption recovery works
- [ ] `.optima/.gitignore` created
- [ ] 10+ unit tests

---

### Step 3: Utility Modules

**Files to create:** `src/utils/hasher.ts`, `src/utils/paths.ts`

**Dependencies:** Step 1

**Estimated time:** 1 hour

**Build Prompt:**

> Create `src/utils/hasher.ts` with SHA-256 content hashing. Create `src/utils/paths.ts` with: path normalization (always forward slashes per Ground Rule 7), `.gitignore` matching using the `ignore` npm package (^7.0.0, per Q5), symlink detection via `fs.lstat`, and path traversal validation (resolved path must start with project root -- reject `../../` escapes per Doc 03 tool step 1).

**Verification:**

- Path normalization converts backslashes to forward slashes
- `.gitignore` patterns correctly exclude matched files
- Symlinks are detected and skipped
- Path traversal attempts throw `OptimaError("PATH_NOT_FOUND")`

**Definition of Done:**

- [ ] Forward-slash normalization works on Windows paths
- [ ] `ignore` package used for gitignore matching
- [ ] Symlink detection via `fs.lstat`
- [ ] Path traversal validation rejects `../../` style escapes
- [ ] 8+ unit tests

---

### Step 4: File Indexer & Entity Extractor

**Files to create:** `src/indexer/file-indexer.ts`, `src/indexer/entity-extractor.ts`

**Dependencies:** Steps 1--3

**Estimated time:** 3 hours

**Build Prompt:**

> Create `src/indexer/file-indexer.ts` with the `walkProject()` function from Doc 03 -- depth-first alphabetical traversal, binary file detection, symlink skipping, `.gitignore` integration via the `ignore` package. Create `src/indexer/entity-extractor.ts` with the `extractEntities()` function from Doc 03 -- Tree-sitter parsing for TypeScript covering all 12 edge cases: async functions, arrow functions, default exports, re-exports, overloaded functions, getters/setters, `declare` statements (skip), `namespace` declarations (skip), generic types, enums, classes, and interfaces.

**Verification:**

- File walker respects `.gitignore`
- File walker skips symlinks
- File walker skips binary files
- Entity extractor handles all 12 edge cases from Doc 03
- Entity extractor produces correct entities for a sample TypeScript file

**Definition of Done:**

- [ ] `walkProject()` matches Doc 03 specification exactly
- [ ] Binary detection works (checks first 512 bytes for null bytes)
- [ ] Secret scanning runs during file indexing (step 5a): `scanForSecrets()` detects all `SECRET_PATTERNS`
- [ ] Secret findings stored with redacted snippets — NEVER the raw secret value
- [ ] Dismissed findings preserved on re-index if match still exists
- [ ] All 12 entity extraction edge cases handled
- [ ] Tree-sitter parses TypeScript files correctly
- [ ] Tests assert against frozen dataset: `test_fixtures/entity_extraction_dataset.json`
- [ ] 15+ unit tests

---

### Step 5: Project Analyzer

**Files to create:** `src/indexer/project-analyzer.ts`

**Dependencies:** Steps 1--4

**Estimated time:** 1.5 hours

**Build Prompt:**

> Create `src/indexer/project-analyzer.ts` that detects tech stack from package.json, tsconfig.json, pyproject.toml, etc. Detect build/test/lint commands from package.json scripts. Detect project purpose from README.md or package.json description. Detect linter/formatter presence using the AUTHORITATIVE linter config list from Doc 02. Store linterDetected as a JSON array of strings (e.g., `'["eslint","prettier"]'`) per Q15. Upsert results into the `project_meta` table.

**Verification:**

- Analyzing a TypeScript project detects: language, framework, test runner, linter
- `linterDetected` stored as JSON array string
- `project_meta` table populated correctly
- `projectPurpose` extracted from README first line or package.json description

**Definition of Done:**

- [ ] Tech stack detection works for TypeScript projects
- [ ] Build/test/lint commands detected from package.json scripts
- [ ] Linter detection uses authoritative list from Doc 02
- [ ] `linterDetected` format matches Q15
- [ ] Key dependencies parsed from package.json and stored in `project_meta.key_dependencies` as JSON array
- [ ] Project purpose extraction follows Doc 03 step 11 logic
- [ ] 8+ unit tests

---

### Step 6: Error Normalization & Sanitization

**Files to create:** `src/memory/error-normalizer.ts`

**Dependencies:** Step 1

**Estimated time:** 1 hour

**Build Prompt:**

> Create `src/memory/error-normalizer.ts` with two functions: `sanitizeError()` (runs FIRST -- scrubs API keys, JWTs, database URLs, bearer tokens from error text before any other processing) and `normalizeError()` (runs AFTER sanitization -- strips Windows paths, strips UUIDs, normalizes whitespace for hash computation). Both functions are specified in Doc 03. The sanitize -> normalize -> hash pipeline is the dedup strategy for gotchas (Q3).

**Verification:**

- `sanitizeError()` removes API keys, JWTs, bearer tokens, database URLs
- `normalizeError()` strips Windows-style paths, UUIDs, normalizes whitespace
- Pipeline: sanitize -> normalize -> hash produces consistent hashes for equivalent errors
- Hash collision handling: on collision, store both entries, emit `HASH_COLLISION` warning (Q3)

**Definition of Done:**

- [ ] `sanitizeError` runs before `normalizeError` (order enforced)
- [ ] API keys/JWTs/bearer tokens/DB URLs scrubbed
- [ ] Windows paths and UUIDs stripped during normalization
- [ ] Hash collision produces both entries + `HASH_COLLISION` warning
- [ ] Tests assert against frozen dataset: `test_fixtures/error_normalization_dataset.json`
- [ ] 10+ unit tests with real-world error string examples

---

### Step 7: Gotcha Ledger & Rules Store

**Files to create:** `src/memory/gotcha-ledger.ts`, `src/memory/rules-store.ts`

**Dependencies:** Steps 1--3, 6

**Estimated time:** 2 hours

**Build Prompt:**

> Create `src/memory/gotcha-ledger.ts` for error->solution CRUD (insert, query by directory with hierarchical prefix match per Q14, increment hit_count, dedup by normalized hash). Create `src/memory/rules-store.ts` for architectural rules/patterns/preferences CRUD with directory scoping precedence (NULL = project-wide -> parent prefix -> most specific first, per Doc 03).

**Verification:**

- Gotcha insertion with normalized error hash
- Gotcha retrieval with hierarchical directory prefix match + file array match
- Dedup by ID, sorted by `hit_count` desc, capped at 10 (Q14)
- `hit_count` incremented on retrieval
- Rules retrieved with directory scoping precedence

**Definition of Done:**

- [ ] Gotcha ledger CRUD operations work
- [ ] Hierarchical directory matching works
- [ ] FTS5 search works: `search_query` finds gotchas/rules across directories
- [ ] FTS5 merge: directory results first, FTS-only results appended, deduped by ID
- [ ] FTS5 error handling: malformed queries fall back to directory-only (no throw)
- [ ] File array matching works alongside directory matching
- [ ] Dedup by ID/sort by `hit_count` desc/cap at 10
- [ ] `hit_count` and `updated_at` updated on retrieval
- [ ] Rules store respects directory scoping precedence
- [ ] Task outcome storage works (insert, query by directory)
- [ ] Task outcomes filtered: only outcomes with non-null learnings included in CLAUDE.md generation
- [ ] Tests assert against frozen datasets: `test_fixtures/gotcha_retrieval_dataset.json` and `test_fixtures/directory_scoping_dataset.json`
- [ ] 12+ unit tests

---

### Step 8: MCP Server & Tool Registration

**Files to create:** `src/index.ts`, `src/server.ts`

**Dependencies:** Steps 1--7

**Estimated time:** 1.5 hours

**Build Prompt:**

> Create `src/index.ts` (entry point) and `src/server.ts` (tool registration) by copying the implementations from Doc 03. Include signal handling (SIGINT, SIGTERM for graceful shutdown), the `wrapError()` pattern for MCP error responses, and registration of all 3 tools: `optima_get_context`, `optima_memorize`, `optima_reindex`. Use the exact tool description strings from the "Tool Descriptions for Tool Search" table in Doc 03 -- these are critical for Claude Code's Tool Search discoverability.

**Verification:**

- Server starts on stdio transport
- All 3 tools registered with correct names and descriptions
- SIGINT/SIGTERM trigger graceful shutdown
- `wrapError()` produces correct MCP error format
- Invalid tool calls return structured error responses

**Definition of Done:**

- [ ] `src/index.ts` matches Doc 03 entry point exactly
- [ ] `src/server.ts` matches Doc 03 tool registration exactly
- [ ] Tool description strings match Doc 03 table exactly
- [ ] Signal handling works
- [ ] `wrapError()` handles both `OptimaError` and unknown errors
- [ ] 5+ unit tests

---

### Step 9: Tool Handlers (get_context, memorize, reindex)

**Files to create:** `src/tools/get-context.ts`, `src/tools/memorize.ts`, `src/tools/reindex.ts`

**Dependencies:** Steps 1--8

**Estimated time:** 4 hours

**Build Prompt:**

> Implement all 3 tool handlers. `get-context.ts` follows the 11-step behavior from Doc 03 exactly: resolve path, validate traversal, check project_meta, query file_index, check mtimes, re-index changed files (with secret scanning step 5a), delete removed files, query gotchas (hierarchical match + FTS5 search if `search_query` provided), query rules (directory scoping + FTS5), query task_outcomes, query security_findings, collect recent_changes, assemble output with directory description using pure string concatenation (no LLM). `memorize.ts` handles all 5 types (error_fix, architectural_rule, pattern, preference, task_outcome) with type-specific validation and routing (error_fix → gotchas, architectural_rule/pattern/preference → rules, task_outcome → task_outcomes), error sanitization/normalization for error_fix type, and triggers CLAUDE.md regeneration after EVERY call. `reindex.ts` forces full re-index, re-runs project analysis, triggers CLAUDE.md regeneration, regenerates feedback rules file.

**Verification:**

- `optima_get_context` returns correct structure (including security_warnings, task_insights, dependency_context)
- Lazy re-indexing only processes changed files
- Gotcha retrieval respects hierarchical matching + FTS5 search merge
- `optima_memorize` stores all 5 types correctly (task_outcome → task_outcomes table)
- `memory_id` is random UUID v4 (NOT database row ID)
- `total_memories` counts gotchas + rules + task_outcomes
- CLAUDE.md regenerated after every memorize call
- `optima_reindex` triggers full re-index + analysis + CLAUDE.md + feedback rules

**Definition of Done:**

- [ ] get-context 11-step behavior matches Doc 03 exactly
- [ ] Path traversal validation rejects escapes
- [ ] Lazy re-indexing works (mtime comparison)
- [ ] Secret scanning runs during re-indexing (step 5a)
- [ ] FTS5 search works when `search_query` provided (merge + dedup)
- [ ] Deleted files removed from `file_index` (cascade deletes entities + security_findings)
- [ ] `recent_changes`: files re-indexed in this call sorted by mtime desc capped at 20
- [ ] `directory_context.description` uses pure string concatenation (no LLM)
- [ ] memorize handles all 5 types with correct validation and routing
- [ ] `memory_id` is UUID v4 not stored is receipt only
- [ ] `total_memories` = gotchas + rules + task_outcomes row count
- [ ] CLAUDE.md regenerated after every memorize call (all 5 types)
- [ ] reindex re-runs project analysis + CLAUDE.md + feedback rules
- [ ] 20+ unit tests across all 3 handlers

---

### Step 10: CLAUDE.md Generator

**Files to create:** `src/generator/claude-md.ts`

**Dependencies:** Steps 1--7

**Estimated time:** 2 hours

**Build Prompt:**

> Create `src/generator/claude-md.ts` implementing CLAUDE.md generation with section markers. Copy the exact marker format, section templates, and regeneration logic from Doc 04. Implement all 7 sections: project_overview, architecture_rules, known_gotchas, patterns, preferences, security_warnings, task_insights. Implement: section marker parsing, content hash comparison (SHA-256, skip write if unchanged), the necessity test filtering (gotchas: hit_count >= 2 per Q4, rules: exclude linter-duplicates, patterns: exclude single-occurrence, preferences: include all, task outcomes: only with non-null learnings), instruction budget enforcement (30-40 instructions total with caps per category: 10 rules, 10 gotchas, 5 patterns, 5 preferences, 5 task insights — security_warnings exempt from budget), and malformed marker handling (orphaned START -> replace to next marker/EOF, orphaned END -> delete and append).

**Verification:**

- Generated CLAUDE.md contains up to 7 sections in order
- Sections with zero entries after filtering are omitted
- Necessity test: gotchas with `hit_count` < 2 excluded, task outcomes without learnings excluded
- Instruction budget: total <= 40 (security_warnings exempt)
- Content hash prevents unnecessary file writes
- Human content outside markers preserved
- Malformed markers handled gracefully

**Definition of Done:**

- [ ] All 7 section templates from Doc 04 implemented
- [ ] Marker format exactly matches Doc 04
- [ ] Necessity test matches Doc 04 (all 5 filter rules)
- [ ] Instruction budget algorithm from Doc 04 implemented (8 steps, budget 30-40)
- [ ] security_warnings section exempt from instruction budget
- [ ] Content hash comparison prevents unnecessary writes
- [ ] Malformed marker handling matches Doc 04
- [ ] Sections with zero entries omitted entirely
- [ ] 15+ unit tests

---

### Step 11: Feedback Rules Generator

**Files to create:** `src/generator/feedback-rules.ts`

**Dependencies:** Step 1

**Estimated time:** 0.5 hours

**Build Prompt:**

> Create `src/generator/feedback-rules.ts` that writes `.claude/rules/optima-feedback.md` with the exact inception payload content from Doc 04. Creates `.claude/rules/` directory if needed. Content is static for MVP. Skips write if file already exists with correct content (content comparison). Returns true if file was written, false if skipped.

**Verification:**

- File created at `.claude/rules/optima-feedback.md`
- Content matches Doc 04 verbatim
- Directory created if missing
- No-op if content unchanged

**Definition of Done:**

- [ ] Output content matches Doc 04 inception payload exactly
- [ ] Creates `.claude/rules/` directory if needed
- [ ] Skips write if file exists with correct content
- [ ] Returns boolean indicating whether file was written
- [ ] 3+ unit tests

---

### Step 12: Integration Testing & Cold Start Verification

**Dependencies:** All previous steps

**Estimated time:** 2 hours

**Build Prompt:**

> Create integration tests that verify the complete cold start bootstrap sequence from Doc 01: (a) Database created on first `optima_get_context` call, (b) Full project analysis runs, (c) Full file walk + entity extraction runs, (d) `.claude/rules/optima-feedback.md` generated, (e) `CLAUDE.md` generated with initial sections, (f) `.optima/` added to `.gitignore`, (g) `GetContextOutput` returned with full project context. Also test the feedback loop: call `optima_memorize` with an error_fix, then call `optima_get_context` for the same directory -- verify the gotcha appears in the response. Test `optima_reindex` -- verify it re-runs everything and regenerates files.

**Verification:**

- Cold start sequence completes in order
- Second session: feedback rules already exist
- Memorize -> get_context round-trip works
- Reindex triggers full rebuild

**Definition of Done:**

- [ ] Cold start bootstrap sequence matches Doc 01 exactly (all 8 steps)
- [ ] Feedback loop works (memorize -> get_context returns the gotcha)
- [ ] Reindex triggers full project analysis + CLAUDE.md + feedback rules
- [ ] All integration tests pass
- [ ] `bun run test` passes (all unit + integration tests)
- [ ] `bun run build` succeeds
