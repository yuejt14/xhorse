# xhorse

A Claude Code plugin that orchestrates three specialized agents (Planner, Generator, Evaluator) through iterative sprint cycles with adversarial evaluation to build applications with higher quality than single-agent passes.

## How It Works

```
User prompt --> Planner --> Product Spec --> Sprint Loop:
                                                 |
                                                 +--> Generator --> Code
                                                 |       |
                                                 |       v
                                                 |    Pre-checks (build, test, lint)
                                                 |       |
                                                 |       v
                                                 +--> Evaluator --> PASS/WARN/FAIL
                                                 |       |
                                                 |       v (if FAIL)
                                                 +--- Rework loop (up to N iterations)
```

By default, all agents use whatever model you're running Claude Code with. Override individual agents via `.xhorse/config.json`.

1. **Planner** analyzes your codebase and converts your prompt into a product spec with sprint decomposition
2. **Generator** implements each sprint, commits incrementally, and produces a self-assessment
3. **Evaluator** independently verifies the implementation against acceptance criteria (cannot modify files)
4. The **orchestrator** decides pass/fail, manages rework loops with ratchet scoring, and advances sprints

## Installation

### From GitHub

```bash
# 1. Add the marketplace
/plugin marketplace add yuejt14/xhorse

# 2. Install the plugin
/plugin install xhorse@yuejt14-xhorse
```

### From local directory

```bash
# For development — loads the plugin for the current session
claude --plugin-dir ./xhorse

# Or install into a project
claude plugin install ./xhorse --scope project
```

## Usage

### Full orchestration

```
/xhorse Build a REST API for managing todo items with authentication
```

Runs the complete pipeline: planning, user approval, sprint loop (generate -> check -> evaluate -> decide), and completion report.

### Planning only

```
/xhorse-plan Build a REST API for managing todo items with authentication
```

Generates a product spec and waits for approval. No code is written.

### Single sprint

```
/xhorse-sprint
```

Runs one sprint cycle (requires an existing planned session from `/xhorse-plan`).

### Check status

```
/xhorse-status
```

Shows the current session state.

## What It Creates

All runtime artifacts are stored in `.xhorse/` in the target project (on a `xhorse/<session-id>` branch):

```
.xhorse/
├── status.json          # State machine (phase, sprint, iteration)
├── config.json          # Settings (models, iteration limits)
├── spec.md              # Product specification
├── current-sprint.md    # Active sprint contract + self-assessment
├── sprints/             # Archived completed sprint contracts
├── evaluations/         # Evaluation reports per sprint/iteration
└── questions.md         # Generator's clarification questions
```

## Configuration

Default settings in `.xhorse/config.json` (created per session):

| Setting | Default | Description |
|---------|---------|-------------|
| `max_iterations_per_sprint` | 3 | Rework attempts before escalating to user |
| `max_sprints` | 10 | Maximum sprints per session |
| `planner_model` | *(current model)* | Override model for planning agent |
| `generator_model` | *(current model)* | Override model for implementation agent |
| `evaluator_model` | *(current model)* | Override model for evaluation agent |

Model fields are omitted from the default config. When absent, agents inherit whatever model you're running Claude Code with. Add a model field to override a specific agent (e.g., `"generator_model": "sonnet"`). If upgrading from an older version, remove any `*_model` fields from existing `.xhorse/config.json` files to use model inheritance.

### Frontend Testing (Optional)

To enable browser-based UI verification using Docker MCP Playwright, add a `frontend_testing` block to `.xhorse/config.json` after session creation:

```json
{
  "frontend_testing": {
    "mcp_server_name": "playwright",
    "dev_server_cmd": "npm run dev",
    "dev_server_url": "http://localhost:3000"
  }
}
```

| Field | Description |
|-------|-------------|
| `mcp_server_name` | Name of the Playwright MCP server in your Claude Code settings |
| `dev_server_cmd` | Command to start the dev server (run in background automatically) |
| `dev_server_url` | URL to health-check and pass to agents for browser navigation |

**Prerequisites**: You must have a [Playwright MCP server](https://github.com/anthropics/anthropic-quickstarts/tree/main/mcp-playwright) registered in your Claude Code MCP settings under the name matching `mcp_server_name`.

When enabled, the orchestrator starts the dev server before sprints, the generator verifies UI changes in a real browser, and the evaluator independently verifies UI acceptance criteria via Playwright. UI criteria are graded under the existing Correctness category. If the MCP server is not available, xhorse halts with a clear error rather than silently degrading.

## Key Design Decisions

- **Branch isolation**: All work happens on `xhorse/<session-id>`. Your original branch is never modified.
- **Ratchet scoring**: If a rework iteration scores lower than the previous best, the code is reverted to the best version before retrying.
- **Pre-checks before evaluation**: Build, test, and lint run before the expensive evaluator agent, catching obvious failures cheaply.
- **Three-tier grading**: PASS / WARN / FAIL (not binary). Warnings are logged but don't block progress.
- **User checkpoints**: The orchestrator pauses for user approval after planning and escalates after max failed iterations.

## Extending with Agent Skills

Agent capabilities are modular. Shared rules that multiple agents need live in `skills/` as non-user-invocable skills (e.g., `skills/frontend-testing/SKILL.md`). Agents read these files when the orchestrator's task prompt directs them to.

To add a new capability:
1. Create `skills/<capability>/SKILL.md` with `user-invocable: false` in frontmatter
2. Add a `Read skills/<capability>/SKILL.md` line to the relevant Agent() prompts in the orchestrator skills

This pattern keeps agent `.md` files focused on role identity and process, while capabilities are composable and independently maintainable.

## License

MIT
