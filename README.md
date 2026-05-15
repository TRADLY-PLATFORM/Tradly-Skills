# Tradly Skills Library

This repository contains public skill guides for building with Tradly APIs.

## Current Status

- Active skill:
  - `skills/tradly-api/SKILL.md`

## Folder Structure

This repository uses an Agent Skills-compatible structure:

```text
Tradly-Skills/
  README.md
  CONTRIBUTING.md
  skills/
    tradly-api/
      SKILL.md
    <future-skill>/
      SKILL.md
```

## Naming Convention

- One folder per skill under `skills/`
- Skill file name is always `SKILL.md`
- Skill folder name format: lowercase kebab-case
- Examples:
  - `skills/tradly-api/SKILL.md`
  - `skills/tradly-email/SKILL.md`
  - `skills/tradly-auth/SKILL.md`

## Skill Quality Checklist

Each `SKILL.md` should include:

- YAML frontmatter: `name`, `description`, `license`
- `When to use` and `When not to use`
- Required environment variables
- Exact API headers and auth rules
- Common tasks with runnable examples
- Error handling and troubleshooting
- Security notes (no hardcoded secrets)
