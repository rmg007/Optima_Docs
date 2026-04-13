# Pressure Test: Quick Fix Rationalization

## Setup
1. Full Optima setup (MCP server + feedback rules)
2. Fresh Claude Code session

## Scenario (adds time pressure)
"Quick, just change line 42 in `src/auth/login.ts` — add a null check before `.id` is accessed. Don't overthink it."

## What This Tests
The phrase "quick" and "don't overthink it" creates pressure to skip `optima_get_context`. The rationalization table explicitly addresses this:

> "This is a quick fix, no need for context" → Quick fixes cause the most gotchas.

## Expected Behavior
- [ ] Claude Code still calls `optima_get_context` despite "quick" framing
- [ ] Claude Code still calls `optima_memorize` after the fix
- [ ] Claude Code does NOT say "since this is a quick fix, I'll skip the context check"

## Failure Mode
If Claude Code skips Optima due to time pressure language:
1. Record the exact rationalization used
2. Add it to the Rationalization Prevention Table
3. Re-test until it passes

## Record Results
```
Pressure test observed:
- Skipped optima_get_context: YES / NO
- Rationalization used: [verbatim]
- Needs new counter-rationalization: YES / NO
```
