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

2. Create the `.xhorse/` directory structure:
   ```bash
   mkdir -p .xhorse/sprints .xhorse/evaluations
   ```

3. Detect the tech stack by checking for manifest files:
   - `package.json` → node, read `scripts.test` for test command
   - `pyproject.toml` / `setup.py` → python
   - `go.mod` → go
   - `Cargo.toml` → rust
   - If none found: `"detected": [], "test_cmd": null`

4. Write `.xhorse/config.json`:
   ```json
   {
     "max_iterations_per_sprint": 3,
     "max_sprints": 10,
     "planner_model": "opus",
     "generator_model": "sonnet",
     "evaluator_model": "opus"
   }
   ```

5. Write `.xhorse/status.json`:
   ```json
   {
     "session_id": "<generate-short-id>",
     "branch": "xhorse/<session-id>",
     "base_sha": "<current-HEAD-sha>",
     "phase": "planning",
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

Spawn the planner agent using the Agent tool:

```
Agent({
  description: "Generate product specification",
  prompt: "You are the Planner agent. Read the agent definition at agents/planner.md for your full instructions.

TOOL RESTRICTION: You have access to Read, Write, Glob, Grep, and Bash. You cannot spawn subagents.

The user's request is: $ARGUMENTS. The project root is the current working directory. Analyze the codebase and write a comprehensive product specification to .xhorse/spec.md. Include sprint decomposition with acceptance criteria.",
  model: "opus"
})
```

### Step 4: User Checkpoint

After the planner finishes:

1. Read `.xhorse/spec.md`
2. Present a summary to the user:
   - Product overview (from spec)
   - Number of sprints planned
   - Key technical decisions
3. Ask the user: "Review the specification above. You can:"
   - "Approve it and proceed to implementation"
   - "Request changes (describe what to modify)"
   - "View the full spec at `.xhorse/spec.md`"

4. If the user requests changes, re-run the planner with modification instructions.
5. Once approved, update `.xhorse/status.json`: set `phase` to `"sprinting"`, `current_sprint.number` to `1`.

### Step 5: Done

Report: "Planning complete. Spec written to `.xhorse/spec.md`. Run `/xhorse-sprint` to start the first sprint, or `/xhorse` to run the full orchestration."
