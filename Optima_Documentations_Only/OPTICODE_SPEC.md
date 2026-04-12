# OptiCode Product Specification

**Version:** 0.2.0 (Web Agent MVP) | **Date:** April 12, 2026 | **Status:** Implementation Phase

---

## 1. Problem

Developers face context fragmentation when using AI models. Past errors are forgotten, architectural conventions are ignored over multiple sessions, and context windows are inefficiently filled with irrelevant files. Existing tools lack a cohesive memory that continuously learns and injects exact context without bloat.

## 2. Solution

OptiCode is a web-based AI coding agent with built-in project intelligence. It serves a React frontend for user interaction while running a robust Bun backend that manages project context, an Anthropic-powered agent loop, and a local SQLite database that logs architectural decisions and error "gotchas". By controlling the agent loop itself, OptiCode injects perfect context before every request and caches it efficiently to lower API costs.

## 3. Philosophy

- **Visible and Interactive.** The developer has a premium, modern chat interface to interact with the model and watch file intelligence.
- **Portability & Ownership.** Intelligence lives locally in `.opticode/opticode.db`. No vendor lock-in.
- **Agent Loop Mastery.** Instead of plugging into an external IDE, OptiCode owns the agent loop. This allows custom model routing, deep prompt caching, and custom tool executions.
- **Continuous Learning.** Post-session, OptiCode studies the tools executed and outcome logs to automatically append knowledge to the project's Gotcha Ledger or Architectural Rules store.

## 4. Architecture

```
Developer ↔ React UI (Web Browser) ↔ Bun Server ↔ Anthropic API (via Messages + Tools)
                                        │
                                        ├── Intelligence Layer (AST Extractor, Gotcha Ledger)
                                        ├── Tool Executors (read_file, write_file, bash)
                                        └── SQLite Database (.opticode/opticode.db)
```

### Prompt Caching and Context Injection
OptiCode uses the `cache_control` parameter natively in the Anthropic SDK. The entire project context (learned gotchas, tech stack, and lazily indexed file chunks) is placed in system messages marked with `cache_control: { type: "ephemeral" }`. Cache is invalidated naturally when local sources change or new intelligence is established.

## 5. Tool Surface

OptiCode defines tools that map Anthropic Requests directly to the underlying OS:
- `read_file`
- `write_file`
- `edit_file`
- `bash`
- `grep`
- `glob`
- `list_files`

All destructive tools (`write_file`, `edit_file`, `bash`) stream a `permission_request` to the React UI via WebSocket and wait for a `permission_response` before proceeding.

## 6. Model Router

OptiCode categorizes tasks prior to loop execution:
- **Claude 3 Haiku**: Fast, cheap, used for context lookups and renaming.
- **Claude 3.5 Sonnet**: Core application development, standard agenting.
- **Claude 3 Opus**: Complex logic refactors or dense bug hunts involving multi-file systems.

## 7. Lazy Indexing Lifecycle

1. A user targets a specific module or asks to fix a specific bug in `src/`.
2. OptiCode checks `fs.stat` mtimes for that directory against `file_index`.
3. Changed files are re-read and entities re-extracted by Tree-sitter.
4. Intelligent context (gotchas scoped to that directory) is pulled.
5. System prompt is bundled up with current intelligence and dispatched to Anthropic API.

## 8. Web Application Design

The React frontend (`/frontend`) must be highly premium and aesthetically jaw-dropping:
- **Stack**: React, Vite, Tailwind CSS, Lucide React icons, shadcn/ui.
- **Routing**: `react-router-dom` with `/chat`, `/context`, `/settings`.
- **Styling**: Sleek dark mode by default, vibrant highlights, glassmorphism, smooth CSS keyframe animations, rich typography (e.g., Inter/Outfit).

## 9. Tech Stack Summary

| Component | Choice |
| --- | --- |
| Server | Bun (`bun run index.ts`) |
| Client | React, Vite, Tailwind |
| LLM | `@anthropic-ai/sdk` |
| Database | SQLite (`better-sqlite3`), Drizzle ORM |
| Protocol | HTTP for static, WebSocket for real-time Agent Loop |
| Indexer | Tree-sitter |

## 10. MVP Exclusions

- No cloud sync functionality.
- No multi-project intelligence aggregation.
- No other LLM providers (strict Anthropic integration).
- No MCP SDK usage (OptiCode is an independent Agent, not an MCP server).

## 11. Security Profile

- **DO NOT** execute bash or write_file without user approval (except when `SafeToAutoRun` allows in specific configurations).
- **DO NOT** index `.env`, `node_modules`, `.git`, or other sensitive/unnecessary directories.