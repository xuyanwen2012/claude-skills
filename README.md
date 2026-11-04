# claude-skills

Agent skills for [Claude Code](https://claude.com/claude-code).

| Skill | What it does |
|---|---|
| [humanize](humanize/SKILL.md) | Run a task through [humanize (hmz)](https://github.com/humanfia/humanize), an agent-flow orchestrator that drives Claude Code, Codex and other agent CLIs in loops such as actor + reviewer (`rlar`). |
| [vk-kernel-wiki](https://github.com/xuyanwen2012/vk-kernel-wiki) (submodule) | Query a cross-referenced wiki on optimizing Vulkan GLSL compute-shader kernels for ML inference (cooperative matrix, quantized linear, attention) across AMD, Intel, NVIDIA, Qualcomm and Arm, with first-party measurements tied to an exact device, driver and date. |

## Install

Copy or symlink a skill directory into `~/.claude/skills/` (all projects) or
`<project>/.claude/skills/` (one project):

```sh
git clone --recurse-submodules https://github.com/xuyanwen2012/claude-skills
ln -s "$PWD/claude-skills/humanize" ~/.claude/skills/humanize
```

The humanize skill's defaults (models, budget) are the author's; edit the
"defaults" table to match yours.
