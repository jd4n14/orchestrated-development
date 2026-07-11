# Agents

Nine specialized Claude Code subagents. The **orchestrated-development** skill launches them; you can also invoke them directly for a single phase.

All agent definitions live in [`agents/`](../agents/).

## Roster

| Agent | Role | Typical model hint | Writes code? |
|-------|------|--------------------|--------------|
| [`task-explorer`](../agents/task-explorer.md) | Reduce ambiguity before planning | sonnet | No (read-only + report) |
| [`impact-analyzer`](../agents/impact-analyzer.md) | Blast radius, risks, characterization need | sonnet | No |
| [`task-planner`](../agents/task-planner.md) | Phased executable plan | sonnet | No |
| [`plan-critic`](../agents/plan-critic.md) | Break the plan before coding | default | No |
| [`characterization-test-designer`](../agents/characterization-test-designer.md) | Freeze current behavior with tests | sonnet | Tests only |
| [`implementation-agent`](../agents/implementation-agent.md) | Implement **one** approved phase | sonnet | Yes |
| [`review-agent`](../agents/review-agent.md) | Audit implementation vs plan | default | No |
| [`state-compressor`](../agents/state-compressor.md) | Compress operational memory | haiku | No |
| [`final-orchestrator-auditor`](../agents/final-orchestrator-auditor.md) | Merge readiness decision | default | No |

Model hints are preferences in frontmatter; Claude Code may map them differently depending on your account settings.

## task-explorer

**When:** Task is ambiguous; area unclear; hidden business rules possible; path not obvious.

**Outputs:** `exploration.md` with CONFIRMED / INFERRED / UNKNOWN, affected flows, risks, questions.

**Does not:** implement, refactor, or produce the final architecture plan.

## impact-analyzer

**When:** After exploration, before planning or implementation.

**Outputs:** `impact.md` — direct/indirect impact, missing tests, data/integration risks, rollback, whether characterization is required.

**Does not:** design the full solution or edit production code.

## task-planner

**When:** Exploration + impact are sufficient; ready to plan.

**Outputs:** `plan.md` — small phases with scope, files, tests, evidence, stop conditions, agent handoffs.

**Does not:** implement. If UNKNOWNs remain, adds validation steps first.

## plan-critic

**When:** After planning, before coding. For FULL triage, run a panel with distinct lenses.

**Outputs:** `plan-review.md` with verdict:

- `APPROVED`
- `APPROVED WITH CHANGES`
- `BLOCKED`

**Does not:** rewrite the plan unless necessary; focuses on risk and gaps.

## characterization-test-designer

**When:** Legacy or critical flows without enough automated coverage may change.

**Outputs:** `characterization-tests.md` + actual test files that assert **current** behavior (not desired ideal).

**Does not:** “fix” production behavior during characterization. Production refactors only with explicit approval if required for testability.

## implementation-agent

**When:** Plan approved; characterization done if required; one phase assigned.

**Outputs:** code changes + `implementation-handoff.md` (files, tests run, deviations, blockers).

**Does not:** expand scope, invent architecture, or skip characterization when required.

## review-agent

**When:** After each implementation phase.

**Outputs:** `review.md` with the same verdict model as plan-critic.

**Does:** re-run verification commands; inspect real diffs.  
**Does not:** implement fixes unless explicitly asked.

## state-compressor

**When:** After major phases, long agent chains, or growing context.

**Outputs:** `state.md` — facts, decisions, blockers, risks, remaining work. No deep analysis, no new decisions.

## final-orchestrator-auditor

**When:** Before declaring the task complete.

**Outputs:** `final-audit.md` with merge readiness:

- `READY TO MERGE`
- `READY WITH KNOWN RISKS`
- `REQUIRES MORE WORK`
- `BLOCKED`

Works primarily from artifacts, not raw code.

## Shared conventions

### Artifact-first communication

If the orchestrator passes an artifact path:

1. Write the **full** report to that path
2. Return only a **short** summary (verdict, top findings, blockers)

### Context protocol

Before reading or editing source inside a repo, load:

1. Workspace rules
2. Repo `CLAUDE.md` / `AGENTS.md`
3. Local docs for the area
4. Spec systems when required
5. Then source

Declare which documents were read in the report/handoff.

### Evidence over confidence

Claims need evidence. Verification commands come from the target repo’s own docs and package scripts.

## Customizing agents

Safe tweaks:

- Adjust `model` frontmatter for cost/quality
- Narrow `tools` further for stricter read-only agents
- Add domain-specific risk checklists (e.g. your payment ledger rules)

Avoid:

- Letting `implementation-agent` own exploration or planning
- Removing epistemic labels (CONFIRMED / INFERRED / UNKNOWN)
- Allowing reviews to skip re-verification of tests

See [contributing.md](contributing.md).
