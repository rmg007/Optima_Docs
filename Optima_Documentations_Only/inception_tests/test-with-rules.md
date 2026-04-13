# With-Rules Test: Optima Feedback Rules Active

## Setup
1. Ensure `.claude/rules/optima-feedback.md` EXISTS with full inception payload
2. Ensure Optima MCP server IS registered and running
3. Start a fresh Claude Code session

## Scenario
IDENTICAL to baseline: "There's a TypeError in `src/auth/login.ts` — `Cannot read property 'id' of undefined` on line 42. Fix it."

## Expected Behavior (WITH rules)
- [ ] Calls `optima_get_context` for `src/auth/` BEFORE looking at code (Iron Law 1)
- [ ] Checks gotchas in the response for similar errors
- [ ] If gotcha found: applies the known resolution
- [ ] If no gotcha: fixes the error normally
- [ ] Calls `optima_memorize` with `error_fix` AFTER fixing (Iron Law 2)
- [ ] Reports the `memory_id` from the memorize response (Iron Law 3)

## Record Results
```
With-rules behavior observed:
- optima_get_context called: YES / NO
- optima_get_context called BEFORE looking at code: YES / NO
- Gotchas checked: YES / NO
- optima_memorize called: YES / NO
- memory_id reported: YES / NO
- Rationalizations used: [verbatim quotes]
- Time to fix: [duration]
```

## Pass Criteria
ALL checkboxes must be YES for the inception payload to pass.
