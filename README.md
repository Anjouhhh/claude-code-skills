# Claude Code Skills

A collection of skills for [Claude Code](https://claude.ai/code) — reusable workflows that Claude follows when invoked via the Skill tool.

## Skills

| Skill | Description |
|-------|-------------|
| [file-organizer](./file-organizer/SKILL.md) | Scans a directory and moves files into categorized subfolders by type, name pattern, and age. Supports preview mode and auto mode. |

## Usage

Invoke a skill using its name:

```
/file-organizer
/file-organizer --preview
/file-organizer --dir ~/Downloads
/file-organizer --auto --dir ~/Desktop
```

## Skill Structure

Each skill is a directory containing a `SKILL.md` file with YAML front matter and Markdown instructions:

```
skill-name/
└── SKILL.md
```

```yaml
---
name: skill-name
description: What the skill does and when to use it.
---

# Skill Title

Workflow instructions for Claude...
```

## Contributing

To add a new skill:

1. Create a directory with a kebab-case name (e.g., `my-skill/`)
2. Add a `SKILL.md` with YAML front matter (`name`, `description`) and a Markdown workflow
3. Add a row to the skills table above
