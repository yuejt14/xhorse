---
name: xhorse-sprint
description: Run one sprint cycle — create contract, generate implementation, run pre-checks, evaluate, and handle the decision gate.
argument-hint:
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Agent
---

# /xhorse-sprint

Run a single sprint cycle of the xhorse harness. This includes contract creation, code generation, pre-checks, evaluation, and the decision gate.

## Instructions

### Step 0: Validate State

1. Read `.xhorse/status.json`. If it doesn't exist: "No active session. Run `/xhorse-plan <prompt>` first."
2. If `phase` is `"planning"`: "Planning not complete. Run `/xhorse-plan` to finish planning."
3. If `phase` is `"complete"`: "All sprints are done. Nothing to run."
4. Verify the xhorse branch exists and is checked out: `git branch --show-current`
5. Read `.xhorse/spec.md` to understand the full scope.
6. Read `.xhorse/config.json` for iteration limits and model settings.
7. **Frontend testing setup** (only if `config.json` contains a `frontend_testing` object):
   a. Validate that all three fields (`mcp_server_name`, `dev_server_cmd`, `dev_server_url`) are non-empty strings. If any are missing, halt: "Frontend testing config incomplete. Missing: {{list}}. Fix `.xhorse/config.json` or remove `frontend_testing`."
   b. Probe MCP: call `mcp__{{mcp_server_name}}__browser_navigate` with `"about:blank"`. If unavailable, halt: "MCP server '{{mcp_server_name}}' not available. Register it in your Claude Code settings, or remove `frontend_testing` from config."
   c. Start the dev server if not already reachable:
      - Check: `curl -sf -o /dev/null {{dev_server_url}}`
      - If not reachable: `Bash(command: "{{dev_server_cmd}}", run_in_background: true)` — do NOT use `<cmd> &`. Then poll `curl` every 1s for up to 30 seconds.
      - If startup fails, halt: "Dev server failed to start. Command: `{{dev_server_cmd}}`, URL: `{{dev_server_url}}`."
   d. **Important**: Agents must NEVER start or stop the dev server.

### Step 1: Create Sprint Contract

Read the spec's "Sprint Decomposition" section. Find the next sprint to implement based on `current_sprint.number` in status.json.

Create `.xhorse/current-sprint.md` from the spec's sprint breakdown:
- Copy the sprint goal, scope, and acceptance criteria from the spec
- Set "Starting commit" to the current HEAD SHA
- Set "Branch" to the xhorse branch name
- List previously completed sprints for context
- Fill in "Expected Files" based on the scope (list files you expect to be created/modified)
- Leave the "Self-Assessment" section empty for the generator to fill

Commit: `git add .xhorse/current-sprint.md && git commit -m "xhorse sprint N: contract"`

### Step 2: Generate

Spawn the generator agent:

```
Agent({
  description: "Implement sprint N",
  prompt: "You are the Generator agent. Read your instructions at agents/generator.md.

TOOL RESTRICTION: You have access to Read, Write, Glob, Grep, and Bash. You cannot spawn subagents.[IF frontend_testing configured, append: ' You also have access to MCP Playwright tools prefixed with mcp__{{mcp_server_name}}__ (e.g., mcp__{{mcp_server_name}}__browser_navigate, mcp__{{mcp_server_name}}__browser_screenshot, mcp__{{mcp_server_name}}__browser_click).']

Implement Sprint N as defined in .xhorse/current-sprint.md. The full product spec is at .xhorse/spec.md. [If rework: Also read the evaluation feedback at .xhorse/evaluations/sprint-NNN-eval-M.md. Fix ONLY the FAIL items.] Follow project conventions from CLAUDE.md. Commit your work incrementally. Fill in the Self-Assessment section of .xhorse/current-sprint.md when done.[IF frontend_testing configured, append: ' Frontend testing enabled. Dev server: {{dev_server_url}}. MCP server: {{mcp_server_name}}. Read skills/frontend-testing/SKILL.md for frontend testing rules — follow the For Generators section.']"
  [, model: "<generator_model from config>" — ONLY if generator_model is set in config.json. If not set, omit the model parameter entirely so the agent inherits the current model.]
})
```

After the generator returns:
1. Check if `.xhorse/questions.md` was created or updated. If so, present the questions to the user and wait for answers. Pass answers back to the generator if needed.
2. Read `.xhorse/current-sprint.md` to see the self-assessment.

### Step 3: Pre-Check Gate

Run deterministic checks before spawning the expensive evaluator:

1. **Build check** (if `tech_stack.build_cmd` exists):
   ```bash
   <build_cmd>
   ```
   If it fails → report the error. Go back to Step 2 (re-spawn generator with the build error). Count this as an iteration.

2. **Test check** (if `tech_stack.test_cmd` exists):
   ```bash
   <test_cmd>
   ```
   If tests fail → report the failures. Go back to Step 2 (re-spawn generator with test failures). Count this as an iteration.

3. **Lint check** (if `tech_stack.lint_cmd` exists):
   Run it. Lint failures are noted but do NOT block — they become warnings for the evaluator.

4. **Dev server health** (only if `frontend_testing` is configured):
   ```bash
   curl -sf -o /dev/null {{dev_server_url}}
   ```
   If unreachable: attempt restart with `Bash(command: "{{dev_server_cmd}}", run_in_background: true)`, then poll for up to 30 seconds. If restart fails, halt: "Dev server is down after generation. The generator's changes may have broken the server. Check for errors."

If all applicable pre-checks pass (or no commands configured), proceed to evaluation.

### Step 4: Evaluate

Spawn the evaluator agent:

**IMPORTANT**: The evaluator MUST NOT have access to Write or Edit tools. Include the tool restriction in the prompt. This is a security boundary — the evaluator judges, it does not modify.

```
Agent({
  description: "Evaluate sprint N",
  prompt: "You are the Evaluator agent. Read your instructions at agents/evaluator.md.

TOOL RESTRICTION: You do NOT have access to Write or Edit tools. Do not attempt to create, modify, or delete any files. You may only read files, search, and run commands. If you find yourself wanting to fix something, describe the fix in your report instead. This restriction is non-negotiable.[IF frontend_testing configured, append: ' You also have access to MCP Playwright tools prefixed with mcp__{{mcp_server_name}}__ (e.g., mcp__{{mcp_server_name}}__browser_navigate, mcp__{{mcp_server_name}}__browser_screenshot, mcp__{{mcp_server_name}}__browser_click). These are read-only verification tools and do not violate your no-write constraint.']

Evaluate Sprint N. Read the sprint contract at .xhorse/current-sprint.md (including the generator's self-assessment). Read the product spec at .xhorse/spec.md. Read the evaluation criteria at skills/xhorse/references/evaluation-criteria.md. Review all code changes since commit <start_sha>: run `git diff <start_sha>..HEAD`. Run the project's tests. Produce your evaluation report as structured text output — do NOT write any files.[IF frontend_testing configured, append: ' Frontend testing enabled. Dev server: {{dev_server_url}}. MCP server: {{mcp_server_name}}. Read skills/frontend-testing/SKILL.md for frontend testing rules — follow the For Evaluators section.']"
  [, model: "<evaluator_model from config>" — ONLY if evaluator_model is set in config.json. If not set, omit the model parameter entirely so the agent inherits the current model.]
})
```

After the evaluator returns:
1. Parse the verdict from the evaluator's output (PASS, PASS_WITH_WARNINGS, or FAIL)
2. Parse the score (X/Y criteria passed)
3. Write the evaluation report to `.xhorse/evaluations/sprint-NNN-eval-M.md`
4. Commit: `git add .xhorse/evaluations/ && git commit -m "xhorse sprint N eval M: <verdict>"`

### Step 5: Decision Gate

**If PASS or PASS_WITH_WARNINGS**:
1. Archive the sprint: copy `.xhorse/current-sprint.md` to `.xhorse/sprints/sprint-NNN.md`
2. Update `status.json`:
   - Add to `completed_sprints` with verdict, iteration count, warnings
   - Check if this was the last sprint from the spec decomposition
   - If more sprints remain: increment `current_sprint.number`, reset `iteration` to 0
   - If all sprints complete: set `phase` to `"complete"`
3. Commit state: `git add .xhorse/ && git commit -m "xhorse sprint N: complete (<verdict>)"`
4. Report the sprint result to the user. Show any warnings.
5. If `phase` is `"complete"`: go to Completion. Otherwise: "Sprint N complete. Run `/xhorse-sprint` for the next sprint."

**If FAIL and iteration < max_iterations_per_sprint**:
1. Increment `current_sprint.iteration` in status.json
2. **Ratchet check**: If `current_sprint.best_score` exists and current score < best_score:
   - Revert to `current_sprint.best_sha`: `git reset --hard <best_sha>`
   - Report: "Score dropped from best. Reverting to best version before retrying."
3. Update best score/SHA if current score >= best_score
4. Report: "Sprint N evaluation FAIL (iteration M/max). Reworking..."
5. Go back to Step 2 (re-spawn generator with evaluation feedback path)

**If FAIL and iteration >= max_iterations_per_sprint**:
1. Report to the user:
   - "Sprint N failed evaluation after M attempts."
   - Show the latest evaluation findings
   - Show the score history across iterations
2. Ask the user:
   - "Continue with more iterations?"
   - "Adjust the spec or sprint contract?"
   - "Skip this sprint and move to the next?"
   - "Stop the harness?"
3. Handle user decision accordingly.

### Completion

When all sprints are complete (`phase` is `"complete"`):

1. Summarize all sprints: number completed, total iterations used, accumulated warnings
2. List all files created/modified across all sprints
3. Show final test results
4. Report: "All sprints complete on branch `xhorse/<session-id>`. To merge: `git checkout <original-branch> && git merge xhorse/<session-id>`"
5. **Frontend testing note** (only if `frontend_testing` was configured):
   > "The dev server (`{{dev_server_cmd}}`) was started in the background and may still be running. Stop it manually if needed."
