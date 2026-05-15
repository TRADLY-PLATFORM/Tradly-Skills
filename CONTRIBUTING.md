# Contributing Skills

## Naming

- One folder per skill under `skills/`
- Skill file name: always `SKILL.md`
- Skill folder name: lowercase kebab-case (example: `tradly-auth`)
- Frontmatter `name`: stable id matching folder name (example: `tradly-auth`)

## Writing Rules

- Keep scope focused (one major workflow per skill).
- Put exact headers, payload shape, and endpoint paths.
- Include at least one end-to-end runnable example.
- Prefer copy-paste-safe snippets over abstract pseudocode.
- Add a troubleshooting section for real failure modes.

## Validation Before Commit

1. Confirm frontmatter exists and is valid YAML.
2. Confirm no real API keys, tokens, or tenant IDs are present.
3. Confirm links and endpoint versions are current.
4. Confirm examples match the stated auth/header requirements.
