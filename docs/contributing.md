# Contributing

Contributions that improve clarity, safety, and portability of the workflow are welcome.

## Local development layout

```text
orchestrated-development/
  agents/           # edit agent prompts here
  skills/.../SKILL.md
  docs/
```

Use the symlink install from [installation.md](installation.md) so Claude Code picks up edits immediately.

## Guidelines

1. **Keep the orchestrator non-implementing.** The skill must not become a general coding agent.
2. **Preserve epistemic labels.** CONFIRMED / INFERRED / UNKNOWN are load-bearing.
3. **Prefer small agent scopes.** One objective, one artifact, clear stop conditions.
4. **Document gate changes.** If you change verdicts or iteration limits, update `docs/workflow.md`.
5. **Avoid product-specific hardcoding** in shared prompts. Domain examples (billing, warehouse) are fine as risk checklists; business rules should stay in the target repo’s docs.
6. **Test by dry-run.** Run a LIGHT triage task in a throwaway repo and confirm artifact creation and agent handoffs.

## Pull requests

- Describe the problem and which agents/skill sections change
- Note any behavior change for orchestrators already mid-task
- Keep diffs focused; separate docs-only PRs when possible

## Versioning

Bump `version` in `.claude-plugin/plugin.json` for meaningful releases:

- **Patch** — wording, docs, bugfix in prompts
- **Minor** — new optional phase, new agent, non-breaking workflow extension
- **Major** — breaking change to artifact names, required gates, or install layout

## License

By contributing, you agree your contributions are licensed under the MIT License (see [LICENSE](../LICENSE)).
