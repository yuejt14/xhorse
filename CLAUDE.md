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

- **Agents** (`agents/`) — Agent definitions with frontmatter (tools, instructions)
  - `planner.md` — Converts user prompts into product specs
  - `generator.md` — Implements sprint contracts
  - `evaluator.md` — Skeptical code reviewer (no Write tool)

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
3. **Evaluator cannot write project files** — Enforced via prompt-level tool restriction in Agent() calls and agent definition
4. **Pre-checks before evaluation** — Cheap deterministic checks (build, test) before expensive evaluator
5. **Branch isolation** — All work on `xhorse/<session>` branch, auto-merged back to original branch on completion
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
│   ├── xhorse-status/SKILL.md        # Show state (/xhorse-status)
│   └── frontend-testing/SKILL.md     # Agent skill: Playwright UI testing
├── agents/                           # Agent definitions
│   ├── planner.md
│   ├── generator.md
│   └── evaluator.md
└── templates/                        # Artifact templates
    ├── product-spec.md
    ├── sprint-contract.md
    └── evaluation-report.md
```

### Plugin Development Notes

These are Claude Code platform constraints that affect how this plugin works. They are not obvious from reading the code.

**Agent invocation and tool enforcement**:
- Skills spawn agents via inline `Agent({description, prompt, model})` calls. The `Agent` tool does NOT accept `tools`, `disallowedTools`, or `maxTurns` parameters. The `subagent_type` parameter only recognizes built-in agents (general-purpose, Explore, Plan), not plugin-defined agents.
- Agent `.md` frontmatter (tools, model, maxTurns) is NOT enforced at runtime when agents are invoked this way. The `.md` files are read as prose by the spawned agent, not parsed as configuration.
- Therefore, tool restrictions (especially the evaluator's no-Write constraint) MUST be stated explicitly in the `Agent()` prompt text. This is behavioral enforcement, not runtime enforcement.
- When adding or modifying Agent() calls, always include a `TOOL RESTRICTION:` block in the prompt.

**Agent skills (extensibility pattern)**:
- Shared agent capabilities live in `skills/` as non-user-invocable skills (`user-invocable: false`). Agents read them via file path references in their `Agent()` prompts.
- To add a new capability: (1) create a skill file in `skills/<capability>/SKILL.md`, (2) add a `Read skills/<capability>/SKILL.md` line to the relevant Agent() prompts in the orchestrator.
- The `frontend-testing` skill demonstrates this pattern — it consolidates rules previously duplicated across `evaluator.md` and `generator.md` into one shared file.

**Subagent nesting**: Subagents cannot spawn other subagents. This fails silently. All three agents (planner, generator, evaluator) are subagents, so their instructions must not require spawning further agents.

**Plugin agent frontmatter**: Plugin agents silently ignore `hooks`, `mcpServers`, and `permissionMode` fields. Do not add these to agent `.md` files.

**Skill descriptions**: Truncated at 250 characters in skill listings. Front-load trigger phrases. Current limit is enforced, not advisory.

**`allowed-tools` in skills**: Pre-approves tools (skips permission prompts) but does NOT restrict which tools are available. Every tool remains callable. It is not a security boundary.

**`agents/` auto-discovery**: The `agents/` directory at plugin root is auto-discovered. No explicit `agents` field needed in `plugin.json`.

**Skill priority and shadowing**: Project-level `.claude/skills/` (priority 3) shadows plugin skills (priority 5) when names collide. Never place duplicate skills in both locations.

### Frontend Testing (Optional)

xhorse optionally supports browser-based UI verification using a Docker MCP Playwright server. This is entirely opt-in via configuration.

**How it works**:
- The user adds a `frontend_testing` block to `.xhorse/config.json` with three required fields: `mcp_server_name`, `dev_server_cmd`, and `dev_server_url`.
- The orchestrator validates all three fields at sprint start and probes the MCP server. It fails loudly if anything is misconfigured (no silent degradation).
- The orchestrator starts the dev server before the sprint loop using `Bash(command: "<cmd>", run_in_background: true)` and re-checks its health between generation and evaluation.
- MCP Playwright tools (e.g., `mcp__<name>__browser_navigate`) are granted to the generator and evaluator via their `TOOL RESTRICTION` blocks in `Agent()` prompts. Detailed usage rules are in `skills/frontend-testing/SKILL.md`.
- Agents NEVER manage the dev server lifecycle. Only the orchestrator starts, checks, and restarts it.
- UI acceptance criteria are graded under the existing Correctness category, not a separate category.

**Config example**:
```json
{
  "frontend_testing": {
    "mcp_server_name": "playwright",
    "dev_server_cmd": "npm run dev",
    "dev_server_url": "http://localhost:3000"
  }
}
```

**Prerequisites**: The user must have a Playwright MCP server registered in their Claude Code MCP settings under the name matching `mcp_server_name`. The plugin cannot configure MCP servers on the user's behalf.

**Design constraints**:
- No changes to `status.json` schema. Frontend testing state is stateless (HTTP health checks only).
- No PID tracking, no nohup, no stdout pattern matching for server readiness.
- Background processes use `run_in_background: true`, never `<cmd> &` (which dies when the Bash shell exits).
- MCP tool names are fully dynamic: `mcp__{{mcp_server_name}}__<tool>`. Never hardcode a server name prefix.
- `planner.md`, `sprint-contract.md`, and `evaluation-criteria.md` are not modified.
- Frontend testing rules for agents are centralized in `skills/frontend-testing/SKILL.md` with role-specific sections. Agent `.md` files contain a pointer to this skill, not the full rules.
