# Agent-Skills

A collection of agent skills that provide specialized knowledge and patterns for specific development scenarios.

## What are Skills?

Skills are markdown files that agents can load to enhance its capabilities when working on particular types of projects. Each skill contains patterns, code examples, and guidelines for a specific technology or architecture.

## Installation

Install skills using [skills.sh](https://skills.sh):

```bash
npx skills add akires47/agent-skills --skill dotnet-best-practices
```

Or reference directly in your agent configuration.

## Available Skills

| Skill                    | Description                                                                                                                                                                           | Rules |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| `dotnet-best-practices`  | Comprehensive .NET development guidelines covering error handling, async patterns, type design, database performance, API design, DI, architecture, serialization, performance, logging, and testing | 94    |

## Skill Structure

The `dotnet-best-practices` skill is organized into 11 priority-ranked categories:

1. **Error Handling** (CRITICAL) - Result pattern, validation, guard clauses
2. **Async Patterns** (CRITICAL) - CancellationToken, ValueTask, IAsyncEnumerable
3. **Type Design** (HIGH) - Records, immutability, pattern matching
4. **Database Performance** (HIGH) - NoTracking, projections, avoiding N+1
5. **API Design** (MEDIUM-HIGH) - Abstractions, return types, readonly collections
6. **Dependency Injection** (MEDIUM) - Extension methods, Options pattern
7. **Architecture** (MEDIUM) - Vertical slices, static handlers, minimal APIs
8. **Serialization** (MEDIUM) - System.Text.Json, Protobuf, wire compatibility
9. **Performance** (LOW-MEDIUM) - Span<T>, frozen collections, string handling
10. **Logging** (LOW-MEDIUM) - Structured logging, correlation IDs
11. **Testing** (LOW) - TestContainers, snapshot testing, AAA pattern

Each rule includes:
- Clear explanation of why it matters
- ❌ Incorrect code example
- ✅ Correct code example  
- Context and cross-references

## Acknowledgments

This collection was inspired by:
- [Aaronontheweb/dotnet-skills](https://github.com/Aaronontheweb/dotnet-skills) - Original .NET skills concept
- [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) - Rules-based skill format

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
