---
name: migrate-aem-to-storyblok
description: Master migration skill that runs the full AEM-to-Storyblok pipeline end-to-end with human review checkpoints between each phase. Orchestrates component inventory, component creation, asset inventory, asset upload, page inventory, and page creation in order. Use this when you want to migrate the content of an AEM project to Storyblok in one guided session.
---

# Skill: migrate-aem-to-storyblok

## Purpose

Migrates the content from AEM to Storyblok headless CMS.
Run the complete AEM-to-Storyblok migration pipeline from start to finish, pausing for human review after each inventory phase before proceeding to the corresponding Storyblok creation phase. This skill orchestrates all six sub-skills in order and requires a connected Storyblok MCP and a target Storyblok space.

---

## Step 0 — Setup and Pre-flight

### 0a — Welcome

Greet the user and explain what is about to happen:

> "Welcome to the AEM → Storyblok migration pipeline. This will guide you through the full migration in six phases:
>
> 1. Component inventory → review → create components in Storyblok
> 2. Asset inventory → review → upload assets to Storyblok
> 3. Page inventory → review → create pages in Storyblok
>
> You will have a chance to review the inventory after each scan before anything is written to Storyblok.
>
> Let's start with a few setup questions."

### 0b — Collect Space ID and Region

Ask the user:

> "What is your Storyblok Space ID and which region is your space in? (eu / us / ap / ca / cn)"

Wait for the answer. Store both values — every Storyblok MCP call in this pipeline uses them. Do not proceed until both are provided.

### 0c — Test Storyblok MCP Connection

Attempt a lightweight read call against the Storyblok MCP to confirm it is connected. Use:

```
search("list components")
```

If the call fails or the MCP tool is not available, stop immediately and tell the user:

> "The Storyblok MCP is not connected. Please connect it in your MCP settings and then restart this skill.
>
> To connect the Storyblok MCP, add it to your MCP configuration with a valid Personal Access Token for the space you want to migrate into."

Do not proceed with any further steps until the connection is confirmed.

If the call succeeds, continue and report:

> "Storyblok MCP connection confirmed. Space ID: [spaceId], Region: [region]."

### 0d — Check for Existing Inventory

Check if the `.inventory/` directory exists and contains any files.

If the folder is not empty, ask the user:

> "I found existing files in the `.inventory/` folder. These might be from a previous run:
>
> [List files found in .inventory/]
>
> Would you like to:
>
> 1. **Keep and use** these files — the pipeline will skip inventory phases if the required files already exist.
> 2. **Delete and restart** — the `.inventory/` folder will be cleared, and all inventory phases will run fresh."

Wait for the user's answer.

- **Keep and use** → do nothing. Proceed to report: `"Keeping existing inventory. Starting Phase 1."`
- **Delete and restart** → delete all files within the `.inventory/` directory. Proceed to report: `"Inventory cleared. Starting Phase 1."`

If the folder is empty or does not exist, proceed directly to report: `"Starting Phase 1."`

---

## Phase 1 — Component Migration

### Step 1a — Component Inventory

Run the `aem-component-inventory` skill in full.

This skill scans the AEM project, resolves the Sling inheritance chain for every component, and writes the result to `.inventory/aem-component-inventory.json`.

Follow all steps in that skill exactly, including its Quality Control checklist. Do not move forward until the QC checklist passes.

When the inventory file is written and validated, report a summary to the user:

> "Component inventory complete.
>
> - Total components: [N]
> - Excluded (hidden/structural): [N]
> - Output: `.inventory/aem-component-inventory.json`"

### Step 1b — Human Review Checkpoint: Components

Pause and present the component list to the user for review. Show a compact table:

```
Component Name        | Fields | Inherits From
----------------------|--------|---------------------------
Accordion             | 3      | core/wcm/components/accordion/v1/accordion
Button                | 4      | core/wcm/components/button/v2/button
Byline                | 3      | (none)
...
```

Then ask:

> "Please review the component inventory above. When you are ready, reply with one of:
>
> - **proceed** — continue with component creation in Storyblok as is
> - **skip** — skip component creation entirely and move to Phase 2 (assets)
> - **abort** — stop the pipeline entirely"
> - **refactor** - implement changes to the inventory

Wait for the user's response before continuing.

- `proceed` → continue to Step 1c
- `skip` → jump to Step 2a
- `abort` → stop the pipeline, report what was completed, and exit
- `refactor` → ask the user what changes they want to implement, do that and then come back to this step

### Step 1c — Create Components in Storyblok

Run the `storyblok-component-creator` skill in full, using the Space ID and Region already collected.

This skill reads `.inventory/aem-component-inventory.json`, converts each component to a valid Storyblok component JSON, validates it, and uploads it via the MCP.

Follow all steps in that skill exactly, including its per-component retry logic and error handling.

When the skill completes, report the summary back to the user:

> "Component creation complete.
>
> - Uploaded: [N]
> - Skipped: [N]
> - Failed: [N]"

Then proceed to Phase 2.

---

## Phase 2 — Asset Migration

### Step 2a — Asset Inventory

Run the `aem-asset-inventory` skill in full.

This skill scans the AEM DAM and page content tree to find every referenced asset — both local binary files and external URLs — and writes the result to `.inventory/aem-asset-inventory.json`.

Follow all steps in that skill exactly, including its Quality Control checklist. Do not move forward until the QC checklist passes.

When the inventory file is written and validated, report a summary to the user:

> "Asset inventory complete.
>
> - Total assets found: [N]
> - Local binaries (uploadable): [N]
> - External references (URL only): [N]
> - Output: `.inventory/aem-asset-inventory.json`"

### Step 2b — Human Review Checkpoint: Assets

Pause and present an asset summary to the user. Show counts by type and a sample list (first 10 assets):

```
Type    | Count
--------|------
Images  | 87
Videos  | 4
PDFs    | 6
Other   | 3

Sample assets:
- /content/dam/wknd/en/adventures/hero.jpg  (local, 1.2 MB)
- /content/dam/wknd/en/adventures/cycling/trail.jpg  (local, 0.8 MB)
- /content/dam/wknd-shared/en/activities/climbing/climber.jpg  (external)
...
```

Then ask:

> "Please review the asset inventory above. When you are ready, reply with one of:
>
> - **proceed** — continue with asset upload to Storyblok
> - **skip** — skip asset upload entirely and move to Phase 3 (pages)
> - **abort** — stop the pipeline entirely"
> - **refactor** - implement changes to the inventory

Wait for the user's response before continuing.

- `proceed` → continue to Step 2c
- `skip` → jump to Step 3a
- `abort` → stop the pipeline, report what was completed, and exit
- `refactor` → ask the user what changes they want to implement, do that and then come back to this step


### Step 2c — Upload Assets to Storyblok

Run the `storyblok-asset-creator` skill in full, using the Space ID and Region already collected.

This skill reads `.inventory/aem-asset-inventory.json`, uploads every local binary to Storyblok via the MCP, maps external asset URLs as-is, and writes `.inventory/storyblok-asset-map.json`.

Follow all steps in that skill exactly, including its error handling and the final asset map file.

When the skill completes, report the summary back to the user:

> "Asset upload complete.
>
> - Uploaded to Storyblok: [N]
> - Mapped (external URL, not uploaded): [N]
> - Errors: [N]
> - Asset map: `.inventory/storyblok-asset-map.json`"

Then proceed to Phase 3.

---

## Phase 3 — Page Migration

### Step 3a — Page Inventory

Run the `aem-page-inventory` skill in full.

This skill walks the AEM content tree, extracts every page with its metadata and component instances, resolves field values, and writes the result to `.inventory/aem-page-inventory.json`.

Follow all steps in that skill exactly, including its Quality Control checklist. Do not move forward until the QC checklist passes.

When the inventory file is written and validated, report a summary to the user:

> "Page inventory complete.
>
> - Total pages: [N]
> - Site trees / locales: [list them]
> - Total component instances: [N]
> - Output: `.inventory/aem-page-inventory.json`"

### Step 3b — Human Review Checkpoint: Pages

Pause and present a page tree summary. Show the top-level structure:

```
Site tree:
content/wknd/
  language-masters/en/  (120 pages)
  us/en/                (120 pages)
  ca/en/                (120 pages)
  ...

Sample pages:
- /content/wknd/us/en  →  slug: us/en  (Home)
- /content/wknd/us/en/adventures  →  slug: us/en/adventures
- /content/wknd/us/en/adventures/ski-touring-mont-blanc  →  slug: ...
```

Then ask:

> "Please review the page inventory above. When you are ready, reply with one of:
>
> - **proceed** — continue with page creation in Storyblok
> - **skip** — skip page creation entirely and finish the pipeline
> - **abort** — stop the pipeline entirely"
> - **refactor** - implement changes to the inventory


Wait for the user's response before continuing.

- `proceed` → continue to Step 3c
- `skip` → jump to Step 4 (cleanup)
- `abort` → stop the pipeline, report what was completed, and exit
- `refactor` → ask the user what changes they want to implement, do that and then come back to this step

### Step 3c — Create Pages in Storyblok

Run the `storyblok-page-creator` skill in full, using the Space ID and Region already collected.

This skill reads `.inventory/aem-page-inventory.json` and `.inventory/storyblok-asset-map.json`, creates folder stories for the site tree structure (Phase 1 of that skill), and then creates all content stories as drafts (Phase 2 of that skill).

Follow all steps in that skill exactly, including internal link resolution and run log writing.

When the skill completes, report the summary back to the user:

> "Page creation complete.
>
> - Folders created: [N]
> - Stories created: [N]
> - Errors: [N]
> - All stories saved as drafts (not published)
> - Run log: `.inventory/storyblok-page-run-log.json` (if written by that skill)"

Then proceed to Step 4.

---

## Step 4 — Optional Cleanup

Ask the user:

> "The migration pipeline is complete. The `.inventory/` folder contains all generated inventory files:
>
> - `.inventory/aem-component-inventory.json`
> - `.inventory/aem-asset-inventory.json`
> - `.inventory/storyblok-asset-map.json`
> - `.inventory/aem-page-inventory.json`
>
> Would you like to:
>
> 1. **Keep** the `.inventory/` folder — useful as documentation of what was migrated
> 2. **Delete** the `.inventory/` folder — cleans up the repository

Wait for the user's answer.

- **Keep** → do nothing. Confirm: `"The .inventory/ folder has been kept."`
- **Delete** → delete the entire `.inventory/` directory and all files within it. Confirm: `"The .inventory/ folder has been removed."`

---

## Step 5 — Final Pipeline Summary

Output a complete summary of everything that was done:

```
AEM → Storyblok Migration Complete
====================================

Phase 1 — Components
  Inventoried:  [N] components
  Uploaded:     [N]
  Skipped:      [N]
  Failed:       [N]

Phase 2 — Assets
  Inventoried:  [N] assets
  Uploaded:     [N]
  Mapped (ext): [N]
  Errors:       [N]

Phase 3 — Pages
  Inventoried:  [N] pages
  Stories created: [N]
  Errors:       [N]

Inventory folder: [kept / removed]
```

If any phase was skipped by the user, note it explicitly:

```
  ⏭️  Phase 2 (Assets) was skipped by user.
```

---

## Error Handling Across Phases

- If any inventory skill's QC fails and cannot be resolved: stop at that phase, report the failure, and ask the user how to proceed before continuing.
- If a Storyblok creation skill fails catastrophically (not just a single-component/asset failure): report the error, stop that creation phase, and offer the user the option to retry from that phase or skip to the next one.
- If the MCP connection is lost mid-pipeline: stop immediately and report which phase was in progress. Advise the user to reconnect the MCP and re-run from the failed phase's creation step (inventory files are already written and do not need to be regenerated).
- Never delete or overwrite an inventory file that was already written successfully. Each phase's inventory is independent and can be reused on resume.

---

## Notes

- The Space ID and Region collected in Step 0 apply to all Storyblok MCP calls throughout the entire pipeline. Never ask for them again.
- Each sub-skill is run in full, including its own QC, error handling, and progress reporting. This master skill does not duplicate their logic — it delegates to them completely.
- Inventory files are written to `.inventory/` relative to the project root. Never change their output paths.
- Pages are always created as **drafts** in Storyblok. Publishing is a manual step for the content team.
- The pipeline runs phases sequentially: components → assets → pages. This order is required because page stories may reference asset URLs from the asset map.
