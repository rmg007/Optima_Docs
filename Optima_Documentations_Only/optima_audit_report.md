# Optima Documentation Suite — Audit Report

## Pass 1: Cross-Document Schema Consistency

| Field/Type | Doc 02 (Drizzle Schema) | Doc 02 (TS Interface) | Doc 03 (Zod Schema) | Doc 04 (Template) | Consistent? (Y/N) | Issue |
|---|---|---|---|---|---|---|
| **Rule Example** | MISSING | MISSING | `example` | `{{rule.example}}` | N | **CRITICAL:** `optima_memorize` (03) accepts `example` for patterns, and templates (04) use it, but `rules` table and `RuleEntry` interface (02) lack an `example` field. Data will be inexplicably dropped. |
| **Project Name** | `name` | `name` | `name` | `{{project_name}}` | N | Template expects `project_name`, but the property is just `name`. Code using this template will render blank names. |
| **Project Purpose** | `projectPurpose` (`project_purpose`) | `purpose` | `purpose` | `{{project_purpose}}` | N | Template uses `project_purpose` while TS and Zod data structures expose `purpose`. |
| **Entity Lines** | `lineStart`, `lineEnd` | `line` | `line` | N/A | N | Database captures start/end, but TS interface and Zod schema only expect a single `line` property. |
| **Entity Description**| `description` | MISSING | MISSING | N/A | N | Drizzle schema contains `description`, but it is completely omitted from the TS interface and Zod output schemas. |
| **Tech Stack** | `techStack` -> `string` (JSON) | `tech_stack: string[]` | `tech_stack: z.array(z.string())`| `{{tech_stack_comma_separated}}` | Y | Consistent, but template explicitly requires pre-processing the array into a comma-separated string `tech_stack_comma_separated`. |
| **Entity Kind Enum** | `text` (no enum constraint) | `"function" [...]` | `z.enum([...])` | N/A | Y | The union types perfectly match across 02 and 03. |
| **Rule Type Enum** | `text` (no enum constraint) | `"architectural_rule" [...]`| `z.enum([...])` | `type = "architectural_rule"`... | Y | The union types match. |
| **Error Codes** | `OptimaErrorCode` type | `OptimaErrorCode` | Zod does not define | N/A | Y | The 8 codes in the 02 type match the 03 taxonomy table exactly. |
| **MemorizeInput** | Union type | Union type | `z.object().refine(...)` | N/A | Y | The TS interface and Zod refine logic are correctly aligned. |
| **Linter Detected** | `linterDetected` -> `string` (JSON) | `linter_detected: string \| null` | `linter_detected: z.string().nullable()` | N/A | Y | Consistent. |

## Pass 2: Resolved Questions Traceability

| Question | Resolution | Expected in Doc(s) | Found? (Y/N) | Discrepancy |
|---|---|---|---|---|
| **Q2 (CLAUDE.md handling)** | Append below, never touch above. | 04 (Regeneration) | Y | "Content outside markers is never modified." |
| **Q3 (Hash collision)** | Warn and store both. | 03 (optima_memorize, Errors) | Y | Specified in error taxonomy and `optima_memorize` dedup logic. |
| **Q4 (Hit_count threshold)**| threshold of 2 hits. | 04 (Necessity test) | **N** | **Discrepancy:** Doc 04 necessity test states: "Exclude gotchas with hit_count=0 older than 30 days". This directly contradicts the expected 2-hit threshold resolution. |
| **Q5 (Ignore npm package)** | Use `ignore` package to respect `.gitignore`. | 00 (Tech Stack), 03 (Exclusions) | Y | Correctly specified in the tech stack table and the file indexer edge cases. |
| **Q6 (better-sqlite3)** | Use `better-sqlite3` strictly. | 00, 02 (Lifecycle), Product Spec | Y | Explicitly called out across all three documents. |
| **Q14 (Gotcha retrieval)** | Hierarchical directory matching. | 03 (Gotcha Retrieval Strategy) | Y | Accurately detailed as Prefix match + File match, deduplicated by ID, and sorted. |

## Pass 3: Implementation Completeness Scan

| File | Key Questions & Implementation Status |
|---|---|
| `src/index.ts` | **PARTIALLY SPECIFIED:** States it creates and starts MCP server. Lacks explicit process signal handling (SIGTERM, SIGINT) and graceful shutdown procedures. |
| `src/server.ts` | **PARTIALLY SPECIFIED:** Missing details on how server instantiation bridges stdio with the specific tool handlers. Assumes implicit MCP SDK knowledge. |
| `src/tools/get-context.ts` | **PARTIALLY SPECIFIED:** Step 11 requires the tool to "synthesize a concise summary" for the directory. MCP servers are logic layers; they do not have embedded LLMs to perform linguistic "synthesis." The algorithmic method for this synthesis is missing. |
| `src/tools/memorize.ts` | **FULLY SPECIFIED:** Logic is clearly mapped to steps. |
| `src/tools/reindex.ts` | **FULLY SPECIFIED:** Unambiguously skips `CLAUDE.md` regeneration, even though that might be conceptually undesirable. |
| `src/indexer/project-analyzer.ts` | **PARTIALLY SPECIFIED:** Tech stack detection algorithm is omitted. What strings exactly go in the array? Is there a dictionary? Missing explicit logic for command extraction from `package.json` scripts. |
| `src/indexer/file-indexer.ts` | **PARTIALLY SPECIFIED:** Does not specify the traversal approach (depth-first vs. breadth-first) or handle parsing of nested `.gitignore` files relative to their subdirectories. |
| `src/indexer/entity-extractor.ts` | **PARTIALLY SPECIFIED:** Describes what to extract but omits the actual Tree-sitter query representations. The agent must invent these complex S-expression queries. |
| `src/memory/gotcha-ledger.ts` | **UNSPECIFIED:** Exposes no CRUD method signatures. Tool handlers simply say "Query table." |
| `src/memory/rules-store.ts` | **UNSPECIFIED:** Exposes no CRUD method signatures. |
| `src/generator/claude-md.ts` | **PARTIALLY SPECIFIED:** Mentions "30-35 instructions total", but has no programmatic mechanism defined to measure or enforce an LLM instruction count. |
| `src/generator/feedback-rules.ts`| **UNSPECIFIED:** Trigger condition states: "regenerated when the feedback rules need updating." Nothing defines when they mathematically/logically "need updating." |
| `src/db/connection.ts` | **FULLY SPECIFIED:** Lazy init sequence is clear. |
| `src/db/migrations.ts` | **FULLY SPECIFIED:** Migration runner transaction/rollback flow is clear. |
| `src/db/schema.ts` | **FULLY SPECIFIED:** Verbatim copy logic given. |
| `src/utils/hasher.ts` | **PARTIALLY SPECIFIED:** Does not specify node `crypto` vs `Bun.crypto`, nor does it dictates input formatting (buffers vs text encoding) prior to hashing. |
| `src/utils/paths.ts` | **FULLY SPECIFIED:** Explicit on symlinks, binary detection, and path slashes. |
| `src/utils/errors.ts` | **FULLY SPECIFIED:** Verbatim code provided. |

## Pass 4: Contradiction Detection

1. **Rule Example Field Existence**
   - **Statement A:** `03_MCP_TOOL_CONTRACTS` (`MemorizeInput`) and `04_INCEPTION_PAYLOAD` templates accept and use an `example` string for patterns.
   - **Statement B:** `02_DATA_MODEL_AND_SCHEMA` rules table and interface strictly omit the `example` field.
   - **Severity:** **P0 Blocking** — Agents cannot store requested structural data.
   - **Recommended Resolution:** Add an `example` column to the `rules` schema.

2. **Wait, Synchronous File I/O vs Rules**
   - **Statement A:** `00_START_HERE` (Ground Rules) states "All file I/O must be async", explicitly mentioning only `better-sqlite3` as an exception.
   - **Statement B:** `02_DATA_MODEL_AND_SCHEMA` dictates `fs.mkdirSync('.optima')`.
   - **Severity:** **P2 Cosmetic** — Agent might strictly try to refactor DB initialization or trip over the rule.
   - **Recommended Resolution:** Relax Ground Rule #4 to allow synchronous directory creations during cold-start.

3. **CLAUDE.md Regeneration Trigger**
   - **Statement A:** `03_MCP_TOOL_CONTRACTS` (`optima_memorize` step 5) triggers `CLAUDE.md` regeneration *strictly* if `type === "architectural_rule"` and `directory` is null.
   - **Statement B:** Logically, adding project-wide patterns or preferences should also update the file.
   - **Severity:** **P1 Confusing** — `CLAUDE.md` will become stale for patterns/preferences unless handled via rules.
   - **Recommended Resolution:** Broaden Step 5 to trigger on any project-wide insertions.

4. **Linter Detection Scope Difference**
   - **Statement A:** `01_PRODUCT_SPEC_MVP` lists ESLint, Prettier, Biome.
   - **Statement B:** `02_DATA_MODEL_AND_SCHEMA` explicitly lists `ruff.toml, pyproject.toml [tool.ruff], .editorconfig` additionally.
   - **Severity:** **P2 Confusing** — Harmless but contradictory constraints for the analyzer module.
   - **Recommended Resolution:** Unify the source list to encompass all configuration formats.

5. **Optima Reindex and Tech Stack changes**
   - **Statement A:** `optima_reindex` removes and re-parses all files, effectively updating the project definition and index.
   - **Statement B:** `optima_reindex` omits `CLAUDE.md` regeneration, meaning structural changes (like deleting an old configuration logic and installing a new framework) won't materialize in `CLAUDE.md` automatically until an arbitrary memory insertion occurs.
   - **Severity:** **P1 Confusing** — System state goes out of sync with generated payload immediately after forced re-indexing.

## Pass 5: Security & Safety Boundary Validation

| Boundary | Enforced in doc(s) | Gaps | Risk Level |
|---|---|---|---|
| **Security Exclusion List Completeness** | 00_START_HERE.md, 03_MCP_TOOL_CONTRACTS.md | Doc 00 lists `**/secrets/**` but Doc 03 omits it from the specific indexer hardcoded step list. Conversely, Doc 03 lists paste and image caches, which Doc 00 omits. | Medium — Inconsistent exclusions might lead the agent to implement only one portion, resulting in leaked secrets. |
| **CLAUDE.local.md Preservation** | 00_START_HERE.md, 03_MCP_TOOL_CONTRACTS.md | Although index and manipulation are forbidden, Doc 04 misses an instruction stating that regeneration MUST NEVER accidentally output to or create `CLAUDE.local.md`. | Low — `claude-md.ts` targets `CLAUDE.md`, but safety bounds shouldn't be implied. |
| **.env File Handling** | 03_MCP_TOOL_CONTRACTS.md | The pattern `.env.*` implicitly blocks `.env.example`, which is generally meant to be safe to read by tools to assess environment shapes. | Low — Functional annoyance, not a security risk. |
| **Scrubbing Sensitives in Gotchas** | 03_MCP_TOOL_CONTRACTS.md | Specifies scrubbing the *hashed* form via regexes but later states: "the raw error_text must also have sensitive secrets redacted before storage". It provides NO deterministic way to redact raw errors programmatically since secret detection isn't built into MCP. | High — Captured tokens or passwords in test output will be stored in SQLite and injected into `CLAUDE.md` in plaintext. |
| **Path Traversal Protection** | 03_MCP_TOOL_CONTRACTS.md | The instruction "Resolve `path` relative to project root" is given. However, there is no validation step to guarantee the result has not broken the project boundary (e.g. escaping the repo via `..`). | High — An agent without a boundary check can read the host `/etc/passwd` filesystem. |

## Pass 6: Spec Generator Prompt Alignment

| Prompt Section | Doc Content | Aligned? (Y/N) | Gap |
|---|---|---|---|
| **Section 1: Cold Start Bootstrap** | 01 (Cold Start) | Y | None |
| **Section 3: Database Init Lifecycle** | 02 (Database Initialization) | Y | None |
| **Section 4: Schema Migration Strategy** | 02 (Schema Migration) | Y | None |
| **Section 5: Gotcha Retrieval Strategy** | 03 (Gotcha Retrieval Strategy) | Y | None |
| **Section 5: Entity Extract Edge Cases** | 03 (Entity Extraction Edge Cases) | Y | None |
| **Section 5: Directory Scoping** | 03 (Directory Scoping Precedence) | Y | None |
| **Section 6: Malformed Marker Handling** | 04 (Malformed marker handling) | Y | None |
| **Section 6: Preferences Template** | 04 (Section: preferences) | Y | None |
| **Section 6: Purpose Extractor priority**| 04 (Project Purpose Extraction) | Y | None |
| **Verbatim Directives Verification** | 02, 03, 04 | Y | All artifacts flagged as verbatim are effectively representable as static blocks format. |

## Pass 7: Agent Failure Mode Prediction

1. **The Pattern Example Blackhole (P0)**
   - **What agent will do wrong:** Implementing `optima_memorize` will either drop the `example` parameter entirely (because `RuleEntry` lacks it) or the agent will violate "copy verbatim" rules and mutate the TypeScript and Drizzle schemas independently. Subsequently, `CLAUDE.md` generator will insert blank/undefined examples into the UI.
   - **Source:** Conflict between `03_MCP_TOOL_CONTRACTS.md` tool inputs and `02_DATA_MODEL_AND_SCHEMA.md` table architecture.
   - **Fix:** Redefine the `rules` SQLite table schema to encompass the `example` column natively.

2. **The LLM "Synthesis" Hallucination (P0)**
   - **What agent will do wrong:** `get-context.ts` is instructed to "synthesize a concise summary of the directory's role." An executing agent will realize it has no API access to an LLM from inside the tool to write creative text, leading to code that either errors out, tries to `fetch()` Anthropic directly (bypassing MCP), or returns unhandled exceptions.
   - **Source:** `03_MCP_TOOL_CONTRACTS.md`, `optima_get_context` step 11.
   - **Fix:** Rewrite step 11 to require a deterministic algorithm (e.g. read the first `<h1>` or 200 chars from the local `README.md`, else return empty string).

3. **Template Variable Mismatch Exceptions (P1)**
   - **What agent will do wrong:** The agent will substitute Handlebars replacements using the `ProjectSummary` interface, causing `{{project_name}}` and `{{project_purpose}}` to resolve as undefined since the object properties are strictly `name` and `purpose`.
   - **Source:** `04_INCEPTION_PAYLOAD.md` markers differing from `02_DATA_MODEL.md`.
   - **Fix:** Ensure template variables map 1:1 natively to the object models.

4. **The Security Scrubbing Gap (P1)**
   - **What agent will do wrong:** Attempts to implement the instruction to "apply a scrubbing pass against the raw error_text" will result in a generic, poorly written hallucinated regex filter because no regex is provided for the raw text pass (only for the hashed error keys), resulting in leaked project credentials in plaintext `CLAUDE.md` files.
   - **Source:** `03_MCP_TOOL_CONTRACTS.md` section on Error Normalization for dedup.
   - **Fix:** Provide a deterministic function for redacting secrets in the raw text, just as it was provided for the hashed deduplication keys.

5. **Path Traversal Sandbox Escape (P1)**
   - **What agent will do wrong:** The `resolve path` logic in `get-context.ts` won't be clamped by boundary checks. Malicious directory requests via MCP will succeed and evaluate system-level files because LLMs don't natively preempt path escape vulnerabilities unless prompted.
   - **Source:** `03_MCP_TOOL_CONTRACTS.md` `optima_get_context` step 1.
   - **Fix:** Dictate explicit security checking to throw if the normalized path does not start with the project root.

6. **The Stale Pattern File Log (P1)**
   - **What agent will do wrong:** A new pattern will be memorized successfully, but because `directory === null` logic in Step 5 strictly targets `architectural_rule` events, `CLAUDE.md` will not show the pattern. The agent writes what is asked.
   - **Source:** `03_MCP_TOOL_CONTRACTS.md`, `optima_memorize` step 5.
   - **Fix:** Expand trigger events to encapsulate `"pattern"` and `"preference"` globally.

7. **Database Pruning Dead-Code (P2)**
   - **What agent will do wrong:** It will run Generation Log pruning when `getDatabase()` is first invoked. Because the connection acts as a singleton cache for the lifetime of the process, pruning will happen exceptionally rarely, bloating the `generation_log` infinitely.
   - **Source:** `02_DATA_MODEL_AND_SCHEMA.md` Retention Policy.
   - **Fix:** Designate pruning to occur deterministically at the conclusion of every `get_context` evaluation lifecycle.

8. **Feedback Rules Re-Generation Trigger Confusion (P2)**
   - **What agent will do wrong:** `feedback-rules.ts` generator will never fire after the first cold-boot because the spec dictates it runs "when the feedback rules need updating" without providing metrics on what flags that boolean reality.
   - **Source:** `04_INCEPTION_PAYLOAD` introduction regarding `.claude/rules/optima-feedback.md`.
   - **Fix:** Clarify the trigger to rely on a version hash or an explicit regeneration utility endpoint.

9. **Tech Stack Detection Bloat (P2)**
   - **What agent will do wrong:** An agent forced to "detect tech stack from `package.json`" without guidance will parse all `dependencies`, producing a polluted array of 100+ nested library names.
   - **Source:** `03_MCP_TOOL_CONTRACTS.md` step 2.
   - **Fix:** Restrict extraction strictly to explicitly whitelisted domains (React, Express, Vue, Django, Next.js, etc.) or just core `devDependencies`.

10. **Entity Line Extraction Collapse (P2)**
    - **What agent will do wrong:** The agent will encounter Drizzle models requesting `lineStart` and `lineEnd` but its TS output schemas only requesting `line`. It will arbitrarily select `lineStart` while losing the span of the code structure silently.
    - **Source:** `02_DATA_MODEL` tables vs types.
    - **Fix:** Expand the TS interface to maintain start/end context mapping out of Drizzle explicitly.

---

### Summary

- **Total issues found:** 18 anomalies identified
  - **P0 (Blocking):** 2
  - **P1 (Bug-inducing):** 6
  - **P2 (Suboptimal):** 10
- **Blocking issues (P0):**
  1. The `rules` schema and TS interfaces completely omit the `example` field, critically breaking Pattern storage and Template visualization rendering workflows entirely.
  2. `get-context.ts` expects the tool implementation logic itself to "synthesize summaries", assuming the existence of native LLM inferencing logic inherently unavailable inside a pure MCP JSON handler.
- **Top 3 highest-risk areas:**
  1. **Schema Mismatches vs Templating Variables** (Data properties don't correspond uniquely to frontend representations).
  2. **Security & Data Sanitization** (Path traversals lack hard boundary constraints; `error_text` scrubbing lacks deterministic algorithms).
  3. **Event Trigger Accuracy** (`CLAUDE.md` and `feedback-rules.md` updates miss core condition evaluations).
- **Overall implementation readiness:** `NOT READY`.
  The core foundation is incredibly detailed and well-thought-out, but an adversarial agent implementing strictly from this context base will suffer a fatal failure during state compilation (`example` schema collision) and a logic halt (the impossible text `synthesis`). It's very close, but the architectural and safety gaps highlighted must be closed natively in the schema and step models before an autonomous toolchain attempts a unified build process.
