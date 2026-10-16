# Agents

Ten specialized Claude Code subagents. The **orchestrated-development** skill launches them; you can also invoke them directly for a single phase.

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
| [`contract-checker`](../agents/contract-checker.md) | Prove cross-boundary references exist | haiku | No |
| [`review-agent`](../agents/review-agent.md) | Audit implementation vs plan | sonnet | No |
| [`state-compressor`](../agents/state-compressor.md) | Compress operational memory | haiku | No |
| [`final-orchestrator-auditor`](../agents/final-orchestrator-auditor.md) | Merge readiness decision | default | No |

Model hints are preferences in frontmatter; Claude Code may map them differently depending on your account settings.

Each agent fills a role with fixed decision permissions ([role permissions](workflow.md#role-permissions)); the model never widens them. The frontmatter model is the default; the orchestrator escalates per phase from [model assignment](workflow.md#model-assignment). **Gate agents** (plan-critic, auditor) use `default` — they inherit the session model, so run orchestration sessions with the strongest reasoning model available (Opus / Fable-class). **Volume agents** (explorer, impact, planner, characterization, implementer, reviewer) stay on sonnet and are escalated to the session model when the phase is hard, critical, or the triage is FULL. **Mechanical agents** (contract-checker, state-compressor) run on haiku: they need exactness, not judgment.

## task-explorer

**When:** Task is ambiguous; area unclear; hidden business rules possible; path not obvious.

**Outputs:** `exploration.md` with CONFIRMED / INFERRED / UNKNOWN, affected flows, risks, questions. Also `context-pack.md`: the distilled conventions, file map, verification commands, and landmines that later agents read instead of re-deriving repo context.

**Does not:** implement, refactor, or produce the final architecture plan.

## impact-analyzer

**When:** After exploration, before planning or implementation.

**Outputs:** `impact.md` — direct/indirect impact, missing tests, data/integration risks, rollback, whether characterization is required.

**Does not:** design the full solution or edit production code.

## task-planner

**When:** Exploration + impact are sufficient; ready to plan.

**Outputs:** `plan.md` — small phases with difficulty (routine/hard), scope, files, tests, evidence, stop conditions, agent handoffs — plus one self-contained `plan/phase-N.md` per phase (phase file + context pack = full implementer input). For EPIC triage it produces `roadmap.md` (sub-task decomposition) instead.

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

**Does not:** expand scope, touch files outside the phase's allowed list, change product or architecture decisions, or skip characterization when required. Makes only local design choices; stops on a contradiction or a false fact in a critical phase.

## contract-checker

**When:** After each implementation phase, before review, at every triage level.

**Outputs:** `contract-check.md` — every HTTP call, table/column, queue/event name, config key, and untyped symbol in the diff, each with the file:line or command output that proves it exists (or MISSING); failing verification commands; a scope check of diff files against the phase's allowed files; a diff fingerprint. Verdict `PASS` / `FAIL` / `INCOMPLETE`.

**Does:** read the real database schema only through a read-only command recorded in the context pack.  
**Does not:** judge design or scope, fix anything, or read secrets.

## review-agent

**When:** After each phase passes its contract check.

**Outputs:** `review.md` with the same verdict model as plan-critic, findings with evidence and a `DEFECT` / `NEEDS CONTEXT` status, and a check of every handoff claim.

**Works as:** an adversarial reviewer from a clean context — it gets the plan's decisions and the diff, never the implementer's reasoning, and tries to break the change instead of grading it.

**Does:** inspect real diffs; reuse the contract check's command results when its diff fingerprint matches, otherwise re-run them.  
**Does not:** implement fixes unless explicitly asked.

## state-compressor

**When:** At every checkpoint — after plan approval, every 2 implementation phases, after gate correction cycles, before ending a session.

**Outputs:** `state.md` — facts, decisions, blockers, risks, remaining work, ending with a **Resume Command** so a fresh session can continue without chat history. No deep analysis, no new decisions.

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

The **first** repo-working agent (normally the explorer) loads full context:

1. Workspace rules
2. Repo `CLAUDE.md` / `AGENTS.md`
3. Local docs for the area
4. Spec systems when required
5. Then source

…and distills it into `context-pack.md`. Later agents read the pack first and fall back to source documents only for gaps. Declare which sources were used in the report/handoff.

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
