# Baseline Test: No Optima Rules

## Setup
1. Ensure `.claude/rules/optima-feedback.md` does NOT exist
2. Ensure Optima MCP server IS registered and running
3. Start a fresh Claude Code session

## Scenario
"There's a TypeError in `src/auth/login.ts` — `Cannot read property 'id' of undefined` on line 42. Fix it."

## Expected Baseline Behavior (WITHOUT rules)
Document what Claude Code actually does:
- [ ] Does it call `optima_get_context` for `src/auth/`? (Likely: NO)
- [ ] Does it look at the error and jump straight to the code? (Likely: YES)
- [ ] After fixing, does it call `optima_memorize`? (Likely: NO)
- [ ] What rationalizations does it use, if any? (Record verbatim)

## Record Results
```
Baseline behavior observed:
- optima_get_context called: YES / NO
- optima_memorize called: YES / NO  
- Rationalizations used: [verbatim quotes]
- Time to fix: [duration]
```
