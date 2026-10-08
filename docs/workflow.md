# Workflow

The **orchestrated-development** skill is the technical orchestrator. It does **not** implement production code. It creates artifacts, launches agents, and enforces gates.

## Task folder

Every non-trivial task gets a folder:

```text
.ai/tasks/<task-id>/
  brief.md
  context-pack.md
  exploration.md
  impact.md
  plan.md
  plan/phase-N.md
  plan-review.md
  characterization-tests.md
  implementation-handoff.md
  review.md
  state.md
  final-audit.md
```

If no id is given, use a short kebab-case name derived from the task.

EPIC-triage work adds an epic folder (`epic-brief.md`, `context-pack.md`, `roadmap.md`, `epic-state.md`) — see [epics.md](epics.md).

### Resume protocol

If `.ai/tasks/<task-id>/` already exists:

1. Read `state.md` first
2. Load only pending / relevant artifacts
3. Continue from the phase indicated by state
4. Do not re-run approved phases unless new information changes the task

If the task belongs to an epic, read `epic-state.md` first. Both state files end with a **Resume Command** — execute it instead of reconstructing context from chat.

## Triage levels

Classify before deep work. Record level + justification in `brief.md`.

| Level | When | Path |
|-------|------|------|
| **LIGHT** | Well-understood area, good coverage, single repo, no critical flows | brief → short plan → implement → review |
| **STANDARD** | Default | Full workflow; skip characterization unless impact requires it |
| **FULL** | Legacy, critical business flows, multi-repo, unknown coverage, irreversible data | Every step |
| **EPIC** | Long feature / multi-session change; plan would exceed ~6 phases | [Epic Protocol](epics.md): decompose into LIGHT/STANDARD sub-tasks, one per session |

Agents may **escalate** triage when they discover risk — including FULL → EPIC when the plan outgrows its sizing. Never **downgrade** without user approval.

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

- Produces `exploration.md` **and** `context-pack.md` (distilled conventions, file map, verification commands, landmines) — later agents read the pack instead of re-deriving repo conventions
- Multi-repo: one explorer **per repo**, launched concurrently, then consolidate into `exploration.md` and one pack with per-repo sections
- Do not plan if exploration has **blocking UNKNOWNs**

### 3. Impact → `impact-analyzer`

After exploration, before planning.

- Direct vs indirect impact
- Missing tests, legacy, integrations, persistence, queues, billing, warehouse, rollback
- If characterization is required → block implementation until tests exist

### 4. Planning → `task-planner`

Phased plan in `plan.md`. Each phase needs:

- objective, **difficulty (routine / hard)**, scope, out of scope
- expected files/modules
- steps, required tests, required evidence
- risks, stop conditions
- suggested agent, handoff requirements

The planner also shards the plan: each phase becomes a self-contained `plan/phase-N.md`. Phase file + context pack must be enough for the implementer — no other artifact required.

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
- Minimal context: `plan/phase-N.md` + `context-pack.md` + stop conditions — not the full plan
- Executor and model come from [model assignment](#model-assignment), using the phase's **difficulty** (routine / hard) and **risk** (normal / critical); the executor's permissions come from [role permissions](#role-permissions) whatever the model
- No scope expansion
- Update `implementation-handoff.md`

### 8. Contract check → `contract-checker`

After each implementation phase, before review, at every triage level. Cheapest gate in the workflow.

- Verifies every cross-boundary reference in the diff exists: HTTP calls ↔ routes, tables/columns ↔ real schema (read-only), queue/event producers ↔ consumers, config keys ↔ declarations
- Uses the phase file's **Contracts** field as checklist and the context pack's **Contract Sources** section to find definitions
- Runs lint / typecheck / tests and reports only failures, plus a diff fingerprint
- Compares the files in the diff with the phase's **allowed files**; any file outside it is `OUT OF SCOPE` and fails the check
- `FAIL` → straight back to the implementer; no review is spent on it. Fix cycles count toward the implementation gate limit

### 9. Review → `review-agent`

After each phase passes its contract check.

- Adversarial, from a clean context: tries to break the change (counterexamples, real data shapes, other callers, new tests run against the base commit)
- Receives the plan's decisions and the diff; the handoff only as claims to reproduce, never the implementer's reasoning or earlier verdicts
- Findings carry evidence and a status (`DEFECT` / `NEEDS CONTEXT`); it states expected behavior, never patches
- Reuse the contract check's command results when its diff fingerprint still matches; otherwise re-verify (diff, lint, tests, typecheck as defined by the repo)
- Same verdict model and gate iteration limit as plan review

### 10. State compression → `state-compressor` (checkpoints)

Mandatory on long tasks, at every checkpoint:

- after plan approval
- after every 2 implementation phases
- after any gate correction cycle
- before ending a session

Produces `state.md` — operational memory ending with a **Resume Command**. After each checkpoint, the session can be cut: `/clear` or a fresh session resumes from `state.md`. Prefer clean-session resumes over letting one session grow until auto-compaction.

### 11. Final audit → `final-orchestrator-auditor`

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

Only the **first** repo-working agent (normally the explorer) loads full context, in this order, before source code:

1. Workspace-level rules (root `CLAUDE.md` / equivalent)
2. Repo `CLAUDE.md` / `AGENTS.md`
3. Local docs for the touched area
4. Formal spec system (e.g. `openspec/`) when the task needs a formal proposal
5. Related source code

It distills the result into `context-pack.md`. Every later agent reads the pack first and goes back to source documents only for what the pack misses. Agents declare which sources they used. Verification evidence comes from each repo’s own scripts (lint, tests, typecheck), recorded in the pack.

## Agent launch rules

Each launch must specify:

- one clear objective
- bounded scope
- input artifacts
- output artifact path (e.g. `.ai/tasks/<id>/exploration.md`)
- stop conditions
- evidence requirements
- context loading instructions when working inside a repo

Agents write the **full report** to the artifact and return a **short summary** (verdict, key findings, blockers — max 15 lines). The orchestrator should not pull full reports into chat unless a gate needs the detail.

### Role permissions

A role is defined by what it may **decide**, not by what a model is good at. The model is a parameter of the role; a stronger model gets no extra permissions. Permissions are enforced with tools and output formats (read-only roles have no edit tools; every report has a place to hand a decision upward), not only with instructions.

**Decision levels**

- **D1 Product and data** — business rules, user-visible behavior, whether existing data is correct or intentional, data repairs. Owner: the user; the orchestrator proposes options.
- **D2 Architecture and contracts** — the approach to a fix, module boundaries, schema, public APIs, cross-repo contracts. Owner: the orchestrator, through planner and plan gate.
- **D3 Local design** — helper extraction, naming, test structure, inside the allowed files. Owner: the executor.

| Role | Agents | May | Must not | Stops and escalates when |
|------|--------|-----|----------|--------------------------|
| Recon | `task-explorer`, `impact-analyzer` | Read code, docs, read-only data; label facts; list options | Make D1/D2 decisions; choose the solution; judge whether data is intentional | Evidence points to a decision → **Decisions Needed**, options and evidence, no choice |
| Planner | `task-planner` | D2 decisions; close every decision the executor needs; name allowed files | D1 decisions; leave D2 open for the executor | A business rule is ambiguous |
| Gate | `plan-critic`, `final-orchestrator-auditor` | Approve, reject, demand changes | Edit plans or code | The verdict depends on a D1 decision |
| Characterizer | `characterization-test-designer` | Freeze current behavior in tests | Change production code | Current behavior looks wrong → record it, keep the test faithful |
| Executor | `implementation-agent` or external | D3 inside the allowed files | Reinterpret scope; touch other files; change D1/D2; commit | Contradiction, impossibility, or a false fact in the phase. Critical risk → stop; normal risk → narrower reading + report |
| Contract checker | `contract-checker` | Verify references, commands, and diff files vs allowed files | Judge design or intent; fix | — |
| Adversarial reviewer | `review-agent` | Break the change: counterexamples, base-commit tests, re-run every claim | Patch, redesign, continue the implementer's work, review its own phase | A finding needs context it lacks → **NEEDS CONTEXT** |
| Compressor | `state-compressor` | Summarize artifacts | Add decisions absent from the artifacts | — |
| Orchestrator | session | D2; integrate, verify, commit when asked | D1 without the user; push D2 onto an executor via a vague brief | — |

Facts in a brief or phase file (counts, names, paths, expected values) are **claims**: any role that can verify one cheaply does so and reports a false one instead of copying it.

**Clean context for the reviewer:** it receives the phase file, the plan's decisions (including other repos' phases), `contract-check.md`, the diff, and the handoff's claims labelled as claims. Never the implementer's reasoning, the orchestrator's opinion, or earlier verdicts. Clean of narrative, not of the contract.

### Model assignment

Decision roles multiply their errors downstream; volume roles do not; mechanical roles only need to be exact. **session** = the orchestration session's model (run it with the strongest reasoning model available), **mid** = sonnet, **cheap** = haiku, **external** = a non-Claude coding agent driven by an executor skill (e.g. a Cursor delegate), only when one is installed.

| Role | Default | Escalate to | Escalate when |
|------|---------|-------------|---------------|
| Orchestrator, Gate | session | — | — |
| Recon | mid | session | exploration of a FULL task returned blocking contradictions |
| Planner | mid | session | triage FULL, or an EPIC roadmap |
| Characterizer | mid | session | the flow under test is critical risk |
| Executor | external, else mid | session | phase is hard **and** critical; hard alone → executor's strongest setting or session |
| Contract checker | cheap | — | — |
| Adversarial reviewer | mid | session | phase is hard or critical, or triage FULL |
| Compressor | cheap | — | — |

- The cheap tier may also fill Recon for evidence-only questions (counts, lookups, tracing a value); the orchestrator still synthesizes before anything is decided.
- Escalate with the model override at launch. Record executor and model per phase in `state.md`.
- **Cross-family review:** if a Claude model implemented a critical phase, the external executor may add a second read-only review. If the implementer was external, `review-agent` already is the second family. The reviewer is never the instance that implemented the phase.
- Never downgrade a row below its default without user approval — one avoided correction cycle saves more than a cheaper model on a decision step.

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
