# xhorse

A Claude Code plugin that orchestrates three specialized agents (Planner, Generator, Evaluator) through iterative sprint cycles with adversarial evaluation to build applications with higher quality than single-agent passes.

## How It Works

```
User prompt --> Planner (opus) --> Product Spec --> Sprint Loop:
                                                     |
                                                     +--> Generator (sonnet) --> Code
                                                     |       |
                                                     |       v
                                                     |    Pre-checks (build, test, lint)
                                                     |       |
                                                     |       v
                                                     +--> Evaluator (opus) --> PASS/WARN/FAIL
                                                     |       |
                                                     |       v (if FAIL)
                                                     +--- Rework loop (up to N iterations)
```

1. **Planner** analyzes your codebase and converts your prompt into a product spec with sprint decomposition
2. **Generator** implements each sprint, commits incrementally, and produces a self-assessment
3. **Evaluator** independently verifies the implementation against acceptance criteria (cannot modify files)
4. The **orchestrator** decides pass/fail, manages rework loops with ratchet scoring, and advances sprints

## Installation

```bash
# From local directory
claude plugin install ./xhorse --scope project

# Or use directly during development
claude --plugin-dir ./xhorse
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
| `planner_model` | opus | Model for planning |
| `generator_model` | sonnet | Model for implementation |
| `evaluator_model` | opus | Model for evaluation |

## Key Design Decisions

- **Branch isolation**: All work happens on `xhorse/<session-id>`. Your original branch is never modified.
- **Ratchet scoring**: If a rework iteration scores lower than the previous best, the code is reverted to the best version before retrying.
- **Pre-checks before evaluation**: Build, test, and lint run before the expensive evaluator agent, catching obvious failures cheaply.
- **Three-tier grading**: PASS / WARN / FAIL (not binary). Warnings are logged but don't block progress.
- **User checkpoints**: The orchestrator pauses for user approval after planning and escalates after max failed iterations.

## License

MIT
