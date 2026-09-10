---
name: storyblok-create-component-definition
description: Convert a single component entry from the AEM component inventory JSON into a valid Storyblok Management API Component JSON object.
  Use when asked to generate a JSON definition of a Storyblok component. Returns only the JSON object, does not upload or call any API.
user-invocable: false
---

# Skill: storyblok-create-component-definition

## Purpose

Take a single component entry from the AEM component inventory JSON and return a valid JSON object that conforms to the Storyblok Management API Component Object. Do not upload or POST anything — only produce the JSON.

---

## Input

A component entry from the inventory JSON, which looks like this:

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

## Output

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

The `schema` field is the most critical part. Its rules are described in detail below.

---

## The `schema` Field — Rules and Structure

### What `schema` MUST be

`schema` MUST be a **plain JSON object** (key-value map). Nothing else is valid.

```json
"schema": {
"fieldKey": { ...field definition object... },
"anotherFieldKey": { ...field definition object... }
}
```

### What `schema` MUST NOT be

- **Not `null`** — `"schema": null` is invalid.
- **Not a number** — `"schema": 0` is invalid.
- **Not an array** — `"schema": []` is invalid.
- **Not a string** — `"schema": "fields"` is invalid.
- **Not empty without reason** — if the component has fields, they must appear in `schema`. An empty object `{}` is only valid for components that truly have no fields.

### Schema Keys

Each key in `schema` is the **technical field name** — in camelCase. It should match the field name from the inventory as closely as possible (e.g. `linkURL` → `link_url`, `fileReference` → `file_reference`).

### Each Field Definition Object

Every value in `schema` must be a **JSON object** with at minimum:

```json
{
  "type": "<field_type_string>",
  "pos": <integer starting at 0>
}
```

Both `type` and `pos` are REQUIRED on every field. `pos` is a zero-based integer that determines display order in the editor. Increment it by 1 for each field.

---

## Field Type Mapping

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

### Enum / Single-Option fields (`type: "option"`)

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

### Asset fields (`type: "asset"`)

```json
"file_reference": {
"type": "asset",
"pos": 0,
"filetypes": ["images"]
}
```

- `filetypes` is an array of strings. Valid values: `"images"`, `"videos"`, `"audios"`, `"texts"`.
- `filetypes` MUST be an array, never `null` or a string.

### Boolean fields (`type: "boolean"`)

```json
"single_expansion": {
"type": "boolean",
"pos": 0
}
```

No extra properties required.

### Blocks fields (`type: "bloks"`)

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

### Link fields (`type: "multilink"`)

Use for any field typed as `string (path)` or that holds a URL/path:

```json
"link_url": {
"type": "multilink",
"pos": 1,
"email_link_type": false,
"asset_link_type": false
}
```

### Richtext fields (`type: "richtext"`)

```json
"description": {
"type": "richtext",
"pos": 3
}
```

---

## Optional but Recommended Field Properties

Add these to field definitions where appropriate:

| Property        | Type    | When to add                                                                            |
|-----------------|---------|----------------------------------------------------------------------------------------|
| `display_name`  | string  | Optional. A human-readable label for the editor, in case the key isn't speaking enough |
| `required`      | boolean | Set `true` when the inventory field has no `?` (i.e. it's not optional)                |
| `description`   | string  | Add a brief hint if the field purpose isn't self-evident                               |
| `translatable`  | boolean | Set `true` for text content fields that editors would translate                        |

---

## Component-Level Properties

| Property       | Type    | Rule                                                                                                   |
|----------------|---------|--------------------------------------------------------------------------------------------------------|
| `name`         | string  | PascalCase. Derived from the component heading (e.g. "Form Button").                                   |
| `display_name` | string  | Optional, for clarity. Human-readable. Use the heading as-is (e.g. `"Form Button"`).                   |
| `is_root`      | boolean | `false` for all nestable/block components. `true` only for page-level content types. Default: `false`. |
| `is_nestable`  | boolean | `true` for all nestable components. Default: `true`.                                                   |
| `schema`       | object  | See above. MUST be a plain object.                                                                     |

---

## AEM `id` Field Handling

Most AEM components include `id?: string`. This is an AEM-internal HTML ID hint. In Storyblok you can include it as a `text` field for parity, but it is optional. If in doubt, include it at the end with a low priority display name:

```json
"id": {
"type": "text",
"pos": 99,
"display_name": "HTML ID (optional)"
}
```

---

## Full Worked Example

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
        "display_name": "Pretitle",
        "translatable": true
      },
      "title": {
        "type": "text",
        "pos": 1,
        "display_name": "Title",
        "translatable": true
      },
      "description": {
        "type": "richtext",
        "pos": 2,
        "display_name": "Description",
        "translatable": true
      },
      "actionsEnabled": {
        "type": "boolean",
        "pos": 3,
        "display_name": "Actions Enabled"
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

## Another Worked Example — Enum Field

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

## Another Worked Example — Asset Field

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
        "filetypes": ["images"]
      },
      "alt": {
        "type": "text",
        "pos": 1,
        "display_name": "Alt Text",
        "translatable": true
      },
      "linkUrl": {
        "type": "multilink",
        "pos": 2,
        "display_name": "Link URL"
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

## Common Mistakes to Avoid

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
