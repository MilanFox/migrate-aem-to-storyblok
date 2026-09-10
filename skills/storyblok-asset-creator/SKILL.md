---
name: storyblok-asset-creator
description: Reads .inventory/aem-asset-inventory.json, uploads every local asset binary to a Storyblok space via the Storyblok MCP, and writes .inventory/storyblok-asset-map.json mapping AEM paths to Storyblok asset URLs. External assets (not on disk) are logged as skipped. Run this skill after aem-asset-inventory and before storyblok-page-creator.
---

# Skill: storyblok-asset-creator

## Purpose

Read `.inventory/aem-asset-inventory.json`, upload every local asset to Storyblok using the Storyblok MCP, then write a mapping file — `.inventory/storyblok-asset-map.json` — that records the Storyblok asset URL for every AEM DAM path. This mapping is consumed by the `storyblok-page-creator` skill when converting `fileReference` fields.

External assets (those without a local binary on disk) are logged as skipped. They will require manual upload or external sourcing.

---

## Prerequisites

1. `.inventory/aem-asset-inventory.json` must exist and pass its QC checklist.
2. The Storyblok MCP must be connected and responding.
3. **Space ID** (numeric) and **Region** (`eu`, `us`, `ap`, `ca`, or `cn`) must be provided by the user.
4. The shell environment must support running `curl` commands (required to push binary data to the S3 pre-signed URL that the MCP returns).

---

## Step 0 — Pre-flight Checks

Run these checks before processing any asset. Do not proceed until all pass.

### Check 1: MCP available?

Call `search("upload asset")` against the Storyblok MCP. If the call fails or returns no results, stop and report:

> "The Storyblok MCP is not connected or not responding. Please verify the MCP server is running and try again."

### Check 2: Space ID and Region confirmed?

You need both values. If either is missing, ask:

> "Before starting: what is your Storyblok Space ID and which region is your space in? (eu / us / ap / ca / cn)"

Store both — every MCP call in this skill uses them.

### Check 3: Inventory file exists?

Read `.inventory/aem-asset-inventory.json`. If the file does not exist, stop and report:

> "The asset inventory file is missing. Please run the `aem-asset-inventory` skill first."

### Check 4: curl available?

Run `curl --version`. If it fails, stop and report:

> "curl is not available in the current shell environment. It is required to upload binary data to Storyblok's S3 storage. Please ensure curl is installed and retry."

---

### Check 5: Has the inventory been reviewed?

This skill writes to a live Storyblok space, and every later phase builds on what it creates. An unreviewed inventory propagates its mistakes into every uploaded asset, and then into every page that references them.

If the `migrate-aem-to-storyblok` skill invoked this one, its review checkpoint has already run — continue.

Otherwise, ask the user and wait for an explicit answer:

> "Has `.inventory/aem-asset-inventory.json` been reviewed and corrected? This will upload assets in Storyblok space <Space ID>. (yes / no)"

If the answer is anything other than a clear yes, stop and report:

> "Stopping. Review the inventory file first, then re-run this skill."

Do not proceed. Exit.

---

## Step 1 — Load the Inventory

Parse `.inventory/aem-asset-inventory.json` fully into memory.

Build three lists:

- **uploadable**: assets where `source === "local"` AND `localFilePath` is not `null` AND `binaryMissing !== true`
- **resolved**: assets where `localFilePath` is `null` AND `resolvedUrl` is not `null` — these are external assets whose URL is already known; they will be mapped directly without uploading
- **skipped**: all other assets (no local binary AND no resolved URL)

Report to the user before starting:

```
Asset inventory loaded.
  Uploadable:  [n] local assets with binaries on disk
  Resolved:    [n] external assets with a known URL (will be mapped directly, no upload)
  Skipped:     [n] assets with no binary and no resolved URL

Starting upload…
```

---

## Step 2 — Create Storyblok Asset Folders (Optional but Recommended)

Storyblok supports organising assets into folders. Mirror the DAM folder structure by creating a folder for each unique path prefix in the inventory.

### Algorithm

1. Collect all unique parent path prefixes from `asset.id` values (e.g. `wknd/en/site` → folders `wknd`, `wknd/en`, `wknd/en/site`).
2. Sort by depth (ascending) so parents are created before children.
3. For each folder path:
   a. Search the Storyblok MCP for an existing asset folder with that name.
   b. If not found, create it. Use the last path segment as the folder name.
   c. Store the returned `id` in a `folderIdMap: Map<folderPath, folderId>`.

**MCP calls:**

```
search("list asset folders")
describe(<operationId>) → verify parameters
execute_readonly(<operationId>, { space_id: <spaceId> }) → check for existing folders

search("create asset folder")
describe(<operationId>)
execute_mutating(<operationId>, { space_id: <spaceId>, body: { asset_folder: { name: "<last segment>" } } })
```

If folder creation fails (e.g. already exists with a different ID), log the issue and continue — asset uploads without a folder ID are still valid (they land in the root).

---

## Step 3 — Upload Each Asset

Process assets from the `uploadable` list **sequentially** (not in parallel). Parallel uploads risk S3 race conditions and make error recovery harder.

For each asset, execute the following sub-steps.

### 3a — Determine the target folder ID

Look up the asset's parent folder path in `folderIdMap` (built in Step 2). If not found (because Step 2 was skipped or folder creation failed), use `null` — the MCP will place the asset in the root.

### 3b — Register the asset with Storyblok MCP

Use the `upload_asset` MCP tool directly (it is provided as a dedicated tool in this environment):

```
mcp__Storyblok__upload_asset({
  space_id: <spaceId>,
  filename: "<asset filename with extension>",   // last segment of asset.aemPath, e.g. "wknd-logo-dk.png"
  title: "<asset.displayName>",
  alt: "<asset.altText>",
  asset_folder_id: <folderIdMap[parentPath] or omit if null>
})
```

This call returns:
- `id` — the Storyblok asset ID (integer)
- `fields` — a map of form fields for the S3 upload
- `post_url` — the S3 endpoint URL to POST to
- `pretty_url` — the final public CDN URL the asset will be accessible at after upload

**Store all returned values** — especially `id`, `post_url`, `fields`, and `pretty_url`. You need them in the next sub-steps.

### 3c — Upload the binary to S3

Use the asset's `localFilePath` directly (only `uploadable` assets reach this step).

The `upload_asset` tool returns a ready-to-use `curl` command in its response. Run it directly from the shell.

If the curl command is not provided, construct it manually from the returned `post_url` and `fields`:

```bash
curl -X POST "<post_url>" \
  -F "Content-Type=<asset.mimeType>" \
  -F "key=<fields.key>" \
  -F "x-amz-credential=<fields.x-amz-credential>" \
  -F "x-amz-algorithm=<fields.x-amz-algorithm>" \
  -F "x-amz-date=<fields.x-amz-date>" \
  -F "x-amz-signature=<fields.x-amz-signature>" \
  -F "policy=<fields.policy>" \
  -F "file=@<asset.localFilePath>"
```

**Exactly match the field names returned by the MCP** — do not guess or add extra fields. The S3 pre-signed URL is strict about the fields present.

A successful S3 upload returns HTTP 204 No Content. Any other status code is an error.

### 3d — Finalise the asset in Storyblok

After a successful S3 upload, call the `upload_asset_finish` MCP tool to tell Storyblok the upload is complete and the asset should be processed:

```
mcp__Storyblok__upload_asset_finish({
  space_id: <spaceId>,
  asset_id: <id returned in 3b>
})
```

On success, Storyblok returns the finalised asset object including its public URL. Extract the URL from the response (`filename` field or equivalent — check the response shape and use the field that contains the CDN URL).

**Store the mapping:** `asset.aemPath → finalised CDN URL`

### 3e — Record the result

After each asset (success or failure), log one line:

```
✅ wknd/en/site/wknd-logo-dk.png → https://a.storyblok.com/f/12345/…/wknd-logo-dk.png
❌ wknd/en/site/not-found.jpg — S3 upload failed (HTTP 403)
```

---

## Step 4 — Error Handling

### On sub-step 3b failure (MCP registration error)

1. Log the error with the full MCP response.
2. Wait 2 seconds and retry once.
3. If the second attempt also fails, mark the asset as `error` and continue to the next asset. Do not abort the entire run.

### On sub-step 3c failure (S3 upload error)

1. Log the HTTP status code and any response body.
2. Do **not** call `upload_asset_finish` — the binary was not uploaded.
3. Mark the asset as `error`, continue to the next asset.

### On sub-step 3d failure (finalise error)

1. Log the error. The binary is on S3 but Storyblok does not know about it.
2. Mark the asset as `partial-error` in the run log (uploaded to S3 but not finalised).
3. Continue to the next asset.

### After 2 failed attempts on a single asset

Stop retrying that asset and mark it `error`. Do not ask the user for each failure — surface all failures in the final summary.

---

## Step 5 — Write the Asset Map File

After all uploads are complete (regardless of errors), write the mapping file.

Output path:

```
<project-root>/.inventory/storyblok-asset-map.json
```

Structure:

```jsonc
{
  "meta": {
    "spaceId": 12345,
    "generatedOn": "YYYY-MM-DDTHH:MM:SSZ",
    "totalAssets": 42,
    "uploaded": 5,
    "skipped": 0,
    "resolved": 37,                            // external assets mapped via resolvedUrl (no upload)
    "errors": 0
  },
  "map": {
    "/content/dam/wknd/en/site/wknd-logo-dk.png": "https://a.storyblok.com/f/12345/abc/wknd-logo-dk.png",
    "/content/dam/wknd/en/site/wknd-logo-light.svg": "https://a.storyblok.com/f/12345/def/wknd-logo-light.svg",
    "/content/dam/wknd-shared/en/activities/climbing/sport-climbing.jpg": "https://wknd.site/content/dam/wknd-shared/en/activities/climbing/sport-climbing.jpg",
    "/content/dam/wknd-shared/en/activities/climbing/truly-missing.jpg": null
  },
  "skipped": [
    {
      "aemPath": "/content/dam/wknd-shared/en/activities/climbing/truly-missing.jpg",
      "reason": "external — no local binary and no resolved URL",
      "usedOnPages": ["/en/adventures/climbing", "…"],
      "componentAltTexts": ["A climber on a cliff face"]
    }
  ],
  "errors": [
    {
      "aemPath": "/content/dam/wknd/en/site/broken.jpg",
      "reason": "S3 upload failed: HTTP 403",
      "storyblokAssetId": null
    }
  ]
}
```

**Key rules for `map`:**

- Every `aemPath` from the inventory must have an entry in `map`.
- For successfully **uploaded** assets, the value is the final public Storyblok CDN URL.
- For **resolved** assets (external, `resolvedUrl` not null), the value is the `resolvedUrl` from the inventory — no upload is performed.
- Use `null` only for assets that are truly unresolvable (skipped or errored).
- The `storyblok-page-creator` skill will read this file to resolve `fileReference` values. Any entry with `null` means the page creator must handle the missing asset gracefully (skip the field or use a placeholder).

---

## Step 6 — Quality Control (Required Final Step)

Re-read both the written map file and the upload log, then verify:

**Coverage:**
- [ ] `meta.totalAssets` equals `meta.uploaded + meta.skipped + meta.resolved + meta.errors`.
- [ ] Every asset from the inventory has exactly one entry in `map`.
- [ ] Every asset with `binaryMissing: true` AND `resolvedUrl: null` appears in `skipped` with `reason` containing "binary missing".
- [ ] Every asset with `source: "external"` AND `resolvedUrl: null` appears in `skipped` with `reason` containing "external".
- [ ] Every asset with `resolvedUrl` not null (and no `localFilePath`) has that `resolvedUrl` in `map` (not `null`).

**Uploaded and resolved assets:**
- [ ] Every non-null value in `map` is a valid HTTPS URL (starts with `https://`).
- [ ] No S3 pre-signed URLs (`X-Amz-*` query params) are present — only final CDN or resolved URLs.
- [ ] `meta.uploaded` matches the count of map values that are Storyblok CDN URLs.
- [ ] `meta.resolved` matches the count of map values that come from `resolvedUrl`.

**Skipped and errors:**
- [ ] `skipped` array length equals `meta.skipped`.
- [ ] `errors` array length equals `meta.errors`.
- [ ] All error entries have a non-empty `reason`.

**File integrity:**
- [ ] The file is valid JSON.
- [ ] The output file exists at exactly `.inventory/storyblok-asset-map.json`.

**Spot-check (do at least 2):**
- [ ] Pick 2 uploaded assets from `map` and verify their Storyblok CDN URLs return HTTP 200 when fetched (or are reachable via a HEAD request).

### Final output to user

After QC, report:

```
Storyblok asset upload complete.

  Uploaded:  [n] assets
  Skipped:   [n] assets (external — no local binary)
  Errors:    [n] assets — see .inventory/storyblok-asset-map.json

Asset map written to .inventory/storyblok-asset-map.json.
This file is required by the storyblok-page-creator skill.
```

If there are resolved assets (external with a known URL), add:

> "[n] assets were not uploaded to Storyblok — their original `resolvedUrl` has been stored in the map directly. These will work as long as the source site remains available. To host them on Storyblok instead, upload them manually and update their entries in `.inventory/storyblok-asset-map.json`."

If there are truly skipped assets (no binary, no resolved URL), add:

> "[n] assets could not be mapped — they have no local binary and no resolved URL. You must source and upload these assets manually and update the `map` entries in `.inventory/storyblok-asset-map.json` from `null` to the correct Storyblok CDN URLs before running `storyblok-page-creator`."

---

## Notes

- **Do not publish assets.** Asset uploads in Storyblok are not published/unpublished in the same way as stories; they are simply available once finalised. No publish call is needed.
- **Filename conflicts:** If Storyblok already has an asset with the same filename in the same folder, the MCP will either update or create a new version. Check the response — if an existing asset ID is returned, record it as-is.
- **SVG files:** Upload SVGs the same as any other binary. Storyblok accepts `image/svg+xml`. Set `mimeType` accordingly in the `Content-Type` field of the curl command.
- **The asset map is append-safe:** If the skill is run a second time (e.g. to retry failed uploads), it should read any existing `storyblok-asset-map.json` and skip assets that already have a non-null URL, only processing the `null` entries. This prevents double-uploading successfully uploaded assets and preserves existing resolved URLs.
