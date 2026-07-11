# Artifacts

Artifacts under `.ai/tasks/<task-id>/` are the **source of truth**. Chat history is disposable.

## Default set

| File | Owner | Purpose |
|------|-------|---------|
| `brief.md` | Orchestrator | Goal, scope, triage, DoD, constraints |
| `exploration.md` | task-explorer | Facts, inferences, unknowns, flows |
| `impact.md` | impact-analyzer | Blast radius, risks, characterization need |
| `plan.md` | task-planner | Phased executable plan |
| `plan-review.md` | plan-critic | Plan gate verdict + findings |
| `characterization-tests.md` | characterization-test-designer | What was frozen and how |
| `implementation-handoff.md` | implementation-agent | What changed, tests, deviations |
| `review.md` | review-agent | Implementation gate verdict |
| `state.md` | state-compressor | Compressed operational memory |
| `final-audit.md` | final-orchestrator-auditor | Merge readiness |

Not every file is required for every triage level. LIGHT tasks may skip exploration, impact, plan-critic, and characterization when risk is low — document the skip and why.

## Design rules

1. **Write full reports to disk** — agents return short summaries in chat.
2. **Pass excerpts, not entire histories** — when launching the next agent, attach only the relevant sections.
3. **Update `state.md` after major phases** — next steps should start from state + needed artifacts.
4. **Never treat chat as durable memory** — if it matters, put it in an artifact.
5. **Version with the product** — consider committing `.ai/tasks/<id>/` for long-running work, or keep it local and gitignore if preferred (this package’s root `.gitignore` ignores `.ai/` for the skill repo itself; product repos can choose either way).

## Suggested brief.md skeleton

```markdown
# Brief: <task title>

## Objective
## User context
## Triage
- Level: LIGHT | STANDARD | FULL
- Justification:
## Affected repos
## Scope
## Non-scope
## Expected outcome
## Constraints
## Known risks
## Open questions
## Definition of done
```

## Suggested state.md skeleton

(Also enforced by `state-compressor`.)

```markdown
# Task State

## Current Objective
## Completed Work
## Active Phase
## Decisions
## Blockers
## Risks
## Pending Work
## Next Recommended Agent
## Open UNKNOWNs
```

## Verdict vocabulary

Used by plan-critic and review-agent:

| Verdict | Meaning |
|---------|---------|
| `APPROVED` | Proceed |
| `APPROVED WITH CHANGES` | Proceed only after listed fixes |
| `BLOCKED` | Do not proceed; fix or escalate |

Used by final-orchestrator-auditor:

| Verdict | Meaning |
|---------|---------|
| `READY TO MERGE` | Complete with evidence |
| `READY WITH KNOWN RISKS` | Acceptable residual risks documented |
| `REQUIRES MORE WORK` | Gaps remain |
| `BLOCKED` | Unsafe to ship |

## Multi-repo notes

When a task spans multiple repositories:

- Identify every affected repo in `brief.md`
- Prefer one explorer (and often one impact pass) **per repo**
- Consolidate into single task-level `exploration.md` / `impact.md` (or clearly named per-repo sections)
- Implementation phases should still be small and ideally single-repo when possible
