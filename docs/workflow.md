# Workflow

The **orchestrated-development** skill is the technical orchestrator. It does **not** implement production code. It creates artifacts, launches agents, and enforces gates.

## Task folder

Every non-trivial task gets a folder:

```text
.ai/tasks/<task-id>/
  brief.md
  exploration.md
  impact.md
  plan.md
  plan-review.md
  characterization-tests.md
  implementation-handoff.md
  review.md
  state.md
  final-audit.md
```

If no id is given, use a short kebab-case name derived from the task.

### Resume protocol

If `.ai/tasks/<task-id>/` already exists:

1. Read `state.md` first
2. Load only pending / relevant artifacts
3. Continue from the phase indicated by state
4. Do not re-run approved phases unless new information changes the task

## Triage levels

Classify before deep work. Record level + justification in `brief.md`.

| Level | When | Path |
|-------|------|------|
| **LIGHT** | Well-understood area, good coverage, single repo, no critical flows | brief → short plan → implement → review |
| **STANDARD** | Default | Full workflow; skip characterization unless impact requires it |
| **FULL** | Legacy, critical business flows, multi-repo, unknown coverage, irreversible data | Every step |

Agents may **escalate** triage when they discover risk. Never **downgrade** without user approval.

## Phases

### 0. Triage

Decide LIGHT / STANDARD / FULL.

### 1. Brief

Write `brief.md` with:

- objective, user context
- triage + justification
- affected repos
- scope / non-scope
- expected outcome, constraints
- known risks, open questions
- definition of done

Ask only minimum clarifying questions if the request is too vague.

### 2. Exploration → `task-explorer`

Use when ambiguity, unclear area, hidden rules, or non-obvious path.

- Multi-repo: one explorer **per repo**, launched concurrently, then consolidate into `exploration.md`
- Do not plan if exploration has **blocking UNKNOWNs**

### 3. Impact → `impact-analyzer`

After exploration, before planning.

- Direct vs indirect impact
- Missing tests, legacy, integrations, persistence, queues, billing, warehouse, rollback
- If characterization is required → block implementation until tests exist

### 4. Planning → `task-planner`

Phased plan in `plan.md`. Each phase needs:

- objective, scope, out of scope
- expected files/modules
- steps, required tests, required evidence
- risks, stop conditions
- suggested agent, handoff requirements

### 5. Plan review → `plan-critic`

- FULL / high-risk: 2–3 critics in parallel (lenses: correctness, data/rollback, sequencing/scope)
- Verdicts: `APPROVED` | `APPROVED WITH CHANGES` | `BLOCKED`
- Gate iteration limit: max **2** correction cycles; on third disagreement, escalate to the user

### 6. Characterization → `characterization-test-designer`

Before implementation when:

- legacy flow may be affected
- coverage missing / partial / unknown
- billing, payments, invoices, crons, queues, webhooks, warehouse, guides, order states, permissions, external APIs, DB writes, historical data

Freeze **current** behavior. If tests cannot be added safely → stop and escalate.

### 7. Implementation → `implementation-agent`

- **One phase at a time**
- Minimal context (phase + excerpts + stop conditions)
- No scope expansion
- Update `implementation-handoff.md`

### 8. Review → `review-agent`

After each implementation phase.

- Compare handoff + diff against plan and risks
- Re-verify evidence (diff, lint, tests, typecheck as defined by the repo)
- Same verdict model and gate iteration limit as plan review

### 9. State compression → `state-compressor`

After major phases or when context grows large. Produces `state.md` — operational memory for the next steps.

### 10. Final audit → `final-orchestrator-auditor`

Before claiming completion. Allowed “done” verdicts:

- `READY TO MERGE`
- `READY WITH KNOWN RISKS`

Otherwise continue the workflow.

## Epistemic rules

Every important claim is labeled:

| Label | Meaning |
|-------|---------|
| **CONFIRMED** | Supported by user, code, tests, logs, docs, or artifact |
| **INFERRED** | Likely from evidence, not proven |
| **UNKNOWN** | Insufficient evidence |

If a plan/implementation depends on UNKNOWN or risky INFERRED claims → validate before continuing. Agents must not invent product requirements, architecture, API behavior, business rules, or legacy behavior.

## Context loading protocol

When work touches a repository, agents load context **in this order** before source code:

1. Workspace-level rules (root `CLAUDE.md` / equivalent)
2. Repo `CLAUDE.md` / `AGENTS.md`
3. Local docs for the touched area
4. Formal spec system (e.g. `openspec/`) when the task needs a formal proposal
5. Related source code

Agents declare which convention docs they read. Verification evidence comes from each repo’s own scripts (lint, tests, typecheck).

## Agent launch rules

Each launch must specify:

- one clear objective
- bounded scope
- input artifacts
- output artifact path (e.g. `.ai/tasks/<id>/exploration.md`)
- stop conditions
- evidence requirements
- context loading instructions when working inside a repo

Agents write the **full report** to the artifact and return a **short summary** (verdict, key findings, blockers). The orchestrator should not pull full reports into chat unless a gate needs the detail.

### Parallelism

| Allowed in parallel | Never parallelize |
|---------------------|-------------------|
| Read-only agents writing **different** artifacts | Agents that edit the same code |
| One explorer per repo | Multiple implementers on same files |
| Plan-critic panel with distinct lenses | Characterization + implementation on same flow without coordination |

## Stop conditions

Stop and ask for a human/orchestrator decision when:

- product behavior is unclear
- data loss is possible
- rollback is not obvious
- unplanned architecture change is required
- tests reveal unexpected behavior
- blocking unknowns appear
- scope expands beyond the approved plan
- characterization is required but missing
- change conflicts with existing behavior

## Completion criteria

A task is complete only when:

- [ ] brief exists
- [ ] exploration is sufficient or explicitly skipped with reason
- [ ] impact is understood
- [ ] plan is approved
- [ ] characterization exists when required
- [ ] implementation handoff exists
- [ ] review passed
- [ ] state is updated
- [ ] final audit passed
- [ ] human review focus is documented

## Final user response

Keep it concise:

- what was completed
- what evidence exists
- remaining risks
- what the human should review
- next recommended action

Avoid dumping internal agent logs.
