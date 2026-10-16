# Installation

This package is designed for **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)**.

It installs as:

1. One **skill** — `orchestrated-development`
2. Ten **agents** — specialized subagents used by the skill

## Prerequisites

- Claude Code CLI installed and authenticated
- Write access to either user config (`~/.claude/`) or the project’s `.claude/` directory

## Option A — User-level (global)

Available in every project on your machine.

```bash
git clone https://github.com/jd4n14/orchestrated-development.git
cd orchestrated-development

mkdir -p ~/.claude/skills ~/.claude/agents

cp -R skills/orchestrated-development ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```

Result:

```text
~/.claude/skills/orchestrated-development/SKILL.md
~/.claude/agents/task-explorer.md
~/.claude/agents/impact-analyzer.md
... (remaining agents)
```

## Option B — Project-level (repo-local)

Shared with teammates via git. Prefer this for team standards.

```bash
# From your application repository root
git clone --depth 1 https://github.com/jd4n14/orchestrated-development.git /tmp/orchestrated-development

mkdir -p .claude/skills .claude/agents
cp -R /tmp/orchestrated-development/skills/orchestrated-development .claude/skills/
cp /tmp/orchestrated-development/agents/*.md .claude/agents/
rm -rf /tmp/orchestrated-development

git add .claude
git commit -m "Add orchestrated-development skill and agents"
```

Or add this repo as a git submodule under something like `tools/orchestrated-development` and symlink/copy into `.claude/`.

## Option C — Symlink (development)

Useful when iterating on the skill itself:

```bash
git clone https://github.com/jd4n14/orchestrated-development.git ~/dev/orchestrated-development

mkdir -p ~/.claude/skills ~/.claude/agents

ln -sfn ~/dev/orchestrated-development/skills/orchestrated-development \
  ~/.claude/skills/orchestrated-development

for f in ~/dev/orchestrated-development/agents/*.md; do
  ln -sfn "$f" ~/.claude/agents/$(basename "$f")
done
```

## Verification

1. Restart Claude Code (or open a new session) so skills/agents are reloaded.
2. Ask Claude to list available skills/agents, or invoke:

   ```text
   Use the orchestrated-development skill for a dry-run brief only:
   "Add a healthcheck endpoint to the API"
   ```

3. Confirm it creates (or would create) a task structure under `.ai/tasks/` and does **not** jump straight into coding.

## Uninstall

User-level:

```bash
rm -rf ~/.claude/skills/orchestrated-development
rm -f ~/.claude/agents/{task-explorer,impact-analyzer,task-planner,plan-critic,characterization-test-designer,implementation-agent,contract-checker,review-agent,state-compressor,final-orchestrator-auditor}.md
```

Project-level: remove the corresponding files under `.claude/skills` and `.claude/agents`.

## Compatibility notes

- Agent frontmatter (`tools`, `model`) follows Claude Code’s agent format. Adjust tool names if your Claude Code version uses different identifiers.
- The skill sets `disable-model-invocation: true` so it is user-invoked rather than auto-triggered on every prompt. Remove or change that frontmatter field if you want automatic invocation.
- Task artifacts live under **`.ai/tasks/<task-id>/`** inside the **working project**, not inside this package.

## Updating

```bash
cd ~/dev/orchestrated-development   # or your clone path
git pull

# Re-copy (or re-run symlink setup)
cp -R skills/orchestrated-development ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```
