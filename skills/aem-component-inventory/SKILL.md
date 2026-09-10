---
name: aem-component-inventory
description: Scans an AEM project, resolves the full Sling inheritance chain of every component, and produces a single JSON file at .inventory/aem-component-inventory.json describing all editor-facing components and their fields. Use this skill before any AEM-to-headless-CMS migration work.
---

# Skill: aem-component-inventory

## Purpose

Scan an AEM project, resolve the full inheritance chain of every component, and produce a single JSON file — `.inventory/aem-component-inventory.json` — that gives an AEM-agnostic description of every component and its editable fields. This file is the single source of truth for downstream migration skills.

---

## Prerequisites

Before starting, orient yourself in the project:

1. Identify the `ui.apps` module root. Content lives under:
   ```
   ui.apps/src/main/content/jcr_root/apps/<project-namespace>/
   ```
   If the project is a single Maven module, the equivalent path is:
   ```
   src/main/content/jcr_root/apps/<project-namespace>/
   ```
2. Note the `<project-namespace>` (e.g. `myproject`, `mysite`) — this is the root of all custom components.
3. Identify any **Core Components** version in use. Check `pom.xml` for a dependency like `core.wcm.components.all`. Core Component dialog definitions live either under `apps/core/wcm/components/` in the repo, or must be fetched from the [Adobe Core Components GitHub repository](https://github.com/adobe/aem-core-wcm-components) matching the version declared in the POM.

---

## Step 1 — Discover All Components

A valid AEM component directory always contains a `.content.xml` file at its root with `jcr:primaryType="cq:Component"`.

**Search pattern:**
```bash
find . -name ".content.xml" | xargs grep -l 'jcr:primaryType="cq:Component"'
```

For each result, the parent directory is one component. Record:

| Field | Where to find it |
|---|---|
| `name` | `jcr:title` attribute in `.content.xml`, fallback to directory name — normalise to PascalCase (e.g. `"My Hero Banner"` → `MyHeroBanner`) |
| `componentGroup` | `componentGroup` attribute — use this to skip internal/hidden components (groups starting with `.` like `.hidden` or `wcm/foundation` are infrastructure, not editor-facing) |
| `resourceType` | The path relative to `/apps/`, e.g. `myproject/components/content/hero` |
| `sling:resourceSuperType` | Inheritance pointer — may be absent |
| `cq:isContainer` | `{Boolean}true` → this component accepts child components |

**Skip these:**
- Components with `componentGroup` starting with `.` (hidden/internal)
- Page/template components (group `wcm/foundation`, `cq/Page`)
- Parsys / layout container internals
- Components with no `_cq_dialog` anywhere in their inheritance chain (pure structural nodes)

---

## Step 2 — Resolve the Inheritance Chain

AEM uses **Sling Resource Type Inheritance** via `sling:resourceSuperType`. A component inherits all dialog fields from its parent unless it overrides them. This chain must be fully resolved to produce a complete field list.

**Algorithm for each component:**

```
chain = [component]
current = component

while current has sling:resourceSuperType:
    parent = resolve(sling:resourceSuperType)
    chain.append(parent)
    current = parent
```

**Resolving a `sling:resourceSuperType` value:**

- Value like `myproject/components/base/page` → look up at `apps/myproject/components/base/page/.content.xml`
- Value like `core/wcm/components/text/v2/text` → this is an **AEM Core Component**. Look it up under `apps/core/wcm/components/` if present in the repo, otherwise consult the Core Components source on GitHub for the matching version.
- Value like `wcm/foundation/components/...` → legacy WCM Foundation Component. Usually has minimal fields; treat as a terminal node.

**Merging fields across the chain:**

Fields defined in a child component's dialog **override** fields with the same `name` attribute from a parent. Merge from the bottom of the chain upward (child wins). The final merged set is the component's complete field surface.

---

## Step 3 — Parse Dialog Fields

Each component's editable fields are defined in:
```
<component-dir>/_cq_dialog/.content.xml
```

This is an XML file describing a JCR node tree. Fields are nested inside tab containers and layout containers. Traverse the entire node tree and collect every node whose `sling:resourceType` maps to a known field type (see mapping below). Skip pure layout/container nodes.

**Key XML attributes per field node:**

| Attribute | Meaning |
|---|---|
| Node name | Maps to the JCR property name saved on content (use as field name, cleaned up) |
| `fieldLabel` | Human-readable label — use as the canonical field name if cleaner than the node name |
| `name` | The actual JCR property path written on save (e.g. `./linkTarget`). Strip leading `./` for the field name |
| `required` | `{Boolean}true` → mark as required |
| `multiple` | `{Boolean}true` on a pathfield/fileupload → treat as array |

**Granite UI → TypeScript type mapping:**

| `sling:resourceType` (partial match) | TypeScript type |
|---|---|
| `form/textfield` | `string` |
| `form/textarea` | `string` |
| `form/numberfield` | `number` |
| `form/checkbox` | `boolean` |
| `form/switch` | `boolean` |
| `form/select` | Enumerate the `items` child nodes → `"value1" \| "value2" \| ...` |
| `form/radiogroup` | Same as select — enumerate values |
| `form/pathfield` | `string` (path) |
| `form/fileupload` | `Image Asset` |
| `form/datepicker` | `string` (ISO date) or `string` (ISO datetime) depending on `type` attribute |
| `form/colorfield` | `string` (color) |
| `form/multifield` | `Array<{ … }>` — recursively describe the nested composite field |
| `form/hidden` | **Skip** — not editor-facing |
| `dialog/richtext` or `rte` | `Richtext` |
| `asset/picker` or `dam/…picker` | `Image Asset` |
| `form/autocomplete` | `string` |
| `form/tagsautocomplete` | `string[]` (tags) |
| `granite/ui/…/include` | Resolve the included dialog fragment and inline its fields |

**Field name normalisation rules:**

- Strip leading `./` from `name` attribute
- `jcr:title` → `title`
- `jcr:description` → `description`
- `cq:tags` → `tags`
- `fileReference` → `image` (or keep if context is clearer)
- Convert all resulting field names to camelCase: split on hyphens, underscores, and spaces, lowercase the first word, capitalise the first letter of each subsequent word (e.g. `link-target` → `linkTarget`, `my_field_name` → `myFieldName`)
- Already-camelCase names stay as-is
- If a node has no `fieldLabel` and the node name is a JCR namespace like `jcr:*`, apply the shorthand above or invent a readable camelCase name that describes the field's purpose

---

## Step 4 — Classify Container Components

Components with `cq:isContainer="true"` (found in Step 1) accept child components at authoring time. They fall into two categories that must be distinguished because they produce fundamentally different content structures:

### Panel containers

**Panel containers** group children into named *items*, each carrying a `cq:panelTitle` (the heading/label visible to end-users). The child components live *inside* each item, not directly under the container itself. This means the panel title is **content** — not just layout metadata — and must be captured in the inventory.

A component is a panel container when its own resource type, or any resource type in its `sling:resourceSuperType` chain, matches one of these patterns:

| Pattern (glob) | Component |
|---|---|
| `*/accordion/v*/accordion` | Accordion |
| `*/tabs/v*/tabs` | Tabs |
| `*/carousel/v*/carousel` | Carousel |

### Layout containers

All other `cq:isContainer="true"` components are **layout containers** (Container, parsys, responsivegrid). Their children are direct component instances with no item-level metadata. These are traversal wrappers only — they are not emitted as component instances in the page inventory.

### Output

Set four fields on the component entry:

| Field | Value |
|---|---|
| `isContainer` | `true` when `cq:isContainer="true"` on the component or any ancestor in the supertype chain; `false` otherwise |
| `containerType` | `"panel"` for Accordion/Tabs/Carousel; `"layout"` for all other containers; `null` if not a container |
| `panelItemFields` | Only for `containerType: "panel"`. An array describing the metadata fields available on each child item node. At minimum: `[{ "name": "panelTitle", "type": "string", "source": "cq:panelTitle or jcr:title", "note": "Heading displayed to end-users for this panel item" }]`. If a specific panel container defines additional item-level attributes, include those too. `null` for non-panel containers. |
| `allowedComponents` | Only for `isContainer: true`. A string array of resource types (without leading `/apps/`) that are allowed as child components, resolved from template policies (see Step 4b below). `null` if no policy is found or no allowed-component list is configured. |

### Step 4b — Resolve Allowed Components from Template Policies

For every container component, look up which component types are allowed inside it. This information lives in the editable-template policy store, **not** in the component definition itself.

**Where to look:**

```
ui.content*/src/main/content/jcr_root/conf/<project>/settings/wcm/policies/.content.xml
```

Also check any template-specific policy mappings:
```
conf/<project>/settings/wcm/templates/<template-name>/policies/.content.xml
```

**Algorithm:**

1. Derive the policy node name from the component's resource type — it is the **last path segment** of the `resourceType` (e.g. `wknd/components/carousel` → `carousel`).
2. In the policies XML, find an element with that tag name (e.g. `<carousel …>`). It may be nested at any depth.
3. Inside that element, find the child policy node (usually named `policy_<timestamp>`). Read the `components` attribute — it is a JCR multi-value string in the format `[/apps/path/one,/apps/path/two]`.
4. For each value:
   - If it starts with `/apps/`, strip the prefix to get the resource type (e.g. `/apps/wknd/components/teaser` → `wknd/components/teaser`).
   - If it starts with `group:`, it references a component group by name (e.g. `group:WKND.Content`). Resolve it by finding all components whose `componentGroup` attribute in their `.content.xml` matches the group name, and add their resource types individually.
   - Values starting with `/libs/` are WCM foundation components — include them as-is without the `/libs/` prefix only if they are also present in the inventory; otherwise skip them.
5. Set `allowedComponents` to the resulting deduplicated array.

If multiple policy nodes exist under the same component name (multiple templates may configure the same component differently), **union** all allowed component lists and deduplicate.

If no policy entry is found, or if the policy element exists but is empty, set `allowedComponents: null`.

---

## Step 5 — Identify the Template File

Each component typically has one or more HTL (`.html`) template files. Record the primary one (same name as the component directory, or `<component-name>.html`).

```
<component-dir>/<component-name>.html   ← primary template
<component-dir>/<variant>.html          ← variants (note separately if present)
```

---

## Step 6 — Write the Inventory File

Output path:
```
<project-root>/.inventory/aem-component-inventory.json
```

Create the `.inventory/` directory if it does not exist.

### File Format

```json
{
  "meta": {
    "project": "<project-namespace>",
    "totalComponents": 0,
    "generatedOn": "YYYY-MM-DD"
  },
  "components": [
    {
      "name": "ComponentName",
      "resourceType": "apps/project/components/content/componentname",
      "template": "relative/path/to/template.html",
      "dialog": "relative/path/to/_cq_dialog/.content.xml",
      "inheritsFrom": ["sling:resourceSuperType", "grandparent/resource/type"],
      "isContainer": false,
      "containerType": null,
      "panelItemFields": null,
      "allowedComponents": null,
      "fields": [
        {
          "name": "fieldName",
          "type": "string",
          "required": true,
          "inherited": false,
          "inheritedFrom": null,
          "note": null
        },
        {
          "name": "anotherField",
          "type": "enum",
          "enumValues": ["option1", "option2"],
          "required": false,
          "inherited": false,
          "inheritedFrom": null,
          "note": null
        },
        {
          "name": "image",
          "type": "ImageAsset",
          "required": false,
          "inherited": true,
          "inheritedFrom": "core/wcm/components/image/v3/image",
          "note": null
        },
        {
          "name": "body",
          "type": "Richtext",
          "required": false,
          "inherited": false,
          "inheritedFrom": null,
          "note": null
        },
        {
          "name": "items",
          "type": "object[]",
          "required": false,
          "inherited": false,
          "inheritedFrom": null,
          "note": null,
          "itemFields": [
            { "name": "label", "type": "string", "required": true },
            { "name": "url", "type": "string (path)", "required": true }
          ]
        }
      ]
    }
  ],
  "excludedComponents": [
    {
      "name": "Component Display Name",
      "resourceType": "project/components/page",
      "reason": "Page template component, group `.hidden`"
    }
  ]
}
```

**Formatting rules:**
- `components` array is sorted alphabetically by `name`.
- `template` and `dialog` are `null` when not present (e.g. pure core-inheriting components).
- `inheritsFrom` is an array of resource type strings in chain order (child first), or `[]` if no inheritance.
- `isContainer` is `true` when the component (or any ancestor in its supertype chain) has `cq:isContainer="true"`; `false` otherwise.
- `containerType` is `"panel"` for Accordion/Tabs/Carousel, `"layout"` for Container/parsys/responsivegrid, `null` for non-containers.
- `panelItemFields` is an array of field descriptors for panel container item metadata (e.g. `panelTitle`); `null` for non-panel containers.
- `allowedComponents` is a string array of resource types (without `/apps/` prefix) that the template policy permits inside this container. `null` for non-containers or when no policy is found.
- `required` is `true` only when the dialog XML explicitly marks the field as required; all other fields are `false`.
- `inherited` is `true` when the field comes from a parent in the `sling:resourceSuperType` chain. `inheritedFrom` names that parent's resource type.
- `note` carries any inline comment about the field (e.g. `"displayed as caption"`, `"datasource-driven"`). Use `null` when there is nothing to annotate.
- `type` values: `"string"`, `"number"`, `"boolean"`, `"string (path)"`, `"string (ISO date)"`, `"string (ISO datetime)"`, `"string (color)"`, `"string[]"`, `"ImageAsset"`, `"Richtext"`, `"enum"`, `"object[]"`.
- When `type` is `"enum"`, include an `"enumValues"` array listing all valid string values.
- When `type` is `"object[]"`, include an `"itemFields"` array of field objects (same shape, without `inherited`/`inheritedFrom`/`note`).
- If a component has no dialog at all (structure/design-only), emit an empty `"fields": []` array.

---

## Step 7 — Quality Control (Required Final Step)

After writing the file, re-read it in full and perform the following checks. Fix any issues found before considering the skill complete.

### Checklist

**Coverage:**
- [ ] Every directory matching `jcr:primaryType="cq:Component"` in the project is either represented in the inventory or explicitly listed in `excludedComponents` with a `reason`.
- [ ] No component in `components` has an empty `fields` array without a structural reason.

**Inheritance:**
- [ ] Every component with a `sling:resourceSuperType` has had its parent chain resolved. If a parent could not be located (e.g. external dependency not in the repo), add a field entry with `"name": "_WARNING"` and `"note": "parent dialog [resourceType] not resolved — fields may be incomplete"`.
- [ ] Core Component fields are present where inheritance applies, with `inherited: true` and `inheritedFrom` set correctly.

**Containers:**
- [ ] Every component with `cq:isContainer="true"` (directly or via supertype) has `isContainer: true`.
- [ ] Accordion, Tabs, and Carousel have `containerType: "panel"` and a non-null `panelItemFields` array.
- [ ] Container, parsys, responsivegrid components have `containerType: "layout"`.
- [ ] Non-container components have `isContainer: false`, `containerType: null`, `panelItemFields: null`, `allowedComponents: null`.
- [ ] Every container component has an `allowedComponents` field. For containers with a matching policy entry, it must be a non-empty string array. If no policy is found, `null` is acceptable — but add a `_WARNING` field noting that allowed components could not be resolved from policy.

**Field Types:**
- [ ] No field has `type` set to a raw Granite `sling:resourceType` string — every field must use one of the defined type values.
- [ ] All `form/select` and `form/radiogroup` fields use `type: "enum"` with a populated `enumValues` array, not `type: "string"`.
- [ ] Multifield shapes use `type: "object[]"` with a fully expanded `itemFields` array.

**Naming:**
- [ ] No `name` value in any `fields` entry contains JCR namespace prefixes (`jcr:`, `cq:`, `./`) — all have been normalised.

**File integrity:**
- [ ] `meta.totalComponents` matches the actual length of the `components` array.
- [ ] The file is valid JSON (parseable without errors).
- [ ] The output file exists at exactly `.inventory/aem-component-inventory.json` relative to the project root.

### After QC

If any check fails, correct the inventory file in place. Re-run the checklist after corrections. Only mark the skill as done when all checks pass or are explicitly noted as unresolvable (with a reason).

---

## Reference: Common AEM Core Component Resource Types

Use this table when resolving Core Component inheritance to find the right dialog source.

| Component | Core Resource Type |
|---|---|
| Text | `core/wcm/components/text/v2/text` |
| Title | `core/wcm/components/title/v3/title` |
| Image | `core/wcm/components/image/v3/image` |
| Button | `core/wcm/components/button/v2/button` |
| Teaser | `core/wcm/components/teaser/v2/teaser` |
| List | `core/wcm/components/list/v4/list` |
| Breadcrumb | `core/wcm/components/breadcrumb/v3/breadcrumb` |
| Navigation | `core/wcm/components/navigation/v2/navigation` |
| Language Navigation | `core/wcm/components/languagenavigation/v2/languagenavigation` |
| Search | `core/wcm/components/search/v2/search` |
| Carousel | `core/wcm/components/carousel/v1/carousel` |
| Tabs | `core/wcm/components/tabs/v1/tabs` |
| Accordion | `core/wcm/components/accordion/v1/accordion` |
| Container | `core/wcm/components/container/v1/container` |
| Content Fragment | `core/wcm/components/contentfragment/v1/contentfragment` |
| Download | `core/wcm/components/download/v2/download` |
| Embed | `core/wcm/components/embed/v2/embed` |
| PDF Viewer | `core/wcm/components/pdfviewer/v2/pdfviewer` |
| Progress Bar | `core/wcm/components/progressbar/v1/progressbar` |
| Separator | `core/wcm/components/separator/v1/separator` |
| Sharing | `core/wcm/components/sharing/v1/sharing` |

For any Core Component not in this table, or to look up exact field names, refer to:  
`https://github.com/adobe/aem-core-wcm-components/tree/main/content/src/content/jcr_root/apps/core/wcm/components/`
