# Agent-Skills

A collection of agent skills that provide specialized knowledge and patterns for specific development scenarios.

## What are Skills?

Skills are markdown files that agents can load to enhance its capabilities when working on particular types of projects. Each skill contains patterns, code examples, and guidelines for a specific technology or architecture.

## Available Skills

| Skill                   | Description                                                                       |
| ----------------------- | --------------------------------------------------------------------------------- |
| `dotnet-vertical-slice` | .NET 10 development using vertical slice (feature) architecture with minimal APIs |

## Skill Format

Each skill file in `skills/` follows this structure:

```yaml
---
name: skill-name
description: When and why to use this skill
---
# Skill Title

Content with patterns, code examples, and guidelines...
```
