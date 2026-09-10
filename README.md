# AEM → Storyblok migration skills

A pipeline of [Claude Code skills](https://code.claude.com/docs/en/skills) that migrates the **content** of an Adobe Experience Manager project into Storyblok: components, assets, and pages.

It is not a one-click migration. It takes the tedious, mechanical parts — scanning the project, resolving inheritance chains, extracting fields, cataloguing assets, creating stories — and gets them to a *good enough to review* state, with a human checkpoint before anything is written to Storyblok.

These are the skill files accompanying the article *I Used an AI Agent to Migrate an AEM Project to Storyblok*. <!-- TODO: link once published -->

## How it works

Every phase follows the same three steps, and never goes directly from AEM to Storyblok:

1. **Abstract** — scan the AEM project, produce a vendor-neutral JSON description of what's there.
2. **Review** — a human reads that file and confirms the interpretation before anything is written.
3. **Rebuild** — a second pass reads the inventory and reconstructs the model in Storyblok.

The agent doing the abstraction is not thinking about Storyblok. The agent doing the rebuild has never seen AEM. Going through a neutral intermediate forces the migration to re-think the content model rather than mechanically transpose it — otherwise you get a Storyblok space that still thinks like AEM, which defeats the point of moving.

## Requirements

- **Claude Code**, with access to the full AEM codebase
- A **Storyblok MCP server** connected, with a Personal Access Token for the target space
- Your Storyblok **Space ID** and **region** (`eu`, `us`, `ap`, `ca`, `cn`)
- **`curl`** on `PATH` — used to push asset binaries to Storyblok's S3 pre-signed URLs

The Storyblok token needs a role with content-management permissions. The *Developer* role cannot create stories.

## Install

Copy the skills into your AEM project:

```bash
mkdir -p /path/to/aem-project/.claude/skills
cp -R skills/. /path/to/aem-project/.claude/skills/
```

Claude Code discovers skills as `.claude/skills/<name>/SKILL.md`. The layout is flat by necessity — nested directories are not discovered as separate skills, and the command name comes from the directory name, so don't rename the folders.

## Usage

From inside your AEM project, just ask:

> Please migrate my content from AEM to Storyblok.

That runs `migrate-aem-to-storyblok`, which asks for your Space ID and region, then walks all six phases in order and pauses for review between each.

Each skill can also be run on its own with `/<skill-name>` — useful when you're re-running a single phase after correcting an inventory.

## The skills

| Skill | Does | Produces |
|---|---|---|
| `migrate-aem-to-storyblok` | Orchestrates all six phases with review checkpoints | — |
| `aem-component-inventory` | Scans components, resolves the Sling inheritance chain | `.inventory/aem-component-inventory.json` |
| `storyblok-component-creator` | Creates the components in Storyblok | Components in your space |
| `aem-asset-inventory` | Finds every asset reference, local and external | `.inventory/aem-asset-inventory.json` |
| `storyblok-asset-creator` | Uploads local binaries, maps external URLs | `.inventory/storyblok-asset-map.json` |
| `aem-page-inventory` | Walks the content tree, extracts pages and field values | `.inventory/aem-page-inventory.json` |
| `storyblok-page-creator` | Creates folders and stories as **drafts** | Stories + `.inventory/storyblok-page-run-log.json` |
| `storyblok-create-component-definition` | Converts one component to Storyblok JSON | Internal helper, not user-invocable |

All inventory files land in `.inventory/` at the project root. They are the reviewable artifacts — read them.

## The review checkpoints are not optional

The pipeline stops three times: after each inventory, before anything is written to Storyblok.

LLMs are good at pattern recognition and following instructions. They are not good at deciding genuinely ambiguous questions — whether a component is a root-level story type or a nestable block, whether a typo in an AEM dialog is intentional. Faced with those, the agent guesses.

And errors accumulate. A misnamed field in the component inventory becomes a mismatched field in every page story. A component miscategorised as nestable means every page using it is structured wrong. By Phase 3, a Phase 1 mistake has multiplied across potentially hundreds of stories.

The three `storyblok-*-creator` skills refuse to run unless their input inventory exists and the review has been confirmed. When the orchestrator invokes them the checkpoint has already happened, so you won't be asked twice — but a standalone `/storyblok-page-creator` will stop and ask before it touches your space.

Pages are always created as **drafts**. Publishing is a deliberate human step.

## What to expect

Component migration is the hardest phase and needs the most guidance. AEM's inheritance is layered — custom components extend Core Components, which extend WCM Foundation — and producing a complete field list means traversing that whole chain and merging dialogs. Expect to correct things here.

Asset migration is the most reliable. It's mostly pattern matching, with clear success criteria.

On the [WKND reference project](https://github.com/adobe/aem-guides-wknd) (~35 components, ~120 pages across all languages, 116 assets of which 111 were external), nothing worked correctly on the first end-to-end run. Each iteration got faster as the inventory files got more accurate.

If your project has significant custom infrastructure, unusual naming conventions, or a non-standard structure, budget time at the review checkpoints.

## What this does not do

- **The frontend.** Rebuilding HTL templates in your Storyblok frontend framework is a separate, larger effort. The same abstract → review → rebuild idea should apply, but HTL carries a lot of AEM-specific assumptions and abstracting intent from implementation is harder for code than for structured data.
- **Publishing.** Everything lands as drafts.
- **Run unsupervised.** There is too much interpretation between the two systems.

## License

<!-- TODO: pick one before publishing -->
