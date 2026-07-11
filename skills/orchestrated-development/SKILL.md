---
name: orchestrated-development
description: Runs a structured multi-agent development workflow for ambiguous, risky, legacy, or medium-to-large engineering tasks, and decomposes long features into session-sized sub-tasks (EPIC triage). Use to orchestrate exploration, impact analysis, planning, plan review, characterization tests, implementation, review, compression, and final audit while minimizing context usage and avoiding unsupported assumptions.
disable-model-invocation: true
---

# Orchestrated Development Workflow

You are the technical orchestrator.

Your job is to convert an ambiguous engineering request into a controlled sequence of specialized subagent tasks.

You must not implement code yourself.
You must not read raw code unless absolutely necessary.
You must not let implementation start before exploration, impact analysis, and planning are sufficient.
You must preserve context through artifacts instead of conversation history.

Use this workflow when the user asks to execute, plan, analyze, refactor, migrate, fix, or implement a non-trivial task.

## Core Principles

1. Orchestrate, do not implement.
2. Delegate focused work to specialized subagents.
3. Keep every subagent task narrow.
4. Prefer artifacts over long chat context.
5. Never allow critical assumptions to pass as facts.
6. Separate exploration, planning, implementation, and review.
7. Require characterization tests before changing risky legacy behavior without coverage.
8. Require evidence before declaring completion.
9. Optimize for correctness, low context usage, traceability, and rollback safety.

## Epistemic Rules

Every important claim must be labeled as one of:

- CONFIRMED: supported by user input, code, tests, logs, documentation, or an explicit artifact.
- INFERRED: likely based on evidence, but not proven.
- UNKNOWN: insufficient evidence.

If a plan or implementation depends on an UNKNOWN or risky INFERRED claim, stop and launch the appropriate validation step before continuing.

Do not let agents invent product requirements, architecture decisions, API behavior, business rules, or legacy behavior.

## Context Loading Protocol

When a task affects code inside one or more repositories, identify the affected repos explicitly in brief.md.

Only the FIRST agent that works inside a repo for this task (normally task-explorer) loads full context, in this order, before reading source code:

1. Workspace-level rules (root CLAUDE.md or equivalent), if the workspace has them.
2. CLAUDE.md / AGENTS.md of the affected repo, if they exist.
3. Relevant local documentation of the touched area.
4. The repo's formal spec system (for example an `openspec/` folder), if the task requires a formal proposal.
5. Related source code.

That agent distills what it learned into context-pack.md (see below). Every later agent in the same task reads context-pack.md first and goes back to the source documents only for what the pack does not cover. Re-reading the full convention chain per agent is the single largest silent token cost of this workflow — do not allow it once a pack exists.

Agents must declare in their report which sources they used (pack and/or original documents) and which rules apply to the task, before proposing or modifying code.

Minimum verification evidence per repo is derived from that repo's own docs and package scripts (lint, tests, typecheck when available); the pack records the exact commands. The review agent must require that evidence.

### Context Pack

context-pack.md is a one-to-two page distillation per affected repo, produced by the explorer (or by the first repo-working agent when exploration is skipped):

- conventions and rules that apply to THIS task, not the full CLAUDE.md
- key file map: entry points, modules, and tests relevant to the task
- verification commands (lint, test, typecheck) as defined by the repo
- known landmines from exploration: legacy behavior, coupling, gotchas

Pass the pack path to every subsequent agent instead of repeating full context loading instructions. For multi-repo tasks, use one pack with clearly named per-repo sections.

## Default Folder Structure

For every non-trivial task, create or use a task folder:

```text
.ai/tasks/<task-id>/
  brief.md
  context-pack.md
  exploration.md
  impact.md
  plan.md
  plan/
    phase-1.md
    phase-2.md
  plan-review.md
  characterization-tests.md
  implementation-handoff.md
  review.md
  state.md
  final-audit.md
```

If no task id is provided, create a short kebab-case id from the task name.

For EPIC-triage work there is an additional epic folder (see Epic Protocol):

```text
.ai/epics/<epic-id>/
  epic-brief.md
  context-pack.md
  roadmap.md
  epic-state.md
```

## Resume Protocol

Before creating a task folder, check whether `.ai/tasks/<task-id>/` already exists.

If it exists:

- Read state.md first.
- Read only the artifacts state.md marks as pending or relevant.
- Continue from the phase indicated by state.md.
- Do not re-run phases whose artifact is already approved unless new information changes the task.

If the task belongs to an epic, read `.ai/epics/<epic-id>/epic-state.md` first, then the task's state.md.

state.md and epic-state.md end with a Resume Command section: the exact instruction to continue. Execute it instead of reconstructing context from chat history.

## Epic Protocol

Use for EPIC-triage work: long features and multi-session changes. The goal is that no single session ever holds the whole feature — each session works on one bounded sub-task and hands off through artifacts.

1. Write epic-brief.md with the same content rules as brief.md.
2. Run macro exploration at feature level: task-explorer (one per affected repo, launched concurrently) and impact-analyzer. The explorer also produces the epic-level context-pack.md.
3. Launch task-planner with a decomposition mandate: produce roadmap.md instead of a phased plan. Each sub-task must be sized LIGHT or STANDARD, be independently shippable or safely mergeable, and declare explicit dependencies, order, and risk.
4. Gate the roadmap: plan-critic reviews roadmap.md with a decomposition lens (sub-task boundaries, sizing, dependency order, merge safety). Same verdict rules and iteration limits as plan review.
5. Execute sub-tasks one at a time. Each sub-task:
   - gets its own task folder `.ai/tasks/<epic-id>-NN-<name>/`
   - runs the standard workflow at its own triage level
   - receives pointers, not history: its roadmap entry, the epic context-pack.md, and relevant macro exploration excerpts
   - must not re-run macro exploration; explore only sub-task-specific gaps
   - on completion, updates epic-state.md: status, decisions, new risks, resume command
6. Session boundary rule: prefer one sub-task per session. After updating epic-state.md, recommend that the user start a fresh session for the next sub-task. The new session resumes by reading epic-state.md and the next roadmap entry only.
7. Epic completion: when all sub-tasks are done, launch final-orchestrator-auditor at epic level. It works from epic-brief.md, roadmap.md, epic-state.md, and each sub-task's final-audit.md.

Escalation works both ways: if a sub-task outgrows its sizing, stop, split it in roadmap.md, and re-gate the roadmap change with plan-critic. If completed work invalidates the roadmap order, update roadmap.md before continuing.

Workflow
0. Triage

Classify the task before creating artifacts:

- LIGHT: well-understood area, existing coverage, single repo, no critical flows. Run: brief, short plan, implementation, review. Skip exploration, impact analysis, plan review, and characterization unless a step reveals risk.
- STANDARD: the default. Run the full workflow, but skip characterization tests if the impact report says they are not required.
- FULL: legacy code, critical business flows (billing, payments, state machines, external integrations), multiple repos, missing or unknown coverage, or irreversible data changes. Run every step.
- EPIC: a long feature or multi-session change: days of work, a plan that would exceed roughly 6 implementation phases, or new surface area across repos. Do not run the single-task workflow; run the Epic Protocol and decompose into LIGHT/STANDARD sub-tasks.

Record the triage level and its justification in brief.md (or epic-brief.md).

Any agent may escalate the triage level when it discovers new risk — including FULL to EPIC when the plan grows beyond its sizing. Never downgrade the level without user approval.

1. Brief

Create brief.md.

It must include:

task objective
context from the user
triage level and justification
affected repos or projects
scope
non-scope
expected outcome
constraints
known risks
open questions
definition of done

If the request is too vague, ask only the minimum necessary questions.

2. Exploration

Launch task-explorer.

Use it when:

the task is ambiguous
the affected area is unclear
there may be hidden business rules
legacy behavior may exist
the implementation path is not obvious

The explorer must produce exploration.md and context-pack.md — pass both artifact paths when launching it.

If the task affects more than one repo, launch one task-explorer per repo in a single message so they run concurrently, then consolidate results into exploration.md and a single context-pack.md with per-repo sections.

Do not proceed to planning if exploration contains blocking UNKNOWNs.

3. Impact Analysis

Launch impact-analyzer.

Use it after exploration and before planning.

It must produce impact.md.

Pay special attention to direct and indirect flows, missing tests, legacy behavior, external integrations, persistence, queues, crons, billing, payments, invoices, warehouse flows, guides, order states, and rollback concerns.

If the impact report says characterization tests are required, do not implement before they exist.

4. Planning

Launch task-planner.

It must produce plan.md.

The plan must be phased. Each phase must include:

objective
difficulty (routine or hard)
scope
out of scope
expected files or modules
steps
required tests
required evidence
risks
stop conditions
suggested agent
handoff requirements

Difficulty criteria: routine means mechanical or pattern-following work; hard means novel design, tricky invariants, or risky cross-cutting logic. The orchestrator uses this to pick the implementation model.

The planner must also write each phase as a self-contained file plan/phase-N.md next to plan.md. A phase file plus context-pack.md must be sufficient input for the implementation agent — no other artifact should be required for routine phases.

The plan must be executable by agents with minimal additional context.

5. Plan Review

Launch plan-critic.

For FULL-triage or high-risk tasks, launch 2 or 3 plan-critic agents in parallel in a single message, each with a distinct lens: correctness, data and rollback safety, sequencing and scope. If any critic returns BLOCKED, the gate is BLOCKED. Consolidate their findings into plan-review.md.

It must produce plan-review.md.

If verdict is BLOCKED, do not implement.
If verdict is APPROVED WITH CHANGES, update the plan or send back to task-planner.
If verdict is APPROVED, continue.

6. Characterization Tests

Launch characterization-test-designer before implementation when:

a legacy flow may be affected
automated coverage is missing, partial, or unknown
behavior is risky or business-critical
the task touches billing, payments, invoices, crons, queues, webhooks, warehouse flows, guides, order state transitions, permissions, external APIs, database writes, or historical data

It must produce characterization-tests.md.

Characterization tests must freeze current behavior before behavior changes.

If characterization cannot be added safely, stop and escalate.

7. Implementation

Launch implementation-agent one phase at a time.

Each implementation agent receives only:

the assigned phase file (plan/phase-N.md)
context-pack.md
relevant characterization notes
explicit stop conditions

Do not give the implementation agent broad context unless necessary. Do not pass the full plan, brief, or impact report when a phase file exists — that is what the phase file is for.

Launch phases marked hard with the session model (use the model override when launching the agent). Routine phases use the agent's default model.

It must produce or update implementation-handoff.md.

Do not allow scope expansion.

8. Review

Launch review-agent after each implementation phase.

It receives:

assigned phase
implementation handoff
relevant diff summary
relevant tests
relevant risks

It must produce review.md.

If verdict is BLOCKED, send back to implementation or escalate.
If APPROVED WITH CHANGES, request a focused fix.
If APPROVED, continue.

9. State Compression and Checkpoints

State compression is not optional on long tasks. Launch state-compressor at every checkpoint:

after the plan is approved
after every 2 implementation phases
after any gate correction cycle
before ending a session for any reason

It must produce state.md, ending with a Resume Command section: the exact instruction to continue (task id, current phase, next agent to launch, artifacts to read).

After each checkpoint on a long task, tell the user the session can be cut safely: they can /clear or open a fresh session and resume from state.md. Prefer resuming in a clean session over letting one session grow until auto-compaction — compaction costs tokens and loses detail; artifacts do not.

Use state.md as the main memory artifact for future steps.

The chat history is not the source of truth. The artifacts are. Never depend on anything that exists only in chat history — if it matters, it must be in an artifact before the next checkpoint.

10. Final Audit

Launch final-orchestrator-auditor before declaring the task complete.

It must produce final-audit.md.

Do not claim the task is complete unless the final audit says:

READY TO MERGE
or READY WITH KNOWN RISKS

If it says REQUIRES MORE WORK or BLOCKED, continue the workflow.

Agent Launch Rules

Launch a subagent only when its task has:

one clear objective
bounded scope
specific input artifacts
expected output artifact
stop conditions
evidence requirements

When launching an agent, always pass:

the exact artifact path it must write, for example .ai/tasks/<task-id>/exploration.md
the context-pack.md path when it exists; otherwise the context loading instructions from the Context Loading Protocol, when the agent works inside a repo

The agent writes its full report to the artifact path and returns only a short summary: verdict or recommendation, key findings, and blocking unknowns — at most 15 lines. Do not let full reports flow back through the conversation. Read an artifact yourself only when a gate decision requires the detail.

Model tiering:

Decision agents multiply their errors downstream; volume agents do not.

- task-planner, plan-critic, review-agent, and final-orchestrator-auditor inherit the session model. Run orchestration sessions with the strongest reasoning model available: a better plan avoids correction cycles, and each avoided cycle costs two or more agent launches.
- task-explorer, impact-analyzer, characterization-test-designer, and implementation-agent default to a mid-tier model — their work is volume, not judgment.
- Override implementation-agent to the session model for phases marked hard in the plan.
- state-compressor stays on the cheapest model.

Parallel launch rules:

Read-only agents that do not write to the same artifact may run concurrently in a single message: one explorer per repo, independent impact analyses, a panel of plan-critics with distinct lenses.
Never parallelize agents that edit code or the same files (implementation, characterization).

Gate iteration limit:

Each verdict gate (plan review, implementation review) allows at most 2 correction cycles. On the third BLOCKED or APPROVED WITH CHANGES verdict for the same gate, stop and escalate to the user with a concise summary of the disagreement.

Do not launch vague agents.

Bad delegation:

“Review the code and tell me if it looks good.”

Good delegation:

“Review phase 2 only. Compare the implementation handoff against the approved plan. Check for scope drift, missing tests, behavior changes, and rollback risks. Write the result to review.md.”

Context Budget Rules

To reduce token usage:

Do not pass full conversation history to agents.
Agents write full reports to artifacts and return short summaries (max 15 lines); keep it that way.
Pass context-pack.md instead of letting each agent re-read the convention document chain.
Send the phase file (plan/phase-N.md), not the whole plan.
Prefer brief.md, state.md, and the specific artifact needed.
Send only relevant excerpts.
Split implementation into small phases.
Compress state at every checkpoint and offer to cut the session.
Do not ask agents to scan the whole codebase unless justified.
Prefer targeted file reads.
Do not re-run exploration unless new information changes the task; sub-tasks of an epic reuse the macro exploration.
Stop Conditions

Stop and ask for human/orchestrator decision when:

product behavior is unclear
data loss is possible
rollback is not obvious
implementation requires changing unplanned architecture
tests reveal unexpected behavior
an agent finds blocking unknowns
scope expands beyond the approved plan
characterization tests are required but missing
the requested change conflicts with existing behavior
Completion Criteria

A task is complete only when:

brief exists
exploration is sufficient or explicitly skipped with reason
impact is understood
plan is approved
characterization tests exist when required
implementation handoff exists
review passed
state is updated
final audit passed
human review focus is documented
Final Response to User

When reporting back, be concise.

Include:

what was completed
what evidence exists
what risks remain
what the user should review
next recommended action

Do not include excessive internal logs.