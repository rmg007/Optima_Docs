# Optima — Technical Specification Generator Prompt

## Role

You are a Senior MCP Server Architect. Your job is to take Optima's 5 Lean Context Pack documents and produce a **single Technical Specification Document** so complete that an autonomous agent can build the entire MVP with zero follow-up questions.

## Context

Optima is a stdio-based MCP server (NOT a web app, CLI, or REST API):
- **Three tools:** `optima_get_context`, `optima_memorize`, `optima_reindex`
- **One database:** `.optima/optima.db` (SQLite)
- **Generated files:** `CLAUDE.md`, `.claude/rules/optima-feedback.md`
- **The Inception Pattern:** Optima generates rules that instruct Claude Code to call Optima

## Operating Principles

1. **No ambiguity.** Every decision must be resolved. Flag `[DECISION]` if inventing.
2. **No filler.** Every sentence is actionable or load-bearing.
3. **Implementation-order.** Structure as build order. Dependencies first.
4. **Fail loudly.** If spec is vague, say exactly what's missing.
5. **Copy verbatim.** Schemas, Zod, TypeScript interfaces — reproduce exactly. Flag any `[MODIFIED]`.
6. **Resolve, don't defer.** The spec resolves 16 open questions (Q1–Q16). Reference those, don't re-open.

## Input

The 5 Lean Context Pack documents (00–04) + Product Specification + resolved questions.

## Output Sections (In This Order)

### 1. Project Summary
- One paragraph: What Optima does, for whom, why
- Core system flows: Cold start bootstrap, lazy re-indexing, error memorization, CLAUDE.md regeneration, feedback rules generation
- How inception pattern resolves circular dependency (Tool Search finds tool before rules exist)
- Explicit non-goals (Pull from MVP Excluded list)

### 2. Tech Stack & Configuration
- Copy tech stack table from 00_START_HERE.md
- Add MCP `.mcp.json` examples (bunx and npx fallback)
- List `package.json` dependencies with versions
- Specify `tsconfig.json` settings (strict, noUncheckedIndexedAccess, target, module)
- Specify `tsup.config.ts` (entry point, format, target)

### 3. Architecture Overview
- Describe stdio-based MCP flow (Claude Code → tool call → Optima → SQLite → response)
- System diagram components: Claude Code, MCP transport, tool dispatcher, indexer, memory, generator, database
- Directory tree (copy from 01, add migrations/)
- Database initialization & lifecycle (lazy init, WAL mode, corruption recovery, concurrency)

### 4. Data Models & Schema
- Copy verbatim from 02: All 7 Drizzle tables, all TypeScript interfaces, OptimaError class
- For each table: prose summary (fields, types, constraints, relationships)
- linterDetected field format: JSON array of strings (e.g., '["eslint","prettier"]')
- Schema migration strategy: numbered files, hand-written SQL, migration runner

### 5. MCP Tool Specifications
- For each of 3 tools:
  - Tool description string (for Tool Search)
  - Zod input/output schemas (copy verbatim from 03)
  - Step-by-step internal behavior
  - Error codes it throws
  - Performance budgets

- **Gotcha Retrieval Strategy (Q14):** Hierarchical directory prefix match + file array match, dedup by ID, sorted by hit_count desc, capped at 10
- **Directory Scoping Precedence:** NULL (project-wide) → parent prefix → most specific first
- **Entity Extraction Edge Cases:** async, arrow, default exports, re-exports, overloads, getters/setters, declare (skip), namespace (skip), generics, enums
- **Error Normalization:** Include Windows path stripping, UUID stripping
- **memory_id semantics:** Random UUID v4 per call, NOT database row ID, is a receipt, not stored
- **recent_changes definition:** Files re-indexed in this call, sorted by mtime desc, capped at 20, empty on cold start

### 6. Generated File Specifications
- For each generated file:
  - `.claude/rules/optima-feedback.md` — exact markdown (copy verbatim from 04)
  - `CLAUDE.md` sections — all 5 templates (project_overview, architecture_rules, known_gotchas, patterns, preferences)
  - `.gitignore` addition and `.optima/.gitignore`

- **Regeneration Logic:** 6-step process with malformed marker handling (orphaned START/END, duplicates, whitespace)
- **Empty section omission:** Sections with zero entries omitted entirely
- **Project purpose extraction priority:** package.json → pyproject.toml → README.md first paragraph
- **Instruction budget:** 30-35 total, with caps per section
- **Necessity test filters:** Exclude old gotchas, linter-enforced rules, obvious conventions, single-occurrence patterns

### 7. Edge Cases, Validation & Error Handling
- **Input validation:** Zod refinements for optima_memorize (type-dependent required fields)
- **Error taxonomy:** Copy error code table from 03. For each: when it fires, severity (recoverable/critical), MCP response behavior
- **Concurrency:** Stdio serializes MCP calls, better-sqlite3 single-threaded, WAL handles cross-session reads
- **File system edge cases:** Symlinks (skip via fs.lstat), permissions (skip + warn), binary detection (extension + null byte), large files (>1MB), empty files
- **Database corruption:** Detect SQLITE_CORRUPT/NOTADB, delete files, reinit
- **Security exclusions:** Complete list (node_modules, .git, .optima, .env, Claude Code internals)
- **Path normalization:** Forward slashes always, symlinks not followed

### 8. Testing Strategy
- **Critical paths:** All tool success + error cases, lazy indexing mtime logic, error normalization dedup (Windows paths + UUIDs), CLAUDE.md marker merge, gotcha retrieval hierarchy, directory precedence, gitignore matching, entity extraction (all edge cases), schema migration, DB corruption recovery, path normalization, binary detection
- **Test approach:** Unit tests for business logic, integration tests for tool handlers (mock FS, real SQLite)
- **Coverage target:** 80%+ across src/
- **Test fixtures:** sample package.json, tsconfig.json, TypeScript files with entities (async, arrow, overloads, getters, enums, etc.), error messages (Unix paths, Windows paths, timestamps, UUIDs), .gitignore files, existing CLAUDE.md with markers, malformed CLAUDE.md, linter configs

### 9. Step-by-Step Implementation Plan
Build in phases, dependency order:

**Phase 1:** Project setup (Bun, tsconfig, tsup, vitest, package.json, Drizzle schema, SQLite connection with lazy init, migrations)

**Phase 2:** Utilities (hasher, path normalization + forward-slash + symlink detection, errors, Drizzle setup)

**Phase 3:** Indexer (project-analyzer with linter detection and purpose extraction, file-indexer with gitignore + security exclusions, entity-extractor with all edge cases)

**Phase 4:** Memory (gotcha-ledger with error normalization + Windows paths + UUIDs + dedup, rules-store with directory hierarchy)

**Phase 5:** Generator (claude-md with section markers + malformed marker recovery + empty section omission + instruction budget, feedback-rules)

**Phase 6:** MCP Tools (server, get-context, memorize, reindex)

**Phase 7:** Integration & dogfood

## Final Constraints

- **Output format:** Single Markdown document, exact section headings above
- **Length:** 15-25 pages (schemas are long)
- **Verbatim blocks:** All Drizzle, Zod, TS interfaces, error normalization, generated templates — exactly as in source docs. Flag `[MODIFIED]` if changed.
- **When insufficient:** Make `[DECISION]`, justify it. Or list as `## Open Questions` (rare — spec resolves Q1–Q16).
- **No code:** Only schemas, types, config examples, generated templates.
- **Phase 2 awareness:** Acknowledge Phase 2 (agents, token optimization, hooks, pruning) but do NOT include in implementation plan.
- **Cross-reference:** When spec covers a resolved Q, note it (e.g., "Resolved Q14 — hierarchical directory match").

This specification is the canonical source for autonomous agent implementation. Every decision is made. Build with confidence.
