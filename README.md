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
