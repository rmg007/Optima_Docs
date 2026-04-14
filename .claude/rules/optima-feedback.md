---
paths: "**/*"
---

**Violating the letter of these rules is violating the spirit of these rules.**

If you find yourself arguing "the spirit of Optima is understanding the codebase, and I already understand it" — that IS the rationalization these rules exist to prevent. Call the tools. Every time. No exceptions.

## Instruction Priority

When instructions conflict, follow this priority order:

1. **Developer's explicit instructions** (verbal commands, CLAUDE.md content OUTSIDE Optima markers) — highest priority
2. **Optima-generated CLAUDE.md content** (inside `<!-- OPTIMA:START -->` markers) — project-learned context
3. **This feedback rules file** (`.claude/rules/optima-feedback.md`) — tool usage instructions
4. **Claude Code's default behavior** — lowest priority

**Examples:**
- Developer says "don't use Optima for this task" → Skip Optima (priority 1 overrides all)
- Developer's CLAUDE.md says "use semicolons" but Optima says "no semicolons" → Follow developer's CLAUDE.md (priority 1 > priority 2)
- Optima's CLAUDE.md says "use ESLint" but developer hasn't mentioned linting → Follow Optima (priority 2 applies)
- This file says "always call optima_get_context" but developer says "just fix this one line" → Call optima_get_context anyway (priority 3 instruction to call tools is not overridden by a task description — "just fix this" is a task, not an instruction to skip tools)

**The user is always in control.** If the developer explicitly says to stop using Optima, stop. But a casual "just fix it" is not an instruction to skip Optima — it's a task description.

## Optima Memory Rules

These rules ensure Optima learns from every session to improve future ones.

### After Fixing Errors

When you successfully resolve a compiler error, runtime exception, or failing test:
- You MUST call the `optima_memorize` tool before concluding your response.
- Include:
  - `type`: `"error_fix"`
  - `error`: The original error message (copy the key part, not the full stack trace)
  - `resolution`: A concise summary of what fixed it
  - `files`: Array of file paths that were modified
  - `directory`: The primary directory involved

### After Making Changes

After modifying code, always verify your work before reporting completion:
- Run the project's test command (if known from `optima_get_context`)
- Run the project's lint command (if known)
- If tests or lint fail, fix the issues before calling `optima_memorize`
- Only log the fix after verification passes

### After Architectural Decisions

When you make a significant design choice (new pattern, structural decision, technology choice):
- Call `optima_memorize` with:
  - `type`: `"architectural_rule"`
  - `rule`: The decision in one sentence
  - `rationale`: Why this approach was chosen
  - `directory`: The directory scope (or omit for project-wide)

### At Task Start

When beginning any task (bug fix, feature, refactor, test, review):
- Call `optima_get_context` with the relevant directory path
- **When encountering or debugging an error:** Pass the error message as `search_query` to enable cross-directory gotcha discovery. Example: `optima_get_context({ path: "src/api/", search_query: "ECONNREFUSED timeout" })`. This finds relevant gotchas even if they were stored against a different directory.
- **When starting a feature or refactor:** Pass descriptive keywords as `search_query`. Example: `optima_get_context({ path: "src/auth/", search_query: "JWT token refresh" })`. This surfaces rules and patterns from related areas.
- Use the returned context (gotchas, rules, entities) to inform your approach
- Pay special attention to gotchas — they contain solutions to previously encountered errors

### After Establishing Patterns

When you establish a reusable code pattern or convention:
- Call `optima_memorize` with:
  - `type`: `"pattern"`
  - `pattern`: Description of the pattern
  - `example`: A brief code example
  - `directory`: Where this pattern applies

### After Completing a Task

When you finish a task (feature, bug fix, refactor, etc.) — regardless of whether it succeeded, partially succeeded, or was abandoned:
- Call `optima_memorize` with:
  - `type`: `"task_outcome"`
  - `task`: One-sentence description of what was attempted
  - `outcome`: `"success"`, `"partial"`, or `"abandoned"`
  - `learnings`: The key insight — what worked, what didn't, what was surprising. **This is the most valuable field.** Skip it only if the task was truly routine.
  - `duration_hint`: Rough time estimate (e.g., `"~45min"`, `"~2hr"`) — helps future estimation
  - `files`: Array of files that were modified
  - `directory`: The primary directory scope

**Why this matters:** Error fixes capture reactive knowledge ("this broke, here's the fix"). Task outcomes capture proactive knowledge ("this approach worked/didn't work for this kind of task"). Future sessions can use these insights to choose better approaches and estimate effort.

**When NOT to log:** Trivial tasks (rename a variable, fix a typo) don't need task outcomes. The threshold: would a future developer benefit from knowing how this task went?

## Iron Laws

These rules are non-negotiable. There are no exceptions. "Violating the letter of these rules is violating the spirit of these rules."

### Iron Law 1: Context Before Code

```
CALL optima_get_context BEFORE WRITING OR MODIFYING ANY CODE IN A DIRECTORY YOU HAVEN'T QUERIED THIS SESSION.
```

No exceptions. Not for "quick fixes." Not for "obvious changes." Not because "I already read CLAUDE.md." The tool gives you directory-specific context that CLAUDE.md cannot contain.

### Iron Law 2: Memorize Before Moving On

```
IF YOU FIXED AN ERROR, CALL optima_memorize WITH TYPE error_fix BEFORE MOVING TO THE NEXT TASK.
```

Not "after I finish everything." Not "I'll batch them." NOW. The moment the error is resolved and tests pass, store it. Future sessions depend on this.

### Iron Law 3: Evidence Before Claims

```
DO NOT CLAIM "CLAUDE.md UPDATED" OR "GOTCHA STORED" WITHOUT SHOWING THE TOOL'S RESPONSE.
```

If you called `optima_memorize`, show the `memory_id` from the response. If CLAUDE.md was regenerated, state whether the content hash changed or the write was skipped. No "should be updated" or "likely stored."

## Red Flags — STOP and Call Optima

If you catch yourself thinking any of these, STOP. You are about to skip Optima and you should not.

**Before writing code:**
- "I know what this error means" → STOP. Check gotchas first. A previous session may have spent 30 minutes debugging this exact error.
- "This directory is simple, no need for context" → STOP. Call `optima_get_context`. Simple directories still have architectural rules and patterns.
- "I already read CLAUDE.md, I have the context" → STOP. CLAUDE.md is a summary. The tool returns directory-specific entities, gotchas, and recent changes. Pass `search_query` to also search across directories.
- "This error is from a different part of the codebase, Optima won't have it" → STOP. Pass the error text as `search_query`. FTS5 search finds gotchas across ALL directories, not just the one you're querying.
- "Let me just look at the code first" → STOP. Looking at code without Optima context means missing the gotchas that explain WHY the code looks that way.

**After fixing an error:**
- "I'll memorize this later" → STOP. Call `optima_memorize` NOW. You will forget. You will move on. The fix will be lost.
- "This fix is too simple to memorize" → STOP. Simple fixes are the ones developers repeat most often. Store it.
- "I'll batch all my memorizations at the end" → STOP. You'll forget half of them. Memorize each fix immediately.

**After memorizing:**
- "CLAUDE.md should be updated now" → STOP. Check the tool response. Did `claude_md_regenerated` return true? Did the content hash change? Evidence, not assumptions.

**General:**
- "This is different because..." → STOP. It is not different. Follow the Iron Laws.
- "I'm being pragmatic by skipping this" → STOP. Skipping Optima is not pragmatic. It's repeating work that previous sessions already did.
- "Just this once" → STOP. There is no "just this once." The rules apply every time.

## Rationalizations for Skipping Optima — All Invalid

Claude Code WILL try to skip calling Optima's tools. These are the excuses it will use and why they are all invalid:

| Claude Code's Excuse | Reality |
|---|---|
| "I already know this codebase" | You don't. Your context resets every session. Optima remembers across sessions. Call the tool. |
| "This is a quick fix, no need for context" | Quick fixes cause the most gotchas. Past developers stored fixes for exactly this scenario. Context takes <200ms. |
| "I'll memorize after I'm done" | You'll forget. You'll move to the next task and the fix will be lost. Memorize immediately when you resolve the error. |
| "The error is obvious, no need to check gotchas" | Obvious errors have non-obvious root causes. A previous session already debugged this for 30 minutes. Check before you repeat that work. |
| "optima_get_context is too slow" | It's <150ms when the index is fresh. You spend more time generating excuses than the tool takes to run. |
| "I'll just check CLAUDE.md instead of calling the tool" | CLAUDE.md is a project-wide summary with a 35-instruction cap. The tool returns directory-specific context, relevant gotchas, entity lists, and recent changes that CLAUDE.md cannot contain. |
| "I already called it for a parent directory" | Directory context is scoped. `src/auth/` has different gotchas than `src/`. Call it for the directory you're actually working in. |
| "This is a new file, there's no context to get" | The directory still has rules, patterns, and gotchas from adjacent files. The project still has architectural rules. Call the tool. |
| "I'm just reading code, not modifying it" | Reading without context means missing the gotchas that explain WHY the code looks that way. Call the tool. |
| "The developer didn't ask me to use Optima" | The developer installed Optima and configured it as an MCP server. That IS the instruction. You don't need per-task permission. |

## Tool Selection Decision Tree

When should you call which Optima tool?

```
Starting any task in a directory?
│
├── Have you called optima_get_context for this directory in this session?
│   ├── NO → Call optima_get_context first (Iron Law 1)
│   └── YES → Proceed with task
│
├── Just fixed an error?
│   └── YES → Call optima_memorize with type "error_fix" (Iron Law 2)
│
├── Discovered a codebase convention?
│   └── YES → Call optima_memorize with type "architectural_rule"
│
├── Noticed a recurring code pattern?
│   └── YES → Call optima_memorize with type "pattern"
│
├── Developer stated a preference?
│   └── YES → Call optima_memorize with type "preference"
│
├── Just finished a non-trivial task?
│   └── YES → Call optima_memorize with type "task_outcome"
│
├── Major refactor, dependency change, or directory restructure?
│   └── YES → Call optima_reindex
│
└── optima_get_context results seem stale (files exist that aren't in the index)?
    └── YES → Call optima_reindex
```

**When in doubt:** Call `optima_get_context`. It takes <200ms. The cost of calling unnecessarily is negligible. The cost of NOT calling is repeating work a previous session already did.
