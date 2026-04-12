# 04 Context Injection and Learning

OptiCode handles context internally, rather than generating files for an external tool like Claude Code. This document outlines how OptiCode constructs the system prompt and learns from completed sessions.

## Context Injection Algorithm

Before every Anthropic API call, the Context Builder intercepts the request and injects a dynamic system prompt.

1. **Lazy Indexing Check**: OptiCode checks the `file_index` and `mtime` for relevant directories.
2. **Retrieve Intelligence**: Queries the `.opticode/opticode.db` to pull project metadata, architecture rules, known gotchas, code patterns, and preferences relevant to the working directories.
3. **Budget Constraint**: Do not overwhelm the Claude model with instructions. The system prompt should inject a maximum of **30-40 instructions** regarding the project's intelligence.
   - `project_overview`: ~5 instructions (name, stack, commands)
   - `architecture_rules`: Cap at 10 rules.
   - `known_gotchas`: Cap at 10 gotchas.
   - `patterns`: Cap at 5 patterns.
   - `preferences`: Cap at 5 preferences.
4. **Construct System Message**: The retrieved intelligence is beautifully formatted into a hidden system prompt that uses `<project_overview>`, `<architecture_rules>`, `<known_gotchas>`, `<patterns>`, and `<preferences>` XML tags to structure the details.
5. **Caching**: This heavy system message is cached using Anthropic's `cache_control: { type: "ephemeral" }` mechanism.

### The Necessity Test
Every rule injected must pass a necessity test:
- Gotchas with `hit_count < 2` that haven't triggered in 30+ days: **exclude** from context.
- Rules that duplicate what the project's linter/formatter enforces: **exclude entirely**.
- Architecture rules that are obvious (e.g., "uses React"): **exclude**.
- Patterns with only 1 occurrence and no `hit_count`: **exclude** until confirmed by reuse.

## Session Learning Algorithm

OptiCode is designed to get smarter independently through post-session review. Once an Agent Loop finishes a task successfully, the **Session Learner** asynchronously kicks in.

### 1. Error Fixes (Gotchas)
OptiCode scans the terminal execution logs or tool results. If it notices a compiler error, failing test, or runtime exception that was subsequently fixed:
- It normalizes the error (stripping line numbers, keys, etc.).
- It extracts a succinct summary of the resolution.
- It writes to the `gotchas` table, associating it to the relevant `directory` and `files`.

### 2. Architectural Decisions
If OptiCode identifies that a new library was installed, a new folder structure was established, or a distinct code pattern was uniquely applied across multiple files:
- It records a new `architectural_rule` or `pattern` to the `rules` table.

### 3. Developer Preferences
If the user corrects the agent (e.g., "Use fetch instead of axios here" or "I prefer styled-components"):
- It extracts the preference and logs it to the `rules` table under the `preference` type.

By automatically running this learner asynchronously at the end of every task, the context builder has a richer dataset for the next interaction, effectively making every session smarter than the last.