---
name: xhorse-plan
description: Run the planning phase only — analyze the codebase, generate a product specification, and wait for user approval before any code is written.
argument-hint: "[description of what to build]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Agent
---

# /xhorse-plan

Run the planning phase of the xhorse harness. This creates a product specification from the user's prompt without writing any implementation code.

## Instructions

### Step 1: Initialize

1. Check if `.xhorse/status.json` already exists:
   - If it exists and `phase` is not `"planning"`: Report "A session is already in progress at phase '{{phase}}'. Run `/xhorse-status` to see current state, or delete `.xhorse/` to start fresh."
   - If it exists and `phase` is `"planning"`: Resume — skip to Step 3.
   - If it does not exist: Continue to initialization.

2. Create the `.xhorse/` directory structure based on `mode` from config.json (if it exists, otherwise default to `"continuous"`):
   - If `mode` is `"continuous"`: `mkdir -p .xhorse/evaluations`
   - If `mode` is `"sprints"`: `mkdir -p .xhorse/sprints .xhorse/evaluations`

3. Detect the tech stack by checking for manifest files:
   - `package.json` → node, read `scripts.test` for test command
   - `pyproject.toml` / `setup.py` → python
   - `go.mod` → go
   - `Cargo.toml` → rust
   - If none found: `"detected": [], "test_cmd": null`

4. Write `.xhorse/config.json` (or merge with existing if user pre-created it):
   ```json
   {
     "mode": "continuous",
     "max_iterations": 3,
     "max_iterations_per_sprint": 3,
     "max_sprints": 10
   }
   ```
   - `mode`: `"continuous"` (default) — single generation pass with one final evaluation. Or `"sprints"` — iterative sprint cycles with per-sprint evaluation.
   - `max_iterations`: rework cap in continuous mode.
   - `max_iterations_per_sprint` / `max_sprints`: used only in sprint mode.
   - Note: Model fields (`planner_model`, `generator_model`, `evaluator_model`) are intentionally omitted. When absent, agents inherit whatever model the user is currently running. Users can add these fields to override specific agents (e.g., `"generator_model": "sonnet"`).

5. Write `.xhorse/status.json`. The `mode` field is authoritative after init — read from config.json and persist here.

   **If `mode` is `"continuous"`**:
   ```json
   {
     "session_id": "<generate-short-id>",
     "branch": "xhorse/<session-id>",
     "base_branch": "<current-branch-name>",
     "base_sha": "<current-HEAD-sha>",
     "phase": "planning",
     "mode": "continuous",
     "user_prompt": "$ARGUMENTS",
     "tech_stack": { ... },
     "config": { ... },
     "generation": {
       "start_sha": null,
       "iteration": 0,
       "best_score": null,
       "best_sha": null
     }
   }
   ```

   **If `mode` is `"sprints"`**:
   ```json
   {
     "session_id": "<generate-short-id>",
     "branch": "xhorse/<session-id>",
     "base_branch": "<current-branch-name>",
     "base_sha": "<current-HEAD-sha>",
     "phase": "planning",
     "mode": "sprints",
     "user_prompt": "$ARGUMENTS",
     "tech_stack": { ... },
     "config": { ... },
     "current_sprint": null,
     "completed_sprints": []
   }
   ```

6. Create the xhorse branch:
   ```bash
   git checkout -b xhorse/<session-id>
   ```

### Step 2: Record baseline
   ```bash
   git add .xhorse/
   git commit -m "xhorse: initialize session <session-id>"
   ```

### Step 3: Run the Planner

Read `mode` from `status.json`. Spawn the planner agent using the Agent tool. The prompt varies by mode:

**If `mode` is `"continuous"`**:
```
Agent({
  description: "Generate product specification",
  prompt: "You are the Planner agent. Read the agent definition at agents/planner.md for your full instructions. The mode is 'continuous' — follow the Continuous Mode quality checklist.

TOOL RESTRICTION: You have access to Read, Write, Glob, Grep, and Bash. You cannot spawn subagents.

The user's request is: $ARGUMENTS. The project root is the current working directory. Analyze the codebase and write a comprehensive product specification to .xhorse/spec.md. Include an Acceptance Criteria section — a flat, numbered list with 'How to verify' instructions and 'Expected files' per criterion. Do NOT decompose into sprints."
  [, model: "<planner_model from config>" — ONLY if planner_model is set in config.json. If not set, omit the model parameter entirely so the agent inherits the current model.]
})
```

**If `mode` is `"sprints"`**:
```
Agent({
  description: "Generate product specification",
  prompt: "You are the Planner agent. Read the agent definition at agents/planner.md for your full instructions.

TOOL RESTRICTION: You have access to Read, Write, Glob, Grep, and Bash. You cannot spawn subagents.

The user's request is: $ARGUMENTS. The project root is the current working directory. Analyze the codebase and write a comprehensive product specification to .xhorse/spec.md. Include sprint decomposition with acceptance criteria."
  [, model: "<planner_model from config>" — ONLY if planner_model is set in config.json. If not set, omit the model parameter entirely so the agent inherits the current model.]
})
```

### Step 4: User Checkpoint

After the planner finishes:

1. Read `.xhorse/spec.md`
2. Present a summary to the user:
   - Product overview (from spec)
   - **Continuous mode**: Number of acceptance criteria. If count > 30, warn: "This spec has {{N}} criteria, above the recommended max of 30 for continuous mode. Consider sprint mode."
   - **Sprint mode**: Number of sprints planned, sprint goals
   - Key technical decisions
3. Ask the user: "Review the specification above. You can:"
   - "Approve it and proceed to implementation"
   - "Request changes (describe what to modify)"
   - "View the full spec at `.xhorse/spec.md`"

4. If the user requests changes, re-run the planner with modification instructions.
5. Once approved, update `.xhorse/status.json` based on mode:
   - **Continuous**: set `phase` to `"generating"`
   - **Sprints**: set `phase` to `"sprinting"`, set `current_sprint` to `{"number": 1, "iteration": 0, "start_sha": null, "best_score": null, "best_sha": null}`

### Step 5: Done

**Continuous mode**: Report: "Planning complete. Spec written to `.xhorse/spec.md`. Run `/xhorse` to start continuous generation."

**Sprint mode**: Report: "Planning complete. Spec written to `.xhorse/spec.md`. Run `/xhorse-sprint` to start the first sprint, or `/xhorse` to run the full orchestration."
