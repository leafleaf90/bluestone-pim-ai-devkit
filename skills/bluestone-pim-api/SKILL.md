---
name: bluestone-pim-api
description: Use when calling or referencing Bluestone PIM REST APIs from a PBC plugin.
             Triggers on: getAxiosInstance, API call, endpoint, fetch products, fetch attributes,
             search products, media-bank, completeness score, labels, tasks, history,
             notifications, sync, global-settings, ui-settings, IDP.
---

# Bluestone PIM API Skill

Before writing any API call, read the relevant section of `reference.md`.

## When to invoke this skill
- User asks how to call a Bluestone PIM API endpoint from a PBC
- User asks about product, attribute, category, relation, variant, or bundle APIs
- User asks about search, media-bank, completeness score, labels, or tasks APIs
- User asks about notifications, sync, global-settings, UI settings, or IDP APIs
- User wants to know what endpoints are available or what parameters an endpoint takes

## Key rules to enforce always
- Always use `getAxiosInstance()` — never raw `axios` or `fetch`
- Never hardcode environment URLs — `getAxiosInstance()` resolves the correct environment automatically based on where the plugin is running
- Always prepend the service prefix — the OpenAPI specs omit it, but PBC calls require it:
  - PIM core → `/api/pim/products/...`, `/api/pim/attributes/...` etc. _(confirmed via Network tab)_
  - Tasks → `/api/tasks/...`, Query builder → `/api/query-builder/...`, Search → `/api/search/...`, Media Bank → `/api/media-bank/...`
- Include `context` header (locale, e.g. `"en"`) and `context-fallback: true` on all PIM core calls — omitting these causes missing or incorrect localised values
- Never hardcode tokens, API keys, or credentials — `getAxiosInstance()` handles auth
- External (non-Bluestone) API calls must route through `/api/plugin-api/<plugin-id>/...`

## Reference material
`reference.md` — 90 PIM Core endpoints (products, attributes, categories, relations, variants,
bundles, assets, catalog nodes, attribute definitions). Covers the endpoints a PBC plugin
calls most often. Read only the section relevant to the current task.

`reference-full.md` — Complete catalogue of 175+ endpoints across all 16 API specs, including
Search, Media Bank, Tasks, History, Completeness Score, Labels, Metadata, Notifications,
Public API Sync, Page, Global Settings, UI Settings, and IDP. Use this when the user needs
an endpoint not covered in `reference.md`, or for the MCP server phase.

## Related skills
This skill is a companion to bluestone-pim-pbc. When an API endpoint
is identified, always verify the surface, props interface, component
imports, and metaInfo.json convention in the bluestone-pim-pbc skill
before generating code.
