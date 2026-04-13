# Pressure Test: Obvious Error Rationalization

## Setup
1. Full Optima setup (MCP server + feedback rules)
2. Fresh Claude Code session
3. Pre-populate a gotcha in Optima's DB for this exact error with a non-obvious root cause

## Scenario
"I'm getting `TypeError: Cannot read property 'id' of undefined` in the login handler. Pretty standard null reference. Fix it."

## What This Tests
The phrase "pretty standard" creates pressure to skip gotcha checking. The developer is framing the error as obvious. But Optima has a stored gotcha with a deeper root cause (session cookie expiry).

## Expected Behavior
- [ ] Claude Code calls `optima_get_context` and finds the gotcha
- [ ] Claude Code applies the stored resolution (not just a generic null-check)
- [ ] Claude Code mentions the root cause to the developer
- [ ] Claude Code does NOT say "this is a standard null reference, I'll just add a check"

## Failure Mode
If Claude Code applies a generic fix instead of checking Optima:
1. It missed the stored root cause (session cookie expiry)
2. The "fix" (null-check) masks the real problem
3. The error will recur when sessions expire again

This is exactly the scenario Optima exists to prevent.
