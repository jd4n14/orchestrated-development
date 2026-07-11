---
name: orchestrated-development
description: Runs a structured multi-agent development workflow for ambiguous, risky, legacy, or medium-to-large engineering tasks. Use to orchestrate exploration, impact analysis, planning, plan review, characterization tests, implementation, review, compression, and final audit while minimizing context usage and avoiding unsupported assumptions.
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

Every agent that analyzes, plans, implements, or reviews code inside a repo must be instructed to load context in this order before reading source code:

1. Workspace-level rules (root CLAUDE.md or equivalent), if the workspace has them.
2. CLAUDE.md / AGENTS.md of the affected repo, if they exist.
3. Relevant local documentation of the touched area.
4. The repo's formal spec system (for example an `openspec/` folder), if the task requires a formal proposal.
5. Related source code.

Agents must declare in their report which convention documents they read and which rules apply to the task, before proposing or modifying code.

Minimum verification evidence per repo is derived from that repo's own docs and package scripts (lint, tests, typecheck when available). The review agent must require that evidence.

## Default Folder Structure

For every non-trivial task, create or use a task folder:

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

If no task id is provided, create a short kebab-case id from the task name.

## Resume Protocol

Before creating a task folder, check whether `.ai/tasks/<task-id>/` already exists.

If it exists:

- Read state.md first.
- Read only the artifacts state.md marks as pending or relevant.
- Continue from the phase indicated by state.md.
- Do not re-run phases whose artifact is already approved unless new information changes the task.

Workflow
0. Triage

Classify the task before creating artifacts:

- LIGHT: well-understood area, existing coverage, single repo, no critical flows. Run: brief, short plan, implementation, review. Skip exploration, impact analysis, plan review, and characterization unless a step reveals risk.
- STANDARD: the default. Run the full workflow, but skip characterization tests if the impact report says they are not required.
- FULL: legacy code, critical business flows (billing, payments, state machines, external integrations), multiple repos, missing or unknown coverage, or irreversible data changes. Run every step.

Record the triage level and its justification in brief.md.

Any agent may escalate the triage level when it discovers new risk. Never downgrade the level without user approval.

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

The explorer must produce exploration.md.

If the task affects more than one repo, launch one task-explorer per repo in a single message so they run concurrently, then consolidate results into exploration.md.

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

the assigned phase
relevant brief excerpt
relevant impact excerpt
relevant plan excerpt
relevant characterization notes
explicit stop conditions

Do not give the implementation agent broad context unless necessary.

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

9. State Compression

Launch state-compressor after long phases, agent chains, or whenever context is getting large.

It must produce state.md.

Use state.md as the main memory artifact for future steps.

The chat history is not the source of truth. The artifacts are.

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
the context loading instructions from the Context Loading Protocol, when the agent works inside a repo

The agent writes its full report to the artifact path and returns only a short summary: verdict or recommendation, key findings, and blocking unknowns. Do not let full reports flow back through the conversation. Read an artifact yourself only when a gate decision requires the detail.

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
Agents write full reports to artifacts and return short summaries; keep it that way.
Prefer brief.md, state.md, and the specific artifact needed.
Send only relevant excerpts.
Split implementation into small phases.
Compress state after every major phase.
Do not ask agents to scan the whole codebase unless justified.
Prefer targeted file reads.
Do not re-run exploration unless new information changes the task.
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