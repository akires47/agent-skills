# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a collection of agent skills - markdown files that provide specialized knowledge and patterns for specific development scenarios. Skills are loaded to enhance its capabilities when working on particular types of projects.

## Structure

```
skills/           # Skill definition files (*.md)
```

Each skill file follows the format:

- YAML frontmatter with `name` and `description`
- Markdown body with patterns, code examples, and guidelines

## Adding New Skills

Skills must include frontmatter:

```yaml
---
name: skill-name
description: When and why to use this skill
---
```

The description should clearly indicate the trigger conditions for when the skill should be activated.
