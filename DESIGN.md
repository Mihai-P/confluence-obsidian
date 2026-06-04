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

### Fidelity (measured: created a page with every tricky construct, pulled as markdown, pushed back, re-read)

**Round-trips with full fidelity** — safe to sync as markdown:

- Headings, bold/italic/inline code, plain links, nested lists, tables, code blocks (with language), task
  lists (`- [ ]`/`- [x]`), emoji, blockquotes.
- **Mermaid** — a ` ```mermaid ` fenced block round-trips exactly and renders natively in Obsidian.
- **Inline ADF nodes — status (with colour), @mentions (with account-id), smart links, dates.** The MCP
  encodes these in markdown as placeholder tags, e.g. `<custom data-type="status" data-id="id-0">Done</custom>`,
  and reconstructs them exactly on push. **Caveat:** they render as raw `<custom>` tags in Obsidian and must
  be preserved verbatim — editing or deleting a tag loses that node.

**Silently flattened to plain paragraphs on the markdown read — and the loss is permanent once pushed back:**

- ❌ Panels (info/note/warning/success/error) → paragraph
- ❌ Expand/collapse → title lost, body becomes a paragraph
- ❌ Multi-column layouts → columns become consecutive paragraphs
- ❌ Decision lists → paragraph

So markdown sync is high-fidelity for ordinary doc content but **destructive to panels / expands / layouts /
decision lists** on a pull→push cycle. The skill guards against this (see §6 "Push guard"). The fully
round-trip-safe alternative is `contentFormat: "html"`, but HTML on disk isn't Obsidian-native, so markdown
stays the default and we accept the documented losses with a guard rather than silent destruction.

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

**Attachments — text-only in v1 (hard MCP constraint).** The Atlassian MCP server exposes **no attachment
upload/download tools** (confirmed: only page/space/search tools exist), so binaries can't be mirrored
through it. Worse, the markdown read mangles image refs: an external `![alt](url)` comes back as
`![](blob:https://media…atl-paas.net/?…&url=<percent-encoded-original>)` — alt text dropped, real URL buried
in a `url=` param; Confluence-hosted images become opaque media blobs with no fetchable path. So v1:
- **Does not** download/upload attachment binaries or maintain a local `attachments/` folder.
- **On pull**, best-effort normalises image refs to something that at least resolves online — decode the
  `url=` param back to `![](<original-url>)` for external images; for Confluence-hosted media leave the
  canonical Confluence URL. These won't render offline in Obsidian (documented limitation).
- **On push**, a newly added local image embed can't be uploaded → warn and skip it (like the panel guard).
Revisit if/when the MCP gains attachment endpoints, or via a separate REST/browser path (see §10).

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

## 5. Links — Obsidian wikilinks on disk, page links on Confluence

On disk, internal page links are **Obsidian wikilinks** (`[[Target Title]]`, or `[[Target Title|display]]`
when the link text differs); the Confluence page-link form exists only transiently during a push. Wikilinks
were chosen over relative `.md` links after the e2e pull: page titles routinely contain spaces and dots
(`04.2 Generated-site lifecycle`), which make relative-path links need ugly `%20`/`<…>` escaping, whereas
wikilinks resolve cleanly by filename. (Trade-off: wikilinks resolve by **filename**, so filenames must be
unique vault-wide; the `--<id>` collision suffix and an `aliases` entry handle that.)

**The id↔path map.** Each run scans the tree and builds `page_id → { path, title }` from frontmatter. This
is the source of truth for rewriting both directions and survives renames (identity = id).

**The MCP emits internal links in two forms** (observed live) — the rewriter must handle both:

1. **Absolute by id:** `https://<site>/wiki/spaces/<KEY>/pages/<id>` — the common case; extract `<id>`.
2. **Relative path:** `./<ancestor>/<target>` with `+`-encoded spaces, no extension, optional `#anchor`.

- **On pull:** for each internal link, resolve its target page id (case 1: read it directly; case 2:
  resolve the relative path against the tree). Look the id up in the id↔path map:
  - **target is in the local mirror** → emit `[[Target Title]]` (or `[[Target Title|original text]]` if the
    link text differs from the title). Anchors become `[[Target Title#Heading]]`.
  - **target is NOT in the mirror** (a page we didn't pull, e.g. another subtree) → **keep the absolute
    Confluence URL** as-is.
  - external `http(s)` links and bare `#anchors` pass through unchanged.
- **On push:** rewrite each `[[Target]]` whose target file has a `confluence.id` to the target's absolute
  Confluence URL (`confluence.url` from its frontmatter); Confluence renders a same-site page URL as a
  proper page link. If the target has no id yet (unpublished), **drop the wikilink, keep the text** — then
  publish that page and re-push. Never send a wikilink or a relative `.md` path to Confluence.

**Two-pass push for new pages.** If a link targets a not-yet-published note, push runs in two passes:
(1) create all new pages to allocate ids, writing each id back to its frontmatter; (2) re-resolve and
rewrite cross-links now that every target has a `confluence.url`.

---

## 6. Sync model — `version` + `body_sha` decision matrix

Because we read and write the **same format** (Markdown) both ways, a hash of the body is a reliable
local-change signal — so we keep the original skill's matrix rather than a shadow-base merge engine.

**Where `version` comes from (observed live):** `getConfluencePage` with `contentFormat: "markdown"`
returns the body but **not** the version number. So source versions from **`getPagesInConfluenceSpace`**,
which returns `{ id, title, parentId, version.number }` per page in one paginated call. That bulk listing is
both the remote-change signal *and* the remote half of the id↔path map — fetch page **bodies** with
`getConfluencePage` only for pages that actually need a pull or a content compare.

Then for each local file with a `confluence.id`:

- `local_changed  = sha256(local_body) != frontmatter.body_sha`
- `remote_changed = listing[id].version != frontmatter.version`

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

### Push guard — don't silently flatten Confluence-only constructs

A markdown pull already drops panels, expands, multi-column layouts, and decision lists (§1); pushing the
flattened markdown back makes that loss permanent. So before any **PUSH** (or the PUSH side of a resolved
CONFLICT), fetch the page once as `contentFormat: "html"` and scan for the markers:

- `data-type="panel-` (info/note/warning/success/error panels)
- `<details` (expand/collapse)
- `data-type="layout-` (multi-column layouts)
- `data-type="decision-` (decision lists)

If any are present, the push would flatten them. **Warn the user and require explicit confirmation per page**
(or skip it). This costs one extra fetch only for pages that actually changed locally. Pages without these
markers push freely. Preserve any inline `<custom data-type=… data-id=…>` placeholders verbatim — they
round-trip correctly only if untouched.

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

- ~~Spot-check lossy constructs~~ **Done** (§1): panels/expand/layouts/decision lists flatten; inline
  status/mention/smartlink/date survive via `<custom>` placeholders; **Mermaid** survives. Guard added (§6).
- ~~Attachments~~ **Resolved as a constraint** (§2): no MCP attachment tools → text-only v1. A future
  high-fidelity path would need the Confluence REST attachment API directly (outside this MCP) or a
  Playwright-driven download using the logged-in session — both are separate efforts, not v1.
- **Conflict diff UX** — *decided*: default to a simple **local-vs-remote** two-way `diff` (both sides are
  Markdown, so it's readable) and let the user pick a side. The optional `.sync/` base enables a nicer
  3-way `git merge-file` later, but is not required for v1. CONFLICT never auto-merges.
- **e2e validated**: pulled `04. Delivery Workflow` + a child into the sibling layout with correct
  frontmatter, `body_sha`, and wikilink rewriting (in-mirror → `[[…]]`, out-of-mirror id kept absolute).
- **Inline `<custom>` placeholders in Obsidian**: leave them raw (safe, slightly ugly) for v1; prettify-on-
  disk + restore-on-push is a possible later nicety.
