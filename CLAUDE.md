# CLAUDE.md

## Project: xhorse

### Overview

A Claude Code plugin that implements a GAN-inspired three-agent harness for long-running application development. Orchestrates Planner, Generator, and Evaluator agents through iterative sprint cycles with adversarial evaluation to produce higher-quality output than single-agent passes.

### Architecture

The plugin consists of:

- **Skills** — Claude Code skills, each in its own directory
  - `skills/xhorse/SKILL.md` — Main orchestrator, invoked via `/xhorse <prompt>`
  - `skills/xhorse-plan/SKILL.md` — Planning phase only, via `/xhorse-plan <prompt>`
  - `skills/xhorse-sprint/SKILL.md` — Single sprint cycle, via `/xhorse-sprint`
  - `skills/xhorse-status/SKILL.md` — Show current state, via `/xhorse-status`
  - `skills/xhorse/references/evaluation-criteria.md` — Evaluation framework

- **Agents** (`agents/`) — Agent definitions with frontmatter (model, tools, instructions)
  - `planner.md` — Converts user prompts into product specs (opus)
  - `generator.md` — Implements sprint contracts (sonnet)
  - `evaluator.md` — Skeptical code reviewer (opus, no Write tool)

- **Templates** (`templates/`) — Markdown templates for structured artifacts
  - `product-spec.md` — Product specification structure
  - `sprint-contract.md` — Sprint negotiation structure
  - `evaluation-report.md` — Evaluation output structure

- **Runtime artifacts** (`.xhorse/` in target project, not in plugin)
  - `status.json` — State machine tracking phase, sprints, iterations
  - `config.json` — User-tunable settings (models, iteration limits)
  - `spec.md`, `current-sprint.md`, `sprints/`, `evaluations/`

### Key Design Principles

1. **Orchestrator owns the loop** — Agents never decide pass/fail
2. **File-based communication** — Agents exchange info through structured files
3. **Evaluator cannot write project files** — Enforced at tool level
4. **Pre-checks before evaluation** — Cheap deterministic checks (build, test) before expensive evaluator
5. **Branch isolation** — All work on `xhorse/<session>` branch
6. **Three-tier grading** — PASS / WARN / FAIL (not binary)
7. **Ratchet scoring** — Revert if quality drops between iterations

### Project Structure

```
xhorse/
├── CLAUDE.md                         # This file
├── .claude-plugin/plugin.json        # Plugin metadata
├── skills/
│   ├── xhorse/                       # Main orchestrator (/xhorse)
│   │   ├── SKILL.md
│   │   └── references/               # Reference material for agents
│   ├── xhorse-plan/SKILL.md          # Planning only (/xhorse-plan)
│   ├── xhorse-sprint/SKILL.md        # Single sprint (/xhorse-sprint)
│   └── xhorse-status/SKILL.md        # Show state (/xhorse-status)
├── agents/                           # Agent definitions
│   ├── planner.md
│   ├── generator.md
│   └── evaluator.md
└── templates/                        # Artifact templates
    ├── product-spec.md
    ├── sprint-contract.md
    └── evaluation-report.md
```
