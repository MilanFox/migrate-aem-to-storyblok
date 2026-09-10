---
name: storyblok-page-creator
description: Reads .inventory/aem-page-inventory.json and .inventory/aem-component-inventory.json, then creates all pages as Storyblok Stories via the Storyblok MCP. Handles folder structure, story slugs, and blok field mapping. Run this skill after both inventory skills are complete.
---

# Skill: storyblok-page-creator

## Purpose

Read the two inventory files produced by the earlier skills and create every page as a Storyblok Story — including the correct folder hierarchy, slug, metadata, and component bloks — using the **Storyblok MCP** (`createStory` / `updateStory` tools).

All stories are saved as **drafts only**. Publishing is intentionally never performed by this skill — editors must publish stories manually.

---

## Prerequisites

1. `.inventory/aem-page-inventory.json` must exist and pass its QC checklist.
2. `.inventory/aem-component-inventory.json` must exist.
3. The following must be available as environment variables or provided by the user before the skill runs:

| Variable               | Description |
|------------------------|---|
| `Storyblok Space ID`   | Numeric ID of the target Storyblok space |
| `Default Content Type` | Storyblok component name to use as the story's root blok (e.g. `"page"`) |
| `Storyblok Region`     | API region: `eu` (default) or `us` |

4. All Storyblok **components** (blok schemas) referenced by `componentName` in the inventory must already exist in the space. This skill creates **stories** (content), not component schemas.
  - If component schemas need to be created first, run the **`storyblok-component-creator`** skill before this one. That skill has authoritative guidance on what valid component field structures look like and how to construct blok objects that match a component's schema.

---

## Companion Skill: storyblok-component-creator

This skill relies on the **`storyblok-component-creator`** skill for all knowledge about how individual blok objects must be structured.

Before building any `content.body` blok in Step 4, consult `storyblok-component-creator` to understand:

- Which fields a component schema defines and which are required vs. optional
- What the valid value shape is for each field type (text, asset, richtext, link, bloks, etc.)
- How to tell whether a field value will be accepted by Storyblok vs. rejected as invalid or empty

**A blok with missing required fields or structurally wrong field values will be stored but will render broken in the editor. Always validate against the component schema before writing the story.**

---

## Story Object Reference

The Storyblok Management API story object has the following writable fields when calling `createStory` or `updateStory`. Only the fields below should be set — do not invent or forward-fill fields that are read-only or server-generated (e.g. `uuid`, `created_at`, `published_at`, `full_slug`, `breadcrumbs`, `last_author`).

Full reference: `https://www.storyblok.com/docs/api/management/stories/the-story-object`

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | **yes** | Human-readable title shown in the editor |
| `slug` | string | **yes** | URL-safe last path segment only (no slashes) |
| `parent_id` | number | **yes** | ID of parent folder; `0` for root |
| `content` | object | **yes** | Must include `component` and `_uid` at minimum |
| `content.component` | string | **yes** | Technical name of the root content type blok |
| `content._uid` | string | **yes** | UUID v4, unique per blok instance |
| `is_startpage` | boolean | no | `true` only for the root story of a folder |
| `path` | string | no | Real path for the Visual Editor (not the slug) |
| `is_folder` | boolean | no | `true` when creating a folder, omit for stories |
| `default_root` | string | no | Default content type for stories inside this folder |
| `tag_list` | string[] | no | Array of tag strings |
| `sort_by_date` | string | no | ISO datestamp for custom sort |
| `meta_data` | object | no | Non-editable key/value data maintained via MAPI only |
| `disable_fe_editor` | boolean | no | Set `true` to disable the Visual Editor for this entry |

> **Never pass** `publish`, `published`, `published_at`, or `first_published_at` — these are read-only or managed by separate publish/unpublish endpoints.

### Minimal valid createStory payload

```json
{
  "story": {
    "name": "About Us",
    "slug": "about-us",
    "parent_id": 12345,
    "content": {
      "component": "page",
      "_uid": "a1b2c3d4-...",
      "body": []
    }
  }
}
```

---

## MCP Setup Check (Run Before Anything Else)

Before processing any pages, verify that the `createStory` tool is available via the Storyblok MCP.

- If `createStory` is **not available**, **stop immediately** and report:

  ```
  ERROR: The Storyblok MCP tool `createStory` is not available.
  This likely means the MCP server is misconfigured. Common cause:
  the connected Storyblok role does not have sufficient permissions —
  for example, the "Developer" role does not have access to story
  creation tools. Check the MCP server setup and ensure the token
  belongs to a role with content management permissions (e.g. Admin
  or Editor).
  ```

  Do **not** fall back to direct API calls.

---

## Pre-flight Checks

### Check 1: Inventory files exist?

This skill needs both inventories. Check each:

- `.inventory/aem-page-inventory.json` — produced by the `aem-page-inventory` skill
- `.inventory/aem-component-inventory.json` — produced by the `aem-component-inventory` skill

If either file is missing, stop and report which one, naming the skill that produces it:

> "The page inventory file is missing. Please run the `aem-page-inventory` skill first."

Do not proceed. Exit.

---

### Check 2: Have the inventories been reviewed?

This skill writes to a live Storyblok space. It is the last phase, so there is no checkpoint after it — an unreviewed inventory lands its mistakes directly in every story created, potentially hundreds.

If the `migrate-aem-to-storyblok` skill invoked this one, its review checkpoint has already run — continue.

Otherwise, ask the user and wait for an explicit answer:

> "Have `.inventory/aem-page-inventory.json` and `.inventory/aem-component-inventory.json` been reviewed and corrected? This will create stories in Storyblok space <Space ID>. (yes / no)"

If the answer is anything other than a clear yes, stop and report:

> "Stopping. Review the inventory files first, then re-run this skill."

Do not proceed. Exit.

---

## Execution Model: Two-Phase Approach

This skill runs in **two distinct phases**. Complete Phase 1 entirely before starting Phase 2. Do not interleave folder creation with story creation.

```
Phase 1 — Structure:   Load inventories → Build folder tree → Create all folders
Phase 2 — Content:     For each page: build bloks → validate fields → create/update story
```

This separation ensures that all `parent_id` values are known before any story is written, eliminating lookup failures and out-of-order errors.

---

## Step 1 — Load Both Inventories

### Page inventory

Parse `.inventory/aem-page-inventory.json` fully into memory.

### Component inventory

Parse `.inventory/aem-component-inventory.json` and load the `components` array into memory for field validation lookups in Phase 2.

### Asset map

Read `.inventory/storyblok-asset-map.json` if it exists. Load the `map` object into memory as `assetMap: Map<aemPath, url>`. This is used in Step 3 to resolve `/content/dam/` paths to their correct Storyblok CDN or resolved URLs.

If the file does not exist, log a warning:

> "WARNING: `.inventory/storyblok-asset-map.json` not found. Asset fields will not be resolved — run the `storyblok-asset-creator` skill first to generate it. Continuing without asset resolution."

Proceed without an asset map — asset fields will be set to `null` where no URL can be derived.

### Component name → Storyblok blok name map

Storyblok component (blok) technical names are typically the `componentName` from the inventory converted to `kebab-case`. Build a lookup:

```
blokName(componentName) = componentName
  .replace(/([a-z])([A-Z])/g, '$1-$2')
  .toLowerCase()
```

Examples: `Hero` → `hero`, `RichText` → `rich-text`, `ImageGallery` → `image-gallery`.

If the space uses a different naming convention the user must supply an override map. Ask before proceeding if there is any ambiguity — do not silently rename bloks.

---

## Phase 1: Step 2 — Build the Complete Folder Hierarchy

> **Do not create any stories until this phase is fully complete.**

Storyblok stores stories in a tree of folders. Every unique path prefix in the page inventory must become a folder before its child stories can be created.

### Algorithm

1. Collect all unique parent paths from the `slug` values in the page inventory.
  - `/en/about/team` → parent paths to ensure: `/en`, `/en/about`
2. Sort them by depth (ascending) so parents are created before children.
3. For each path, check if it already exists via the MCP.
4. If not found, create it using `createStory`:

```json
{
  "story": {
    "name": "<last path segment, title-cased>",
    "slug": "<last path segment>",
    "is_folder": true,
    "parent_id": "<parent folder id or 0 for root>"
  }
}
```

5. Store the returned `id` in a `folderIdMap: Map<slug, id>` for use when creating stories.

**Phase 1 is complete when every folder path in the inventory has a confirmed entry in `folderIdMap`.**

Report to the user before proceeding:
```
Phase 1 complete: [n] folders created, [n] already existed.
Starting Phase 2 — story creation.
```

---

## Phase 2: Step 3 — Map Field Values

For each component instance, convert AEM field values to Storyblok-compatible values.

### Field type transformations

| AEM field characteristic | Storyblok field value |
|---|---|
| Plain string | string as-is |
| `number` | number as-is |
| `boolean` | boolean as-is |
| String that starts with `/content/dam/` | Asset object — look up the path in `assetMap` first. If found and the value is non-null, use it as `filename`: `{ "filename": "<assetMap[path]>", "alt": "" }`. If not found in the map or the value is `null`, set the field to `null` and log: `ASSET UNRESOLVED [slug] field=[fieldName]: /content/dam/… not in asset map`. Do **not** construct a URL by hand — only use values from `assetMap`. |
| String that starts with `/content/` (internal link) | Story link object: `{ "linktype": "story", "story": { "url": "<slug>" } }` — convert the AEM path to a slug using the same stripping rule from the page inventory skill |
| External URL (`https?://`) | URL link object: `{ "linktype": "url", "url": "…" }` |
| Richtext string (HTML) | Storyblok richtext object — see richtext conversion below |
| Array of strings | Array of strings |
| Array of objects (multifield) | Array of blok objects — see multifield conversion below |

### Richtext conversion

Storyblok richtext is a structured document object, not raw HTML. Convert HTML to the Storyblok richtext schema:

```json
{
  "type": "doc",
  "content": [
    { "type": "paragraph", "content": [{ "type": "text", "text": "…" }] }
  ]
}
```

Map common HTML tags:

| HTML | Storyblok node type |
|---|---|
| `<p>` | `paragraph` |
| `<h1>`–`<h6>` | `heading` with `attrs.level` 1–6 |
| `<ul>` | `bullet_list` |
| `<ol>` | `ordered_list` |
| `<li>` | `list_item` |
| `<a href="…">` | `text` node with mark `link` |
| `<strong>` / `<b>` | `text` node with mark `bold` |
| `<em>` / `<i>` | `text` node with mark `italic` |
| `<br>` | `hard_break` |
| `<img>` | `image` node with `attrs.src` |

If the HTML is complex or deeply nested, use a lightweight HTML-to-JSON parser (e.g. a minimal recursive descent over the DOM). Do not silently drop markup — flag any tags that could not be converted with a `// CONVERSION WARNING` comment in the output log.

### Multifield conversion

Each object in a multifield array becomes a Storyblok blok:

```json
{
  "component": "<blokName for the parent component>__item",
  "_uid": "<generate a UUID v4>",
  "fieldA": "…",
  "fieldB": "…"
}
```

If a dedicated sub-blok name is not defined in the space, use the pattern `<parent-blok-name>__item`. Flag this in the run log so a developer can rename it later.

---

## Phase 2: Step 4 — Build, Validate, and Create Each Story

Process pages in `slug` order, shortest first.

### 4a — Build the story body

```json
{
  "story": {
    "name": "<page.title>",
    "slug": "<last segment of page.slug>",
    "parent_id": "<folderIdMap[parent path] or 0>",
    "content": {
      "component": "{STORYBLOK_DEFAULT_CONTENT_TYPE}",
      "_uid": "<UUID v4>",
      "body": []
    },
    "is_startpage": "<true if page.slug === '/'>",
    "path": "<page.slug>"
  }
}
```

Note: the `publish` parameter is **never** passed to `createStory` or `updateStory`. Stories must remain as drafts. Do not add it under any circumstances.

### 4b — Build the blok array

Iterate `page.components` ordered by `order` ascending. For each component, build a blok object and append it to `body`:

```json
{
  "component": "<blokName(componentName)>",
  "_uid": "<UUID v4>",
  "<fieldKey>": "<mapped field value>",
  …
}
```

Apply all field transformations from Step 3 to every field present on the component.

**Unmapped components:** if `componentName` is `null` and `unmappedResourceType` is set, do **not** skip the component. Instead, include it as a best-effort blok using the raw resource type converted to kebab-case as the component name, and carry over all available fields as-is:

```json
{
  "component": "<kebab-case of unmappedResourceType>",
  "_uid": "<UUID v4>",
  "<fieldKey>": "<raw field value>",
  …
}
```

Log a warning: `UNMAPPED [slug] order=[n]: used raw type [unmappedResourceType] as blok name — verify schema exists in space.`

### 4c — Field Validation (Required Before Every Story Write)

Before calling `createStory` or `updateStory` for a page, validate the assembled story body against the following rules. **Do not skip this step.**

Consult the **`storyblok-component-creator`** skill to determine which fields are required vs. optional for each component. Apply these checks per blok in `content.body`:

**Per-blok checks:**
- [ ] `component` is a non-empty string
- [ ] `_uid` is a valid UUID v4
- [ ] Every **required** field defined in the component's schema has a non-null, non-empty value
  - String fields: not `""` or `null`
  - Asset fields: `filename` is not `""` or `null`
  - Richtext fields: `content` array is not empty — at minimum `[{ "type": "paragraph", "content": [{ "type": "text", "text": " " }] }]`
  - Link fields: `linktype` is set and the target value (`url` or `story.url`) is not empty
  - Bloks fields (nested arrays): not an empty array if the field is required
- [ ] **Optional** fields that are empty may be omitted entirely from the blok object — do not write `""`, `null`, or `[]` for optional fields unless the schema explicitly requires a default

**Story-level checks:**
- [ ] `name` is non-empty
- [ ] `slug` is non-empty and contains no slashes
- [ ] `parent_id` is a valid number (resolved from `folderIdMap`)
- [ ] `content.component` matches the configured Default Content Type
- [ ] `content._uid` is a valid UUID v4

**If any required field fails validation:**
1. Do **not** write the story.
2. Log: `VALIDATION ERROR [slug] blok=[component] order=[n]: field "[fieldName]" is required but has no value.`
3. Attempt to recover: if a sensible fallback exists (e.g. a richtext field can be initialized to a minimal empty doc, a string field can be set to `" "`), apply it and re-validate once.
4. If recovery is not possible, mark the page as `error` in the run log and continue to the next page.

### 4d — Create or Update via MCP

- **Check** whether a story with the same slug already exists.
  - If **not found**: call `createStory` with the validated body from 4a–4c.
  - If **found**: call `updateStory` with the existing story's `id` and the same body.

On success, store the returned `story.id` in a `storyIdMap: Map<slug, id>`.

On failure, log the full error with `ERROR [slug]: <detail>` and continue with the next page. Do not abort the entire run on a single failure.

### 4d (continued) — Per-story progress output

After each story attempt, output one status line immediately (do not batch them up):

```
✅ /en/about/team — created (id: 98765, 4 bloks)
🔄 /en/existing   — updated (id: 98700, 2 bloks)
⚠️  /en/contact   — created with warnings (id: 98766, 1 unresolved asset)
❌ /en/broken     — error: createStory returned 422 — slug already taken
⏭️  /en/empty     — skipped (no components)
```

Rules:
- Use ✅ for clean creates, 🔄 for updates, ⚠️ for creates/updates with warnings (unresolved assets, unmapped components, validation recoveries), ❌ for hard errors, ⏭️ for skipped pages.
- Include the story `id` and blok count on success lines.
- For Phase 1 (folder creation), log one line per folder:
  ```
  📁 /en — created (id: 12300)
  📁 /en/about — already existed (id: 12301)
  ```
- Keep lines scannable — do not dump JSON into the progress output.

### 4e — Internal Link Resolution (Second Pass)

After all stories are created, any story link field whose target slug was not yet created at the time of the first pass may be unresolved. Perform a second pass:

1. Collect all stories that contain story link fields.
2. For each, look up the target slug in `storyIdMap`.
3. If found, call `updateStory` with the updated content to embed the resolved `story.id` into the link object.

---

## Step 5 — Write a Run Log

Output path:
```
.inventory/storyblok-page-run-log.json
```

```jsonc
{
  "runDate": "YYYY-MM-DDTHH:MM:SSZ",
  "spaceId": 12345,
  "totalPages": 42,
  "created": 38,
  "updated": 2,
  "skipped": 1,
  "errors": 1,
  "results": [
    { "slug": "/en/about/team", "status": "created", "storyId": 98765 },
    { "slug": "/en/existing", "status": "updated", "storyId": 98700 },
    { "slug": "/en/legacy", "status": "skipped", "reason": "page has no components at all" },
    { "slug": "/en/broken", "status": "error", "detail": "…" }
  ],
  "validationErrors": [
    { "slug": "/en/contact", "blok": "hero", "order": 1, "field": "headline", "detail": "required field is empty — story not written" }
  ],
  "conversionWarnings": [
    { "slug": "/en/about", "order": 2, "field": "body", "warning": "Unsupported HTML tag <table> dropped" }
  ],
  "unmappedComponentWarnings": [
    { "slug": "/en/about", "order": 3, "usedBlokName": "some-aem-component", "originalResourceType": "project/components/some-aem-component" }
  ]
}
```

---

## Step 6 — Quality Control (Required Final Step)

Re-read the log file and check:

**Coverage:**
- [ ] `created + updated + skipped + errors` equals `totalPages`.
- [ ] Every page from the inventory appears in `results` with a status.

**Errors:**
- [ ] All `status: "error"` entries have a non-empty `detail`.
- [ ] No story was silently dropped (absent from `results`).

**Publish check:**
- [ ] Confirm no story was published — all must remain as drafts. The `publish` parameter must never have been passed to `createStory` or `updateStory`.

**Component completeness:**
- [ ] Every page's `body` array contains a blok for every component entry in the inventory, including those with unmapped resource types.
- [ ] No component was silently dropped.
- [ ] No blok in any story has an empty required field (cross-check `validationErrors` — if any exist, the affected stories were not written and are logged as `error`).

**Warnings:**
- [ ] All `conversionWarnings`, `unmappedComponentWarnings`, and `validationErrors` have been reviewed. Warnings are acceptable but must be surfaced to the user in a summary at the end of the skill run.

**Post-run verification (spot check):**
- [ ] Pick 3 random stories from `storyIdMap` and retrieve them via MCP to confirm the content structure matches the inventory.
- [ ] Verify folder structure and confirm all expected folders exist.

### Final output to user

After QC, report:

```
Storyblok import complete.
  Phase 1 — Folders:   [n] created, [n] already existed
  Phase 2 — Stories:
    Created:            [n] stories
    Updated:            [n] stories (already existed)
    Skipped:            [n] (pages with no components at all)
    Errors:             [n] — see .inventory/storyblok-page-run-log.json
  Validation errors:    [n] — required fields were empty; affected stories were not written.
  Conversion warnings:  [n] — review richtext/link fields that may need manual attention.
  Unmapped components:  [n] — bloks created with raw AEM resource type as name; verify schemas exist in space.
  All stories saved as drafts — nothing has been published.
```

Full API reference: `https://www.storyblok.com/docs/api/management/` (for data structures, since the MCP takes the same data structures as the MAPI)
