# 03 Agent and Tool Specs

OptiCode relies entirely on the Anthropic SDK (`@anthropic-ai/sdk`) for tool usage, model generation, and caching.

## Agent Loop Architecture

1. **Initialization**: When the user provides a prompt, the system first updates the Intelligence Layer indexes and retrieves relevant context.
2. **Context Injection**: The context is embedded as early messages / system prompts, tagged with `cache_control: { type: "ephemeral" }`.
3. **Execution Loop**:
    - The server calls the Anthropic Messages API, passing the user messages and the standard tools.
    - If the model returns a text response, it streams to the UI.
    - If the model requests a tool call, the server pauses parsing, streams the tool request to the UI, executes the tool, and returns the result as a new message.
    - If a tool requires user permission (e.g. `bash` or `write_file`), the server suspends execution, waits for a WebSocket approval from the client, and then either executes or blocks the tool based on user input.
4. **Completion**: The model loop exits when no further tool calls are generated. 
5. **Session Learning**: Post-completion, the Session Learner analyzes the outcome and writes to the Gotcha Ledger or updates the file indexes.

## Model Router Strategy

Before the Agent Loop begins, the task is classified into three complexity tiers:

- **Claude 3 Haiku (`claude-3-haiku-20240307`)**: Used for minor typo fixes, formatting, gathering quick file listings, or low-context questions.
- **Claude 3.5 Sonnet (`claude-3-5-sonnet-latest`)**: The standard model for feature implementations, debugging, and most agentic tasks.
- **Claude 3 Opus (`claude-3-opus-latest`)**: Reserved for broad architectural overhauls, deep-seated bugs affecting multiple systems, or logic crossing significantly complex domains.

## Prompt Caching Strategy

OptiCode heavily utilizes prompt caching to minimize costs and latency:

- The initial System Prompt containing the tech stack, instructions, and error ledgers is marked with `cache_control: { type: "ephemeral" }`.
- Consecutive messages containing large file read outputs or bash outputs > 1000 tokens are also marked with cache controls.
- Cache invalidation occurs automatically when the system prompt contents change, which typically happens when the Intelligence Layer re-indexes files or updates gotchas.

## Tool Definitions (JSON Schemas)

OptiCode passes these directly into the `tools` array for the Anthropic API.

### `read_file`
Read completely or partially from a file.
```json
{
  "name": "read_file",
  "description": "Reads the contents of a file or a specific line range.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "startLine": { "type": "number", "description": "1-indexed optional start line" },
      "endLine": { "type": "number", "description": "1-indexed optional end line" }
    },
    "required": ["path"]
  }
}
```

### `write_file`
Create a new file or completely overwrite an existing one. Requires User Approval.
```json
{
  "name": "write_file",
  "description": "Writes absolute content to a file, overwriting if it exists.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["path", "content"]
  }
}
```

### `edit_file`
Replaces a specific block of code within an existing file.
```json
{
  "name": "edit_file",
  "description": "Edits an existing file by replacing a strict target block with replacement content.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "targetContent": { "type": "string", "description": "Exact character-for-character sequence to replace" },
      "replacementContent": { "type": "string" }
    },
    "required": ["path", "targetContent", "replacementContent"]
  }
}
```

### `bash`
Execute terminal commands. Requires User Approval.
```json
{
  "name": "bash",
  "description": "Executes standard terminal bash/powershell commands.",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": { "type": "string" }
    },
    "required": ["command"]
  }
}
```

### `list_files`
List directory entries.
```json
{
  "name": "list_files",
  "description": "List the files and subdirectories inside a directory.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "description": "Directory to list, defaulting to project root" }
    }
  }
}
```

### `grep`
Search across files for text patterns.
```json
{
  "name": "grep",
  "description": "Search literal or regex patterns across the codebase.",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": { "type": "string" },
      "includePaths": { "type": "array", "items": { "type": "string" } }
    },
    "required": ["pattern"]
  }
}
```

### `glob`
Find multiple files by glob patterns.
```json
{
  "name": "glob",
  "description": "Search for file paths matching a given glob pattern.",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": { "type": "string" }
    },
    "required": ["pattern"]
  }
}
```

## Tool Restrictions & Security

- `read_file`, `list_files`, `grep`, and `glob` execute immediately without permissions.
- `bash` and `write_file` are inherently dangerous operations that prompt the user.
- If the model tries to read sensitive Claude state (like `~/.claude/session-env/`), the tool interceptor immediately blocks it and returns a simulated "Path not found" to the model or instructs it to avoid reading those directories.