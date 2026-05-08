# Devicebase Skills

Claude Code skills for the [Devicebase](https://devicebase.cn) CLI — a cross-platform Go CLI for remote Android, HarmonyOS, and iOS device control via HTTP API.

## Available Skills

| Skill | Description |
|-------|-------------|
| [devicebase](devicebase/SKILL.md) | Devicebase CLI reference — tap, swipe, text input, app launching, screenshots, UI hierarchy inspection, and more |

## Usage

Install Devicebase CLI if not already installed:

```bash
which devicebase || curl -fsSL https://downloads.devicebase.cn/cli/install.sh | bash
```

Install skills to your Claude Code project or user directory:

```bash
# install skills
npx skills add https://github.com/devicebase/skills --skill devicebase

# Project-level (recommended)
cp -r ~/.agents/skills/devicebase /your-project/.laude-code/.claude/skills/

# User-level (available across all projects)
cp -r ~/.agents/skills/devicebase ~/.claude/skills/
```

Then activate in your Claude Code session by describing your task — the skill triggers automatically when working with mobile device automation.

## Links

- [Devicebase](https://devicebase.cn) — Remote device control platform
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) — How skills work in Claude Code
