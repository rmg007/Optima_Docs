# Optima Observability Guide

Optima logs every significant operation to **stderr** in newline-delimited JSON. This makes it trivial to capture, parse, and analyze what the server is doing at runtime — essential for improving the system, debugging failures, and understanding real-world usage patterns.

---

## Why stderr, Not stdout

Optima is a **stdio MCP server**. The MCP protocol uses `stdout` as a bidirectional JSON-RPC channel. Writing anything to `stdout` other than valid JSON-RPC frames corrupts the protocol stream and breaks the connection. All log output therefore goes to `stderr`, which Claude Code (and any MCP host) leaves separate from the protocol stream.

**This is a hard constraint, not a preference.** Any `console.log()` in the server code is a protocol bug.

---

## Log Format

Every log line is a self-contained JSON object:

```json
{
  "ts":        "2025-04-14T05:43:36.812Z",
  "level":     "INFO",
  "component": "get-context",
  "msg":       "response assembled",
  "path":      "src/auth",
  "is_cold_start": false,
  "files_reindexed": 2,
  "files_removed": 0,
  "entities": 14,
  "gotchas": 3,
  "rules": 2,
  "security_warnings": 0,
  "duration_ms": 47
}
```

| Field       | Type   | Description |
|-------------|--------|-------------|
| `ts`        | string | ISO 8601 UTC timestamp |
| `level`     | string | `DEBUG` \| `INFO` \| `WARN` \| `ERROR` |
| `component` | string | Which module emitted this log (see table below) |
| `msg`       | string | Human-readable event description |
| `...`       | mixed  | Structured fields specific to the event |

---

## Log Levels

| Level   | Used for |
|---------|----------|
| `DEBUG` | Verbose internal state — file walks, entity counts per file, hash comparisons, individual hit-count increments. Useful for deep debugging, too noisy for normal use. |
| `INFO`  | Normal lifecycle events — tool invocations, cold start, index sync results, project analysis, CLAUDE.md written, memory stored. Default in production. |
| `WARN`  | Recoverable problems — secret pattern detected, FTS5 query error (fell back), concurrent write conflict, database corruption recovery. Always investigate. |
| `ERROR` | Fatal/unrecoverable — tool handler failed, migration failed, unhandled promise rejection. Requires immediate attention. |

### Controlling Log Level

Set the `OPTIMA_LOG_LEVEL` environment variable before starting the server:

```bash
# Normal use — INFO and above
OPTIMA_LOG_LEVEL=INFO node dist/index.js

# Verbose debugging
OPTIMA_LOG_LEVEL=DEBUG node dist/index.js

# Quiet — warnings and errors only
OPTIMA_LOG_LEVEL=WARN node dist/index.js
```

Default is `DEBUG` when the variable is unset (single-user development phase — maximum verbosity).

---

## Components and Their Events

### `server`
The MCP server lifecycle.

| Event | Level | Key Fields |
|-------|-------|------------|
| `optima MCP server starting` | INFO | `version` |
| `transport connected — ready for tool calls` | INFO | — |
| `tools registered` | INFO | `tools` (array of tool names) |
| `shutdown initiated` | INFO | `signal` (SIGINT/SIGTERM/SIGHUP) |
| `database closed` | INFO | — |
| `unhandled rejection` | ERROR | `reason`, `stack` |

### `tool`
Every tool invocation is bracketed by two log lines: one on entry, one on completion or failure.

| Event | Level | Key Fields |
|-------|-------|------------|
| `optima_get_context called` | INFO | `path`, `search_query` |
| `optima_get_context completed` | INFO | `path`, `is_cold_start`, `files_reindexed`, `files_removed`, `gotchas`, `architectural_rules`, `security_warnings`, `entities`, `duration_ms` |
| `optima_get_context failed` | ERROR | `path`, `error`, `code`, `duration_ms` |
| `optima_memorize called` | INFO | `type`, `directory` |
| `optima_memorize completed` | INFO | `type`, `stored`, `duplicate_detected`, `hit_count_updated`, `total_memories`, `claude_md_regenerated`, `duration_ms` |
| `optima_memorize failed` | ERROR | `type`, `error`, `code`, `duration_ms` |
| `optima_reindex called` | INFO | `path` |
| `optima_reindex completed` | INFO | `path`, `files_indexed`, `entities_found`, `project_analysis_updated`, `claude_md_regenerated`, `duration_ms` |
| `optima_reindex failed` | ERROR | `path`, `error`, `code`, `duration_ms` |
| `optima_dismiss_warning called` | INFO | `finding_id`, `reason` |
| `optima_dismiss_warning completed` | INFO | `finding_id`, `file`, `pattern_name`, `dismissed`, `was_already_dismissed`, `remaining_warnings`, `duration_ms` |
| `optima_dismiss_warning failed` | ERROR | `finding_id`, `error`, `code`, `duration_ms` |
| `optima_forget called` | INFO | `memory_id`, `reason` |
| `optima_forget completed` | INFO | `memory_id`, `deleted_from`, `total_memories`, `claude_md_regenerated`, `duration_ms` |
| `optima_forget failed` | ERROR | `memory_id`, `error`, `code`, `duration_ms` |

### `db`
Database initialization and connection management.

| Event | Level | Key Fields |
|-------|-------|------------|
| `initializing database` | INFO | `dbPath` |
| `database ready` | INFO | `dbPath` |
| `database corruption detected — recovering` | WARN | `dbPath`, `error` |
| `database recreated after corruption recovery` | INFO | `dbPath` |
| `failed to open database` | ERROR | `dbPath`, `error` |
| `pragmas applied` | DEBUG | `journal_mode`, `foreign_keys` |
| `pruned old records` | DEBUG | `generation_log_rows_deleted`, `task_outcomes_rows_deleted` |
| `created .optima/.gitignore` | DEBUG | — |
| `connection closed` | DEBUG | — |

### `migrations`
Schema migration runner.

| Event | Level | Key Fields |
|-------|-------|------------|
| `schema up to date` | DEBUG | `version` |
| `running pending migrations` | INFO | `current_version`, `pending` (array of `{version, description}`) |
| `migration applied` | INFO | `version`, `description`, `duration_ms` |
| `migration failed` | ERROR | `version`, `description`, `error` |

### `get-context`
The `handleGetContext` handler internals.

| Event | Level | Key Fields |
|-------|-------|------------|
| `cold start detected — running project analysis` | INFO | `path`, `project_root` |
| `warm start` | DEBUG | `path` |
| `file walk complete` | DEBUG | `path`, `files_found`, `already_indexed` |
| `entities extracted` | DEBUG | `file`, `count` |
| `entity extraction failed` | WARN | `file`, `error` |
| `secret pattern detected` | WARN | `file`, `line`, `pattern`, `severity` |
| `removed deleted file from index` | DEBUG | `file` |

### Secret Pattern Severity Reference

When `secret pattern detected` is logged, the `severity` field is one of `critical`, `high`, or `medium`. The mapping is hardcoded in `src/indexer/file-indexer.ts`:

| `pattern_name` | Detection heuristic | `severity` |
|---|---|---|
| `aws_key` | Matches `AKIA[0-9A-Z]{16}` (AWS access key ID prefix) | `critical` |
| `private_key` | Matches `-----BEGIN ... PRIVATE KEY-----` (PEM header) | `critical` |
| `github_token` | Matches `ghp_[a-zA-Z0-9]{36}` | `critical` |
| `connection_string` | Matches `postgres://`, `mysql://`, `mongodb://`, `redis://` URI patterns | `critical` |
| `generic_api_key` | Matches `api[_-]?key\s*[:=]\s*['"][^'"]{8,}['"]` | `high` |
| `jwt` | Matches three base64url segments separated by dots (JWT structure) | `high` |
| `generic_secret` | Matches `(secret\|password\|token)\s*[:=]\s*['"][^'"]{8,}['"]` | `medium` |

**Severity definitions:**
- `critical` — Likely real credential that grants direct access to a system. Should be rotated immediately.
- `high` — Likely a real secret. High confidence it is not a placeholder.
- `medium` — Pattern matches secret assignment syntax, but could be a test value or placeholder. Investigate before assuming it's real.

**What is never logged:** The actual matched value. Only `pattern_name`, `file`, `line`, and `severity` appear in logs and in the database. See Security Policy section.
| `index sync complete` | INFO | `path`, `files_reindexed`, `files_removed`, `entities_extracted`, `secrets_found` |
| `memory query complete` | DEBUG | `path`, `search_query`, `gotchas_found`, `rules_found`, `task_insights_found` |
| `active security warnings present` | WARN | `path`, `count`, `by_severity` |
| `cold start bootstrap — generating files` | INFO | — |
| `cold start bootstrap complete` | INFO | `feedback_rules_written`, `claude_md_written`, `claude_md_instruction_count` |
| `response assembled` | INFO | `path`, `is_cold_start`, `files_reindexed`, `files_removed`, `entities`, `key_files`, `gotchas`, `rules`, `security_warnings`, `duration_ms` |
| `appended .optima/ to .gitignore` | DEBUG | — |

### `reindex`
The `handleReindex` handler internals.

| Event | Level | Key Fields |
|-------|-------|------------|
| `starting reindex` | INFO | `path` |
| `cleared stale index` | DEBUG | `path`, `files_deleted` |
| `file walk complete` | DEBUG | `files_found` |
| `entities extracted` | DEBUG | `file`, `count` |
| `entity extraction failed` | WARN | `file`, `error` |
| `secret pattern detected` | WARN | `file`, `line`, `pattern`, `severity` |
| `indexing complete` | INFO | `files_indexed`, `entities_found`, `parse_failures`, `secrets_by_severity` |
| `reindex finished` | INFO | `files_indexed`, `entities_found`, `claude_md_regenerated`, `claude_md_instruction_count`, `feedback_rules_written`, `duration_ms` |

### `memorize`
The `handleMemorize` handler internals.

| Event | Level | Key Fields |
|-------|-------|------------|
| `error_fix pipeline` | DEBUG | `original_length`, `sanitized_length`, `normalized_length`, `hash` (12-char prefix), `directory` |
| `duplicate error_fix detected — hit count incremented` | INFO | `existing_id`, `directory` |
| `rule stored` | INFO | `type`, `id`, `directory` |
| `task outcome stored` | INFO | `id`, `outcome`, `has_learnings`, `directory` |
| `post-store generation` | DEBUG | `claude_md_regenerated`, `claude_md_instruction_count`, `feedback_rules_written`, `total_memories` |

### `project-analyzer`
Tech stack and project metadata detection.

| Event | Level | Key Fields |
|-------|-------|------------|
| `project analysis complete` | INFO | `name`, `tech_stack`, `build_command`, `test_command`, `lint_command`, `linters_detected`, `key_dependencies_count`, `has_purpose` |

### `claude-md`
CLAUDE.md generator.

| Event | Level | Key Fields |
|-------|-------|------------|
| `data fetched for generation` | DEBUG | `gotchas`, `arch_rules`, `patterns`, `prefs`, `task_rows`, `security_rows` |
| `instruction budget trimmed` | DEBUG | `before`, `after`, `trimmed` |
| `content unchanged — skipping write` | DEBUG | `path`, `instruction_count` |
| `concurrent write detected — re-reading` | DEBUG | `attempt` |
| `CLAUDE.md concurrent write conflict — skipping` | WARN | `path` |
| `CLAUDE.md written` | INFO | `path`, `instruction_count`, `sections` (array of section names) |

### `feedback-rules`
Feedback rules file generator.

| Event | Level | Key Fields |
|-------|-------|------------|
| `feedback rules unchanged — skipping write` | DEBUG | `path` |
| `feedback rules written` | INFO | `path` |

### `gotcha-ledger`
Error memorization storage.

| Event | Level | Key Fields |
|-------|-------|------------|
| `hit_count incremented` | DEBUG | `directory`, `ids_incremented` |
| `FTS5 query error — falling back to directory-only matching` | WARN | `query`, `error` |

### `rules-store`
Architectural rules and patterns storage.

| Event | Level | Key Fields |
|-------|-------|------------|
| `FTS5 query error — falling back to directory-only matching` | WARN | `query`, `error` |

### `entity-extractor`
Tree-sitter entity extraction from source files.

| Event | Level | Key Fields |
|-------|-------|------------|
| `extraction complete` | DEBUG | `file`, `total_entities`, `by_kind` (object with counts per entity kind) |

### `error-normalizer`
Error sanitization and normalization pipeline.

| Event | Level | Key Fields |
|-------|-------|------------|
| `sanitizeError` | DEBUG | `original_length`, `sanitized_length`, `redactions_applied` |
| `normalizeError` | DEBUG | `input_length`, `normalized_length` |

### `paths`
Path resolution, validation, and ignore filter building.

| Event | Level | Key Fields |
|-------|-------|------------|
| `resolveAndValidate` | DEBUG | `input`, `resolved` |
| `path does not exist` | WARN | `input`, `resolved` |
| `buildIgnoreFilter loaded .gitignore` | DEBUG | `rules` (count of non-comment rules) |
| `no .gitignore found` | DEBUG | `root` |
| `failed to read .gitignore` | WARN | `path` |

### `file-indexer`
File system walker for project indexing.

| Event | Level | Key Fields |
|-------|-------|------------|
| `walk complete` | DEBUG | `root`, `files_found`, `skipped_ignored`, `skipped_binary`, `skipped_symlinks` |

---

## Security Policy: What Is Never Logged

The following are **never** written to logs, regardless of log level:

- **Raw secret values** — when a secret pattern is detected, only the pattern name (`aws_key`, `github_token`, etc.), file path, line number, and severity are logged. The actual value is never logged, even in DEBUG.
- **File contents** — no file content is ever written to logs.
- **Full error hashes** — error dedup hashes are truncated to 12 characters in DEBUG logs (e.g., `"a3f9b2c1d4e5…"`).
- **Sanitized or normalized error text** — lengths are logged, but not the text itself.
- **API keys, tokens, or credentials** from any source.

This policy ensures that log files can be shared for debugging without risk of leaking secrets.

---

## How to Capture Logs

### Claude Code MCP configuration

Add `stderr` capture to your MCP server config:

```json
{
  "mcpServers": {
    "optima": {
      "command": "node",
      "args": ["C:/path/to/Optima/dist/index.js"],
      "env": {
        "OPTIMA_LOG_LEVEL": "DEBUG"
      }
    }
  }
}
```

Claude Code routes MCP server stderr to its own log viewer. Check **Claude Code → Settings → MCP Logs** (or the equivalent in your version) to view the output.

### Tailing logs from a terminal (when running manually)

```bash
# Run server with stderr to a file
OPTIMA_LOG_LEVEL=DEBUG node dist/index.js 2>optima.log

# Tail in another terminal
tail -f optima.log | jq .

# Filter to WARN and ERROR only
tail -f optima.log | jq 'select(.level == "WARN" or .level == "ERROR")'

# Watch tool invocations
tail -f optima.log | jq 'select(.component == "tool")'

# Watch all secret detections
tail -f optima.log | jq 'select(.msg | contains("secret"))'

# Watch cold starts
tail -f optima.log | jq 'select(.msg | contains("cold start"))'
```

---

## Using Logs to Improve Optima

The log structure is designed to answer these questions directly:

### Performance
```bash
# Slowest tool calls (duration_ms > 500)
jq 'select(.component == "tool" and .duration_ms > 500)' optima.log

# Average duration per tool
jq -r 'select(.component == "tool" and .duration_ms) | [.msg, .duration_ms] | @tsv' optima.log
```

### Index health
```bash
# Files being re-indexed on every call (indicates mtime comparison issue)
jq 'select(.component == "get-context" and .files_reindexed > 0) | {path, files_reindexed, duration_ms}' optima.log

# Files removed (deleted but still indexed)
jq 'select(.component == "get-context" and .files_removed > 0)' optima.log
```

### Memory patterns
```bash
# What types of memories are being stored most?
jq 'select(.component == "memorize" and .msg == "optima_memorize completed") | .type' optima.log | sort | uniq -c

# Duplicate error fixes (same error seen multiple times)
jq 'select(.component == "tool" and .duplicate_detected == true)' optima.log

# When does CLAUDE.md actually change vs. stay the same?
jq 'select(.component == "tool" and .msg == "optima_memorize completed") | {type, claude_md_regenerated}' optima.log
```

### Security
```bash
# All secret detections grouped by pattern
jq 'select(.msg | contains("secret pattern detected")) | {file, pattern, severity}' optima.log | sort

# Active security warnings per session
jq 'select(.msg == "active security warnings present") | {path, count, by_severity}' optima.log
```

### CLAUDE.md effectiveness
```bash
# How full is the instruction budget?
jq 'select(.msg == "CLAUDE.md written") | {instruction_count, sections}' optima.log

# How often is the budget trimmed?
jq 'select(.msg == "instruction budget trimmed")' optima.log
```

### Errors and regressions
```bash
# All errors since last deploy
jq 'select(.level == "ERROR")' optima.log

# FTS5 query failures (degrade gracefully but worth fixing)
jq 'select(.msg | contains("FTS5 query error"))' optima.log

# Migration failures
jq 'select(.component == "migrations" and .level == "ERROR")' optima.log
```

---

## Adding New Log Points

When adding new functionality, follow these guidelines:

1. **Import the logger** — `import { logger } from "../utils/logger.js";`
2. **Use the right level** — INFO for successful operations, DEBUG for internal state, WARN for graceful degradation, ERROR for failures.
3. **Never log secrets** — log counts, severities, pattern names, lengths. Never raw values.
4. **Log durations** — calculate `Date.now() - t0` at function entry and include `duration_ms` in completion events.
5. **Use consistent component names** — match the module name (e.g., `"get-context"`, `"claude-md"`, `"migrations"`).
6. **Structured fields over string interpolation** — `{ file: relPath, count: 42 }` not `"indexed 42 entities in " + relPath`.
7. **Test that logs don't break anything** — the logger writes to stderr synchronously; it should never throw. If it did, it would crash the server.

---

## Implementation Notes

### Logger internals (`src/utils/logger.ts`)

The logger is a thin wrapper around `process.stderr.write()`. It:
- Serializes each entry as `JSON.stringify(entry) + "\n"` — newline-delimited JSON (NDJSON), parseable with `jq`
- Checks `LEVELS[level] < LEVELS[activeLevel]` before serializing — zero-cost filtering
- Exposes `setLogLevel()` / `resetLogLevel()` for test isolation (tests run at WARN to keep output clean)
- Never throws — failures in logging must not crash the server

### Why not a logging library?

Optima is a single-binary stdio server with zero external runtime dependencies beyond `better-sqlite3` and `tree-sitter`. Adding a logging library (winston, pino, etc.) would add weight without meaningful benefit. The logger is 60 lines and does exactly what's needed.
