# Tradly Skills

Reusable skills for AI coding agents building on the Tradly platform.

> **Skills vs MCP:** Tradly Skills is for *building* — it gives your agent the knowledge and patterns to write correct Tradly-powered apps from the start. [Tradly MCP](https://tradly.app/mcp) is for *operating* — it lets agents query and manage a live marketplace in natural language. Use both.

## Available Skills

### Tradly API

Path: `skills/tradly-api/SKILL.md`

Use this skill to build marketplace, storefront, booking, directory, and commerce applications with the Tradly headless API platform. Covers auth flows, API patterns, product management, user flows, and payment processing.

## What you can build

- Online marketplace
- Booking & appointments platform
- Grocery or food delivery app
- Service marketplace
- Rental platform
- B2B wholesale store
- Freelance marketplace
- Subscription commerce

Tradly provides auth, listings, payments, escrow, email, orders, CMS, and a web editor out of the box — agents focus on what's unique to the product.

## Repository Structure

```text
Tradly-Skills/
  README.md
  CONTRIBUTING.md
  skills/
    tradly-api/
      SKILL.md
```

## Usage

Install or reference the skill from `skills/tradly-api/SKILL.md` in an agent environment that supports skills.

For Claude Code:

```bash
git clone https://github.com/TRADLY-PLATFORM/Tradly-Skills .tradly-skills
```

Then reference in your `CLAUDE.md`:

```
See .tradly-skills/skills/tradly-api/SKILL.md for Tradly API patterns and auth flows.
```

For Codex-style local usage:

```text
~/.codex/skills/tradly-api/SKILL.md
```

Additional Tradly skills may be added under `skills/` as separate folders.

## Related

- [Tradly MCP](https://tradly.app/mcp) — connect agents to a live marketplace via MCP (`@tradly/mcp`)
- [Build with AI agents](https://tradly.app/agents) — how to build marketplaces using AI and agents
- [Tradly API docs](https://developer.tradly.app) — full API reference
- [Tradly](https://tradly.app) — the headless commerce platform
