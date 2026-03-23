# Bluestone PIM External Plugin — Developer Reference

A complete reference for building, registering, and shipping Bluestone PIM UI plugins (PBCs — Packaged Business Capabilities).

---

## 1. What is a Bluestone PBC?

A Bluestone PIM UI plugin (internally called a PBC — Plugin-Based Component) is a **React/TypeScript bundle** that is dynamically loaded by the Bluestone PIM UI at runtime based on the logged-in user's organization. The bundle is served either from `localhost` (dev) or a remote HTTPS server (staging/prod) and is fetched fresh on every page load.

**What it can do:**
- Render UI inside any of the predefined **surfaces** (slots) in the PIM UI
- Call any PIM API endpoint as the currently logged-in user
- Call external APIs via the platform proxy (with JWT passthrough)
- Interact with core UI via the `providedEnv` object (navigate, open modals, show toasts, reload grids)
- Read/write to `sessionStorage` for state that must survive modal transitions

**What it cannot do:**
- Access API keys or secrets from within the plugin code (bundle is public to all users)
- Host endpoints in Bluestone infrastructure
- Call external APIs directly — must go through the proxy path
- Use any PIM API endpoint not approved in the review process

**Runtime constraint:** React is pinned to **16.14.0** — hooks work, but some React 17+ features do not.

---

## 2. Project Structure

```
index.ts              ← Plugin entry point — registers all plugin parts
metaInfo.json         ← Plugin manifest — describes surfaces and metadata
src/PluginParts/      ← One file per surface implementation
package.json          ← Dependencies (all @bluestone-ext/* packages)
```

The plugin's `index.ts` always ends with:
```typescript
export default pluginPartCreator.exportPluginParts();
```

---

## 3. Vendor Packages

All private packages ship under the `@bluestone-ext` scope. In the portable distribution they are pre-packed tarballs in `vendor/`; in the standard template they resolve from AWS CodeArtifact.

| Package | Purpose | Key exports |
|---|---|---|
| `@bluestone-ext/plugin-framework` | Build tool + runtime framework | `pluginPartCreator`, `getAxiosInstance`, `isFilterSelection`, `isQuerySelection` |
| `@bluestone-ext/ui-components` | Bluestone component library | See §6 |
| `@bluestone-ext/types` | Shared TypeScript types | `ProvidedEnv`, `UserProfile`, `SystemMessage` |
| `@bluestone-ext/icons` | Icon components | Named exports per icon (e.g. `LightDelete`) |

**Node.js requirement:** 22.18 (enforced by `engines` in `package.json`).

---

## 4. All Plugin Surfaces

### 4.1 `fullPage` — Full-page view with left-nav button

**Use case:** Custom standalone tools, dashboards, data management pages.

**Props:** `CombinedFullPageProps` from `@bluestone-ext/plugin-framework/dist/definitions/InjectedProps`

Notable props: `providedEnv`, `pluginId`, `pluginPartId`

```typescript
import {CombinedFullPageProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

export default function MyPage(props: CombinedFullPageProps) {
    const {providedEnv} = props;
    return <div>Full page content</div>;
}
```

**Registration:**
```typescript
pluginPartCreator.createFullPagePlugin({id: 'fullpage-plugin-part-id', module: MyPage});
```

**metaInfo.json key:** `fullPage`

---

### 4.2 `settings` — Full-page view in the Settings menu

**Use case:** Plugin configuration, admin controls, data management.

**Props:** Same as `fullPage` — `CombinedFullPageProps`

```typescript
pluginPartCreator.createSettingsPlugin({id: 'settings-plugin-part-id', module: Settings});
```

**metaInfo.json key:** `settings`

---

### 4.3 `productPanelTab` — Tab in the product details panel

**Use case:** Display or edit product-specific data, enrich attributes, show related assets.

**Props:** `CombinedProductTabProps`

Notable props: `productInfo.id`, `productInfo.name`, `providedEnv`

```typescript
import {CombinedProductTabProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

function ProductDetailsTab({productInfo, providedEnv}: CombinedProductTabProps) {
    return <div>Product: {productInfo.id} — {productInfo.name}</div>;
}
```

**Registration:**
```typescript
pluginPartCreator.createProductTabPlugin({id: 'product-details-tab-plugin-part-id', module: ProductDetailsTab});
```

**metaInfo.json key:** `productPanelTab`

---

### 4.4 `catalogPanelTab` — Tab in the category details panel

**Use case:** Category-specific metadata, custom category configuration.

**Props:** `CombinedCategoryTabProps`

Notable props: `catalogId`, `catalog.name`, `providedEnv`

```typescript
import {CombinedCategoryTabProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

export default class CategoryDetailsTab extends React.Component<CombinedCategoryTabProps> {
    render() {
        const {catalogId, catalog} = this.props;
        return <div>{catalogId} — {catalog.name}</div>;
    }
}
```

**Registration:**
```typescript
pluginPartCreator.createCategoryTabPlugin({id: 'category-details-tab-plugin-part-id', module: CategoryDetailsTab});
```

**metaInfo.json key:** `catalogPanelTab`

---

### 4.5 `relationPanelTab` — Tab in the relation definition panel

**Use case:** Custom views for relation definitions, relationship metadata.

**Props:** `CombinedRelationTabProps`

Notable props: `relationId`, `relation.name`, `providedEnv`

```typescript
import {CombinedRelationTabProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

export default class RelationDetailsTab extends React.Component<CombinedRelationTabProps> {
    render() {
        const {relationId, relation} = this.props;
        return <div>{relationId} — {relation.name}</div>;
    }
}
```

**Registration:**
```typescript
pluginPartCreator.createRelationTabPlugin({id: 'relation-details-tab-plugin-part-id', module: RelationDetailsTab});
```

**metaInfo.json key:** `relationPanelTab`

---

### 4.6 `productDetailsAction` — Action menu item on product

**Not a React component** — a config object with a `functionModule`.

**Use case:** Trigger an action for the currently open product (e.g. export, sync, validate).

```typescript
import {ProductDetailsActionPluginConfig} from '@bluestone-ext/plugin-framework/dist/definitions/PluginPartCreator';
import {ProvidedEnv} from '@bluestone-ext/types';

export const productDetailsActionPlugin: ProductDetailsActionPluginConfig = {
    id: 'product-details-action-plugin-part-id',
    functionModule: {
        name: 'My Action',                          // label shown in the menu
        action: (productId: string, providedEnv: ProvidedEnv) => {
            // called when the user clicks the action
        },
        disabled: (productId: string, providedEnv: ProvidedEnv) => {
            return false;                           // return true to disable the item
        }
    }
};
```

**Registration:**
```typescript
pluginPartCreator.createProductDetailsActionPlugin(productDetailsActionPlugin);
```

**metaInfo.json key:** `productDetailsAction`

---

### 4.7 `gridAction` — Action for selected products in the grid

**Not a React component** — a config object with a `functionModule`.

**Use case:** Bulk operations on selected products (export, tag, publish, etc.).

The `selection` parameter tells you how products were selected: individual IDs, a saved filter, or a query.

```typescript
import {GridActionPluginConfig} from '@bluestone-ext/plugin-framework/dist/definitions/PluginPartCreator';
import {SelectionInformation} from '@bluestone-ext/plugin-framework/dist/definitions/GridSelectionInformation';
import {isFilterSelection, isQuerySelection} from '@bluestone-ext/plugin-framework/dist';
import {ProvidedEnv} from '@bluestone-ext/types';

export const gridActionPlugin: GridActionPluginConfig = {
    id: 'demo-grid-action-plugin-part-id',
    functionModule: {
        name: 'My Grid Action',
        shouldShow: (selection: SelectionInformation, providedEnv: ProvidedEnv) => {
            return !selection.isBrowsingArchived;  // hide when browsing archive
        },
        action: (selection: SelectionInformation, providedEnv: ProvidedEnv) => {
            if (isFilterSelection(selection)) { /* filter-based selection */ }
            else if (isQuerySelection(selection)) { /* query-based selection */ }
            else { /* individual IDs */ }
        }
    }
};
```

**Registration:**
```typescript
pluginPartCreator.createGridActionPlugin(gridActionPlugin);
```

**metaInfo.json key:** `gridAction`

---

### 4.8 `dashboard` — Dashboard widget

**Use case:** Summary metrics, KPIs, quick-access panels on the PIM dashboard.

Widgets can have a companion settings component (see §4.9) — the persisted settings are passed as `props.settings`.

**Props:** `CombinedDashboardProps<YourSettingsType>`

```typescript
import {CombinedDashboardProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

interface Settings { title: string; showCount: boolean; }
type Props = CombinedDashboardProps<Settings>;

export default class MyWidget extends React.Component<Props> {
    render() {
        const title = this.props.settings?.title ?? 'My Widget';
        return <div>{title}</div>;
    }
}
```

**Registration:**
```typescript
pluginPartCreator.createDashboardPlugin({
    id: 'demo-dashboard-widget-plugin-part-id',
    module: MyWidget,
    testId: 'demo-dashboard-widget-plugin-part-id'
});
```

**metaInfo.json key:** `dashboard`

---

### 4.9 `settingsForPluginPart` — Per-widget settings panel

**Use case:** Configuration form for a dashboard widget (or other plugin part). Paired with a `dashboard` part via matching IDs.

**Props:** `CombinedSettingsForPluginPartProps<YourSettingsType>`

Notable props: `settings` (current values), `onChangeSettings(newSettings)` (save callback)

```typescript
import {CombinedSettingsForPluginPartProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

interface Settings { title: string; }
type Props = CombinedSettingsForPluginPartProps<Settings>;

class MyWidgetSettings extends React.Component<Props> {
    render() {
        return (
            <InputFieldV2
                value={this.props.settings?.title ?? ''}
                onChange={(v) => this.props.onChangeSettings({title: v})}
            />
        );
    }
}
```

**Registration:**
```typescript
pluginPartCreator.createSettingsForPluginPart({
    id: 'demo-dashboard-widget-settings-plugin-part-id',
    module: MyWidgetSettings
});
```

**metaInfo.json key:** none directly — referenced by the dashboard part's settings ID

---

### 4.10 `fullPageModal` — Full-page modal overlay

**Use case:** Complex workflows that should appear as a modal over any surface (e.g. advanced editors, wizards). Not listed in `PLUGIN_SURFACES.md` — discovered from source.

Opened programmatically from any plugin part via `providedEnv.layout.modals.pluginFullpageModal.openPluginFullpageModal(pluginId, partId, options)`.

**Props:** `CombinedFullPageModalPluginProps`

```typescript
import {CombinedFullPageModalPluginProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

export function MyModalContent(props: CombinedFullPageModalPluginProps) {
    return <div>Modal content</div>;
}

// Footer actions are rendered separately
export const renderFooterActions = (isLoading, setLoadingState, providedEnv) => [
    <ButtonV2 onClick={() => providedEnv.layout.modals.pluginFullpageModal.closePluginFullpageModal()}>
        Close
    </ButtonV2>
];
```

**Registration:**
```typescript
pluginPartCreator.createFullPageModalPlugin({
    id: 'fullpage-modal-plugin-part-id',
    module: MyModalContent,
    renderFooterActions: renderFooterActions,
    testId: 'fullpage-modal-plugin-part-id',
    defaultTitle: 'Modal title'
});
```

**Opening from another plugin part:**
```typescript
const {openPluginFullpageModal, closePluginFullpageModal} = providedEnv.layout.modals.pluginFullpageModal;
openPluginFullpageModal('your-plugin-id', 'fullpage-modal-plugin-part-id', {
    title: 'Override title',        // optional
    subTitle: 'Subtitle',           // optional
    noAnimation: false,             // optional, useful when stacking modals
    onClickClose: () => { closePluginFullpageModal(); }  // optional override of X button
});
```

**metaInfo.json key:** `fullPageModal`

---

### 4.11 `mediaPreview` — Custom media asset preview

**Use case:** Render custom players or viewers for asset types (e.g. MP3 audio, custom video formats). Not listed in `PLUGIN_SURFACES.md` — discovered from source.

**Props:** `CombinedMediaPreviewProps`

Notable props: `mediaURL` — the URL of the asset to preview

```typescript
import {CombinedMediaPreviewProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';

export default function MediaPreviewMP3({mediaURL}: CombinedMediaPreviewProps) {
    return <audio controls src={mediaURL} style={{width: '100%'}} />;
}
```

**Registration:**
```typescript
pluginPartCreator.createMediaPreviewPlugin({
    id: 'media-preview-mp3-plugin-part-id',
    module: MediaPreviewMP3,
    testId: 'media-preview-mp3-plugin-part-id'
});
```

**metaInfo.json key:** `mediaPreview`

---

## 5. Minimal Scaffold

### `index.ts`

```typescript
import {pluginPartCreator} from '@bluestone-ext/plugin-framework';
import MyPage from './src/PluginParts/MyPage';

pluginPartCreator.createFullPagePlugin({id: 'my-page-part-id', module: MyPage});

export default pluginPartCreator.exportPluginParts();
```

### `src/PluginParts/MyPage.tsx`

```typescript
import React from 'react';
import {CombinedFullPageProps} from '@bluestone-ext/plugin-framework/dist/definitions/InjectedProps';
import {getAxiosInstance} from '@bluestone-ext/plugin-framework';
import {ButtonV2, CroppedText} from '@bluestone-ext/ui-components';

export default function MyPage({providedEnv}: CombinedFullPageProps) {
    const [count, setCount] = React.useState<number | null>(null);

    React.useEffect(() => {
        getAxiosInstance()
            .get('/api/tasks/tasks')
            .then((res) => setCount(res.data?.data?.length ?? 0))
            .catch(() => providedEnv.utils.showSystemMessage({text: 'Failed to load', type: 'error'}));
    }, []);

    return (
        <div style={{padding: 24}}>
            <CroppedText textType="HeadingH1">My Plugin</CroppedText>
            {count === null ? 'Loading...' : `Tasks: ${count}`}
            <ButtonV2 onClick={() => providedEnv.navigation.goToPath('/tasks')}>Go to Tasks</ButtonV2>
        </div>
    );
}
```

### `metaInfo.json`

```json
{
  "id": "your-plugin-id",
  "name": "My Plugin",
  "creator": "Your Company",
  "description": "A brief description shown in the PIM UI.",
  "structureVersion": "v2",
  "permissions": [],
  "contains": {
    "fullPage": [
      { "id": "my-page-part-id", "displayName": "My Plugin" }
    ]
  }
}
```

**Every entry in `contains` must have a matching `pluginPartCreator` call in `index.ts`, and vice versa.**

---

## 6. UI Component Library (`@bluestone-ext/ui-components`)

Browse the live demo at `https://ui-external.test.bluestonepim.com/`

Components seen in use across the template:

| Category | Components |
|---|---|
| **Layout** | `ScrollableView`, `LabeledContainer`, `Section` |
| **Typography** | `CroppedText` |
| **Buttons** | `ButtonV2`, `TextButton`, `ButtonWithDropdown` |

**ButtonV2 props** (confirmed from type definitions):
```tsx
// type is a plain string — NO static ButtonV2.Type enum
// testId is REQUIRED
<ButtonV2
    type="primary"       // 'primary' | 'secondary' | 'clear' | 'danger' | 'text' | 'tertiary'
    testId="my-button"   // required
    onClick={(e) => {}}  // required
    disabled={false}
>
    Label
</ButtonV2>
```
| **Actions** | `Action`, `Actions` |
| **Forms** | `InputFieldV2`, `TextareaV2`, `Checkbox`, `DatePickerV2`, `DateTimeEditorV2`, `InlineEditorV2`, `Switch`, `MarkdownEditor` |

**InputFieldV2 props** (confirmed from type definitions):
```tsx
// onChange receives the value string directly — NOT a React event
<InputFieldV2
    value={str}
    onChange={(value: string) => setState(value)}  // string, not event
    onPressEnter={(value: string) => submit()}      // Enter key — not onKeyDown
    onPressEsc={() => cancel()}                     // Escape key
    focusOnMount                                    // use instead of autoFocus
    placeholder="…"
    disabled={false}
/>
```

**Switch props** (confirmed from type definitions):
```tsx
<Switch
    value={booleanState}           // NOT "checked" — prop is "value"
    onChange={() => toggle()}      // no argument needed; component calls onChange with no useful args
    labelOnText="On"               // optional built-in label when on
    labelOffText="Off"             // optional built-in label when off
    disabled={false}               // optional
/>
```
| **Selects** | `SelectText` + `SelectOptionText`, `SelectWithIcon` + `SelectOptionWithIcon`, `MultiSelectText` + `MultiSelectOptionText`, `MultiSelectTextWithIcon` + `MultiSelectOptionWithIcon` |
| **Segmented** | `SegmentedControlV2` + `SegmentedControlV2Option` |
| **Dropdowns** | `DropDownMenuV2` + `DropDownItem` |
| **Feedback** | `MessageBox`, `BadgeV2`, `Spinner`, `EmptyState`, `PopoverV2` |
| **Navigation** | `Breadcrumbs`, `TabsV2` |
| **Data** | `BasicTable`, `PagingV2` |
| **Misc** | `UserPicture` |

**Pattern:** Select + Option components are always paired — the parent holds state, options are children.

---

## 7. Icons (`@bluestone-ext/icons`)

Browse the live demo at `https://icons.test.bluestonepim.com/`

```typescript
import {LightDelete, LightRefresh, LightTasks} from '@bluestone-ext/icons';

<LightRefresh color="DarkGrey70" width="24px" />
```

Each icon accepts only a predefined set of `color` and `width` values. Do not pass arbitrary strings.

---

## 8. API Requests

### The one rule: always use `getAxiosInstance()`

```typescript
import {getAxiosInstance} from '@bluestone-ext/plugin-framework';
```

Using any other HTTP client (`import axios from 'axios'`, `fetch`, etc.) will:
1. Break local dev proxying
2. Cause automatic rejection during Bluestone code review

### Calling Bluestone PIM APIs

Send to `/api/<service-prefix>/...` — the proxy authenticates as the current user. **Always include the service prefix.**

#### Confirmed PIM API path prefixes

| Service | Prefix | Example |
|---|---|---|
| PIM core (products, attrs, categories, relations, variants, bundles, assets, catalog) | `/api/pim/` | `/api/pim/products/{id}` |
| Tasks | `/api/tasks/` | `/api/tasks/tasks` |
| Query builder | `/api/query-builder/` | `/api/query-builder/products/count` |
| Search | `/api/search/` | `/api/search/products/search` |
| Media Bank | `/api/media-bank/` | `/api/media-bank/assets/{id}` |

> `/api/pim/` was confirmed via browser DevTools Network tab inspection — it does not appear in the public OpenAPI spec paths, which list endpoints without this prefix.

```typescript
// Fetch a product (note /api/pim/ prefix)
const {data} = await getAxiosInstance().get(`/api/pim/products/${productId}`, {
    headers: {'context': 'en', 'context-fallback': 'true'}
});

// Count tasks
const res = await getAxiosInstance().get('/api/tasks/tasks');
const numberOfTasks = res.data?.data?.length ?? 0;

// Product count via query builder
const {data} = await getAxiosInstance().post('/api/query-builder/products/count', {
    query: {
        matchRule: 'all',
        children: [
            {termType: 'updated_date', filterType: 'date_relative', unit: 'days_ago', value: 7}
        ]
    }
});
```

### Calling an external/custom backend

All external traffic goes through the proxy path — never call external URLs directly:

```
/api/plugin-api/<plugin-id>/<path>
```

Set `backendUrl` in `metaInfo.json` and the platform handles routing + JWT injection:

```typescript
// Calls your backendUrl + /my-endpoint
await getAxiosInstance().get('/api/plugin-api/your-plugin-id/my-endpoint');
```

For local dev, set `backendUrl` to `http://localhost:<port>` — the proxy forwards requests there automatically.

### MAPI vs PAPI — two API layers

Bluestone PIM exposes two distinct API layers:

| Layer | Prefix | Character |
|---|---|---|
| **MAPI** (Management API) | `/api/pim/`, `/api/tasks/`, `/api/query-builder/`, etc. | Internal API used by the PIM UI itself. Raw, ID-heavy responses. Confirmed via DevTools. |
| **PAPI** (Partner/Public API) | `/api/v1/` | Public OpenAPI-documented API. May return richer, more human-readable responses. Document confirmed endpoints here as discovered. |

> When designing a plugin feature, consider whether PAPI offers a richer response for the same data before defaulting to MAPI.

---

### MAPI attribute data model — `/api/pim/products/{id}/attributes`

MAPI attribute responses are intentionally raw. They contain **no human-readable names** — all labels, display values, and dictionary entries must be resolved separately.

#### Response shape

```typescript
interface RawAttribute {
    definitionId: string;    // ID of the attribute definition — name/label lives in the definition, not here
    values?: string[];       // Present for scalar types (text, number, boolean, date) AND select/multiselect.
                             //   - Plain text or number-as-string for simple types: "100", "some description"
                             //   - For single_select/multi_select: option IDs — resolve via definition.restrictions.enum.values (no extra API calls)
                             //   Mutually exclusive with `dictionary`
    dictionary?: string[];   // Present for dictionary attribute types. Always 24-char hex ObjectIDs —
                             // always require a separate resolution call to get display labels.
                             // Mutually exclusive with `values`
                             // Neither field present = attribute exists on product but has no value set
}

interface AttributesResponse {
    data: RawAttribute[];
}
```

#### Example response

```json
{
    "data": [
        { "definitionId": "abc123", "values": ["100"] },
        { "definitionId": "def456", "values": ["A long text description..."] },
        { "definitionId": "ghi789", "dictionary": ["id-of-dictionary-entry-1", "id-of-dictionary-entry-2"] },
        { "definitionId": "jkl000" }
    ]
}
```

#### To display human-readable attributes from MAPI you need at minimum:
1. `GET /api/pim/products/{id}/attributes` — raw values
2. `GET /api/pim/definitions` — to get human-readable names per `definitionId`
3. For `dictionary[]` entries only: one call per unique `{definitionId, valueId}` pair via `GET /api/pim/definitions/dictionary/{definitionId}/values/{valueId}`. For `single_select`/`multi_select`: options are already in the definitions response under `restrictions.enum.values` — **no extra calls**.

> **Key insight:** `dictionary[]` and `single_select`/`multi_select` (`values[]`) are resolved differently:
> - **`dictionary`** → one API call per value via `GET /api/pim/definitions/dictionary/{definitionId}/values/{valueId}`
> - **`single_select` / `multi_select`** → options are already in the definition under `restrictions.enum.values` — **zero extra calls**

#### Confirmed full resolution pattern:

```typescript
// Step 1: fetch attributes + definitions in parallel
const [attrsRes, defsRes] = await Promise.all([
    getAxiosInstance().get(`/api/pim/products/${productId}/attributes`, { headers }),
    getAxiosInstance().get('/api/pim/definitions', { headers })
]);
const attrs: RawAttribute[] = attrsRes.data?.data ?? [];
const defs: AttributeDefinition[] = defsRes.data?.data ?? [];

// Step 2: build select label map from definitions — no extra API calls
// Confirmed: options are at definition.restrictions.enum.values[].{ valueId, value }
const selectLabels: Record<string, string> = {};
const definitionMap: Record<string, AttributeDefinition> = {};
defs.forEach((def) => {
    definitionMap[def.id] = def;
    if (def.dataType === 'single_select' || def.dataType === 'multi_select') {
        def.restrictions?.enum?.values?.forEach((opt) => {
            selectLabels[opt.valueId] = opt.value;
        });
    }
});

// Step 3: resolve dictionary[] values — one call per unique {definitionId, valueId}
// CONFIRMED response shape: { value: { value: { [languageId]: string } } } — no data wrapper
const dictLabels: Record<string, string> = {};
const pairs: Array<{definitionId: string; valueId: string}> = [];
const seen = new Set<string>();
attrs.forEach((attr) => {
    attr.dictionary?.forEach((valueId) => {
        const key = `${attr.definitionId}/${valueId}`;
        if (!seen.has(key)) { seen.add(key); pairs.push({ definitionId: attr.definitionId, valueId }); }
    });
});
await Promise.all(
    pairs.map(async ({ definitionId, valueId }) => {
        try {
            const { data } = await getAxiosInstance().get(
                `/api/pim/definitions/dictionary/${definitionId}/values/${valueId}`,
                { headers: { context: languageId, 'context-fallback': 'true' } }
            );
            dictLabels[valueId] = data.value.value[languageId] ?? data.value.value['en'] ?? valueId;
        } catch { dictLabels[valueId] = valueId; }
    })
);

// Step 4: render
attrs.map((attr) => {
    if (attr.dictionary?.length) return attr.dictionary.map(id => dictLabels[id] ?? id).join(', ');
    if (attr.values?.length)     return attr.values.map(v => selectLabels[v] ?? v).join(', ');
    return '—';
});
```

#### Writing dictionary attribute values from a PBC

Two separate operations are needed — creating the value in the definition, then assigning it to the product:

```typescript
// Step 1: create new value in the dictionary definition
// Body: { value: string (required), number?: string, metadata?: string }
// New value ID comes back in the Resource-Id response HEADER (lowercase), not the body
const res = await getAxiosInstance().post(
    `/api/pim/definitions/dictionary/${definitionId}/values`,
    { value: labelText },
    { headers: { context: languageId, 'context-fallback': 'true' } }
);
const newValueId = res.headers['resource-id'];

// Step 2: assign to product — sends the FULL set of valueIds (replaces, not appends)
// Include all existing dictionary[] IDs plus the new one
await getAxiosInstance().post(
    `/api/pim/products/${productId}/attributes/dictionary/${definitionId}/values`,
    { valueIds: [...existingValueIds, newValueId] },
    { headers: { context: languageId, 'context-fallback': 'true' }, params: { forceVla: false } }
);
// Response: 204 No Content
```

> **Pitfall:** `POST .../products/.../dictionary/{defId}/values` **replaces** the full value set.
> Always send existing IDs + the new one — never just the new one alone.

> **Pitfall:** The `Resource-Id` header key is lowercase (`resource-id`). Access as `res.headers['resource-id']`.

To update an existing dictionary value's label:
```typescript
// PATCH — wrap the value field in { value: ... }
await getAxiosInstance().patch(
    `/api/pim/definitions/dictionary/${definitionId}/values/${valueId}`,
    { value: { value: newLabel } },
    { headers }
);
// Response: 204 No Content
```

To remove a value from the definition entirely:
```typescript
await getAxiosInstance().delete(
    `/api/pim/definitions/dictionary/${definitionId}/values/${valueId}`
);
// Response: 202 Accepted (async)
```

### Debugging unknown API paths

If an endpoint returns empty or unexpected results, or you are unsure of the correct path prefix, use browser DevTools → Network tab to inspect what path the Bluestone PIM UI itself calls for that data. That is the correct proxy path. This is the authoritative method — the public OpenAPI specs omit the `/api/<prefix>/` layer.

### Common pitfalls

| Anti-pattern | Fix |
|---|---|
| `import axios from 'axios'` | `import {getAxiosInstance} from '@bluestone-ext/plugin-framework'` |
| `getAxiosInstance().get('https://external.com/api')` | `getAxiosInstance().get('/api/plugin-api/<plugin-id>/path')` |
| Missing `backendUrl` in `metaInfo.json` | Add the correct URL for dev/prod |
| `/api/products/{id}` (no prefix) | `/api/pim/products/{id}` |
| Omitting `context` / `context-fallback` headers on PIM core calls | Add `headers: {'context': 'en', 'context-fallback': 'true'}` |
| Assuming MAPI `/api/pim/` attribute responses include human-readable names | They do not — `definitionId`, `values[]`, and `dictionary[]` are all raw IDs or untyped strings. Resolve definitions and dictionary entries separately, or consider using PAPI (`/api/v1/`) if it returns richer data for your use case. |
| Guessing response field names (`name`, `label`, `value`) when the schema is not in the reference | Check `reference-full.md` → "Confirmed Response Body Shapes" section first. If the type is not there, flag to the developer and ask them to verify in DevTools before shipping. |
| Accessing `data.name` on `DictionaryAttributeResponse` | Confirmed shape is `data.value.value[languageId]` — no `name` field, no `data` wrapper. |
| Assuming `values[]` in product attributes is always plain text | For `single_select` and `multi_select` definitions, `values[]` contains option IDs, not labels. Resolve by checking `definition.dataType` and looking up `definition.restrictions.enum.values[].{ valueId, value }` — the options are already in the definitions list, no extra API calls needed. |
| Calling the dictionary endpoint for `single_select`/`multi_select` values | Returns 404 — the dictionary endpoint only works for `dataType: "dictionary"` definitions. Select/multiselect options live on the definition itself under `restrictions.enum.values`. |

---

## 9. `providedEnv` Reference

Every React surface receives `providedEnv` in its props. Non-React surfaces receive it as a function argument.

```typescript
// Components
const {LanguageSelector, DetailPanel} = providedEnv.layout.components;

// Modals
providedEnv.layout.modals.productSelectionModal.openNewProductSelectionModal(params);
providedEnv.layout.modals.confirmModal.openConfirmModal({title, text, confirmText, cancelText, onConfirm, onCancel});
providedEnv.layout.modals.confirmModal.closeConfirmModal();
providedEnv.layout.modals.categorySelectionModal.openCategorySelectionModal(params);
providedEnv.layout.modals.mediaAssetSelectionModal.openAssetSelectionModal(params);
providedEnv.layout.modals.productsPreviewModal.openProductsPreviewModal(params);
providedEnv.layout.modals.pluginFullpageModal.openPluginFullpageModal(pluginId, partId, options);
providedEnv.layout.modals.pluginFullpageModal.closePluginFullpageModal();

// Products grid
providedEnv.layout.products.reloadProducts();
providedEnv.layout.products.reloadProductsWithPanels();

// Navigation
providedEnv.navigation.goToPath('/some/path');
providedEnv.navigation.goToProducts(filtersOrSavedView);
providedEnv.navigation.goToExternalJob(jobId);
providedEnv.navigation.goToMediaAssets(filtersOrSavedView);
providedEnv.navigation.goToCategorySettingsTab(categoryId);

// Query builder
providedEnv.queryBuilder.goToQueryBuilderWithSetup(params);

// Session
providedEnv.session.languageId;   // current language
providedEnv.session.user;         // logged-in user profile

// System messages (toasts) — use instead of console.error
providedEnv.utils.showSystemMessage({text: 'Done!', type: 'success'});
providedEnv.utils.showSystemMessage({text: 'Failed', type: 'error'});
providedEnv.utils.showSystemMessage({text: 'Note', type: 'info'});
providedEnv.utils.closeSystemMessage(messageRef);
```

**`DetailPanel` usage** — tabs with icons, actions, and content:

```tsx
<DetailPanel
    testId="my-panel"
    expandable
    closingCallback={() => setOpen(false)}
    detailsPanelContent={{
        header: 'Panel header',
        tabs: [{
            title: 'Tab 1',
            icon: <LightTasks color="DarkGrey70" />,
            content: <div>Tab content</div>,
            permissions: [],
            actions: [{icon: (ref) => <div ref={ref}><LightRefresh color="DarkGrey70" /></div>, onClick: fn, testId: 'x'}],
            addButtonConfig: {onAddClick: fn, addButtonTooltipText: 'Add item'},
            beforeActions: <div/>,
            afterActions: <div/>
        }]
    }}
/>
```

---

## 10. Key Conventions

### Must follow (enforced at review)

1. **Always use `getAxiosInstance()`** — never bypass with another HTTP client
2. **Never use `console.error` for user-facing errors** — use `showSystemMessage` instead
3. **Every plugin part registered in `index.ts` must have a matching entry in `metaInfo.json`** and vice versa
4. **Never change the `id` field in `metaInfo.json`** — it is assigned by Bluestone and must not be modified
5. **Never store API keys or secrets in the plugin** — the bundle is accessible to all users
6. **External API calls must route through `/api/plugin-api/<plugin-id>/...`** — never call external URLs directly

### Development workflow

- After any code save → **reload the browser** (the dev server rebuilds but the UI caches the old bundle)
- `npm run start` → dev server with watch + rebuild
- `npm run build` then `npm run serve` → static HTTPS-ready server on port 8188 (for remote testing)
- `npm run serve` does **not** watch for changes — rebuild explicitly

### State management between plugin parts

Use `sessionStorage` to pass data between a fullPage plugin and its modal (or other parts). This is the supported pattern — no shared global state mechanism exists outside of it.

### Versioning via PR title

| PR title contains | Version bump |
|---|---|
| `[MAJOR]` | Major — resets minor and patch to 0 |
| `[MINOR]` | Minor — resets patch to 0 |
| _(neither)_ | Patch only |

### Review checklist (Bluestone will check)

- [ ] `getAxiosInstance()` used for all API calls
- [ ] No excessive/polling API calls
- [ ] No infinite loops or memory leaks
- [ ] No `console.error` — replaced with `showSystemMessage`
- [ ] SonarQube scan passes
- [ ] Plugin loads and renders without errors
- [ ] `metaInfo.json` fields populated: `name`, `creator`, `description`
- [ ] `README.md` present (per `docs/README_TEMPLATE.md`)
- [ ] `id` in `metaInfo.json` unchanged

---

## 11. `metaInfo.json` — Complete Reference

```json
{
  "id": "plugin-id-assigned-by-bluestone",
  "name": "Human-readable name (shown in UI)",
  "creator": "Company or individual name",
  "description": "Max 500 characters. Shown next to the plugin in UI.",
  "structureVersion": "v2",
  "backendUrl": "http://localhost:4666",
  "needHelpLink": "https://optional-help-url.example.com",
  "permissions": [],
  "contains": {
    "fullPage":             [{"id": "...", "displayName": "..."}],
    "settings":             [{"id": "...", "displayName": "..."}],
    "productPanelTab":      [{"id": "...", "displayName": "..."}],
    "catalogPanelTab":      [{"id": "...", "displayName": "..."}],
    "relationPanelTab":     [{"id": "...", "displayName": "..."}],
    "productDetailsAction": [{"id": "...", "displayName": "..."}],
    "gridAction":           [{"id": "...", "displayName": "..."}],
    "dashboard":            [{"id": "...", "displayName": "..."}],
    "fullPageModal":        [{"id": "...", "displayName": "..."}],
    "mediaPreview":         [{"id": "...", "displayName": "..."}]
  }
}
```

---

## 12. Quick Reference

| Task | Command / Location |
|---|---|
| Install dependencies | `npm install` |
| Start dev server | `npm run start` |
| Build for production | `npm run build` |
| Serve built bundle remotely | `npm run serve` (port 8188) |
| Run tests | `npm run test` |
| Plugin entry point | `index.ts` |
| Plugin manifest | `metaInfo.json` |
| Surface components | `src/PluginParts/` |
| Component library demo | `https://ui-external.test.bluestonepim.com/` |
| Icon library demo | `https://icons.test.bluestonepim.com/` |
| Test environment | `https://app.test.bluestonepim.com` |
| Full docs index | `docs/DOCUMENTATION_INDEX.md` |
