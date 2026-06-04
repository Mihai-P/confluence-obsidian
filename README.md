# confluence-obsidian

A Claude Code skill that keeps a local Markdown mirror in two-way sync with an Atlassian Confluence space — so your Confluence pages live as plain `.md` files you can read, edit, and version in any Markdown editor (Obsidian, your IDE, etc.) and push back up.

The sync runs entirely through the **Atlassian MCP server**. There is no helper binary, no snapshot file, and no batch importer — every read and write hits the live Confluence API directly.

## Why

Confluence is a fine place to *publish*, but a poor place to *author* and *version*. This project flips that: the source of truth lives on disk as Markdown, where it can be edited offline, diffed, reviewed in pull requests, and dropped straight into an Obsidian vault. Confluence becomes a render/publish target that stays current in both directions.

## What it does

Given a local tree of Markdown files mirroring a Confluence space:

```
confluence/<SPACE>/
├── index.md                 # space root
├── 01-some-section/
│   ├── index.md             # a Confluence page with children
│   └── a-leaf-page.md       # a leaf Confluence page
└── ...
```

…the skill walks the tree, compares each file against its live Confluence counterpart, and decides per page whether to **skip, push, pull, or flag a conflict**. Directory layout maps to Confluence parent/child page hierarchy: a folder's `index.md` is the parent of the pages beside it.

## How it works

Each file carries a sync watermark in its YAML frontmatter:

```yaml
---
title: "..."
confluence:
  id: "262260"            # canonical page id (null until first push)
  url: "https://<site>.atlassian.net/wiki/spaces/SD/pages/262260"
  space_key: "SD"
  space_id: "262148"
  parent_id: "..."        # inferred from directory layout
  version: 5              # the remote version we last synced to
  last_modified: "..."
  body_sha: "<sha256>"    # hash of the body that matched `version`
---
```

On each run, for every page the skill computes:

- `local_changed`  — does the body's `sha256` differ from the stored `body_sha`?
- `remote_changed` — has the live Confluence `version` moved since last sync?

…and applies a decision matrix:

| local changed | remote changed | action |
| --- | --- | --- |
| no  | no  | **skip** |
| yes | no  | **push** (`updateConfluencePage`) |
| no  | yes | **pull** (overwrite local body) |
| yes | yes | **conflict** — stop, show a diff, let you pick a side (never auto-merge) |

New local files (`id: null`) are **created** in Confluence with the parent resolved from the folder structure; new remote pages with no local file are **pulled down** into the matching directory. After any write, the frontmatter watermark (`version`, `last_modified`, `body_sha`) is rewritten so the next run can tell exactly what moved.

Bodies are exchanged in Confluence **`storage` format** (canonical XHTML) so they round-trip losslessly.

### Modes

- **Full sync** (default) — push, pull, and surface conflicts for review.
- **Pull-only** — remote wins; refresh the local mirror from Confluence.
- **Push-only** — local wins; publish local edits, overwriting concurrent remote changes.

Pull-only and push-only both silently overwrite one side, so neither is the default.

## Repository layout

```
.claude-plugin/
└── marketplace.json          # makes this repo an installable Claude Code marketplace
skills/
└── confluence-sync/
    ├── SKILL.md              # the skill definition (frontmatter + core workflow)
    └── references/           # progressive-disclosure detail, loaded on demand
        ├── new-pages.md      # creating pages that exist on only one side
        └── conflict-resolution.md
template/
└── SKILL.md                  # scaffold for authoring a new skill
```

This mirrors the layout of [`anthropics/skills`](https://github.com/anthropics/skills): a top-level `skills/` directory of self-contained skill folders, a `template/` scaffold, and a `.claude-plugin/` manifest. `SKILL.md` follows the [Agent Skills](https://agentskills.io) open standard and the [Claude Code skills](https://code.claude.com/docs/en/skills) conventions — a `references/` folder holds detail the skill loads only when a task needs it, so it costs no context until then.

### Using it

Either copy `skills/confluence-sync/` into your project's `.claude/skills/` (or `~/.claude/skills/` for all projects), or install the whole repo as a marketplace plugin:

```
/plugin marketplace add <path-or-git-url-to-this-repo>
/plugin install confluence-sync@confluence-obsidian
```

The skill is **manual-only** (`disable-model-invocation: true`) because it writes to live Confluence — invoke it explicitly:

> `/confluence-sync` — then ask to sync, pull, publish, check whether the local mirror is up to date, or see what changed.

## Requirements

- **Claude Code** with the **Atlassian MCP server** connected and authenticated against your Confluence site.
- A local Markdown mirror under `confluence/<SPACE>/` whose files carry the frontmatter contract above. Files predating the skill that lack `body_sha` are treated as uninitialised: the first sync pulls remote as authoritative, then writes the watermark.

## Status & roadmap

Today the skill syncs Confluence ↔ a local Markdown tree. Because that tree is just plain Markdown, it already drops into an **Obsidian** vault — the `confluence-obsidian` name points at where this is headed: making an Obsidian vault a first-class, comfortable home for Confluence content (link rewriting, attachments/images, and vault-friendly frontmatter are the natural next steps).

## License

Personal project — no license granted yet.
