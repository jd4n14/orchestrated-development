# Orchestrated Development

Structured **multi-agent development workflow** for Claude Code.

Turns ambiguous, risky, legacy, or medium-to-large engineering tasks into a controlled sequence of specialized subagents — with artifacts as the source of truth, not chat history.

| | |
|---|---|
| **Skill** | `orchestrated-development` — technical orchestrator (does not implement code) |
| **Agents** | 9 specialized subagents for explore → plan → implement → review → audit |
| **Target** | [Claude Code](https://docs.anthropic.com/en/docs/claude-code) |

## Why this exists

Ad-hoc multi-agent runs often:

- invent requirements instead of labeling unknowns
- implement before impact is understood
- burn context on full conversation history
- skip characterization tests on legacy flows
- declare “done” without evidence

This package enforces a gated workflow, narrow agent scopes, and file-based handoffs under `.ai/tasks/<task-id>/`.

## Quick install (Claude Code)

```bash
# Clone
git clone https://github.com/jd4n14/orchestrated-development.git
cd orchestrated-development

# Install skill (user-level)
mkdir -p ~/.claude/skills
cp -R skills/orchestrated-development ~/.claude/skills/

# Install agents (user-level)
mkdir -p ~/.claude/agents
cp agents/*.md ~/.claude/agents/
```

Project-level install (repo-local, shareable with the team):

```bash
mkdir -p .claude/skills .claude/agents
cp -R skills/orchestrated-development .claude/skills/
cp agents/*.md .claude/agents/
```

See [docs/installation.md](docs/installation.md) for details and verification.

## Usage

In Claude Code, invoke the skill when the task is non-trivial:

```text
Use the orchestrated-development skill for: <describe the task>
```

Or ask for a full orchestration of a ticket, refactor, migration, or bug in a critical flow.

The orchestrator will:

1. **Triage** the task (`LIGHT` / `STANDARD` / `FULL`)
2. Write a **brief** and create the task folder
3. Launch only the agents required for that triage level
4. Gate progress on artifacts and review verdicts
5. Finish with a **final audit** before claiming completion

## Workflow at a glance

```text
Triage → Brief → Explore → Impact → Plan → Plan review
       → Characterization (if needed) → Implement (per phase)
       → Review → Compress state → Final audit
```

| Phase | Agent | Artifact |
|-------|--------|----------|
| Explore | `task-explorer` | `exploration.md` |
| Impact | `impact-analyzer` | `impact.md` |
| Plan | `task-planner` | `plan.md` |
| Plan review | `plan-critic` | `plan-review.md` |
| Characterization | `characterization-test-designer` | `characterization-tests.md` |
| Implement | `implementation-agent` | `implementation-handoff.md` |
| Review | `review-agent` | `review.md` |
| Compress | `state-compressor` | `state.md` |
| Final audit | `final-orchestrator-auditor` | `final-audit.md` |

Full detail: [docs/workflow.md](docs/workflow.md) · Agents: [docs/agents.md](docs/agents.md) · Artifacts: [docs/artifacts.md](docs/artifacts.md)

## Repository layout

```text
.
├── agents/                          # Claude Code subagent definitions
│   ├── task-explorer.md
│   ├── impact-analyzer.md
│   ├── task-planner.md
│   ├── plan-critic.md
│   ├── characterization-test-designer.md
│   ├── implementation-agent.md
│   ├── review-agent.md
│   ├── state-compressor.md
│   └── final-orchestrator-auditor.md
├── skills/
│   └── orchestrated-development/
│       └── SKILL.md                 # Orchestrator skill
├── docs/                            # Documentation
├── .claude-plugin/
│   └── plugin.json
├── LICENSE
└── README.md
```

## Core principles

1. **Orchestrate, do not implement** (skill level)
2. **Delegate narrow work** to specialized agents
3. **Artifacts over chat** as memory
4. **Label claims**: `CONFIRMED` / `INFERRED` / `UNKNOWN`
5. **Characterization before** risky legacy changes
6. **Evidence before** declaring completion
7. **Context budget**: short agent summaries, full reports on disk

## When to use it

| Use | Skip / LIGHT triage |
|-----|---------------------|
| Ambiguous tickets | One-line renames |
| Legacy / untested flows | Pure docs edits |
| Billing, payments, warehouse, state machines | Trivial UI copy |
| Multi-repo changes | Well-covered single-file fix |
| Migrations / irreversible data | Exploratory spikes with no delivery goal |

## Documentation

- [Installation](docs/installation.md)
- [Workflow](docs/workflow.md)
- [Agents](docs/agents.md)
- [Artifacts](docs/artifacts.md)
- [Contributing / customizing](docs/contributing.md)

## License

[MIT](LICENSE) © Diego Parra ([@jd4n14](https://github.com/jd4n14))
