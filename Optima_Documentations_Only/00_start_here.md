# 00 Start Here

**Read this file first. Then read the other four documents in order.**

## North Star

A web-based AI coding agent with built-in project intelligence that indexes your project, remembers what errors you've hit and how you fixed them, and structurally injects context into the Anthropic API to make every session smarter than the last.

## Reading Order

| Order | File | Purpose |
| --- | --- | --- |
| 1 | `00_start_here.md` | This file. Rules, tech stack, constraints. |
| 2 | `01_product_spec_mvp.md` | What OptiCode is, how it works, what to build, project structure. |
| 3 | `02_data_model_and_schema.md` | Drizzle schemas, TypeScript interfaces, SQLite tables. |
| 4 | `03_AGENT_AND_TOOL_SPECS.md` | Anthropic tool definitions, agent loop, and caching. |
| 5 | `04_CONTEXT_INJECTION_AND_LEARNING.md` | Exact rules for context injection and post-session learning. |

## Tech Stack (Locked)

| Component | Choice | Command |
| --- | --- | --- |
| Runtime | Bun | `bun run`, `bun test`, `bunx` |
| Language | TypeScript | Strict mode. No `any`. |
| LLM Integration | `@anthropic-ai/sdk` | Direct Messages API calls with Tool Use. |
| Frontend | React + Vite + Tailwind | Served by Bun. |
| Database | SQLite via `better-sqlite3` | Single file at `.opticode/opticode.db`. Uses better-sqlite3 for runtime portability. |
| ORM | Drizzle ORM (`drizzle-orm`  • `drizzle-kit`) | Typed queries. SQLite dialect. |
| AST Parsing | `tree-sitter`  • `tree-sitter-typescript` | For entity extraction. |
| Gitignore Parsing | `ignore` (npm, ^7.0.0) | Correct gitignore semantics for file indexing exclusions. |
| Bundler | `tsup` (Backend) / `vite` (Frontend) | |
| Test Runner | Vitest | `bun run test` via vitest. |

## Ground Rules

1. **No file exceeds 300 lines.** Split into separate modules if approaching the limit.
2. **Copy verbatim.** When a spec provides a fenced TypeScript, SQL, or Zod block, reproduce it byte-for-byte.
3. **The agent loop calls the Anthropic Messages API directly.**
4. **Model routing is a first-class component.** Switch between Haiku, Sonnet, and Opus depending on the task mapping.
5. **The frontend is a React SPA served by the Bun server.**
6. **All cross-module inputs validated with Zod.** No exceptions.
7. **All file I/O must be async.** Never use synchronous fs methods in the hot path. (Exception: `better-sqlite3` database queries are intentionally synchronous by design).
8. **Tests are mandatory.** Every module ships with tests. Minimum 80% coverage.
9. **Errors use a closed taxonomy.** Every thrown error must use one of the error codes defined in the specification documents. Use `OptiCodeError`.
10. **Paths are always forward-slash normalized.** All file paths stored in SQLite use forward slashes (`src/utils/foo.ts`), regardless of OS. Normalize on ingestion using `path.replace(/\\/g, '/')`. Symlinks are NOT followed - skip silently.

## Project Structure

```
opticode/
├── package.json
├── tsconfig.json
├── vite.config.ts              # Frontend build config
├── tsup.config.ts              # Backend build config
├── vitest.config.ts
├── .env.example                # ANTHROPIC_API_KEY=sk-ant-...
├── src/
│   ├── index.ts                # Entry point: starts Bun server
│   ├── agent/                  # Core agent loop, router, cache, and Anthropic tools
│   ├── tools/                  # Executors: read, write, edit, bash, grep, glob, permissions
│   ├── server/                 # Bun HTTP server, WebSocket, and protocol types
│   ├── intelligence/           # Indexer, Memory (Gotchas, Rules), Context Builder, Learner
│   ├── db/                     # Connection, schemas, migrations
│   └── utils/                  # Hasher, paths, errors
├── frontend/                   # React app, chatting UI, tree viewer, intelligence dashboard
│   ├── index.html
│   ├── src/
│   └── tailwind.config.ts
├── test/
└── .opticode/                  # Per-project data directory generated at runtime
```

## Claude Code State Awareness

OptiCode operates alongside the user's workspace, and understanding Claude Code's internal state architecture is still critical to avoid conflicts or indexing private/global agent state if a user happens to use it.

**Claude Code's local state lives in two places:**

- `~/.claude/` — Global user state (session transcripts, file-history, caches, global settings). NEVER index this.
- `<project>/.claude/` — Project-level config (settings.json, agents/, commands/, skills/, rules/).

**Security-sensitive paths OptiCode must NEVER index:**

- `~/.claude/session-env/` — contains plaintext environment variables (API keys, tokens)
- `~/.claude/projects/` — full session transcripts in JSONL
- `~/.claude/file-history/` — before/after file edit snapshots
- `CLAUDE.local.md` — developer's personal instruction overrides. OptiCode must never read, modify, or generate this file.
- `.claude/settings.local.json` — personal permission overrides (gitignored)
- Any `.env` file or `**/secrets/**` path

## DO NOT

- **DO NOT use the Claude Agent SDK.** It requires separate API billing. Rely ONLY on the Anthropic SDK (`@anthropic-ai/sdk`).
- **DO NOT implement MCP transport.** OptiCode is not an MCP server. It manages the agent loop purely via HTTP/WebSockets.
- **DO NOT generate CLAUDE.md files.** OptiCode injects context directly into the Anthropic API's system prompt to avoid cluttering workspaces.
- **DO NOT implement a file watcher.** OptiCode uses lazy indexing via `fs.stat` mtime checks inside context building. No background processes, no `chokidar`, no `fs.watch`.
- **DO NOT index Claude Code's internal state.** Never read or index `~/.claude/`, `.claude/session-env/`, `.claude/projects/`, or `.claude/file-history/`.
- **DO NOT use `npm`, `npx`, or `node` commands.** Bun only.
- **DO NOT use `bun:sqlite` directly.** Use `better-sqlite3` for runtime portability.

## When Stuck

If a spec is ambiguous or two documents seem to contradict each other:

1. Trust `02_data_model_and_schema.md` for data shapes.
2. Trust `03_AGENT_AND_TOOL_SPECS.md` for tool behavior and agent loop.
3. Trust `04_CONTEXT_INJECTION_AND_LEARNING.md` for context generation logic.
4. If still unclear, stop and ask. Do not improvise.