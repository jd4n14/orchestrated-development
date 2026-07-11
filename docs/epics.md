# Epics

**EPIC** is the triage level for long features and multi-session changes: days of work, a plan that would exceed roughly 6 implementation phases, or new surface area across repos.

The core problem with running a long feature through the single-task workflow is not correctness — it is that one session ends up holding the entire feature: every gate verdict, every handoff summary, every correction cycle. The session grows until auto-compaction, and the usage window is spent re-carrying context instead of doing work.

The Epic Protocol fixes this structurally: **no single session ever holds the whole feature.** Each session works on one bounded sub-task and hands off through artifacts.

## Folder

```text
.ai/epics/<epic-id>/
  epic-brief.md     objective, scope, constraints, affected repos, definition of done
  context-pack.md   distilled conventions, file map, verification commands, landmines
  roadmap.md        ordered sub-tasks with dependencies, sizing, and risk
  epic-state.md     cross-task memory: statuses, decisions, risks, resume command
```

Each sub-task gets a normal task folder named after the epic:

```text
.ai/tasks/<epic-id>-01-<name>/
.ai/tasks/<epic-id>-02-<name>/
```

## Protocol

1. **Epic brief** — same content rules as `brief.md`, at feature level.
2. **Macro exploration** — `task-explorer` (one per affected repo, concurrent) plus `impact-analyzer`, at feature level. The explorer also produces the epic's `context-pack.md`.
3. **Roadmap** — `task-planner` runs with a decomposition mandate: produce `roadmap.md` instead of a phased plan. Every sub-task must be sized LIGHT or STANDARD, be independently shippable or safely mergeable, and declare explicit dependencies, order, and risk. Phase-level detail belongs to each sub-task's own plan later, not to the roadmap.
4. **Roadmap gate** — `plan-critic` reviews the decomposition itself: sub-task boundaries, sizing, dependency order, merge safety. Same verdicts and iteration limits as plan review.
5. **Sub-task execution** — one at a time, in roadmap order. Each sub-task:
   - runs the standard workflow at its own triage level
   - receives pointers, not history: its roadmap entry, the epic `context-pack.md`, relevant macro exploration excerpts
   - never re-runs macro exploration; it explores only sub-task-specific gaps
   - on completion, updates `epic-state.md` (status, decisions, new risks, resume command)
6. **Session boundary** — prefer **one sub-task per session**. After `epic-state.md` is updated, recommend the user start a fresh session for the next sub-task. The new session resumes from `epic-state.md` and the next roadmap entry only.
7. **Epic audit** — when all sub-tasks are done, `final-orchestrator-auditor` runs at epic level, working from `epic-brief.md`, `roadmap.md`, `epic-state.md`, and each sub-task's `final-audit.md`.

## Re-planning

Escalation works both ways:

- A sub-task that outgrows its sizing **stops**, gets split in `roadmap.md`, and the roadmap change is re-gated with `plan-critic`.
- Completed work that invalidates the roadmap order means updating `roadmap.md` **before** continuing.

Never silently absorb extra scope into a running sub-task.

## Suggested roadmap.md skeleton

```markdown
# Roadmap: <epic title>

## Decomposition rationale

Why these boundaries; key dependency constraints.

## Sub-tasks

### 01-<name>
- Goal:
- Triage: LIGHT | STANDARD
- Depends on: none
- Risk: low | medium | high
- Definition of done:

### 02-<name>
...

## Order and milestones

Execution order, natural checkpoints, and what is shippable after each.

## Cross-task risks

Risks that span sub-tasks and who watches them.
```

## Suggested epic-state.md skeleton

```markdown
# Epic State: <epic-id>

## Sub-task Status
| Sub-task | Status | Audit verdict |
|----------|--------|---------------|

## Decisions
## Cross-task Risks
## Open Questions
## Resume Command
Exact instruction for a fresh session: next sub-task, artifacts to read.
```

## Optional: parallel sub-tasks

Sub-tasks with no dependency edge between them *can* run in parallel sessions using isolated git worktrees. This saves wall-clock time, not tokens, and it moves integration risk to merge time — use it only when sub-tasks touch disjoint files and the roadmap explicitly marks them independent. The default is sequential.

## Relationship to spec systems

If the product repo uses a formal spec system (for example OpenSpec), the epic maps naturally onto it: the proposal document is the source for `epic-brief.md`, and roadmap sub-tasks should reference the proposal's requirements instead of restating them.
