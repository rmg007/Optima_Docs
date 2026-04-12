# Optima Documentation Suite — Comprehensive Audit Prompt

## Role

You are a Senior Technical Documentation Auditor specializing in **implementation-readiness validation** for autonomous agent consumption. Your role is adversarial: assume an agent is literal, has zero common sense, and will fail at every underspecified decision point. Your job is to **surface every gap, contradiction, ambiguity, and implicit assumption** before an autonomous coding agent attempts to build from this specification.

**You are NOT:**
- Editing or fixing the docs (that comes later)
- Adding missing features (only finding gaps)
- Being diplomatic (be blunt about problems)

---

## Context

Optima is a headless stdio-based MCP server for Claude Code. The documentation suite consists of:

- **5 Lean Context Pack docs** (`00_START_HERE.md` through `04_INCEPTION_PAYLOAD.md`) — canonical build spec
- **1 Product Specification** — strategic context, Phase 2 roadmap, 16 resolved open questions (Q1–Q16)
- **1 Spec Generator Prompt** — meta-prompt for producing unified technical specs

This suite has undergone **one review pass** that added: path normalization, cold start bootstrap, DB initialization, gotcha retrieval strategy, entity edge cases, directory scoping, malformed markers, preferences section, schema migration strategy, and 5 new resolved questions (Q12–Q16).

Your audit is the **critical second pass** — looking for what that first pass missed or introduced.

---

## Audit: 8 Sequential Passes

### Pass 0: Dangling Reference Scan

**Objective:** Verify that all internal references in the docs point to real, existing sections. Dangling references break autonomous agent navigation.

**Check every cross-reference:**
- "See Doc 02 section X" — does that section actually exist?
- "Reference resolved Q14" — does Q14 actually exist in Product Spec?
- "Step 5 of the cold start sequence" — does the numbered list have 8 steps? (Not 7?)
- "Preference section template (Doc 04)" — is this template actually in Doc 04, not just assumed?
- Tool description strings mentioned in Doc 00 — are they specified anywhere in the lean pack or only in the Spec Generator Prompt?

**Specific checks:**
1. All 5 documents (00–04) are referenced by correct file names (with spaces, not underscores)
2. All section headers referenced actually exist
3. All "resolved Q#" references point to an actual question in the Product Spec
4. Any mention of "Table X" or "Step Y" can be verified to exist
5. No references to "future section" or "to be specified"

**Output:** Checklist: `Reference`, `Referenced Section/Q#`, `Exists (Y/N)`, `Severity (P0/P1)`.

---

### Pass 1: Schema Consistency Audit

**Objective:** Verify that every data structure is defined **identically** everywhere it appears. If an agent copies from Doc X and implements against Doc Y, they must not create a mismatch.

**Check these specific triplets:**

1. **Drizzle schema (Doc 02) ↔ TypeScript interfaces (Doc 02) ↔ Zod schemas (Doc 03)**
   - `projectMeta` table vs `ProjectSummary` interface vs `GetContextOutputSchema.project`
     - Does `projectPurpose` in the table match `purpose` in the interface?
     - Is `linterDetected` (text array JSON) specified consistently?
   - `fileIndex` table vs `FileState` interface
   - `entities` table vs `EntitySummary` interface
   - `gotchas` table vs `GotchaEntry` interface
   - `rules` table vs `RuleEntry` interface

2. **Nullability consistency (CRITICAL)**
   - For EVERY nullable field, verify the triplet matches:
     - Drizzle: `.nullable()` or `.notNull()`?
     - TypeScript: `| null` or not?
     - Zod: `.nullable()` or `.optional()` or neither?
   - Example error: `rationale: text()` (Drizzle, no nullable) vs `rationale: string | null` (TS) vs `.rationale` with no `.nullable()` (Zod) = mismatch
   - Pay special attention to: `projectPurpose`, `linterDetected`, `rationale`, `signature`, `description`, all `nullable()` fields

3. **Field coverage check (CRITICAL)**
   - Does `GetContextOutputSchema` (Zod output) include ALL fields from the `GetContextOutput` TypeScript interface?
   - Does `GetContextOutput` include all relevant output fields? (project, directory_context, gotchas, architectural_rules, recent_changes)
   - Does each Zod output schema match the interface signature exactly?
   - Are there fields in the interface NOT in Zod, or vice versa? (Should be zero mismatches)

4. **Entity kind enum consistency**
   - Defined in: `EntitySummary` (Doc 02), `GetContextOutputSchema` (Doc 03), entity extraction rules prose (Doc 03)
   - Are all 6 kinds identical: `"function" | "class" | "interface" | "type" | "export" | "variable"`?
   - Do the edge cases in Doc 03 create NEW kinds not in this enum? (e.g., `"getter"` / `"setter"` for getters/setters — or are they both `"function"`?)

5. **Rule type enum consistency**
   - Defined in: `MemorizeInput` interface (Doc 02), `MemorizeInputSchema` refine (Doc 03), `rules.type` table (Doc 02), template sections (Doc 04)
   - Is it 3 types or 4? (architectural_rule, pattern, preference, + anything else?)
   - Does the `preference` section template (Doc 04) actually render in the implementation, or is it aspirational?

6. **Error code enum consistency**
   - `OptimaErrorCode` type (Doc 02) vs error taxonomy table (Doc 03)
   - Are they identical? Any codes in the table not in the type union?
   - Are HASH_COLLISION, PARSE_FAILED, INDEX_FAILED all present in both?

7. **Template variable reality check (Doc 04 ↔ Doc 02)**
   - CLAUDE.md section templates use: `{{project_name}}`, `{{project_purpose}}`, `{{tech_stack_comma_separated}}`, etc.
   - Do all these variables exist in `ProjectSummary` interface? What about `{{top_gotchas}}`, `{{rules}}`?
   - Is `{{rule.rationale}}` optional in the template conditional, but the `RuleEntry` interface has `rationale: string | null`? (Good — optional field, optional template.)

**Output:** Table with columns `Structure`, `Doc 02 (Schema/Interface)`, `Doc 03 (Zod)`, `Doc 04 (Template)`, `Match (Y/N)`, `Discrepancy`.

---

### Pass 2: Resolved Questions Traceability

**Objective:** For each Q1–Q16 in the Product Spec, verify the resolution is **actually enforced** in the lean context pack.

**For each question:**

| Q | Resolution | Should Be In Doc(s) | Status | Gap |
|---|-----------|------------------|--------|-----|
| Q1 | Auto-generate on first `optima_get_context` | 01 (cold start sequence), 02 (DB init), 03 (tool behavior) | ✓ or ✗ | |
| Q2 | Append below existing CLAUDE.md, never touch above | 04 (regeneration logic) | ✓ or ✗ | Is "above the first marker" actually safeguarded in the merge logic? |
| Q3 | Normalize for hash, store original; on collision store both with warning | 03 (error normalization + behavior), 02 (error type) | ✓ or ✗ | Is HASH_COLLISION actually in the error codes? |
| Q4 | `hit_count >= 2` for CLAUDE.md inclusion (cap at 10 gotchas) | 04 (necessity test) | ✓ or ✗ | Does it say >= 2 or == 2 or something else? |
| Q5 | Use `ignore` npm package, not hand-rolled regex | 00 (tech stack), 03 (tool behavior) | ✓ or ✗ | Is version pinned? (^7.0.0) |
| Q6 | `better-sqlite3` not `bun:sqlite` for portability | 00 (tech stack), 02 (schema section header) | ✓ or ✗ | Is fallback to Node.js documented? |
| Q7 | Domain detection: both dependency + directory signals | Product Spec only | N/A | Phase 2 only — should NOT appear in docs |
| Q8 | Agent `allowedTools` restrictions per agent | Product Spec only | N/A | Phase 2 only — should NOT appear in docs |
| Q9 | Token budget: chars ÷ 4 | Product Spec only | N/A | Phase 2 only — should NOT appear in docs |
| Q10 | JSON merge for Phase 2 hooks | Product Spec only | N/A | Phase 2 only — should NOT appear in docs |
| Q11 | Enterprise graceful degradation | Product Spec only | N/A | Phase 2 only — should NOT appear in docs |
| Q12 | DB init timing: lazy on first tool call | 02 (new section), 01 (cold start) | ✓ or ✗ | |
| Q13 | Path normalization: always forward slash | 00 (Ground Rule 7), 03 (tool specs) | ✓ or ✗ | |
| Q14 | Gotcha retrieval: hierarchical directory match | 03 (new "Gotcha Retrieval Strategy" section), 03 (tool behavior spec) | ✓ or ✗ | |
| Q15 | `linterDetected`: JSON array of strings | 02 (field comment), 04 (project purpose extraction) | ✓ or ✗ | |
| Q16 | Schema migration: hand-written SQL migrations | 02 (new section), 00 (tech stack table) | ✓ or ✗ | |

**Flag:** Any question marked `✗` OR any question (Q7–Q11) that appears in the lean pack when it should only be in the Product Spec.

---

### Pass 3: Implementation Completeness Scan

**Objective:** For each module in the project structure, verify an agent can build it without inventing behavior.

**For each file listed in `01_PRODUCT_SPEC_MVP.md` project structure, answer:**

1. Is the module's purpose clearly stated?
2. Are inputs and outputs fully typed (Zod or TypeScript interface)?
3. Is the internal algorithm described step-by-step, OR is it deferred to another section?
4. Are error conditions enumerated with specific error codes?
5. Are edge cases covered?
6. **Can an agent build this using ONLY these docs?**

**Module audit table:**

| Module | Purpose Clear? | Input/Output Spec'd? | Algorithm Detailed? | Error Handling? | Edge Cases? | Buildable? |
|--------|----------------|------------------|------------------|-----------------|-------------|-----------|
| `src/index.ts` | Entry point, creates server | Yes/No | Yes/No | Yes/No | Yes/No | Y/N |
| `src/server.ts` | Tool registration | Yes/No | Yes/No | Yes/No | Yes/No | Y/N |
| `src/tools/get-context.ts` | 11-step behavior | Yes/No | Yes/No (11 steps) | Yes/No | Yes/No | Y/N |
| ... (all 15 modules) | | | | | | |

**High-risk modules to focus on:**
- `src/indexer/entity-extractor.ts` — Tree-sitter queries specified? Or just prose rules?
- `src/generator/claude-md.ts` — Full merge algorithm step-by-step? Malformed marker handling? Empty section omission?
- `src/tools/memorize.ts` — When does CLAUDE.md regenerate? Only for project-wide rules? What triggers it?
- `src/db/migrations.ts` — Transaction/rollback behavior specified?

**Flag:** Modules marked `N` in "Buildable?" column.

---

### Pass 4: Contradiction Detection

**Objective:** Find statements in one doc that **directly contradict** another. These are the most dangerous — agent follows whichever doc it reads last.

**Known contradiction-prone areas:**

1. **Q4 hit_count threshold:** 
   - Product Spec says: "A gotcha useful twice has proven value" → `hit_count >= 2`
   - Doc 04 necessity test says: "exclude gotchas where `hit_count = 0` AND `created_at` is older than 30 days"
   - These are different filters! One is "exclude if < 2 hits", the other is "exclude if 0 hits AND stale". Which is correct?

2. **When is `optima-feedback.md` regenerated?**
   - Doc 01: "written on first `optima_get_context` call"
   - Doc 04: "regenerated when the feedback rules need updating"
   - When exactly? Only on cold start? Every call? After memorize?

3. **Linter detection config files:**
   - Doc 01 lists: "ESLint, Prettier, Biome"
   - Doc 04 also lists: "Ruff, pyproject.toml [tool.ruff], .editorconfig"
   - Are these supposed to be the same list? Is there a unified authoritative list?

4. **File I/O async rule:**
   - Ground Rule 4 (Doc 00): "All file I/O must be async"
   - `better-sqlite3` is synchronous
   - Is there a documented carve-out for database I/O?

5. **CLAUDE.md regeneration triggers:**
   - Doc 03 step 5: "If the new entry is an `architectural_rule` and `directory` is null (project-wide), trigger CLAUDE.md regeneration"
   - What about new gotchas? New patterns? Do they trigger regeneration too?

6. **`optima_reindex` side effects:**
   - Doc 03 doesn't mention CLAUDE.md regeneration after reindex
   - But logically the project context changed — should it regenerate?

7. **`recent_changes` population:**
   - Doc 01: "files whose mtime changed since last index"
   - Doc 03 tool step: "Collect `recent_changes` — file paths re-indexed in steps 5-7"
   - Same thing, or different? (Seems the same, but phrased differently.)

8. **Generation log pruning trigger:**
   - Doc 02: "Prune on database initialization"
   - Does this mean: every `getDatabase()` call? Only cold start? First call each session?

9. **Path normalization scope:**
   - Ground Rule 7: "All paths stored in SQLite use forward slashes"
   - Does this apply to paths in `files` JSON array in `gotchas` and `rules` tables? What about error messages?

10. **Tool Search descriptions location (GAP, not contradiction):**
    - Doc 00 mentions tools are "tuned for Tool Search discoverability"
    - Doc 03 mentions Zod schemas should have `.describe()` strings for discoverability
    - BUT: No actual tool description strings are provided anywhere in the lean pack
    - The Spec Generator Prompt says "Tool names and `.describe()` strings (tuned for Claude Code's Tool Search discoverability)"
    - Are these supposed to be in the Zod schemas (Doc 03) or in server.ts? Not specified.
    - **Flag:** Tool description specification is missing or vague

**Output:** For each contradiction: `Conflicting statements`, `Doc A quote`, `Doc B quote`, `Severity (P0/P1/P2)`, `Resolution options`.

---

### Pass 5: Security & Safety Boundary Validation

**Objective:** Verify all security rules are airtight — no exceptions, no carve-outs, no "developer responsibility."

1. **Exclusion list unified check:**
   - Compare across: Doc 00 (Security-sensitive paths), Doc 03 (gitignore + security exclusions), Product Spec section 12
   - Missing entries? Format inconsistencies (e.g., `~/.claude/session-env/` vs `.claude/session-env/`)?

2. **CLAUDE.local.md protection:**
   - "Never read, modify, or generate" — is this enforced in:
     - File indexer (Doc 03)? ✓ or ✗
     - CLAUDE.md regeneration (Doc 04)? ✓ or ✗
     - DO NOT list (Doc 00)? ✓ or ✗

3. **`.env` pattern coverage:**
   - `.env`, `.env.*`, `.env.local`, `.env.production`, `.env.example`
   - Doc 03 says `.env`, `.env.*` — but is `.env.example` intended to be excluded? (Often safe, kept in repo for reference.)

4. **Generated file content safety:**
   - Could `CLAUDE.md` sections ever expose secrets?
   - Example: if an error message in a gotcha contains an API key, it appears in CLAUDE.md's `known_gotchas` → exposed in a file that might be committed
   - Is there error message sanitization before storing in gotcha ledger?

5. **Settings.json write safety:**
   - MVP spec says Optima never writes `.claude/settings.json`
   - Is this enforced with explicit DO NOT? Or just implied because Phase 2?

6. **Path traversal prevention:**
   - `optima_get_context({path: "../../.ssh/"})` — could this escape project root?
   - Is there input validation that constrains paths to project subtree?

7. **Sensitive data in database:**
   - If an error message contains `DATABASE_URL=postgres://user:pass@host`, is it sanitized before storing?
   - Or does the app assume error messages are already sanitized?

**Output:** For each boundary: `Boundary`, `Enforced in doc(s)`, `Gaps`, `Risk level (low/medium/high)`.

---

### Pass 6: Spec Generator Prompt Alignment

**Objective:** Does the Spec Generator Prompt correctly request all the new content added in the recent review pass?

1. **Checklist of new sections:** Does the prompt request:
   - Cold Start Bootstrap Sequence? (Section 1, item 1)
   - Database Initialization & Lifecycle? (Section 3)
   - Gotcha Retrieval Strategy? (Section 5)
   - Entity Extraction Edge Cases? (Section 5)
   - Directory Scoping Precedence? (Section 5)
   - File System Edge Cases? (Section 7)
   - Malformed Marker Handling? (Section 6)
   - Preference Section Template? (Section 6)
   - Project Purpose Extraction Priority? (Section 6)
   - Schema Migration Strategy? (Section 4)

2. **Resolved questions:**
   - Does the prompt reference Q1–Q16 or only Q1–Q11?
   - Are the new questions (Q12–Q16) mentioned in the Input or Context sections?

3. **Verbatim directives:**
   - Are all "copy verbatim" blocks in the docs flagged as such in the prompt?
   - Any new code blocks added in this review that should be flagged?

4. **Implementation plan coverage:**
   - Does Section 9 (implementation phases) mention all modules in the project structure?
   - Are the new considerations (malformed markers, preferences, path normalization, etc.) included in the relevant phases?

5. **Testing strategy:**
   - Does Section 8 mention: Windows paths, malformed markers, preferences section, entity edge cases, directory hierarchy, project purpose extraction?

6. **Verbatim directives alignment:**
   - Does the prompt request "copy verbatim" for all code blocks added in the recent review pass?
   - New code blocks to check:
     - Error normalization function with Windows paths (Doc 03)
     - Migration runner logic (Doc 02)
     - Malformed marker handling algorithm (Doc 04)
     - Project purpose extraction priority list (Doc 04)
     - Schema migration file convention (Doc 02)
   - Any new code block not flagged as "copy verbatim" in the prompt = risk of paraphrasing

**Output:** Checklist: `Prompt Section`, `Should Mention`, `Does Mention (Y/N)`, `Gap`.

---

### Pass 7: Agent Failure Mode Prediction

**Objective:** Predict the top 10–15 places where an autonomous agent is most likely to make the **wrong choice** or **get stuck**.

For each predicted failure:
1. **What the agent does wrong**
2. **Why** (what in the docs caused it?)
3. **Which doc/section**
4. **Severity**: `P0` (blocks build), `P1` (bug), `P2` (suboptimal)

**Predicted failure modes to evaluate:**

1. **`optima_memorize` regeneration trigger ambiguity**
   - Agent reads Doc 03 step 5: only project-wide `architectural_rule` triggers regeneration
   - Agent assumes new `gotchas`, `patterns`, `preferences` do NOT trigger regeneration
   - Reality: Is this correct? Or should every new entry trigger at least a partial regeneration?
   - **Risk:** Silent bug — no CLAUDE.md updates after storing gotchas for a month

2. **Entity extraction edge cases gone wrong**
   - Agent reads entity extraction rules, but Tree-sitter queries not specified
   - Agent guesses at query syntax → extracts wrong entities or misses some
   - **Risk:** Entity list incomplete/wrong → reduced context quality

3. **Gotcha retrieval hierarchy confusion**
   - Agent reads "most specific directory first" but implements longest-match incorrectly
   - Agent sorts by `created_at` for precedence instead of specificity
   - **Risk:** Wrong gotcha returned for nested directories

4. **Malformed marker recovery not implemented**
   - Agent only handles well-formed markers
   - Existing projects have orphaned `<!-- OPTIMA:START -->` without END
   - Agent crashes or overwrites content
   - **Risk:** Data loss, silent corruption of CLAUDE.md

5. **hit_count vs hit_count >= 2 threshold**
   - Agent reads Doc 04 necessity test: "exclude if hit_count = 0 AND >30 days"
   - Agent doesn't read Q4 in Product Spec: "2 hits for proven value"
   - Agent includes single-hit gotchas in CLAUDE.md
   - **Risk:** CLAUDE.md polluted with unproven fixes

6. **Path normalization inconsistency**
   - Agent normalizes paths on ingestion but stores `\` paths in some code path
   - Agent queries with forward-slash paths don't match backslash-stored paths
   - **Risk:** Gotchas/rules not found when they should be

7. **Database init vs connection lifecycle**
   - Agent doesn't understand lazy init
   - Agent tries to create DB in `index.ts` startup instead of first tool call
   - Agent doesn't handle cold start bootstrap properly
   - **Risk:** Inception pattern breaks — rules file doesn't exist on first call

8. **Tool Search description missing**
   - Agent doesn't find where tool descriptions are specified
   - Agent hardcodes vague descriptions
   - Claude Code's Tool Search doesn't load tools
   - **Risk:** Optima tools not discoverable

9. **CLAUDE.md section ordering**
   - Doc 04 specifies section order but agent doesn't enforce it
   - Agent appends in random order
   - CLAUDE.md is harder to read
   - **Risk:** UX issue, harder to maintain (minor)

10. **Symlink infinite loop**
    - Agent uses `fs.readdirSync` instead of checking `fs.lstat().isSymbolicLink()`
    - Symlinks not detected → infinite recursion if `.git` is symlinked
    - **Risk:** Indexer hangs

11. **`hit_count` vs `files` array length confusion**
    - Agent reads: "increment `hit_count` when gotcha is returned" (Doc 03)
    - Agent confuses: "hit_count is how many files have this error"
    - Reality: `hit_count` tracks how many times `optima_get_context` returned this gotcha, not file count
    - Agent stores wrong semantic meaning in comments/docs
    - **Risk:** CLAUDE.md gotchas are mislabeled (e.g., "encountered in 3 files" when actually "hit 3 times")

12. **`directory` scope vs `files` array conflation**
    - Doc 03 Gotcha Retrieval Strategy mentions both `directory` matching AND `files` array matching
    - Agent confuses: `directory` is the same as `files`? Or different filters?
    - Reality: `directory` is the primary scope; `files` is the secondary match mechanism
    - Agent implements either one or the other, not both
    - **Risk:** Gotchas not retrieved correctly for nested directories

13. **`memory_id` is ephemeral, not stored**
    - Doc 03 says: "Generate a random UUID v4 for `memory_id`... it is NOT the database row's `id`... not stored"
    - Agent misinterprets: `memory_id` is the unique identifier for the gotcha
    - Agent tries to use it to retrieve the gotcha later: `optima_get_context` returns `memory_id`, agent tries to call `optima_memorize` with the same `memory_id`
    - Reality: `memory_id` is a one-time receipt, discarded after the memorize call returns
    - **Risk:** Silent bug — agent's mental model of how memorize works is wrong, may not detect issues until testing

14. **CLAUDE.md regeneration timing ambiguity**
    - Doc 03 step 5: "only project-wide `architectural_rule` triggers regeneration"
    - Agent assumes: only `architectural_rule` type; new `gotchas` and `patterns` do NOT trigger
    - Does Doc 04 contradict this? Or does it imply all new entries regenerate?
    - **Risk:** CLAUDE.md not updated when new gotchas are stored; context stale by design

**Output:** Numbered list (11–14), each with: `Failure`, `Root cause in docs`, `Doc/section`, `Severity`, `Detection hint (what would expose the bug?)`.

---

## Output Structure

Produce audit results as a single document with these 8 sections (Pass 0 through Pass 7). End with:

### Summary Section

- **Total issues by severity:** # P0 (blocking), # P1 (bugs), # P2 (suboptimal)
- **Blocking issues (P0 only):** List them
- **Top 3 riskiest areas:** Where do issues cluster?
- **Readiness verdict:** 
  - `READY` (no P0, <3 P1)
  - `READY WITH CAVEATS` (1–2 P0, manageable)
  - `NOT READY` (3+ P0, or systematic gaps)
- **One-paragraph justification** of the verdict

---

## Audit Constraints

- **Be specific.** "Doc X is vague" ≠ useful. Quote the exact vague sentence.
- **Quote sources.** When flagging a contradiction, quote both conflicting sentences with doc/section.
- **No fixes — only diagnosis.** State what's wrong, not how to fix it.
- **Assume worst-case agent.** If a reasonable human "figures it out" but docs don't say it explicitly, flag it.
- **Compare across ALL 7 docs.** Don't miss contradictions hiding in different docs.
- **Pass 0 is critical.** Start with dangling reference scan before auditing content — broken links block everything.
- **Check Pass 8 predictions against Pass 4 contradictions.** If you found a contradiction in Pass 4, does it appear in agent failure predictions in Pass 8?
