# confluence-obsidian — design

How the two-way sync works: an **Obsidian-native Markdown vault** on disk, a **Confluence space** on the
remote, and a sync engine that keeps them in agreement without losing data in either direction.

This is the contract the `confluence-sync` skill implements. It supersedes the original "store Confluence
`storage` XHTML in the file body" approach, which round-tripped losslessly but was unreadable and unusable
in Obsidian — defeating the project's whole purpose.

> Status: agreed design, June 2026. Validated against a working reference (`docs-as-code`, Roo agents) and a
> live fidelity test on the real SD space.

---

## 1. Key decision: Markdown via the Atlassian MCP

The Atlassian MCP server converts Confluence content **to and from Markdown server-side**
(`getConfluencePage` / `updateConfluencePage` with `contentFormat: "markdown"`). We use that directly. This
is how the working `docs-as-code` setup operates, and a live test on a complex SD page confirmed the
fidelity is high:

- Tables (including large/complex ones), fenced code blocks, blockquotes, nested lists, bold/italic, and
  emoji all round-trip cleanly.
- Internal page links come back as **hierarchy-relative paths** (`./<ancestor title>/<target title>`,
  spaces as `+`, no `.md`, with `#anchors`); external links stay absolute.

**Consequences — this deletes most of the original complexity:**

- No ADF, no client-side converter (`marklas`/`md2cf`), no `storage` XHTML on disk.
- No shadow-base three-way merge engine. We reuse the original skill's `version` + `body_sha` matrix
  (§6), because reading and writing the **same format** (Markdown) both directions makes a content hash a
  reliable local-change signal again.

**Residual fidelity risk:** the test page used Markdown-native constructs and manual emoji, not Confluence
**panels / expand / status / mentions**. Those should be spot-checked on a panel-heavy page before relying
on them; whatever the MCP's markdown does with them is what we get (we are not preserving them by hand).

---

## 2. On-disk layout — sibling folder notes

The vault mirrors the Confluence space tree. A page that has **both content and children** is stored as a
**sibling folder note**: a `.md` file holding the page's own content, **beside** a folder of the same name
holding its children.

```
confluence/<SPACE>/
├─ 01. Pipeline Architecture.md        # a leaf page (no children)
├─ 02. Tech Stack Boilerplate.md       # a page WITH children: its own content…
├─ 02. Tech Stack Boilerplate/         # …and its children in a sibling folder
│  ├─ 02.5 Generated-site conversion spec.md
│  └─ …
├─ attachments/
│  └─ <attachment_id>.<ext>            # images/files, named by Confluence attachment id
└─ .sync/                              # optional local cache (gitignored) — see §6
```

**Why sibling, not `name/name.md` or `name/index.md`:**

- The MCP's own Markdown links are **hierarchy-relative and assume this exact layout**. A link from
  `01. Pipeline Architecture.md` to a child of `02. Tech Stack Boilerplate` is emitted as
  `./02.+Tech+Stack+Boilerplate/02.5+…` — i.e. it expects `02.5` to live *inside a folder beside `01`*.
  Sibling makes the files line up with the links, so link rewriting is cosmetic (encoding only), not path
  recomputation. `name/name.md` would put every file one level deeper than its links expect.
- `index.md` / `README.md` parents are rejected: Obsidian resolves links by **filename**, so a vault full
  of identical `index`/`README` notes makes links ambiguous and the graph view noise.
- Matches the proven `docs-as-code` vault.

**Cost of sibling:** a page's content file (`Foo.md`) and its children folder (`Foo/`) are two entries that
must be renamed/moved together. Minor, and mechanical (identity is the page id, so it's a move, not a
delete+create).

**Leaf ↔ parent transition.** A leaf is just `Foo.md`. When it gains its first child remotely, create the
sibling `Foo/` folder and write the child inside it — `Foo.md` itself doesn't move. When the last child is
removed, delete the empty `Foo/` folder.

**Filenames.** Filename = page title with filesystem-illegal characters escaped. On a title collision in
the same parent folder, suffix `--<id>`. Keep the rule stable so existing files keep matching their pages.

**Attachments.** Downloaded into a per-space `attachments/` folder, named by Confluence attachment id
(globally unique). Referenced from notes as normal Markdown images `![alt](attachments/<id>.png)` (or
Obsidian embeds `![[<id>.png]]`).

---

## 3. Frontmatter (per note)

Minimal identity + watermark, namespaced under `confluence:` so it doesn't collide with Obsidian keys.
Obsidian tolerates arbitrary frontmatter and hides it from the rendered note.

```yaml
---
title: "01. Pipeline Architecture"
confluence:
  id: "917505"                          # canonical page id — the ONLY durable identity
  url: "https://xstep.atlassian.net/wiki/spaces/SD/pages/917505"
  space_key: "SD"
  space_id: "262148"
  parent_id: "262260"                   # null only for the space root
  version: 12                           # remote version we last synced TO
  last_synced: "2026-06-04T10:22:00Z"
  body_sha: "<sha256 of the local markdown body>"
---
```

`version` + `body_sha` together describe the last known agreement between local and remote — `version` is
the authoritative remote-change signal, `body_sha` detects local edits. (`aliases`, `tags` may be added
later to mirror labels / keep links stable on rename.)

---

## 4. Body rules (disk vs. Confluence)

The on-disk Markdown is the human-friendly, Obsidian-readable form; the body sent to Confluence is trimmed.

- **On disk:** keep the YAML frontmatter and the leading `# H1` (the H1 reads as the note title in
  Obsidian and equals `frontmatter.title`).
- **On push:** strip the frontmatter **and** the leading `# H1` before sending — Confluence renders the
  page title itself, so an H1 in the body would duplicate it. Send `contentFormat: "markdown"`.
- **On pull:** the MCP returns the body with its `# H1`; write it as-is (frontmatter is added/updated by
  the sync).

`body_sha` always hashes the **local** body (frontmatter excluded, H1 included) — the exact bytes stored on
disk — so local-change detection is independent of the push-time trimming.

---

## 5. Links — Obsidian on disk, page links on Confluence

On disk, links are Obsidian-resolvable relative Markdown; the Confluence page-link form exists only
transiently during a push.

**The id↔path map.** Each run scans the tree and builds `page_id → { path, title }` from frontmatter. This
is the source of truth for rewriting both directions and survives renames (identity = id).

- **On pull:** the MCP emits internal links as `./<ancestor>/<target>` with `+`-encoded spaces, no
  extension, and `#anchors`. Rewrite each to an Obsidian-resolvable relative link: decode `+`→space, append
  `.md`, and resolve the target via the id↔path map (path comes out right because the layout is sibling).
  External `http(s)` links and `#anchors` pass through unchanged.
- **On push:** rewrite each relative `.md` link to the **target page's absolute Confluence URL**
  (`confluence.url` from the target file's frontmatter); Confluence renders a same-site page URL as a
  proper page link. If the target has no id yet (unpublished), **drop the link but keep the text** — then
  publish that page and re-push. Never send a relative `.md` link to Confluence; it breaks there.

**Two-pass push for new pages.** If a link targets a not-yet-published note, push runs in two passes:
(1) create all new pages to allocate ids, writing each id back to its frontmatter; (2) re-resolve and
rewrite cross-links now that every target has a `confluence.url`.

---

## 6. Sync model — `version` + `body_sha` decision matrix

Because we read and write the **same format** (Markdown) both ways, a hash of the body is a reliable
local-change signal — so we keep the original skill's matrix rather than a shadow-base merge engine.

For each local file with a `confluence.id`, fetch the remote with `contentFormat: "markdown"` and compute:

- `local_changed  = sha256(local_body) != frontmatter.body_sha`
- `remote_changed = remote.version != frontmatter.version`

| local_changed | remote_changed | Action | What to do |
| --- | --- | --- | --- |
| no  | no  | **SKIP** | Nothing to do. |
| yes | no  | **PUSH** | Strip frontmatter + H1, rewrite links → absolute, `updateConfluencePage`. On success, write back the new `version` (from the response), `last_synced`, and `body_sha = sha256(local_body)`. |
| no  | yes | **PULL** | Overwrite local body with the remote markdown (rewrite links → relative `.md`). Update `version`, `last_synced`, `body_sha`. |
| yes | yes | **CONFLICT** | Stop. Show a diff. Let the user pick a side; then PUSH or PULL. Never auto-merge. |

Plus:

- `confluence.id == null` → **CREATE_REMOTE** (new local page; see new-pages flow).
- Remote page with no local file holding its id → **CREATE_LOCAL** (pull it down into the right folder).
- Missing `body_sha` (pre-existing/untracked file) → treat the first sync as PULL-authoritative, then write
  `body_sha` to initialise tracking.

**Why no false conflicts:** `remote_changed` is version-based (exact, not content-based), so the
Markdown round-trip (push → Confluence re-renders → next pull may differ in bytes) never fabricates a
remote change. After our own push, we store the new `version`, so the next run sees `remote_changed = no`.

The `.sync/` cache is optional here — used only to show a richer diff on CONFLICT (stash the
last-synced body per id). It is not required for change detection.

---

## 7. New pages, moves, identity

Everything keys on **page id**, never path or title.

- **New local page** (`id: null`): resolve the parent id from the directory layout (the sibling `Foo.md`
  for a file in `Foo/`, walking up; the space root for top-level files), `createConfluencePage` with that
  parent, then write the returned `id`/`url`/`version` back to frontmatter.
- **New remote page** (id not present locally): find the local file whose `id == parentId`; write the new
  page beside it (as a leaf `Title.md`, or promote to a `Title.md` + `Title/` pair if it has descendants).
- **Remote move/rename**: `parent_id` or `title` differs from frontmatter → move/rename the local file (and
  its sibling folder, if any) to match.
- **Local move**: a file's directory changed but `confluence.id` is unchanged → re-parent on Confluence.
  (v2 `updateConfluencePage` can change the title; use the v1 move endpoint for re-parenting if needed.)

---

## 8. Confluence / MCP specifics

- **Format**: `contentFormat: "markdown"` on both `getConfluencePage` and `updateConfluencePage`.
- **cloudId**: the site host works directly — `"xstep.atlassian.net"` (cloud id
  `e059e03c-a728-4f8b-9b40-fbdf8ff31285`). Space `SD` = id `262148`, home page `262260`.
- **Optimistic locking**: an update must advance `version.number`; a stale version → HTTP 409. Re-fetch and
  re-check the version immediately before each write; on 409, reclassify as a conflict and re-pull. Trust
  the version returned in the update **response** when writing the watermark (a quick re-GET can lag).
- **pageId is a string**; pass it quoted.
- **Error handling**: on any MCP error (401/403/timeout/etc.), **stop, do not auto-retry, ask the user to
  re-authenticate** — expired sessions are the common cause (per the docs-as-code rule).
- **Discovering changes cheaply** on large spaces: CQL `space = SD and lastModified > "<watermark>"` to get
  the changed-since set instead of walking every page.

---

## 9. Migration from the current scheme

The existing skill stores `storage` XHTML in the body with `index.md` parents and a `body_sha` over XHTML.
Per space, one time:

1. Pull every page fresh with `contentFormat: "markdown"`; write the sibling folder-note tree (`Title.md`
   + `Title/`), replacing `index.md`.
2. Rewrite the MCP's relative links to Obsidian relative `.md` links (§5).
3. Reset `body_sha` to the sha of the new Markdown body; keep `version` from the remote.
4. Treat any page lacking a tracked `body_sha` as PULL-authoritative on the first run.

---

## 10. Open items

- **Spot-check lossy constructs** (panels, expand, status, mentions, Mermaid/diagram macros) on a
  panel-heavy page to see what the MCP markdown does with them; decide whether any need special handling.
- **Attachments**: download-on-pull / upload-on-push and id stability are sketched (§2) but need their own
  pass.
- **Conflict diff UX**: how much to lean on the optional `.sync/` base for a 3-way-style diff vs. a simple
  local-vs-remote diff.
