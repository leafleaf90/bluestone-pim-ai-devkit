---
name: bluestone-pim-pbc
description: Use when building, scaffolding, or debugging a Bluestone PIM PBC plugin.
             Triggers on: PBC, plugin, PluginParts, surface, metaInfo, pluginPartCreator,
             dashboard widget, fullPage, productPanelTab, gridAction.
---

# Bluestone PIM PBC Skill

Before writing any PBC code, read `reference.md` in full.

## When to invoke this skill
- User asks to scaffold, build, or modify a Bluestone PIM PBC
- User mentions any surface type (fullPage, dashboard, gridAction, productPanelTab, etc.)
- User asks about `metaInfo.json`, `pluginPartCreator`, or `providedEnv`
- User asks about `@bluestone-ext` packages

## Key rules to enforce always
- Always use `getAxiosInstance()` from `@bluestone-ext/plugin-framework` — never `axios` directly or `fetch`
- Never store secrets or API keys in plugin code — the bundle is accessible to all users
- Every part registered in `index.ts` must have a matching entry in `metaInfo.json`, and vice versa
- React is pinned to **16.14.0** — no React 17+ features
- External APIs must route through `/api/plugin-api/<plugin-id>/...`, never called directly
- Never use `console.error` for user-facing errors — use `providedEnv.utils.showSystemMessage` instead
- Never change the `id` field in `metaInfo.json` — it is assigned by Bluestone

## Reference material
See `reference.md` for the complete surface catalogue, props interfaces, component library,
`providedEnv` API, scaffold patterns, and review checklist.

## Related skills
When the PBC needs to call any Bluestone PIM API endpoint, always
also read the bluestone-pim-api skill. That skill contains the full
endpoint catalogue, correct /api/pim/ path prefixes, request shapes,
response shapes, and getAxiosInstance() usage examples. Never guess
an endpoint path — look it up in bluestone-pim-api/reference.md or
reference-full.md first.
