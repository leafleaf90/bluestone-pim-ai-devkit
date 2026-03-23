# Bluestone PIM AI DevKit

A collection of [Agent Skills](https://skills.sh) that help developers build Bluestone PIM plugins (PBCs — Plugin-Based Components) faster and with fewer mistakes.

## What is this?

The Bluestone PIM AI DevKit packages the knowledge needed to build Bluestone PIM PBC plugins into structured skill files that any compatible AI coding agent can load during development. Instead of looking up surface types, props interfaces, API conventions, and component names while you code, your AI agent has them at hand and enforces them automatically.

## Requirements

- Any [Agent Skills](https://skills.sh)-compatible AI coding agent
- A Bluestone PIM plugin project based on the external plugin template

## Installation

### Via skills.sh

```bash
# Orchestrator — load this first; routes between the two skills below
npx skills add bluestone-pim/bluestone-pim-ai-devkit@bluestone-pim-orchestrator

# PBC plugin development skill
npx skills add bluestone-pim/bluestone-pim-ai-devkit@bluestone-pim-pbc

# API reference skill
npx skills add bluestone-pim/bluestone-pim-ai-devkit@bluestone-pim-api
```

> Once listed on [skills.sh](https://skills.sh), the exact install command will be shown on the listing page.

### Manual

Clone or download this repository:

```bash
git clone https://github.com/bluestone-pim/bluestone-pim-ai-devkit.git
```

Then reference the `skills/` directory in your Claude Code project config. Consult the [Claude Code documentation](https://docs.anthropic.com/claude-code) for the current configuration format, as it may evolve.

## What's Included

### `bluestone-pim-pbc`

Everything needed to build a Bluestone PIM PBC plugin.

**Activates when you:**
- Ask your agent to scaffold, build, or modify a PBC plugin
- Mention any surface type (`fullPage`, `dashboard`, `gridAction`, `productPanelTab`, etc.)
- Ask about `metaInfo.json`, `pluginPartCreator`, or `providedEnv`
- Ask about `@bluestone-ext` packages

**What it enforces:**
- Always uses `getAxiosInstance()` — never raw `axios` or `fetch`
- Routes external API calls through `/api/plugin-api/<plugin-id>/...`
- Keeps `index.ts` and `metaInfo.json` in sync
- Respects the React 16.14.0 constraint
- Uses `showSystemMessage` instead of `console.error`

[Full reference](skills/bluestone-pim-pbc/reference.md)

---

### `bluestone-pim-api`

A complete REST API reference for all Bluestone PIM endpoints a PBC plugin may call.

**Activates when you:**
- Ask how to call a Bluestone PIM API from a PBC
- Work with products, attributes, categories, relations, or variants via the API
- Use search, media assets, completeness scoring, labels, or tasks
- Need webhook/notification, sync, settings, or identity provider APIs

**What it provides:**
- Endpoint signatures, required parameters, and response types for all 16 API specs
- Ready-to-use `getAxiosInstance()` examples for every endpoint
- Covers: PIM core, Search, Media Bank, Tasks, History, Completeness, Labels, Metadata, Notifications, Sync, Page, Global Settings, UI Settings, IDP

[Full reference](skills/bluestone-pim-api/reference.md)

---

## Orchestrator

The orchestrator is a skill (`skills/bluestone-pim-orchestrator/`) that sits above the two
skills and handles routing between them. Install and load it the same way as the other skills:

```bash
npx skills add bluestone-pim/bluestone-pim-ai-devkit@bluestone-pim-orchestrator
```

It defines:
- When to use `bluestone-pim-pbc` alone (scaffolding, surface types, component questions)
- When to use `bluestone-pim-api` alone (endpoint lookups, request shapes)
- When to use both together (most real PBC work involving API calls)
- Rules that apply universally regardless of which skill is active

## Usage Example

Once installed, just describe what you want to build:

```
Scaffold a productPanelTab that fetches the current product's attributes from the Bluestone PIM API and displays them in a LabeledContainer list.
```

Your AI agent will generate code that:
- Uses the correct `CombinedProductTabProps` interface with `productInfo.id`
- Calls `getAxiosInstance()` routed through the proxy path
- Imports components from `@bluestone-ext/ui-components`
- Shows errors via `providedEnv.utils.showSystemMessage`
- Includes the matching `metaInfo.json` entry

## Resources

- [Bluestone PIM External Plugin Documentation](https://app.test.bluestonepim.com)
- [UI Component Library](https://ui-external.test.bluestonepim.com/)
- [Icon Library](https://icons.test.bluestonepim.com/)
- [Plugin Template Repository](https://bitbucket.org/bluestonepim)

## Contributing

Found an endpoint path or response shape that doesn't match reality?
Open a PR updating the relevant `.claude/skills/bluestone-pim-api/reference.md` or `reference-full.md`.
Real-world confirmed shapes are more valuable than spec-derived guesses.