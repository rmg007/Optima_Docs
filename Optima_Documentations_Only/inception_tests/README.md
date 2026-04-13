# Inception Payload Pressure Tests

These test scenarios validate that Optima's inception payload (`.claude/rules/optima-feedback.md`) actually changes Claude Code's behavior. Inspired by Superpowers' writing-skills TDD methodology for documentation.

## Methodology

1. **RED (Baseline):** Run each scenario WITHOUT `optima-feedback.md` present. Document Claude Code's natural behavior.
2. **GREEN (With Payload):** Run the same scenario WITH `optima-feedback.md`. Verify behavior changes.
3. **REFACTOR (Close Loopholes):** If Claude Code found a way to rationalize skipping Optima despite the payload, add a counter-rationalization and re-test.

## Test Files

| Test | What It Tests | Key Behavior |
|---|---|---|
| `test-baseline-no-rules.md` | Behavior without any Optima rules | Does Claude Code call Optima tools naturally? (Expected: No) |
| `test-with-rules.md` | Behavior with optima-feedback.md | Does Claude Code call optima_get_context before coding? (Expected: Yes) |
| `test-pressure-quick-fix.md` | Time pressure rationalization | Does Claude Code skip Optima when told "just fix line 42"? |
| `test-pressure-obvious-error.md` | Obviousness rationalization | Does Claude Code skip gotcha check for "obvious" TypeError? |
| `test-memorize-discipline.md` | Memorization follow-through | Does Claude Code call optima_memorize after fixing an error? |
