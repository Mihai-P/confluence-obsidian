---
name: confluence-sync
description: Two-way sync between an Obsidian-native Markdown vault under confluence/<SPACE>/ and an Atlassian Confluence space, using the Atlassian MCP server's server-side Markdown conversion. Invoke with /confluence-sync (manual-only — it writes to live Confluence) to publish local edits, pull remote changes, check whether the local mirror is up to date, or see what changed. Reads/writes pages with contentFormat "markdown"; tracks each page with version + body_sha in YAML frontmatter and decides per-page whether to skip, push, pull, or surface a conflict. Stores pages as sibling folder notes (Title.md beside a Title/ folder of children); rewrites links to Obsidian relative form on pull and to absolute Confluence page URLs on push.
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Glob, Bash(awk *), Bash(sha256sum *), Bash(diff *), Bash(mkdir *), Bash(mv *), mcp__claude_ai_Atlassian__getConfluencePage, mcp__claude_ai_Atlassian__updateConfluencePage, mcp__claude_ai_Atlassian__createConfluencePage, mcp__claude_ai_Atlassian__getPagesInConfluenceSpace, mcp__claude_ai_Atlassian__getConfluencePageDescendants, mcp__claude_ai_Atlassian__searchConfluenceUsingCql
---

# confluence-sync

Two-way sync between an **Obsidian-native Markdown vault** under `confluence/<SPACE>/` and an Atlassian
Confluence space. Every read and write goes through the Atlassian MCP server using its **server-side
Markdown conversion** (`contentFormat: "markdown"`), plus `Read` / `Edit` / `Write` / `Glob` on the local
files. No client-side format converter, no snapshot file, no batch importer.

The full design rationale lives in `DESIGN.md` at the repo root. This file is the operational contract.

## Required reading

- `DESIGN.md` (repo root) — the why: layout, link model, sync matrix, fidelity notes.
- The frontmatter at the top of any existing `.md` file under `confluence/<SPACE>/` — same shape applies to
  every page.

## Reference files

Detailed procedures load on demand:

- `references/new-pages.md` — when the matrix produces a `CREATE_REMOTE` (local-only) or `CREATE_LOCAL`
  (remote-only) page.
- `references/conflict-resolution.md` — when a page lands in the `CONFLICT` bucket.

## Constants (this site)

- cloudId: `xstep.atlassian.net` (the host works directly as cloudId).
- Space `SD` → `space_id: 262148`, home page id `262260`. Other spaces: read from frontmatter.

## On-disk layout — sibling folder notes

A page with **both content and children** is a `.md` file beside a same-named folder of its children:

```
confluence/SD/
├─ 02. Tech Stack Boilerplate.md       # the page's own content
├─ 02. Tech Stack Boilerplate/         # its children, in a sibling folder
│  └─ 02.5 Generated-site conversion spec.md
├─ 01. Pipeline Architecture.md        # a leaf (no children)
└─ attachments/<attachment_id>.<ext>
```

A leaf is just `Title.md`. When it gains its first child, create the sibling `Title/` folder and write the
child inside — `Title.md` does not move. Filename = page title, filesystem-illegal characters escaped
(notably `/`,`\` → `-`); when the filename differs from the true title, add the true title as an `aliases`
entry. On a title collision in the same folder, suffix `--<id>`.

## State stored in frontmatter

```yaml
---
title: "01. Pipeline Architecture"
confluence:
  id: "917505"                       # canonical id; null only for not-yet-pushed local pages
  url: "https://xstep.atlassian.net/wiki/spaces/SD/pages/917505"
  space_key: "SD"
  space_id: "262148"
  parent_id: "262260" | null         # null only for the space root
  version: 12                        # remote version we last synced TO
  last_synced: "2026-06-04T10:22:00Z"
  body_sha: "<sha256 of the local markdown body>"
---
```

`version` + `body_sha` describe the last known agreement: `version` is the authoritative remote-change
signal, `body_sha` detects local edits. A file with no `body_sha` is **uninitialised** — on first sync take
remote as authoritative (PULL), then write `body_sha`.

## Body rules — disk vs. Confluence

A leading `# H1` is **not guaranteed** — some pages return markdown starting with `# Title`, others start
with a paragraph. So:

- **On disk:** store the body exactly as returned. Keep a leading `# Title` if present; don't synthesise one
  if absent (the filename is the title).
- **On push:** strip the frontmatter, and strip the leading `# H1` **only if** the first content line is an
  H1 whose text equals `frontmatter.title` (else Confluence would show the title twice). Otherwise send the
  body as-is.
- **On pull:** write the returned body as-is, then add/update frontmatter.

`body_sha` always hashes the **local** body — everything after the frontmatter, whatever H1 it has — so
local-change detection is independent of push-time trimming.

## Computing the local body sha

`Read` the file, split off the body (everything after the second `---`), hash with sha256. In Bash:

```bash
awk 'fence==2 {print} /^---$/ && fence<2 {fence++}' <file> | sha256sum | awk '{print $1}'
```

Use that exact incantation so a sha taken right after a `Write` matches a sha taken before the next sync.

## Format — Markdown via the MCP

Use `contentFormat: "markdown"` for **both** `getConfluencePage` and `updateConfluencePage`. The MCP server
does the Confluence↔Markdown conversion. `pageId` is a **string** — pass it quoted.

**`version` does NOT come from the page read.** `getConfluencePage` (markdown) returns the body but not the
version number. Get versions from **`getPagesInConfluenceSpace`**, which returns `{ id, title, parentId,
version.number }` per page in one paginated call — that listing is both the remote-change signal and the
remote half of the id↔path map. Fetch page **bodies** only for pages that need a pull or a content compare.

## Links — rewrite both directions

On disk, internal links are **Obsidian wikilinks** `[[Target Title]]` (or `[[Target Title|display text]]`
when the text differs). Build an `id → { path, title }` map by scanning every file's frontmatter once per
run. The MCP emits internal links in **two forms** — handle both:

1. **Absolute by id** — `https://<site>/wiki/spaces/<KEY>/pages/<id>` (the common case; extract `<id>`).
2. **Relative path** — `./<ancestor>/<target>` with `+`-encoded spaces, no extension, optional `#anchor`.

- **On pull:** resolve each internal link's target id, then look it up in the map:
  - **in the local mirror** → emit `[[Title]]` (or `[[Title|original text]]`; anchors → `[[Title#Heading]]`).
  - **not in the mirror** (a page we didn't pull) → **keep the absolute Confluence URL**.
  - external `http(s)` and bare `#anchor` links pass through unchanged.
- **On push:** rewrite each `[[Target]]` whose target file has a `confluence.id` to the target's absolute
  Confluence URL (`confluence.url`). If the target has no id yet, **drop the wikilink, keep the text**. Never
  send a wikilink or relative `.md` path to Confluence.

For new local pages that link to each other, push in **two passes**: (1) create all new pages to allocate
ids, writing each id back to frontmatter; (2) re-resolve and rewrite cross-links.

## Decision matrix

Get the remote `version` per page from `getPagesInConfluenceSpace` (not from the page read). Then for each
local file with a `confluence.id`:

- `local_changed  = sha256(local_body) != frontmatter.body_sha`
- `remote_changed = listing[id].version != frontmatter.version`

| local_changed | remote_changed | Action | What to do |
| --- | --- | --- | --- |
| no  | no  | **SKIP** | Nothing to do. |
| yes | no  | **PUSH** | Run the **push guard** (below). Then strip frontmatter + H1, rewrite links → absolute, `updateConfluencePage`. On success write back `version` (from response), `last_synced`, `body_sha = sha256(local_body)`. |
| no  | yes | **PULL** | Overwrite local body with remote markdown (rewrite links → `[[wikilinks]]`). Update `version`, `last_synced`, `body_sha`. |
| yes | yes | **CONFLICT** | Stop. Show the diff. User picks a side; then PUSH or PULL. Don't auto-merge. |

Also: `id == null` → **CREATE_REMOTE**; remote page whose id is in no local file → **CREATE_LOCAL** (see
`references/new-pages.md`). Missing `body_sha` → first sync is PULL-authoritative, then write `body_sha`.

`remote_changed` is version-based, so the Markdown round-trip never fabricates a remote change; after a
PUSH we store the new `version`, so the next run sees `remote_changed = no`.

## Fidelity & the push guard

Markdown via the MCP is high-fidelity for ordinary content — headings, bold/italic/code, links, nested
lists, **tables, code blocks (with language), task lists, emoji, blockquotes, Mermaid** (a ` ```mermaid `
fence round-trips exactly), and **inline status / @mentions / smart links / dates** (these arrive as
`<custom data-type="…" data-id="id-N">…</custom>` placeholder tags and round-trip exactly **if left
verbatim** — don't mangle them).

But a markdown pull **silently flattens these Confluence-only constructs to plain paragraphs**, and pushing
the flattened text back makes the loss permanent: **panels** (info/note/warning/success/error),
**expand/collapse**, **multi-column layouts**, **decision lists**.

**Attachments / images are text-only in v1.** The MCP has no attachment up/download tools, so binaries
aren't mirrored. Image refs come back as `![](blob:…&url=<encoded>)` — on pull, best-effort decode the
`url=` param back to `![](<url>)` for external images and leave Confluence-hosted media as its canonical URL
(won't render offline). On push, a newly added local image embed can't be uploaded → warn and skip it.

**Push guard.** Before any PUSH (including the PUSH side of a resolved CONFLICT), fetch the page once with
`contentFormat: "html"` and scan for these markers:

- `data-type="panel-` · `<details` · `data-type="layout-` · `data-type="decision-`

If any are present, pushing would flatten them. **Stop, warn the user which constructs would be lost, and
require explicit confirmation for that page** (or skip it). Pages without these markers push freely. This is
one extra fetch only for pages that changed locally.

## Workflow — full sync (default)

Input: a target directory, default `confluence/SD/`.

1. **Walk locally.** `Glob` `confluence/<SPACE>/**/*.md`. `Read` each; parse the `confluence:` block; capture
   `id`, `version`, `body_sha`; compute the current body sha. Build the id↔path map.
2. **List remotes.** `getPagesInConfluenceSpace(spaceId)` (paginated) → `{ id, title, parentId, version }`
   per page. This gives `remote.version` for every page and the remote half of the id↔path map. Fetch a page
   **body** with `getConfluencePage(pageId, contentFormat=markdown)` only when a page needs a PULL or a
   content compare (CONFLICT) — not for SKIP/PUSH-only decisions.
3. **Classify** into PUSH / PULL / CONFLICT / SKIP / CREATE_LOCAL / CREATE_REMOTE. Don't mutate yet.
4. **Report the plan** before mutating anything:

   ```text
   confluence-sync plan:
     SKIP:  N pages
     PUSH:  M pages
       - confluence/SD/01. Pipeline Architecture.md  (local v12 → remote v12)
     PULL:  K pages
     CONFLICT: J pages (will stop on first)
   ```

5. **Apply, halting on the first error:** PULL first (can't lose data — remote is source of truth), then
   PUSH, then CONFLICT (see reference). Re-fetch + re-check `version` immediately before each PUSH; on a 409
   reclassify as CONFLICT and re-pull.
6. **After each PUSH**, write back the new `version`/`last_synced` from the response and
   `body_sha = sha256(local_body)`.
7. **After each PULL**, write the file (frontmatter + remote body, links rewritten) and
   `body_sha = sha256(new local_body)`.
8. **Report** the output contract block below.

## Workflow — pull-only

Full sync but skip PUSH and treat CONFLICT as PULL (remote wins). Use after out-of-band Confluence edits.
Don't make it the default — it silently overwrites local edits.

## Workflow — push-only

Full sync but skip PULL and treat CONFLICT as PUSH (local wins). Use when local edits must go up and you
accept overwriting concurrent Confluence edits. Same caveat.

## New pages

`CREATE_REMOTE` (a local file with `id: null`) and `CREATE_LOCAL` (a remote page with no local mirror) both
need parent resolution from the sibling layout and stable filenames. Read **`references/new-pages.md`**.

## Conflict resolution

When a page lands in `CONFLICT` (both sides moved), follow **`references/conflict-resolution.md`**: print the
watermarks, show a `diff`, let the user pick a side. Never auto-merge.

## MCP error handling

On **any** error from an Atlassian MCP call (401/403, timeout, connection, "session expired", unexpected
page-not-found): **stop immediately, do not auto-retry, report it, and ask the user to re-authenticate the
Atlassian MCP connection.** Wait for confirmation before resuming. Expired sessions are the usual cause.

## Don't

- Don't update local frontmatter `version` until the corresponding MCP write has succeeded.
- Don't trust `local.version` alone — always fetch the live page, even if it feels redundant.
- Don't send frontmatter, a leading `# H1`, or relative `.md` links to Confluence.
- Don't PUSH a page that contains panels/expand/layout/decision constructs without running the push guard
  and getting confirmation — markdown flattens them permanently.
- Don't rewrite or "tidy" `<custom data-type=… data-id=…>` placeholder tags — they round-trip only verbatim.
- Don't create remote pages whose `parent_id` you can't resolve from the local tree — stop and ask.
- Don't auto-retry through an MCP error — re-authenticate first.

## Output contract

End every run with this block, exactly:

```text
CONFLUENCE_SYNC_RESULT:
  scanned:         <N total local files>
  skipped:         <N>
  pushed:          <N>    [paths...]
  pulled:          <N>    [paths...]
  created_remote:  <N>    [paths...]
  created_local:   <N>    [paths...]
  conflicts:       <N>    [paths..., or 'none']
  errors:          <N>    [paths + reason, or 'none']
  status:          SUCCESS | CONFLICT | ERROR
```

`status: CONFLICT` if anything stayed unresolved in the conflict bucket. `status: ERROR` if any MCP call or
write failed. Otherwise `SUCCESS`.
