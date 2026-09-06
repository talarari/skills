# Skills

A personal collection of [Claude Code skills](https://github.com/anthropics/skills).

## Structure

```
skills/
  <category>/
    <skill-name>/
      SKILL.md        # Skill definition (required)
      scripts/        # Supporting scripts (optional)
      references/     # Reference docs (optional)
```

## Usage

Copy a skill folder into one of:

- `~/.claude/skills/` — available in all projects
- `<project>/.claude/skills/` — scoped to a single project

## Skills

| Skill | Description |
|-------|-------------|
| [pr-interactive-review](skills/pr-interactive-review/) | Interactive code review walkthrough for a GitHub PR — fetches into a worktree, walks through changes step by step, and posts the review to GitHub |
| [playwright-sandbox](skills/playwright-sandbox/) | Fix Playwright/Chromium failing with `ERR_CONNECTION_RESET` in a Claude Code cloud sandbox — one command, plus a `SessionStart` hook to make it stick |
| [cloud-scheduled-runs](skills/cloud-scheduled-runs/) | Create recurring Routines in a Claude Code cloud session that actually fire and actually have repo access — session binding, UTC cron, unattended prompts, and how to verify one is really working |
