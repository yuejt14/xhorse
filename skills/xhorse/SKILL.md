---
name: xhorse
description: Build complex applications using a three-agent harness (Planner, Generator, Evaluator) with iterative sprint cycles and adversarial evaluation. Use for full apps, large features, or multi-sprint development.
argument-hint: "[description of what to build]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Agent
---

# /xhorse — Three-Agent Development Harness

You are the orchestrator of the xhorse harness. You coordinate three agents (Planner, Generator, Evaluator) through iterative sprint cycles to build applications with higher quality than single-agent passes.

**You own the loop.** Agents do work. You make decisions. You never delegate decision-making to an agent.

## Phase 0: Resume Detection

Check if `.xhorse/status.json` exists in the current working directory.

**If it exists**: Read it. Check the `phase` field.
- If `phase` is `"complete"`: Report "Previous session is complete. Delete `.xhorse/` to start a new one, or inspect the results on branch `{{branch}}`."
- If `phase` is `"planning"` or `"sprinting"`: Ask the user: "Found an existing xhorse session (phase: {{phase}}, sprint: {{current_sprint.number}}). Resume this session or start fresh?" If resume, checkout the xhorse branch and skip to the appropriate phase. If fresh, confirm deletion of `.xhorse/` then proceed to Phase 1.

**If it does not exist**: Proceed to Phase 1.

## Phase 1: Initialization

1. Generate a short session ID (8 chars, alphanumeric). Use:
   ```bash
   head -c 4 /dev/urandom | xxd -p
   ```

2. Detect the tech stack:
   ```bash
   ls package.json pyproject.toml setup.py go.mod Cargo.toml build.gradle pom.xml 2>/dev/null
   ```
   Based on what exists, set:
   - `test_cmd`: the test command (e.g., `npm test`, `pytest`, `go test ./...`)
   - `lint_cmd`: the lint command if available (e.g., `npx eslint .`, `ruff check .`)
   - `build_cmd`: the build/compile command if available (e.g., `npx tsc --noEmit`)
   - For `package.json`: read it and use `scripts.test` if defined

3. Record the current HEAD SHA:
   ```bash
   git rev-parse HEAD
   ```

4. Create the directory structure:
   ```bash
   mkdir -p .xhorse/sprints .xhorse/evaluations
   ```

5. Write `.xhorse/config.json` with defaults:
   ```json
   {
     "max_iterations_per_sprint": 3,
     "max_sprints": 10
   }
   ```
   Note: Model fields (`planner_model`, `generator_model`, `evaluator_model`) are intentionally omitted. When absent, agents inherit whatever model the user is currently running. Users can add these fields to override specific agents (e.g., `"generator_model": "sonnet"`).

   **Optional — Frontend testing**: Users can add a `frontend_testing` block to enable browser-based UI verification via a Playwright MCP server:
   ```json
   {
     "frontend_testing": {
       "mcp_server_name": "playwright",
       "dev_server_cmd": "npm run dev",
       "dev_server_url": "http://localhost:3000"
     }
   }
   ```
   When present, the orchestrator manages the dev server and grants MCP Playwright tools to agents. The user must have the MCP server registered in their Claude Code settings. Validated at sprint start, not at initialization — users add this after session creation (same pattern as model overrides).

6. Write `.xhorse/status.json` with:
   ```json
   {
     "session_id": "<id>",
     "branch": "xhorse/<id>",
     "base_sha": "<sha>",
     "phase": "planning",
     "user_prompt": "$ARGUMENTS",
     "tech_stack": {
       "detected": [...],
       "test_cmd": "...",
       "lint_cmd": "...",
       "build_cmd": "..."
     },
     "config": { ... },
     "current_sprint": null,
     "completed_sprints": []
   }
   ```

7. Create the xhorse branch and commit initialization:
   ```bash
   git checkout -b xhorse/<session-id>
   git add .xhorse/
   git commit -m "xhorse: initialize session <session-id>"
   ```

## Phase 2: Planning

1. Spawn the **planner agent**:
   ```
   Agent({
     description: "Generate product specification",
     prompt: "You are the Planner agent for the xhorse harness.

   Read your full instructions at agents/planner.md.

   TOOL RESTRICTION: You have access to Read, Write, Glob, Grep, and Bash. You cannot spawn subagents.

   USER REQUEST: $ARGUMENTS

   TASK:
   1. Analyze the existing codebase — read CLAUDE.md, check the directory structure, understand existing patterns and tech stack.
   2. Write a comprehensive product specification to .xhorse/spec.md
   3. The spec MUST include a Sprint Decomposition section with ordered sprints, each having: goal, scope, and 3-7 acceptance criteria with 'How to verify' instructions.
   4. Use the template structure from templates/product-spec.md as a guide.

   Return a summary under 300 words: what the spec covers, number of sprints, key decisions."
     [, model: "<planner_model from config>" — ONLY if planner_model is set in config.json. If not set, omit the model parameter entirely so the agent inherits the current model.]
   })
   ```

2. After the planner returns, read `.xhorse/spec.md`.

3. **User checkpoint**: Present the spec summary to the user. Show:
   - Product overview
   - Number of sprints planned
   - Sprint goals (one line each)
   - Key technical decisions

   Ask: "Review the specification. Approve to proceed, or describe what to change."

4. If the user requests changes: re-spawn the planner with modification instructions appended to the original prompt. Repeat until approved.

5. Once approved:
   ```bash
   git add .xhorse/spec.md
   git commit -m "xhorse: product specification approved"
   ```
   Update `status.json`: set `phase` to `"sprinting"`, set `current_sprint` to `{"number": 1, "iteration": 0, "start_sha": null, "best_score": null, "best_sha": null}`.

## Phase 3: Sprint Loop

Read `status.json` to get `current_sprint.number`. Read `.xhorse/spec.md` to get the sprint decomposition. Read `.xhorse/config.json` for settings.

**Frontend Testing Setup** (only if `config.json` contains a `frontend_testing` object):

1. **Validate config.** All three fields (`mcp_server_name`, `dev_server_cmd`, `dev_server_url`) must be non-empty strings. If any are missing or empty, halt:
   > "Frontend testing config incomplete. Missing field(s): {{list}}. Fix `.xhorse/config.json` or remove the `frontend_testing` block to disable."

2. **Probe the MCP server.** Call `mcp__{{mcp_server_name}}__browser_navigate` with URL `"about:blank"`. If the tool is not available or returns an error, halt:
   > "Frontend testing requires the '{{mcp_server_name}}' MCP server but `mcp__{{mcp_server_name}}__browser_navigate` is not available. Register the MCP server in your Claude Code settings, or remove `frontend_testing` from config."

   Do NOT silently degrade. The user explicitly opted in — missing infrastructure is an error.

3. **Start the dev server** (if not already running):
   - Check if `dev_server_url` is already reachable:
     ```bash
     curl -sf -o /dev/null {{dev_server_url}}
     ```
     If reachable: skip startup — the server is already running.
   - If not reachable: start the dev server in the background:
     ```
     Bash(command: "{{dev_server_cmd}}", run_in_background: true)
     ```
     **Do NOT use `<cmd> &`** — the process dies when the shell exits. Use the `run_in_background: true` parameter.
   - Poll until ready (max 30 seconds):
     ```bash
     for i in $(seq 1 30); do curl -sf -o /dev/null {{dev_server_url}} && exit 0; sleep 1; done; exit 1
     ```
     If this exits non-zero, halt:
     > "Dev server failed to start within 30 seconds. Command: `{{dev_server_cmd}}`. URL: `{{dev_server_url}}`. Check the command and port, then retry."

**Important**: Agents must NEVER start or stop the dev server. Only the orchestrator manages the dev server lifecycle.

Loop until all sprints are complete or the user stops.

### 3a: Create Sprint Contract

1. Find the current sprint in the spec's Sprint Decomposition section.
2. If no more sprints exist in the spec: go to Phase 4 (Completion).
3. Record the starting SHA:
   ```bash
   git rev-parse HEAD
   ```
4. Write `.xhorse/current-sprint.md` using the template structure from `templates/sprint-contract.md`:
   - Fill in: sprint number, goal, scope (in/out), acceptance criteria with "How to verify", technical approach
   - Set starting commit SHA
   - Set expected files based on scope
   - List completed sprints for context
   - Leave self-assessment empty
5. Update `status.json`: set `current_sprint.start_sha`.
6. Commit:
   ```bash
   git add .xhorse/
   git commit -m "xhorse sprint <N>: contract"
   ```

### 3b: Generate

1. Determine if this is a first attempt or a rework iteration:
   - First attempt (iteration 0): no evaluation feedback
   - Rework (iteration > 0): include path to latest evaluation report

2. Spawn the **generator agent**:
   ```
   Agent({
     description: "Implement sprint <N>",
     prompt: "You are the Generator agent for the xhorse harness.

   Read your full instructions at agents/generator.md.

   TOOL RESTRICTION: You have access to Read, Write, Glob, Grep, and Bash. You cannot spawn subagents.[IF frontend_testing configured, append: ' You also have access to MCP Playwright tools prefixed with mcp__{{mcp_server_name}}__ (e.g., mcp__{{mcp_server_name}}__browser_navigate, mcp__{{mcp_server_name}}__browser_screenshot, mcp__{{mcp_server_name}}__browser_click).']

   TASK: Implement Sprint <N>.
   - Product spec: .xhorse/spec.md
   - Sprint contract: .xhorse/current-sprint.md
   [IF REWORK: - Evaluation feedback: .xhorse/evaluations/sprint-<NNN>-eval-<M>.md — Fix ONLY the FAIL items.]
   - Project conventions: CLAUDE.md (if exists)
   [IF frontend_testing configured: - Frontend testing enabled. Dev server: {{dev_server_url}}. MCP server: {{mcp_server_name}}. Read skills/frontend-testing/SKILL.md for frontend testing rules — follow the 'For Generators' section.]

   Implement the sprint scope. Commit incrementally. Run tests. Fill in the Self-Assessment section of .xhorse/current-sprint.md.

   Return a summary under 300 words: what was implemented, test results, any concerns."
     [, model: "<generator_model from config>" — ONLY if generator_model is set in config.json. If not set, omit the model parameter entirely so the agent inherits the current model.]
   })
   ```

3. After the generator returns:
   - Check if `.xhorse/questions.md` exists and has new content. If so, present questions to the user. Wait for answers. If critical, pass answers back to generator (re-spawn) or note them for the evaluator.
   - Read `.xhorse/current-sprint.md` to confirm self-assessment was filled in.

### 3c: Pre-Check Gate

Run deterministic checks before the expensive evaluator. Read `tech_stack` from `status.json`.

1. **Build** (if `build_cmd` is set):
   ```bash
   <build_cmd>
   ```
   If fails: increment `current_sprint.iteration`. If iteration < max, go back to 3b with the build error. If iteration >= max, go to 3e (user escalation).

2. **Tests** (if `test_cmd` is set):
   ```bash
   <test_cmd>
   ```
   If fails: increment `current_sprint.iteration`. If iteration < max, go back to 3b with test failures. If iteration >= max, go to 3e (user escalation).

3. **Lint** (if `lint_cmd` is set): Run it. Note results as warnings — do NOT block on lint.

4. **Dev server health** (only if `frontend_testing` is configured):
   ```bash
   curl -sf -o /dev/null {{dev_server_url}}
   ```
   If unreachable: attempt to restart the server using `Bash(command: "{{dev_server_cmd}}", run_in_background: true)`, then poll for up to 30 seconds. If restart also fails, halt:
   > "Dev server is down after generation and could not be restarted. The generator's changes may have broken the server. Check the application for errors before re-running."

If all applicable pre-checks pass (or not configured), proceed to 3d.

### 3d: Evaluate

1. Get the diff scope:
   ```bash
   git diff <start_sha>..HEAD --stat
   ```

2. Spawn the **evaluator agent**:

   **IMPORTANT**: The evaluator MUST NOT have access to Write or Edit tools. Include the tool restriction in the prompt. This is a security boundary — the evaluator judges, it does not modify.

   ```
   Agent({
     description: "Evaluate sprint <N>",
     prompt: "You are the Evaluator agent for the xhorse harness.

   Read your full instructions at agents/evaluator.md.

   TOOL RESTRICTION: You do NOT have access to Write or Edit tools. Do not attempt to create, modify, or delete any files. You may only read files, search, and run commands. If you find yourself wanting to fix something, describe the fix in your report instead. This restriction is non-negotiable.[IF frontend_testing configured, append: ' You also have access to MCP Playwright tools prefixed with mcp__{{mcp_server_name}}__ (e.g., mcp__{{mcp_server_name}}__browser_navigate, mcp__{{mcp_server_name}}__browser_screenshot, mcp__{{mcp_server_name}}__browser_click). These are read-only verification tools and do not violate your no-write constraint.']

   TASK: Evaluate Sprint <N>, Iteration <M>.
   - Product spec: .xhorse/spec.md
   - Sprint contract: .xhorse/current-sprint.md
   - Evaluation criteria: skills/xhorse/references/evaluation-criteria.md
   - Changes since sprint start: run `git diff <start_sha>..HEAD`
   - Test command: <test_cmd>
   [IF frontend_testing configured: - Frontend testing enabled. Dev server: {{dev_server_url}}. MCP server: {{mcp_server_name}}. Read skills/frontend-testing/SKILL.md for frontend testing rules — follow the 'For Evaluators' section.]

   Independently verify each acceptance criterion. Run tests. Check scope drift. Grade each criterion PASS/WARN/FAIL.

   Return your evaluation report as structured text. Do NOT write any files.
   Format: use the structure defined in your agent instructions."
     [, model: "<evaluator_model from config>" — ONLY if evaluator_model is set in config.json. If not set, omit the model parameter entirely so the agent inherits the current model.]
   })
   ```

3. After the evaluator returns:
   - Parse the verdict (PASS, PASS_WITH_WARNINGS, FAIL) from the output
   - Parse the score (X/Y criteria passed)
   - Write the evaluation report to `.xhorse/evaluations/sprint-<NNN>-eval-<M>.md`
   - Commit:
     ```bash
     git add .xhorse/evaluations/
     git commit -m "xhorse sprint <N> eval <M>: <verdict>"
     ```

### 3e: Decision Gate

Read the verdict from the evaluation.

**PASS or PASS_WITH_WARNINGS**:
1. Archive: copy `.xhorse/current-sprint.md` to `.xhorse/sprints/sprint-<NNN>.md`
2. Update `status.json`:
   - Append to `completed_sprints`: `{"number": N, "iterations": M, "verdict": "<verdict>", "warnings": [...]}`
   - Determine if more sprints remain in the spec
   - If more: set `current_sprint` to `{"number": N+1, "iteration": 0, "start_sha": null, "best_score": null, "best_sha": null}`
   - If none remain: set `phase` to `"complete"`, `current_sprint` to null
3. Commit:
   ```bash
   git add .xhorse/
   git commit -m "xhorse sprint <N>: complete (<verdict>)"
   ```
4. Report sprint result to user. Show warnings if any.
5. If phase is `"complete"`: go to Phase 4.
6. Otherwise: loop back to 3a for the next sprint.

**FAIL (iteration < max_iterations_per_sprint)**:
1. Increment `current_sprint.iteration`
2. **Ratchet check**:
   - If `best_score` is set and current score < best_score:
     - Report: "Score dropped ({{current}} < {{best}}). Reverting to best version."
     - Run: `git reset --hard <best_sha>`
   - If current score >= best_score (or best_score is null):
     - Update `best_score` to current score
     - Update `best_sha` to current HEAD
3. Update `status.json`
4. Report: "Sprint {{N}} FAIL (iteration {{M}}/{{max}}). Sending feedback to generator..."
5. Loop back to 3b with the evaluation report path.

**FAIL (iteration >= max_iterations_per_sprint)**:
1. Report to the user:
   - "Sprint {{N}} failed evaluation after {{max}} attempts."
   - Show the latest evaluation findings (key FAIL items)
   - Show score history across iterations
2. Ask the user to choose:
   - "Give it more iterations (increases max for this sprint)"
   - "Adjust the spec or sprint contract and retry"
   - "Skip this sprint and continue to the next"
   - "Stop the harness"
3. Handle:
   - More iterations: update max, loop back to 3b
   - Adjust: let user edit, loop back to 3b with iteration reset
   - Skip: archive as skipped, move to next sprint
   - Stop: update phase to "stopped", report current state

## Phase 4: Completion

1. Read all completed sprints from `status.json`.
2. Get full diff from base:
   ```bash
   git diff <base_sha>..HEAD --stat
   ```
3. Run final test suite:
   ```bash
   <test_cmd>
   ```
4. Report to the user:
   - **Summary**: X sprints completed, Y total iterations, Z warnings
   - **Files changed**: list from git diff stat
   - **Test results**: final test output
   - **Warnings**: accumulated warnings from all sprints
   - **Next steps**: "Your changes are on branch `xhorse/<session-id>`. To merge:"
     ```
     git checkout <original-branch>
     git merge xhorse/<session-id>
     ```
     "Or to squash merge: `git merge --squash xhorse/<session-id>`"
     "To discard: `git branch -D xhorse/<session-id>`"

5. **Frontend testing note** (only if `frontend_testing` was configured):
   > "The dev server (`{{dev_server_cmd}}`) was started in the background and may still be running. Stop it manually if needed (e.g., find the process on the dev server port and kill it)."

## Orchestrator Rules

1. **You own the loop.** Never ask an agent to decide whether to continue, skip, or stop. That's your job.
2. **File-based communication.** Pass file paths to agents, not content. Agents read files themselves. This preserves context window.
3. **Concise agent outputs.** Every agent prompt includes the 300-word return limit. Detailed output goes to files.
4. **User checkpoints at natural boundaries.** After planning (spec review). After sprint failures (escalation). Never mid-generation.
5. **State in status.json.** Every state transition updates status.json. If interrupted, the next invocation can resume.
6. **Branch isolation.** All work happens on `xhorse/<session-id>`. The user's original branch is never modified.
