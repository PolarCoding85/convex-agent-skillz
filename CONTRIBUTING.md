# Contributing to Convex Agent Skills

Thanks for your interest in contributing! This project provides Claude Code skills for Convex backend development, and contributions help keep them accurate and comprehensive.

## How to Contribute

1. Fork this repository
2. Create a branch for your changes (`git checkout -b my-change`)
3. Make your changes
4. Test by using the skills in a real Convex project
5. Submit a pull request with a clear description

## Skill Structure

Each skill lives in `.claude/skills/<skill-name>/` and follows this structure:

```
.claude/skills/<skill-name>/
├── SKILL.md              # Main entry point (required)
└── references/           # Topic-specific guides
    ├── TOPIC_A.md
    └── TOPIC_B.md
```

### SKILL.md Format

Every skill needs YAML frontmatter:

```yaml
---
name: skill-name
description: 'What this skill does and when to use it.'
---

# Skill Title

Instructions, patterns, and examples...
```

### Writing Guidelines

- **Keep SKILL.md under 500 lines** — move detailed content to `references/`
- **Use complete, runnable code examples** — not pseudocode
- **Include "Do" and "Don't" patterns** where helpful
- **Explain the "why"** not just the "how"
- **Keep content actionable** — Claude uses this to write code, not just explain concepts

## What Makes a Good Contribution

- Fills a gap in current Convex coverage
- Based on real-world usage patterns
- Tested against the current Convex version
- Includes working code examples

## Types of Contributions

- **New skills** — Cover a Convex topic not yet documented
- **Skill improvements** — Better examples, updated patterns, bug fixes
- **New reference docs** — Add depth to existing skills
- **Bug reports** — Open an issue if a skill gives incorrect guidance

## Code of Conduct

Be respectful and constructive. We're all here to make Convex development better.
