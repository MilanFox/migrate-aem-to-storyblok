---
name: aem-page-inventory
description: Scans an AEM content tree, extracts every page and its component instances with their field values, and produces a single JSON file at .inventory/aem-page-inventory.json. Requires .inventory/aem-component-inventory.json to already exist. Use this skill after aem-component-inventory and before any Storyblok page creation work.
---

# Skill: aem-page-inventory

## Purpose

Walk the AEM content tree, extract every published page and its component instances (with all field values), and write a single machine-readable JSON file — `.inventory/aem-page-inventory.json` — that a downstream agent can consume directly to recreate the site in a headless CMS (Storyblok).

Component names in the output are taken verbatim from `aem-component-inventory.json` so the two files form a consistent reference.

The output is **content-migration-first**: every field carries its actual authored value (or `null` if absent). AEM-specific structures are abstracted into generalised, Storyblok-friendly equivalents wherever doing so improves portability — see the *Structural Abstraction* section below.

---

## Prerequisites

1. `.inventory/aem-component-inventory.json` must exist and be complete.
2. Identify the content root. Pages live under:
   ```
   ui.content/src/main/content/jcr_root/content/<project-namespace>/
   ```
   or in a single-module project:
   ```
   src/main/content/jcr_root/content/<project-namespace>/
   ```
3. Note the `<project-namespace>` — it is the root segment stripped when computing page slugs.

---

## Step 1 — Load the Component Inventory

Read `.inventory/aem-component-inventory.json` and build an in-memory lookup:

```
componentMap: Map<resourceType, componentName>
```

where `resourceType` is the `resourceType` value from each entry in the `components` array (e.g. `myproject/components/content/hero`) and `componentName` is the `name` value (e.g. `Hero`).

This map is used throughout to translate AEM `sling:resourceType` values into inventory component names.

---

## Step 2 — Discover All Pages

A valid AEM page directory contains a `.content.xml` with `jcr:primaryType="cq:Page"`.

```bash
find . -name ".content.xml" | xargs grep -l 'jcr:primaryType="cq:Page"'
```

For each result the **parent directory** is one page. Record its absolute file-system path.

**Skip these directories:**
- The content root itself (it is a site root, not a renderable page)
- Scaffold / experience fragment roots (`/content/experience-fragments/`, `/content/dam/`)
- Any path segment starting with `_` or containing `rep:policy`

---

## Step 3 — Extract Page Metadata

Each page's metadata lives in its `jcr:content` node. In a VLT/FileVault export this is either:

- A sibling file `_jcr_content/.content.xml`, or
- A nested `<jcr:content …>` element inside the page's own `.content.xml`

Parse whichever form is present and read the following attributes from the `jcr:content` node itself (not its children):

| Output field | Source attribute | Notes |
|---|---|---|
| `title` | `jcr:title` | Fallback to directory name |
| `description` | `jcr:description` | `null` if absent |
| `template` | `cq:template` | Full template path string; `null` if absent |
| `language` | `jcr:language` | `null` if absent; inherit from ancestor |
| `hideInNav` | `hideInNav` | `{Boolean}true` → `true`, absent → `false` |
| `canonicalUrl` | `canonicalUrl` | `null` if absent |
| `metaDescription` | `metaDescription` or `og:description` | `null` if absent |
| `robots` | `cq:robotsTags` | `null` if absent; may be array |

All fields are always present in the output. Missing source attributes produce `null`, never a missing key.

---

## Step 4 — Compute the Page Slug

The slug is the file-system path of the page directory, relative to the content root, with the `<project-namespace>` prefix stripped.

```
fs path:   .../jcr_root/content/myproject/en/about/team
slug:      /en/about/team
```

- Preserve all path segments as-is.
- Do **not** strip language segments — the consuming agent decides how to handle localisation.
- Root page (the project namespace directory itself) → slug `/`.

---

## Step 5 — Extract Component Instances

The content tree under `jcr:content` contains the component instances placed by authors. The nesting mirrors the parsys / container structure. Most components are emitted into a flat ordered array, but **panel containers** (Accordion, Tabs, Carousel) preserve their nested item structure.

### Prerequisite: Build a container lookup

From `aem-component-inventory.json`, build a second lookup alongside `componentMap`:

```
containerLookup: Map<resourceType, { containerType: "panel" | "layout", panelItemFields: [...] | null }>
```

Populate it from every component entry that has `isContainer: true`. Also include the known AEM built-in container patterns listed below. This lookup drives the traversal decisions.

### Finding component nodes

Recursively walk every child node of `jcr:content`. A node is a **component instance** when its `sling:resourceType` appears in `componentMap`.

**Layout containers** are traversal wrappers — they are **not** emitted as component instances. Keep descending into their children. A node is a layout container when:

- Its `sling:resourceType` matches `containerLookup` with `containerType: "layout"`, OR
- Its `sling:resourceType` matches one of these built-in patterns:
  - `wcm/foundation/components/parsys`
  - `wcm/foundation/components/iparsys`
  - `core/wcm/components/container/v*/container`
  - `core/wcm/components/responsivegrid/…`
  - Any type ending in `/parsys` or `/responsivegrid`
- Its `sling:resourceType` maps to a component named `Container` in `componentMap`

**Panel containers** (Accordion, Tabs, Carousel) — identified by `containerType: "panel"` in `containerLookup` — **are** emitted as component instances, but with special nested handling. See *Panel container traversal* below.

### Panel container traversal

When you encounter a node whose `sling:resourceType` maps to a panel container:

1. **Emit the component itself** with its own config fields (e.g. `singleExpansion`, `id`, `expandedItems` for Accordion).
2. **Add a `children` array** to the emitted instance. This array contains one entry per panel item.
3. **Iterate the component's direct child nodes.** Each child that is an `nt:unstructured` node and is not a JCR metadata node (skip `cq:responsive`, `cq:childrenOrder`, `cq:panelOrder`, and any node name starting with `cq:` or `jcr:`) is a **panel item**.
4. For each panel item, create an item object:
   ```json
   {
     "itemName": "<node-name>",
     "panelTitle": "<cq:panelTitle or jcr:title fallback, null if neither present>",
     "components": [ /* child component instances, in document order */ ]
   }
   ```
5. **Recursively traverse** the panel item node to find component instances inside it. These child components are collected into the item's `components` array — they are **not** added to the page's flat component list. Each child component gets a local zero-based `order` within its item.
6. Inside a panel item, layout containers are still traversed-through (not emitted). Panel containers can nest (e.g. an Accordion inside a Tab) — apply the same panel container rules recursively.

**Critical:** Components inside panel items must **only** appear in the `children[].components` arrays, never as siblings in the page-level flat list. This prevents the downstream agent from seeing orphaned components with opaque container paths.

### Tracking container path

As you recurse, maintain a **path stack** of node names descended through. When you emit a component instance, set its `container` field to the slash-joined stack at the point the component was found — relative to `jcr:content`, not including the component node itself.

Examples:
- Component directly under `root` → `container: "root"`
- Component inside `root/container` → `container: "root/container"`
- Component inside `root/container/inner` → `container: "root/container/inner"`

**Never emit an empty string for `container`.** If a component is found directly under `jcr:content` (no intermediate nodes), use `"root"` as the container value.

**Panel item components** do not need a `container` field — their position is already defined by their placement inside `children[n].components`. If you do include `container` on them for debugging, keep it relative to the panel item node, not the page root.

### Determining order

Use the JCR `jcr:primaryType="nt:unstructured"` child ordering as it appears in the XML source. If the XML parser returns attributes in document order, honour that. For sibling nodes at the same container level, order is document order top-to-bottom.

For the **page-level** flat list: assign each component instance a zero-based `order` integer scoped to the whole page, incremented as you encounter instances in depth-first traversal. Panel containers themselves get a page-level `order`, but their child components do **not** — those get a separate zero-based `order` within their `children[n].components` array.

### Reading field values

For each component instance, read all attributes from its node (and any composite child nodes for multifields) and map them to field values.

**All declared fields must be present in the output.** Cross-reference the component's field list from `aem-component-inventory.json`. Any field that has no authored value in the XML must appear in `fields` with value `null` — never omitted.

**Type coercion rules:**

| Raw AEM value | Output JSON type |
|---|---|
| `{Boolean}true` / `{Boolean}false` | `true` / `false` |
| `{Long}42` | `42` |
| `{Double}3.14` | `3.14` |
| Plain string | `"string"` |
| `[val1,val2]` (multi-value) | `["val1","val2"]` |
| `{Date}2024-01-15T00:00:00.000+00:00` | `"2024-01-15T00:00:00.000Z"` |
| Absent / empty string | `null` |

**Skip these attributes entirely:**
- `jcr:primaryType`, `jcr:createdBy`, `jcr:created`, `jcr:lastModified`, `jcr:lastModifiedBy`
- `cq:lastModified`, `cq:lastModifiedBy`, `cq:lastReplicationAction`, `cq:lastReplicated`, `cq:lastReplicatedBy`
- `sling:resourceType` (already captured as `componentName`)

**Normalise field names** using the same rules as the component inventory skill:
- Strip leading `./`
- `jcr:title` → `title`, `jcr:description` → `description`, `cq:tags` → `tags`
- `fileReference` → `image`

**Multifield child nodes:** If a component node has child nodes that are themselves `nt:unstructured` (the typical multifield pattern), collect them as an array of objects under the parent field name. Each child object follows the same attribute-reading, coercion, and null-filling rules above. An empty multifield → `[]`, not `null`.

**Distinguishing multifield children from panel item children:** When a component is a panel container, its child nodes are panel items (not multifield entries). Use the `containerLookup` to decide: if the parent component has `containerType: "panel"`, treat child `nt:unstructured` nodes as panel items, not multifield values.

---

## Step 6 — Structural Abstraction for Storyblok

AEM has its own conventions that have no direct equivalent in Storyblok. Rather than dumping raw AEM artefacts into the JSON, **make a reasoned decision** about the most portable representation and document the choice in `meta.abstractionNotes`.

The goal is: *as close to AEM intent as possible, as generic as necessary for Storyblok to consume without AEM knowledge.*

### Guiding principles

1. **Prefer semantic meaning over AEM mechanism.** A `cq:template` path encodes the page type — extract the final path segment and map it to a human-readable `pageType` string (e.g. `/conf/myproject/settings/wcm/templates/content-page` → `"content-page"`).
2. **Flatten where nesting adds no value.** If a container holds a single component, represent it as a direct child rather than a one-element wrapper array.
3. **Normalise references.** DAM paths (`/content/dam/…`) are kept as-is since they are genuine asset references; internal page links (`/content/<namespace>/…`) are rewritten to slug form (e.g. `/en/about/team`).
4. **Represent structure, not AEM plumbing.** Parsys / responsivegrid nesting is meaningful only as grouping context. Encode it as a `container` label, not as AEM resource types.
5. **When uncertain, preserve and annotate.** If the correct abstraction is unclear, keep the raw value, flag it in the field with an `_aemRaw` sibling key, and add a note to `meta.abstractionNotes`.

### Specific transformations (apply unless a better option is evident)

| AEM construct | Abstracted output |
|---|---|
| `cq:template` full path | `pageType`: last path segment, stripped of version suffixes (e.g. `content-page`) |
| `/content/<ns>/…` internal link | Rewrite to slug: `/en/about/team` |
| `/content/dam/…` asset path | Keep as-is; tag with `"_type": "asset"` on the parent field object if helpful |
| `cq:tags` array of taxonomy paths | `tags`: array of tag leaf names (e.g. `myproject:topic/ai` → `"ai"`) |
| `cq:robotsTags` array | `robots`: plain string array, e.g. `["index","follow"]` |
| Experience Fragment path | `{ "_type": "fragment", "path": "/en/fragments/footer" }` |
| Multifield of inline items | Array of objects (already handled in Step 5) |
| Layout container with a `layout` attribute | Capture as `"layout"` key on the containing `components` group if the group is emitted, otherwise discard |
| `hideInNav` | Kept as boolean on the page; omit from individual components |

### Resolving `*FromPage` fields

Some components (e.g. Teaser) support inheriting their title or description from a linked page at runtime via boolean flags such as `titleFromPage` and `descriptionFromPage`. For a headless CMS these flags are meaningless — the actual content value must be present.

**Rule:** After collecting all field values for a component, check for any field matching the pattern `<fieldName>FromPage` that is `true`. When found:

1. Identify the companion link field (typically `link`) on the same component.
2. Convert the link value to a slug (apply the same `/content/<ns>/…` → slug rewrite as other internal links).
3. Look up that slug in the **already-collected pages** array for the current run.
4. If the page is found, replace the `null` value of `<fieldName>` with the matching page-level field (e.g. `titleFromPage: true` → use `page.title`; `descriptionFromPage: true` → use `page.description`).
5. Keep the `<fieldName>FromPage` boolean field in the output as-is (it records the original AEM authoring intent).
6. If the linked page is not found in the collected pages (e.g. it was skipped or lives outside the content root), leave `<fieldName>` as `null` and add a note to `meta.abstractionNotes`.

**Important:** This resolution must use the page data collected during *this run* — do not attempt to read additional files.

### Tree structure

Where the page's component list has natural section groupings (inferred from container node paths — e.g. `root/header`, `root/main`, `root/footer`), emit the components as a **nested tree** rather than a flat array:

```jsonc
"sections": {
  "header": [ /* components in order */ ],
  "main":   [ /* components in order */ ],
  "footer": [ /* components in order */ ]
}
```

If no meaningful grouping exists, fall back to the flat `"components": []` array. Both shapes are valid output; choose whichever is more informative.

Use judgment: if AEM's container naming is opaque (e.g. `par_1`, `par_2`) and gives no semantic hint, do not create artificial sections — use the flat list.

---

## Step 7 — Build the JSON Structure

```jsonc
{
  "meta": {
    "project": "<project-namespace>",
    "contentRoot": "<relative path to jcr_root/content/<namespace>>",
    "totalPages": 42,
    "generatedOn": "YYYY-MM-DD",
    "abstractionNotes": [
      "cq:template paths shortened to pageType leaf segment.",
      "Internal /content/myproject/ links rewritten to slugs.",
      "responsivegrid containers mapped to 'main' section; no header/footer containers detected."
    ],
    "skippedPages": [
      { "path": "jcr_root/content/myproject/scaffold", "reason": "Scaffold root, not a renderable page" }
    ]
  },
  "pages": [
    {
      "slug": "/en/about/team",
      "title": "Our Team",
      "description": "Meet the people behind the project.",
      "pageType": "content-page",
      "language": "en",
      "hideInNav": false,
      "meta": {
        "metaDescription": "…",
        "robots": ["index", "follow"],
        "canonicalUrl": null
      },
      "sections": {
        "main": [
          {
            "order": 0,
            "componentName": "Hero",
            "container": "root/responsivegrid",
            "fields": {
              "title": "We build great things",
              "image": "/content/dam/myproject/hero.jpg",
              "ctaLabel": "Learn more",
              "ctaLink": "/en/contact",
              "subtitle": null,
              "backgroundColor": null
            }
          },
          {
            "order": 1,
            "componentName": "Text",
            "container": "root/responsivegrid",
            "fields": {
              "body": "<p>Rich text content here</p>",
              "alignment": null
            }
          },
          {
            "order": 2,
            "componentName": "Accordion",
            "container": "root/responsivegrid",
            "fields": {
              "id": null,
              "singleExpansion": false,
              "expandedItems": null
            },
            "children": [
              {
                "itemName": "item_1",
                "panelTitle": "How does it work?",
                "components": [
                  {
                    "order": 0,
                    "componentName": "Text",
                    "fields": {
                      "body": "<p>Panel content here.</p>",
                      "alignment": null
                    }
                  }
                ]
              },
              {
                "itemName": "item_2",
                "panelTitle": "Can I contribute?",
                "components": [
                  {
                    "order": 0,
                    "componentName": "Text",
                    "fields": {
                      "body": "<p>Yes! Here's how…</p>",
                      "alignment": null
                    }
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  ]
}
```

**`container` field:** Record the slash-separated node path from `jcr:content` down to (but not including) the component node itself. This preserves structural context without requiring the consuming agent to understand parsys nesting.

**`children` field (panel containers only):** Present on Accordion, Tabs, and Carousel instances. An array of item objects, each with `itemName` (the JCR node name), `panelTitle` (from `cq:panelTitle` or `jcr:title`, `null` if neither), and `components` (an ordered array of child component instances). Components inside panel items do **not** appear in the page-level flat list — they exist only within `children`. Non-panel-container components do not have a `children` field.

**`fields` completeness:** Every field declared for the component in `aem-component-inventory.json` must appear. Fields with no authored value are `null`.

**Unknown `sling:resourceType`:** If a node has a `sling:resourceType` not found in `componentMap`, record it as:

```json
{
  "order": 3,
  "componentName": null,
  "unmappedResourceType": "myproject/components/legacy/oldwidget",
  "container": "root/responsivegrid",
  "fields": { … }
}
```

Do not silently drop unknown components.

---

## Step 8 — Write the Output File

Output path:
```
<project-root>/.inventory/aem-page-inventory.json
```

Write valid, pretty-printed JSON (2-space indent). The `pages` array must be sorted by `slug` alphabetically.

---

## Step 9 — Quality Control (Required Final Step)

Re-read the written file and verify:

**Coverage:**
- [ ] Page count in `meta.totalPages` matches the length of the `pages` array.
- [ ] Every page directory discovered in Step 2 is either in the output or listed in `meta.skippedPages`.

**Components:**
- [ ] No component instance with a known `sling:resourceType` has `componentName: null`.
- [ ] All `fields` objects contain only normalised key names — no `jcr:` or `cq:` prefixes, no leading `./`.
- [ ] Every field declared in the component inventory is present in `fields`; missing values are `null`, not absent.
- [ ] Multifield arrays are arrays of objects, not arrays of raw strings; empty multfields are `[]`.

**Panel containers (Accordion, Tabs, Carousel):**
- [ ] Every Accordion, Tabs, and Carousel instance has a `children` array (may be empty if no items authored).
- [ ] Each item in `children` has `itemName` (string), `panelTitle` (string or null), and `components` (array).
- [ ] Panel titles (`cq:panelTitle` / `jcr:title`) are captured — no item has `panelTitle: null` when the source XML has a title attribute.
- [ ] Components inside panel items appear **only** in `children[].components`, not as siblings in the page-level flat list.
- [ ] No component in the page-level list has a `container` path containing an accordion/tabs/carousel item node name (e.g. `accordion/item_1`) — this would indicate the component was incorrectly flattened instead of nested.

**Types:**
- [ ] No field value is a raw AEM type string like `{Boolean}true` — all have been coerced.
- [ ] All `{Date}` values have been converted to ISO 8601 strings.
- [ ] No empty strings — convert to `null`.

**Abstraction:**
- [ ] No `cq:template` full paths in the output — only `pageType` leaf values.
- [ ] No `/content/<namespace>/` internal links — rewritten to slugs.
- [ ] `meta.abstractionNotes` documents every non-trivial structural decision made.
- [ ] No AEM-internal field names (`cq:`, `sling:`, `wcm:`) appear as field keys in any `fields` object.

**File integrity:**
- [ ] The file is valid JSON (parseable without errors).
- [ ] The output file exists at exactly `.inventory/aem-page-inventory.json`.

Fix any issues before considering the skill complete.
