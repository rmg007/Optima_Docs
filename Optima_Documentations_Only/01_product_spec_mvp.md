# 01 Product Spec MVP

## What OptiCode Is

OptiCode is a web-based AI coding agent with built-in project intelligence. It runs locally, uses the Anthropic Messages API with tool_use, provides a React frontend for user interaction, and transparently manages your project's context without generating CLAUDE.md files or relying on Claude Code.

## Architecture & Transport

OptiCode consists of three main systems communicating together:

1. **Frontend (UI)**: A React + Vite SPA showing chat, file viewers, tool execution, and an intelligence dashboard.
2. **Backend (Server)**: A Bun server providing HTTP endpoints for static assets and WebSocket for bidirectional real-time communication (chat streaming, continuous tool updates, permission modals).
3. **Agent Loop (Core)**: The loop calling the Anthropic API, executing tools via local filesystem/bash commands, managing the prompt cache, and utilizing the Intelligence Layer for deep project awareness.

## How It Works

1. User runs `npx opticode` in their project directory.
2. Bun server starts on `localhost:4040` and opens the browser.
3. User enters their `ANTHROPIC_API_KEY` in the settings (stored in `.opticode/config.json`, gitignored).
4. Intelligence layer indexes the project (lazy, triggered on the first interaction).
5. User types a message in the chat UI.
6. **Model Router** classifies the task complexity → selects the appropriate model (Haiku, Sonnet, or Opus).
7. **Context Builder** assembles the system prompt from the Intelligence DB (tech stack, gotchas, rules, entity context).
8. **Agent Loop** calls the Anthropic API with standard tools + the cached system prompt.
9. Tool calls stream to the frontend via WebSocket so the user sees exactly what’s happening in real-time.
10. Destructive operations (like `bash` or `write_file`) require user approval via a permission modal.
11. After completion, the **Session Learner** analyzes the results and updates the Intelligence DB (extracting error/fix patterns, updating the file index).

## Agent Loop & Tool Executors

OptiCode does not rely on MCP tool search. Instead, the Agent Loop directly provides Anthropic tools to the model:

- `read_file`: Read specific lines or whole files.
- `write_file`: Write strings directly to new or existing files (safely truncates/overwrites).
- `edit_file`: Search-and-replace text safely.
- `bash`: Execute arbitrary commands with streaming stdout/stderr.
- `grep`: Pattern match text across files.
- `glob`: Find files by path patterns.
- `list_files`: Show directories/files.

Destructive or unsafe commands trigger a permission request to the frontend via WebSocket. The Agent Loop waits for `PermissionResponse` before execution.

## Model Router Strategy

The router intercepts the task before the Anthropic API is called and selects a model based on complexity signals:

- **Claude 3 Haiku ($1/$5 per MTok)**: Used for trivial commands, renaming, typos, fast context gathering.
- **Claude 3.5 Sonnet ($3/$15 per MTok)**: The default for most feature additions and general work.
- **Claude 3 Opus ($5/$25 per MTok)**: Reserved for heavy architectural rewrites or complex debugging where the prompt mentions highly complex keywords ("refactor", "architect") or massive file trees.

## Prompt Caching Strategy

OptiCode relies heavily on Anthropic's SDK `cache_control` to minimize costs:
- The base System Prompt and project context blocks are bundled into a single message block marked with `cache_control: { type: "ephemeral" }`.
- Cache invalidation happens transparently when the Intelligence Layer updates the file index or gotcha ledger, so context always stays fresh.

## Lazy Indexing Lifecycle

Indexing happens internally inside the Context Builder before an API call:

1. The user asks to fix a bug in `src/auth/login.ts`.
2. OptiCode checks `fs.stat` mtime for the touched files and relevant directories against the `mtime_ms` column in `file_index`.
3. Files with newer mtimes are re-indexed (content hashed, entities re-extracted via Tree-sitter).
4. Unchanged files are skipped.
5. Context is assembled from the fresh index and injected into the System Prompt.

## MVP Excluded (Phase 2+)

Do not implement any of the following:

- Additional language support (Python, Dart, Go, etc.)
- Token optimization and AI-driven relevance scoring
- Adaptive prompt templates based on user style
- Historical edit tracking and code quality scoring
- Dependency and security auditing
- Team collaboration features
- Multi-project support
- Claude Code integration or MCP support

## Success Criteria

1. Running `npx opticode` successfully spins up the React UI and Bun backend.
2. The Agent Loop completes tasks seamlessly, appropriately routing to Haiku/Sonnet/Opus depending on the prompt.
3. The Context Builder accurately injects the latest tech stack overview, gotchas, and architectural rules perfectly.
4. Prompt caching saves significant tokens across multi-turn conversations.
5. A repeated error is resolved faster because OptiCode provides the historical fix from the Intelligence Database directly into the prompt context.