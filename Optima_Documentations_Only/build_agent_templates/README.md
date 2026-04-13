# Build Agent Templates

These prompt templates are for building Optima using Claude Code's subagent delegation (inspired by Superpowers' subagent-driven-development pattern).

Use these templates when delegating build steps to subagents. Each template is designed for a specific role:

| Template | Role | When to Use |
|---|---|---|
| `implementer-prompt.md` | Implements a build step | Dispatch per build step from Doc 05 |
| `spec-reviewer-prompt.md` | Verifies spec compliance | After implementer reports DONE |
| `quality-reviewer-prompt.md` | Verifies code quality | After spec compliance passes |

## Workflow

1. Read build step from Doc 05
2. Dispatch implementer subagent with full task text
3. If implementer reports DONE → dispatch spec reviewer
4. If spec reviewer approves → dispatch quality reviewer
5. If quality reviewer approves → mark step complete, proceed to next
6. If any reviewer finds issues → implementer fixes → re-review
