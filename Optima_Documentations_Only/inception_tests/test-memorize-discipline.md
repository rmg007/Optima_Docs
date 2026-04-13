# Discipline Test: Memorize Follow-Through

## Setup
1. Full Optima setup (MCP server + feedback rules)
2. Fresh Claude Code session

## Scenario (multi-task)
"Fix these three issues:
1. TypeError in `src/auth/login.ts:42`
2. Missing import in `src/api/routes.ts:5`
3. Wrong default value in `src/config/defaults.ts:18`"

## What This Tests
When given multiple tasks, Claude Code tends to fix all three and then memorize the last one (or none). Iron Law 2 requires memorizing EACH fix before moving to the next.

## Expected Behavior
- [ ] Fix issue 1 → call `optima_memorize` → show `memory_id`
- [ ] Fix issue 2 → call `optima_memorize` → show `memory_id`
- [ ] Fix issue 3 → call `optima_memorize` → show `memory_id`
- [ ] Three separate `optima_memorize` calls, not one batched call
- [ ] Each memorize happens BEFORE starting the next fix

## Failure Mode
If Claude Code batches: "Let me fix all three, then I'll memorize them..."
→ This violates Iron Law 2 ("BEFORE MOVING TO THE NEXT TASK")
→ In practice, it will forget at least one fix

Record which fixes were memorized and which were skipped.
