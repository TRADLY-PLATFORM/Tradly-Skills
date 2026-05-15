# Tradly Skills Library

This repository is a skills library for agent workflows around Tradly.

## Current Status

- Existing skill source: `tradly-api-SKILL.md`
- Library baseline added: discovery docs + reusable template

## Recommended Skill Layout

Use one folder per skill:

```text
skills/
  tradly-api/
    SKILL.md
  <next-skill>/
    SKILL.md
```

This keeps skills portable and easy for agents to scan.

## Skill Quality Checklist

Each `SKILL.md` should include:

- YAML frontmatter: `name`, `description`, `license`
- `When to use` and `When not to use`
- Required environment variables
- Exact API headers and auth rules
- Common tasks with runnable examples
- Error handling and troubleshooting
- Security notes (no hardcoded secrets)

## Next Steps

1. Move or copy `tradly-api-SKILL.md` into `skills/tradly-api/SKILL.md`.
2. Split large guides into focused skills when needed (auth, catalog, checkout, content/layers).
3. Keep examples framework-agnostic unless a section is explicitly framework-specific.
