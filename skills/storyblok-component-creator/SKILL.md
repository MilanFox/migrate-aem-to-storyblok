---
name: storyblok-component-creator
description: Migrate all components from a component inventory file to a Storyblok space via the Storyblok MCP.
  Reads project-root/.inventory/aem-component-inventory.json, converts each component using the
  storyblok-create-component-definition skill, and uploads each one via the MCP.
  Use when asked to migrate a component inventory (specifically) to Storyblok.
---

# Skill: storyblok-component-creator

## Purpose

Read every component from `(project-root)/.inventory/aem-component-inventory.json`, generate a valid Storyblok component JSON for each one using the `storyblok-create-component-definition` skill, and upload each component to the target Storyblok space via the Storyblok MCP.

---

## Step 0 — Pre-flight Checks

Run these checks before touching any component. Do not proceed until all pass.

### Check 1: Storyblok MCP connected?

Attempt a `search("list components")` call against the Storyblok MCP. If the call fails or the MCP is not available, stop and tell the user:

> "The Storyblok MCP is not connected. Please connect it via the MCP settings and then re-run this skill."

Do not proceed. Exit.

### Check 2: Space ID and Region provided?

You need both:

- **Space ID** — a numeric Storyblok space ID (e.g. `12345`)
- **Region** — one of `eu`, `us`, `ap`, `ca`, `cn`

If either is missing from the current context or conversation, ask:

> "Before we start: what is your Storyblok Space ID and which region is your space in? (eu / us / ap / ca / cn)"

Do not proceed until the user has provided both. Store them — every MCP call for the rest of this skill uses them.

---

### Check 3: Inventory file exists?

Read `project-root/.inventory/aem-component-inventory.json`. If the file does not exist, stop and report:

> "The component inventory file is missing. Please run the `aem-component-inventory` skill first."

Do not proceed. Exit.

---

### Check 4: Has the inventory been reviewed?

This skill writes to a live Storyblok space, and every later phase builds on what it creates. An unreviewed inventory propagates its mistakes into every component, and then into every page that uses them.

If the `migrate-aem-to-storyblok` skill invoked this one, its review checkpoint has already run — continue.

Otherwise, ask the user and wait for an explicit answer:

> "Has `.inventory/aem-component-inventory.json` been reviewed and corrected? This will create components in Storyblok space <Space ID>. (yes / no)"

If the answer is anything other than a clear yes, stop and report:

> "Stopping. Review the inventory file first, then re-run this skill."

Do not proceed. Exit.

---

## Step 1 — Read the Inventory

Read the file at `project-root/.inventory/aem-component-inventory.json`.

Parse the JSON and extract the `components` array. Ignore `excludedComponents`.

Build an ordered list of all components to process. Report to the user how many were found before starting:

> "Found 28 components to migrate. Starting now…"

---

## Step 2 — Process Each Component

For each component in the list, run the following loop. Track state per component: `pending`, `success`, `failed`, `skipped`.

### 2a — Generate the JSON

#### Input

A component entry from the inventory, which looks like this:

```json
{
  "name": "Button",
  "resourceType": "wknd/components/button",
  "template": null,
  "dialog": null,
  "inheritsFrom": ["core/wcm/components/button/v2/button"],
  "fields": [
    { "name": "title", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/button/v2/button", "note": null },
    { "name": "link", "type": "string (path)", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/button/v2/button", "note": null },
    { "name": "icon", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/button/v2/button", "note": null },
    { "name": "id", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/button/v2/button", "note": null }
  ]
}
```

---

#### Output

A single valid JSON object with the shape:

```json
{
  "component": {
    "name": "string",
    "display_name": "string",
    "is_root": false,
    "is_nestable": true,
    "schema": { ... }
  }
}
```
The `display_name` is optional and gives an editor more context in case the name isn't speaking enough.
The `name` should be in PascalCase if not told otherwise.
The `schema` field is the most critical part. Its rules are described in detail below.

---

#### The `schema` Field — Rules and Structure

##### What `schema` MUST be

`schema` MUST be a **plain JSON object** (key-value map). Nothing else is valid.

```json
"schema": {
"field_key": { ...field definition object... },
"another_field_key": { ...field definition object... }
}
```

##### What `schema` MUST NOT be

- **Not `null`** — `"schema": null` is invalid.
- **Not a number** — `"schema": 0` is invalid.
- **Not an array** — `"schema": []` is invalid.
- **Not a string** — `"schema": "fields"` is invalid.
- **Not empty without reason** — if the component has fields, they must appear in `schema`. An empty object `{}` is only valid for components that truly have no fields.

##### Schema Keys

Each key in `schema` is the **technical field name** — in camelCase. It should match the field name from the inventory as closely as possible.

##### Each Field Definition Object

Every value in `schema` must be a **JSON object** with at minimum:

```json
{
  "type": "<field_type_string>",
  "pos": <integer starting at 0>
}
```

Both `type` and `pos` are REQUIRED on every field. `pos` is a zero-based integer that determines display order in the editor. Increment it by 1 for each field.

---

#### Field Type Mapping

Map field `type` values from the inventory JSON to Storyblok field types using this table. Never invent a type not in this list.

| Inventory `type` value    | Storyblok `type` value | Notes |
|---------------------------|------------------------|-------|
| `"string"`                | `"text"`               | Default for plain strings |
| `"string (path)"`         | `"multilink"`          | Internal links/paths |
| `"number"`                | `"number"`             |  |
| `"boolean"`               | `"boolean"`            |  |
| `"Richtext"`              | `"richtext"`           |  |
| `"ImageAsset"`            | `"asset"`              | Add `"filetypes": ["images"]` |
| `"string[]"`              | `"options"`            | Multi-select; use `"source": "undefined"` with explicit options if values are known, otherwise omit `source` |
| `"enum"` (use `enumValues` array) | `"option"`   | Single-select; map `enumValues` to `options` array |
| `"object[]"` (use `itemFields`)   | `"bloks"`    | Nested components |
| `id` field (`type: "string"`)     | omit or `"text"` | AEM `id` fields are usually CMS-internal; include as `"text"` only if content editors need to set it |

##### Enum / Single-Option fields (`type: "option"`)

When a field has a fixed set of string values (e.g. `"h1" | "h2" | "h3"`), use `type: "option"` and list the options:

```json
"type_field": {
"type": "option",
"pos": 2,
"options": [
{ "name": "H1", "value": "h1" },
{ "name": "H2", "value": "h2" },
{ "name": "H3", "value": "h3" }
]
}
```

- Each option object MUST have `"name"` (display label) and `"value"` (stored value).
- `"name"` can be a human-readable version of `"value"`.
- The `options` value MUST be an **array of objects**. It MUST NOT be `null`, a string, or a flat array of strings.

##### Asset fields (`type: "asset"`)

```json
"file_reference": {
"type": "asset",
"pos": 0,
"filetypes": ["images"],
"allow_external_url": true
}
```

- `filetypes` is an array of strings. Valid values: `"images"`, `"videos"`, `"audios"`, `"texts"`.
- `filetypes` MUST be an array, never `null` or a string.
- **Always set `"allow_external_url": true`** on every `asset` field. This lets editors paste an external URL instead of uploading to the DAM — important when migrating content that may reference external assets. See [Storyblok docs — The Component Schema Field Object](https://www.storyblok.com/docs/api/management/components/the-component-schema-field-object):
  > *"Allow loading assets from an external URL in Asset and Multi-Assets fields. Defaults to `false`."*
  ```json
  "image": {
    "type": "asset",
    "pos": 2,
    "filetypes": [],
    "asset_folder_id": null,
    "allow_external_url": true
  }
  ```

##### Boolean fields (`type: "boolean"`)

```json
"single_expansion": {
"type": "boolean",
"pos": 0
}
```

No extra properties required.

##### Blocks fields (`type: "bloks"`)

Use when a field holds an array of nested components (e.g. `items`, `expandedItems` typed as complex arrays):

```json
"items": {
"type": "bloks",
"pos": 0,
"minimum": 0,
"maximum": 0
}
```

`maximum: 0` means unlimited. Set a real number to cap it.

If the inventory provides an `allowedComponents` list for this field, set `component_whitelist` to the PascalCase component names derived from those resource types (e.g. `wknd/components/teaser` → `"Teaser"`). If `allowedComponents` is `null`, omit `component_whitelist` entirely (no restriction).

```json
"items": {
"type": "bloks",
"pos": 0,
"minimum": 0,
"maximum": 0,
"component_whitelist": ["ExperienceFragment", "Image", "Teaser"]
}
```

##### Link fields (`type: "multilink"`)

Use for any field typed as `string (path)` or that holds a URL/path:

```json
"link_url": {
"type": "multilink",
"pos": 1,
"email_link_type": false,
"asset_link_type": false
}
```

##### Richtext fields (`type: "richtext"`)

```json
"description": {
"type": "richtext",
"pos": 3
}
```

---

#### Optional but Recommended Field Properties

Add these to field definitions where appropriate:

| Property        | Type    | When to add                                                                  |
|-----------------|---------|------------------------------------------------------------------------------|
| `display_name`  | string  | A human-readable label for the editor, in case the key isn't speaking enough |
| `required`      | boolean | Set `true` when the inventory field has no `?` (i.e. it's not optional)      |
| `description`   | string  | Add a brief hint if the field purpose isn't self-evident                     |
| `translatable`  | boolean | Set `true` for text content fields that editors would translate              |

---

#### Component-Level Properties

| Property       | Type    | Rule                                                                                                   |
|----------------|---------|--------------------------------------------------------------------------------------------------------|
| `name`         | string  | PascalCase. Derived from the component heading (e.g. "Form Button" → `"FormButton"`, "Teaser" → `"Teaser"`). |
| `display_name` | string  | Human-readable. Use the heading as-is (e.g. `"Form Button"`). Optional, overrides "name" in the UI.    |
| `is_root`      | boolean | `false` for all nestable/block components. `true` only for page-level content types. Default: `false`. |
| `is_nestable`  | boolean | `true` for all nestable components. Default: `true`.                                                   |
| `schema`       | object  | See above. MUST be a plain object.                                                                     |

---

#### AEM `id` Field Handling

Most AEM components include `id?: string`. This is an AEM-internal HTML ID hint. In Storyblok you can include it as a `text` field for parity, but it is optional. If in doubt, include it at the end with a low priority display name:

```json
"id": {
"type": "text",
"pos": 99,
"display_name": "HTML ID (optional)"
}
```

---

#### Full Worked Example

**Input (from inventory):**

```json
{
  "name": "Teaser",
  "resourceType": "wknd/components/teaser",
  "inheritsFrom": ["core/wcm/components/teaser/v2/teaser"],
  "fields": [
    { "name": "pretitle", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/teaser/v2/teaser", "note": null },
    { "name": "title", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/teaser/v2/teaser", "note": null },
    { "name": "description", "type": "Richtext", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/teaser/v2/teaser", "note": null },
    { "name": "actionsEnabled", "type": "boolean", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/teaser/v2/teaser", "note": null },
    { "name": "id", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/teaser/v2/teaser", "note": null }
  ]
}
```

**Output:**

```json
{
  "component": {
    "name": "Teaser",
    "is_root": false,
    "is_nestable": true,
    "schema": {
      "pretitle": {
        "type": "text",
        "pos": 0,
        "translatable": true
      },
      "title": {
        "type": "text",
        "pos": 1,
        "translatable": true
      },
      "description": {
        "type": "richtext",
        "pos": 2,
        "translatable": true
      },
      "actionsEnabled": {
        "type": "boolean",
        "pos": 3,
      },
      "id": {
        "type": "text",
        "pos": 4,
        "display_name": "HTML ID (optional)"
      }
    }
  }
}
```

---

#### Another Worked Example — Enum Field

**Input:**

```json
{
  "name": "Title",
  "resourceType": "wknd/components/title",
  "inheritsFrom": ["core/wcm/components/title/v3/title"],
  "fields": [
    { "name": "text", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/title/v3/title", "note": null },
    { "name": "type", "type": "enum", "enumValues": ["h1", "h2", "h3", "h4", "h5", "h6"], "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/title/v3/title", "note": null },
    { "name": "linkUrl", "type": "string (path)", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/title/v3/title", "note": null },
    { "name": "id", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/title/v3/title", "note": null }
  ]
}
```

**Output:**

```json
{
  "component": {
    "name": "Title",
    "display_name": "Title",
    "is_root": false,
    "is_nestable": true,
    "schema": {
      "text": {
        "type": "text",
        "pos": 0,
        "display_name": "Text",
        "translatable": true
      },
      "type": {
        "type": "option",
        "pos": 1,
        "display_name": "Heading Level",
        "options": [
          { "name": "H1", "value": "h1" },
          { "name": "H2", "value": "h2" },
          { "name": "H3", "value": "h3" },
          { "name": "H4", "value": "h4" },
          { "name": "H5", "value": "h5" },
          { "name": "H6", "value": "h6" }
        ]
      },
      "linkUrl": {
        "type": "multilink",
        "pos": 2,
        "display_name": "Link URL"
      },
      "id": {
        "type": "text",
        "pos": 3,
        "display_name": "HTML ID (optional)"
      }
    }
  }
}
```

---

#### Another Worked Example — Asset Field

**Input:**

```json
{
  "name": "Image",
  "resourceType": "wknd/components/image",
  "inheritsFrom": ["core/wcm/components/image/v3/image"],
  "fields": [
    { "name": "file", "type": "ImageAsset", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/image/v3/image", "note": null },
    { "name": "alt", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/image/v3/image", "note": null },
    { "name": "linkUrl", "type": "string (path)", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/image/v3/image", "note": null },
    { "name": "isDecorative", "type": "boolean", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/image/v3/image", "note": null },
    { "name": "id", "type": "string", "required": false, "inherited": true, "inheritedFrom": "core/wcm/components/image/v3/image", "note": null }
  ]
}
```

**Output:**

```json
{
  "component": {
    "name": "Image",
    "is_root": false,
    "is_nestable": true,
    "schema": {
      "file_reference": {
        "type": "asset",
        "pos": 0,
        "display_name": "Image Asset",
        "filetypes": ["images"],
        "allow_external_url": true
      },
      "alt": {
        "type": "text",
        "pos": 1,
        "display_name": "Alt Text",
        "translatable": true
      },
      "linkUrl": {
        "type": "multilink",
        "pos": 2
      },
      "isDecorative": {
        "type": "boolean",
        "pos": 3,
        "display_name": "Decorative (no alt required)"
      },
      "id": {
        "type": "text",
        "pos": 4,
        "display_name": "HTML ID (optional)"
      }
    }
  }
}
```

---

---

#### Container Components (`isContainer: true`)

When the inventory entry has `"isContainer": true`, the component accepts child components at authoring time. You **must** add a `bloks` field to its schema to represent this slot — without it, editors have nowhere to drop child blocks.

##### Rules

1. **Add a `bloks` field** as the first field in the schema (pos 0). Shift all other fields' `pos` values up by 1.

2. **Name the field** based on `containerType`:
   - `containerType: "panel"` → key `"items"`, with a contextual `display_name` derived from the component name:
     - Component name contains "Carousel" → `"Slides"`
     - Component name contains "Accordion" → `"Panels"`
     - Component name contains "Tab" → `"Tabs"`
     - Otherwise → `"Items"`
   - `containerType: "layout"` → key `"components"`, `display_name: "Components"`

3. **Restrict by `allowedComponents`** when the inventory provides a non-null list:
   - Convert each resource type to PascalCase component name (last path segment, PascalCase): `wknd/components/experience-fragment` → `ExperienceFragment`, `wknd/components/teaser` → `Teaser`
   - Set `component_whitelist` to that array
   - If `allowedComponents` is `null`, omit `component_whitelist` entirely (unrestricted)

4. **It is fine if whitelisted components don't exist in Storyblok yet.** The `component_whitelist` will take effect once they are created. Do not skip or defer the whitelist for this reason.

##### Example — Carousel (panel container with allowedComponents)

Inventory entry:
```json
{
  "name": "Carousel",
  "isContainer": true,
  "containerType": "panel",
  "allowedComponents": ["wknd/components/experiencefragment", "wknd/components/image", "wknd/components/teaser"],
  "fields": [
    { "name": "autoplay", "type": "boolean", ... },
    { "name": "delay", "type": "number", ... }
  ]
}
```

Output schema:
```json
"schema": {
  "items": {
    "type": "bloks",
    "pos": 0,
    "display_name": "Slides",
    "minimum": 0,
    "maximum": 0,
    "component_whitelist": ["ExperienceFragment", "Image", "Teaser"]
  },
  "autoplay": {
    "type": "boolean",
    "pos": 1,
    "display_name": "Autoplay"
  },
  "delay": {
    "type": "number",
    "pos": 2,
    "display_name": "Autoplay Delay (ms)"
  }
}
```

##### Example — Container (layout container, no allowedComponents)

```json
"schema": {
  "components": {
    "type": "bloks",
    "pos": 0,
    "display_name": "Components",
    "minimum": 0,
    "maximum": 0
  }
}
```

---

#### Common Mistakes to Avoid

| Mistake | Why it fails | Correct approach |
|---|---|---|
| `"schema": null` | `schema` must be an object | Use `"schema": {}` at minimum, or a populated object |
| `"schema": []` | Arrays are not valid for `schema` | `schema` is always `{}` with named keys |
| `"schema": 0` | Numbers are not valid for `schema` | Same as above |
| `"options": ["a", "b"]` | Options must be objects, not strings | `[{ "name": "A", "value": "a" }]` |
| `"options": null` | Options cannot be null | Either omit the property or provide an array |
| `"filetypes": "images"` | filetypes must be an array | `"filetypes": ["images"]` |
| Missing `"type"` on a field | Every field needs a type | Always include `"type"` |
| Missing `"pos"` on a field | Every field needs a position | Always include `"pos"` as a zero-based integer |
| Using `"type": "string"` | Not a valid Storyblok type | Use `"type": "text"` |
| Using `"type": "array"` | Not a valid Storyblok type | Use `"type": "bloks"` or `"type": "options"` |
| Missing `"allow_external_url": true` on asset field | External asset URLs will be blocked by default | Always add `"allow_external_url": true` to every `asset` field |



### 2b — Pre-upload Validation

Before calling the MCP, validate the JSON locally against the rules from `storyblok-create-component-definition`. This is not optional — the MCP's error messages are not descriptive enough to debug from. Catch problems here first.

**Mandatory checks:**

- `component` key exists and is a plain object.
- `name` is a non-empty string, PascalCase, no spaces.
- `display_name` is a non-empty string.
- `is_root` is a boolean.
- `is_nestable` is a boolean.
- `schema` exists, is a plain object (not `null`, not an array, not a number, not a string).
- Every key in `schema` maps to a plain object (not `null`, not a primitive).
- Every field object in `schema` has `type` (string) and `pos` (integer ≥ 0).
- `type` is one of the valid values: `text`, `textarea`, `richtext`, `markdown`, `number`, `boolean`, `datetime`, `asset`, `multiasset`, `multilink`, `option`, `options`, `bloks`, `section`, `table`, `custom`.
- If `type` is `option` or `options` and `options` is present: it must be an array of objects each with `name` and `value` string keys.
- If `type` is `asset` or `multiasset` and `filetypes` is present: it must be an array of strings.
- If `type` is `asset` or `multiasset`: `allow_external_url` MUST be `true`.
- If the inventory component has `isContainer: true`: `schema` MUST contain exactly one `bloks` field (named `items` for panel containers, `components` for layout containers).
- If `component_whitelist` is present on a `bloks` field: it must be an array of non-empty strings.

If any check fails: go back to `storyblok-create-component-definition` with the specific violation noted (see Step 3 — Error Handling).

### 2c — Upload via MCP

Use the Storyblok MCP discovery-first flow:

1. `search("create component")` — find the correct operation ID.
2. `describe(<operationId>)` — confirm the request body schema.
3. `execute_mutating(<operationId>, { spaceId: <Space ID>, body: { component: { ... } } })` — send the component.

The body to send is exactly the `component` object from the generated JSON (not the outer wrapper):

```json
{
  "component": {
    "name": "Button",
    "display_name": "Button",
    "is_root": false,
    "is_nestable": true,
    "schema": { ... }
  }
}
```

If the upload returns a success response (HTTP 200 or 201, or a component object in the response): mark this component `success` and log it.

---

## Step 3 — Error Handling

### On any failure (validation or MCP error)

1. Note which attempt number this is for this component (1, 2, or 3).
2. Re-read the failing JSON against the full rules in here. Do not guess — go back to that skill and re-derive the field definitions from scratch for the failing component.
3. Re-run pre-upload validation (Step 2b) on the new JSON before trying the MCP again.
4. Retry the upload (Step 2c).

### After 3 failed attempts on a single component

Stop retrying. Ask the user:

> "I was unable to migrate the **[ComponentName]** component after 3 attempts. The last error was: [error message or validation failure description].
>
> What would you like to do?
> 1. Tell me what to change and I'll try again
> 2. Skip this component and continue

Wait for the user's response before continuing to the next component.

- If they give instructions: apply them, reset the attempt counter to 0 for this component, and retry from Step 2a.
- If they choose skip: mark this component `skipped` and continue with the next one.

---

## Step 4 — Progress Reporting

After each component (success, skip, or exhausted retries waiting for input), log a one-line status:

```
✅ button — uploaded
✅ teaser — uploaded
⚠️  accordion — waiting for user input (attempt 3 failed)
⏭️  breadcrumb — skipped
```

Do not dump full JSON into the progress log. Keep it scannable.

---

## Step 5 — Final Summary

After all components have been processed, output a summary:

```
Migration complete.

✅ Uploaded:  24
⏭️  Skipped:    2
❌ Failed:     0

Failed or skipped components:
- breadcrumb (skipped by user)
- content-fragment-list (skipped by user)
```

If any components were skipped or failed, remind the user:

> "Skipped or failed components were not created in Storyblok. You can re-run this skill for individual components or handle them manually."

---

## Notes

- Process components sequentially, not in parallel. Parallel uploads risk race conditions and make error recovery harder to follow.
- Do not attempt to deduplicate against existing Storyblok components. If a component with the same `name` already exists, the MCP will return an error — treat it as a normal failure and surface it to the user via the 3-attempt flow.
- Never modify the inventory file.
- This skill is the single source of truth for what a valid component JSON looks like. When in doubt about a field, schema structure, or type mapping, go back to it — do not improvise.
