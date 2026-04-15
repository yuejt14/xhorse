---
name: xhorse-status
description: Show xhorse session status.
---

# /xhorse-status

Show status of the xhorse harness.

## Instructions

1. Read `.xhorse/status.json`. If it doesn't exist: "No active xhorse session. Run `/xhorse <prompt>` to start one."

2. Display session info:
   - Session ID, branch, base branch
   - Mode: continuous or sprints
   - Phase: planning, generating, sprinting, complete, or stopped

3. Display mode-specific progress:

   **Continuous mode** (`mode` is `"continuous"`):
   - Phase: {{phase}}
   - Iteration: {{generation.iteration}} / {{max_iterations}}
   - Best score: {{generation.best_score}} (or "none yet")
   - Show latest evaluation report path if any exist in `.xhorse/evaluations/`

   **Sprint mode** (`mode` is `"sprints"` or absent):
   - Phase: {{phase}}
   - Current sprint: {{current_sprint.number}} (iteration {{current_sprint.iteration}} / {{max_iterations_per_sprint}})
   - Completed sprints: list from `completed_sprints` with verdicts
   - Best score (current sprint): {{current_sprint.best_score}} (or "none yet")

4. Show tech stack info: detected stack, test/build/lint commands.

5. If frontend testing is configured, show: MCP server name, dev server command, dev server URL.
