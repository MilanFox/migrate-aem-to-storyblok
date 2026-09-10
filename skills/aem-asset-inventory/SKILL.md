---
name: aem-asset-inventory
description: Scans the AEM DAM tree and all page content, inventories every asset (local and externally referenced) with its metadata and binary file path, then produces .inventory/aem-asset-inventory.json. Run this skill before any Storyblok asset upload work. Requires no other inventory files to exist first.
---

# Skill: aem-asset-inventory

## Purpose

Walk the AEM Digital Asset Manager (DAM) content tree and all page content to produce a single machine-readable JSON file — `.inventory/aem-asset-inventory.json` — that captures every asset: its metadata (title, alt text, dimensions, MIME type), the local file-system path to its highest-resolution binary (or a note that it is external), and every page that references it.

This file is the single source of truth for the downstream `storyblok-asset-creator` skill.

---

## Key Concepts: AEM DAM on Disk

The AEM content repository is exported to disk using the **FileVault / VLT** format. Understanding this format is essential for finding assets correctly.

### Asset node layout

Each asset in the DAM maps to a **directory** on disk. The directory name is the asset's filename (including extension), e.g. `wknd-logo-dk.png/`. Inside every asset directory:

```
wknd-logo-dk.png/
  .content.xml                         ← asset root node (jcr:primaryType="dam:Asset")
  _jcr_content/
    .content.xml                       ← asset content node (jcr:primaryType="dam:AssetContent")
                                          contains: dc:title, dc:description, dam:MIMEtype,
                                          tiff:ImageWidth, tiff:ImageLength, dam:size, dam:sha1
    renditions/
      original                         ← BINARY FILE: the source upload (always present)
      original.dir/
        .content.xml                   ← metadata node for "original" rendition
      cq5dam.thumbnail.48.48.png       ← BINARY FILE: 48×48 thumbnail
      cq5dam.thumbnail.48.48.png.dir/
        .content.xml
      cq5dam.thumbnail.140.100.png     ← BINARY FILE: 140×100 thumbnail
      cq5dam.thumbnail.319.319.png     ← BINARY FILE: 319×319 thumbnail
      cq5dam.web.1280.1280.png         ← BINARY FILE: web-optimised large rendition (PNG)
      cq5dam.web.1280.1280.jpeg        ← BINARY FILE: web-optimised large rendition (JPEG)
```

**Critical rule for identifying binary files vs. metadata directories:**

In a FileVault export, JCR binary properties are stored as plain files. The directory that holds a binary's metadata has the **same name as the binary with `.dir` appended**. Therefore:

- A path **without** a `.dir` suffix (and not a directory) → **actual binary file** (the asset data)
- A path **with** a `.dir/` suffix → a metadata directory, not a binary; contains only `.content.xml`

When looking for binaries, always check that the path has no `.dir` suffix and is a file, not a directory.

### Rendition priority

Use the following priority order when selecting the binary to record as `localFilePath` in the inventory (highest quality first):

1. `original` — always the source, highest quality, native format
2. `cq5dam.web.1280.1280.png` — large web-optimised PNG
3. `cq5dam.web.1280.1280.jpeg` — large web-optimised JPEG (fallback if no PNG)
4. `cq5dam.thumbnail.319.319.png` — medium thumbnail (last resort)

If none of the above exist, record `localFilePath: null` and note `binaryMissing: true`.

### DAM root location

The DAM content tree lives at:

```
<project-root>/aem-guides-wknd/ui.content.sample/src/main/content/jcr_root/content/dam/
```

If this path does not exist, also check:

```
<project-root>/aem-guides-wknd/ui.content/src/main/content/jcr_root/content/dam/
```

The first location (`ui.content.sample`) is the sample-content module and typically contains the bulk of demo assets. The second is the base content module and may contain structural-only entries.

### Page content root

`fileReference` usages are found in the page content tree at:

```
<project-root>/aem-guides-wknd/ui.content.sample/src/main/content/jcr_root/content/wknd/
```

## Step 0 — Pre-flight Checks

Run these checks before processing any asset. Do not proceed until all pass.

### Check 1: AEM Base URL (Required)

External `fileReference` values (assets not present in the local DAM) must be resolved to absolute URLs so that downstream skills can fetch their binaries. Ask the user for the AEM base URL before doing any other work:

> "What is the AEM base URL for this project? (e.g. `https://author-p123-e456.adobeaemcloud.com`) This is required to resolve external asset references to absolute URLs."

Wait for the answer. **Do not proceed until a URL is provided.** Store it as `aemBaseUrl` (trim any trailing slash).

---

## Step 1 — Discover Local Assets

A directory is a DAM asset if its `.content.xml` contains `jcr:primaryType="dam:Asset"`.

**Search command:**

```bash
find . -name ".content.xml" | xargs grep -l 'jcr:primaryType="dam:Asset"'
```

Run this search inside the DAM root directory identified above.

For each match, the **parent directory** is one asset. Record:

| Field | Source |
|---|---|
| `aemPath` | The JCR path: replace the file-system prefix up to and including `jcr_root` with nothing, so `…/jcr_root/content/dam/wknd/en/site/wknd-logo-dk.png` → `/content/dam/wknd/en/site/wknd-logo-dk.png` |
| `assetDir` | Absolute file-system path to the asset directory (for resolving binary paths below) |

**Skip these:**

- Directories with `jcr:primaryType="dam:AssetContent"` (these are the child `_jcr_content` nodes, not the asset root)
- Any path containing `rep:policy` or `_oak_index`
- Folder thumbnail assets (path segment `folderThumbnail`)

---

## Step 2 — Read Asset Metadata

For each discovered asset, parse its metadata. Metadata lives in two locations:

### Primary: asset content node

Read `[assetDir]/_jcr_content/.content.xml`. This file has `jcr:primaryType="dam:AssetContent"`. The attributes on this node (or its `metadata` child node) provide:

| Output field | XML attribute | Notes |
|---|---|---|
| `title` | `dc:title` | Human-readable name; fall back to asset filename (without extension) if absent |
| `description` | `dc:description` | `null` if absent |
| `mimeType` | `dam:MIMEtype` | e.g. `"image/png"`, `"image/jpeg"`, `"image/svg+xml"` |
| `width` | `tiff:ImageWidth` | Strip `{Long}` prefix → integer; `null` for non-image types |
| `height` | `tiff:ImageLength` | Strip `{Long}` prefix → integer; `null` for non-image types |
| `fileSize` | `dam:size` | Strip `{Long}` prefix → integer bytes; `null` if absent |
| `sha1` | `dam:sha1` | String; `null` if absent |

**Note on XML attribute location:** In some exports, properties are on the `_jcr_content/.content.xml` node itself; in others they appear on a nested `<metadata …>` child element. Read both locations and merge — the child `metadata` node attributes take precedence.

**Namespace prefixes used in the XML:**

| Prefix | Namespace |
|---|---|
| `dc:` | Dublin Core — `http://purl.org/dc/elements/1.1/` |
| `dam:` | AEM DAM — `http://www.day.com/dam/1.0` |
| `tiff:` | TIFF/Exif — `http://ns.adobe.com/tiff/1.0/` |

### Alt text derivation

AEM separates the DAM metadata (`dc:title`) from component-level alt text (`alt=` on the image component). Both are useful:

- `altText` → use `dc:description` first (most likely to be descriptive), then `dc:title`, then the asset filename without extension
- `title` → use `dc:title`, then the asset filename without extension

---

## Step 3 — Resolve Binary File Paths

For each asset, locate the highest-priority binary rendition by walking the `renditions/` directory and checking for binary files using the priority order from the Key Concepts section.

**Algorithm:**

```
renditionsDir = assetDir + "/_jcr_content/renditions"

for each candidateName in ["original", "cq5dam.web.1280.1280.png", "cq5dam.web.1280.1280.jpeg", "cq5dam.thumbnail.319.319.png"]:
    candidate = renditionsDir + "/" + candidateName
    if candidate exists AND is a file AND does NOT end in ".dir":
        localFilePath = candidate (absolute file-system path)
        break

if no candidate found:
    localFilePath = null
    binaryMissing = true
```

Also collect **all** available binary renditions for the asset (for completeness):

```
renditions: [
  { "name": "original", "localFilePath": "…/renditions/original" },
  { "name": "cq5dam.web.1280.1280.png", "localFilePath": "…/renditions/cq5dam.web.1280.1280.png" },
  …
]
```

Only include entries where the binary file actually exists on disk.

---

## Step 4 — Scan Page Content for References

All component instances reference assets via the `fileReference` attribute. Scan all `.content.xml` files inside the **page content root** directory for this attribute.

**Search command (run from the project root):**

```bash
grep -r 'fileReference=' --include="*.xml" -l <page-content-root>
```

Then extract every `fileReference` value from every matching file.

**For each reference found:**

1. Note the value (e.g. `/content/dam/wknd/en/site/wknd-logo-dk.png` or `/content/dam/wknd-shared/en/activities/climbing/sport-climbing.jpg`)
2. Note which page it came from — derive the page slug by stripping the `jcr_root/content/wknd` prefix and all `_jcr_content/…` suffix parts from the containing file's path.
3. Also note any `alt` attribute on the same XML node — this is a component-level override for alt text.

Build a usage map:

```
usageMap: Map<aemPath, { pages: Set<slug>, componentAltTexts: Set<string> }>
```

---

## Step 5 — Identify External Assets

Many `fileReference` values in the page content will point to DAM paths that are **not present** in the local project DAM tree (e.g. `/content/dam/wknd-shared/…`). These are assets served from a shared or external DAM and have no local binary.

For each `fileReference` value collected in Step 4:

- If the path was found in the local DAM in Step 1 → already handled, `source: "local"`
- If the path was NOT found in Step 1 → it is an external reference. Create an entry with `source: "external"` and `localFilePath: null`.

**URL Resolution:**

Construct a `resolvedUrl` for every asset (both local and external) using `aemBaseUrl` from Step 0:

1. Start with `aemBaseUrl` (no trailing slash).
2. Append the `aemPath` (which begins with `/`).

Example: `https://author-p123-e456.adobeaemcloud.com/content/dam/wknd-shared/en/activities/climbing/sport-climbing.jpg`

**`resolvedUrl` is mandatory for all external assets.** If `aemBaseUrl` was not captured in Step 0, stop here and ask for it before continuing.

For local assets, also set `resolvedUrl` using the same pattern — it is useful for downstream skills even when a local binary exists.

For external assets, the metadata fields (`title`, `altText`, `mimeType`, etc.) cannot be read from the DAM. Infer what you can from the path:

- `name` → last path segment without extension (e.g. `sport-climbing`)
- `displayName` → `name`, humanised (replace hyphens with spaces, title-case): `"Sport Climbing"`
- All other fields → `null`

Record the `componentAltTexts` from Step 4 in the `usages` object — these are the only alt text values available for external assets.

---

## Step 6 — Build the Output JSON

### Asset ID

The `id` field is the `aemPath` with the leading `/content/dam/` stripped:

```
/content/dam/wknd/en/site/wknd-logo-dk.png → wknd/en/site/wknd-logo-dk.png
```

This ID is stable, human-readable, and unique within the inventory.

### JSON structure

```jsonc
{
  "meta": {
    "project": "<project-namespace>",          // e.g. "wknd"
    "aemBaseUrl": "<baseUrl or null>",
    "damRoot": "<relative path to jcr_root/content/dam>",
    "pageContentRoot": "<relative path to jcr_root/content/<namespace>>",
    "totalAssets": 42,
    "localAssets": 5,                          // assets with binaries on disk
    "externalAssets": 37,                      // referenced but not in project DAM
    "missingBinaries": 0,                      // local assets where no binary was found on disk
    "generatedOn": "YYYY-MM-DD",
    "notes": [
      "37 external assets reference /content/dam/wknd-shared/ which is not part of this project. These must be sourced externally before upload.",
      "…"
    ]
  },
  "assets": [
    {
      "id": "wknd/en/site/wknd-logo-dk.png",
      "name": "wknd-logo-dk",
      "displayName": "WKND Logo Dark",
      "aemPath": "/content/dam/wknd/en/site/wknd-logo-dk.png",
      "source": "local",                       // "local" | "external"
      "resolvedUrl": "https://<aemBaseUrl>/content/dam/...", // constructed URL or null
      "localFilePath": "/abs/path/to/renditions/original",  // null if external or missing
      "binaryMissing": false,                  // true if local asset has no binary on disk
      "mimeType": "image/png",
      "width": 301,
      "height": 128,
      "fileSize": 3106,
      "sha1": "abc123…",
      "title": "WKND Logo Dark",
      "altText": "WKND Logo Dark",
      "description": null,
      "renditions": [
        {
          "name": "original",
          "localFilePath": "/abs/path/to/renditions/original",
          "mimeType": "image/png"
        },
        {
          "name": "cq5dam.web.1280.1280",
          "localFilePath": "/abs/path/to/renditions/cq5dam.web.1280.1280.png",
          "mimeType": "image/png"
        }
      ],
      "usages": {
        "totalCount": 3,
        "pages": ["/en/adventures/climbing", "/en/adventures/hiking"],
        "componentAltTexts": ["A person rock climbing"]  // alt text overrides found in component XML
      }
    },
    {
      "id": "wknd-shared/en/activities/climbing/sport-climbing.jpg",
      "name": "sport-climbing",
      "displayName": "Sport Climbing",
      "aemPath": "/content/dam/wknd-shared/en/activities/climbing/sport-climbing.jpg",
      "source": "external",
      "localFilePath": null,
      "binaryMissing": null,
      "mimeType": null,
      "width": null,
      "height": null,
      "fileSize": null,
      "sha1": null,
      "title": null,
      "altText": null,
      "description": null,
      "renditions": [],
      "usages": {
        "totalCount": 12,
        "pages": ["/en/adventures/climbing", "…"],
        "componentAltTexts": ["A climber on a cliff face", "Sport climbing in the mountains"]
      }
    }
  ]
}
```

**Sorting:** Sort `assets` array alphabetically by `aemPath`. Local assets first within each group is not required — alphabetical by `aemPath` is sufficient and predictable.

---

## Step 7 — Write the Output File

Output path:

```
<project-root>/.inventory/aem-asset-inventory.json
```

Write valid, pretty-printed JSON (2-space indent).

---

## Step 8 — Quality Control (Required Final Step)

Re-read the written file and verify:

**Coverage:**
- [ ] `meta.totalAssets` equals `meta.localAssets + meta.externalAssets`.
- [ ] `meta.localAssets` equals the count of assets from Step 1.
- [ ] `meta.externalAssets` equals the count of unique external `fileReference` values found in Step 4 that have no local DAM entry.
- [ ] `meta.missingBinaries` equals the count of local assets where `localFilePath` is `null`.
- [ ] Every asset from Step 1 is present in the `assets` array.
- [ ] Every unique external `fileReference` from Step 4 is present in the `assets` array.

**Fields:**
- [ ] Every `source: "external"` asset has a non-null `resolvedUrl` that is a well-formed absolute URL.
- [ ] Every `source: "local"` asset has a `mimeType` value (not `null`).
- [ ] Every `source: "local"` asset has a non-empty `renditions` array (at least the `original` entry if the file exists).
- [ ] No asset with `source: "local"` and an existing `renditions/original` binary has `localFilePath: null`.
- [ ] All `width` and `height` values are integers, not raw AEM strings like `{Long}301`.
- [ ] All `fileSize` values are integers in bytes.
- [ ] Every asset has a `displayName` — no `null` display names.

**Usages:**
- [ ] `usages.totalCount` equals the length of `usages.pages`.
- [ ] All page slugs in `usages.pages` are normalised (no `/content/wknd/` prefix, no `_jcr_content` suffix).
- [ ] No AEM-internal paths appear anywhere in `usages.pages`.

**File integrity:**
- [ ] The file is valid JSON (parseable without errors).
- [ ] The output file exists at exactly `.inventory/aem-asset-inventory.json` relative to the project root.

Fix any issues before considering the skill complete.

---

## Reference: AEM Rendition Naming Conventions

| Rendition filename pattern | Description |
|---|---|
| `original` | The source file as uploaded. Always the highest quality. No file extension in FileVault export. |
| `cq5dam.web.1280.1280.png` | AEM-generated web rendition, max 1280×1280, PNG format |
| `cq5dam.web.1280.1280.jpeg` | Same as above, JPEG format (generated for SVGs and some other types) |
| `cq5dam.thumbnail.319.319.png` | Medium thumbnail (319×319) |
| `cq5dam.thumbnail.140.100.png` | Small thumbnail (140×100) |
| `cq5dam.thumbnail.48.48.png` | Tiny thumbnail (48×48) |

The `cq5dam.*` renditions are AEM-generated derivatives. The `original` is always the authoritative source file.
