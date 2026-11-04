# claude-skills

Agent skills for [Claude Code](https://claude.com/claude-code).

| Skill | What it does |
|---|---|
| [humanize](humanize/SKILL.md) | Run a task through [humanize (hmz)](https://github.com/humanfia/humanize), an agent-flow orchestrator that drives Claude Code, Codex and other agent CLIs in loops such as actor + reviewer (`rlar`). |

## Install

Copy or symlink a skill directory into `~/.claude/skills/` (all projects) or
`<project>/.claude/skills/` (one project):

```sh
git clone https://github.com/xuyanwen2012/claude-skills
ln -s "$PWD/claude-skills/humanize" ~/.claude/skills/humanize
```

The humanize skill's defaults (models, budget) are the author's; edit the
"defaults" table to match yours.
