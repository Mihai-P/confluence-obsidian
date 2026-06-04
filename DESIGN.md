# confluence-obsidian — design

How the two-way sync works: an **Obsidian-native Markdown vault** on disk, a **Confluence space** on the
remote, and a sync engine that keeps them in agreement without losing data in either direction.

This document is the contract the `confluence-sync` skill implements. It supersedes the original
"store Confluence `storage` XHTML in the file body" approach, which round-tripped losslessly but was
unreadable and unusable in Obsidian — defeating the project's whole purpose.

> Status: agreed design, June 2026. Drives the rewrite of `skills/confluence-sync/`.

---

## 1. Goals & non-goals

**Goals**

- On disk, every page is **Obsidian-native Markdown**: wikilinks, embeds, callouts, folder-note hierarchy.
- **Two-way sync** with safe conflict handling — neither side silently loses edits.
- **No false conflicts**: editing nothing, or editing only on one side, never reports a spurious conflict.
- **No silent data loss**: Confluence constructs with no Markdown equivalent (macros, panels, layouts,
  status, mentions) survive a round-trip.

**Non-goals**

- Real-time sync. This runs on demand (CLI/agent), polling Confluence — not webhooks.
- Rendering parity. We preserve *content and structure*, not pixel-exact Confluence rendering.
- Supporting Confluence Server/Data Center first-class. We target **Confluence Cloud** (ADF). Storage
  format is kept as a fallback path only.

---

## 2. On-disk layout

The vault mirrors the Confluence space tree. A page that has **both content and children** is stored as a
**same-name folder note**: a folder named after the page, containing a `.md` of the same name for the
page's own content, with child pages beside it.

```
confluence/<SPACE>/
├─ <Space Home>.md                 # space root page content (or Home/Home.md if it has children)
├─ Architecture/
│  ├─ Architecture.md              # the "Architecture" page's OWN content (folder note)
│  ├─ API Gateway.md               # a leaf child page
│  └─ Data Model/
│     ├─ Data Model.md             # content for a child that ALSO has children
│     └─ Schemas.md
├─ attachments/
│  └─ <attachment_id>.<ext>        # images/files, named by Confluence attachment id (globally unique)
└─ .sync/                          # sync state cache (gitignored) — see §6
```

**Why same-name folder notes** (`Page/Page.md`) and not `index.md`/`README.md`:

- Obsidian resolves `[[wikilinks]]` by **filename**, vault-wide. A vault full of `README.md`/`index.md`
  files means `[[README]]` is ambiguous and the graph view is noise. Naming the file after the page keeps
  links unambiguous.
- A page and its whole subtree live in **one folder** → a remote move/rename is a single atomic local
  folder move. Good for sync mechanics.
- It is the most common Obsidian folder-note convention and the default of the Folder Notes plugin
  (clicking the folder opens its note), so it works for a human vault user with zero config.

**Leaf ↔ parent promotion.** A leaf page is just `Leaf.md`. When it gains its first child remotely, the
sync **promotes** it: `Leaf.md` → `Leaf/Leaf.md`, then writes the child beside it. The reverse (last child
removed) demotes `Leaf/Leaf.md` → `Leaf.md`. Identity is the page id (frontmatter), so the promotion is a
move, never a delete+create.

**Slugs / filenames.** Filename = page title, with filesystem-illegal characters escaped. On a title
collision within the same parent folder, suffix `--<id>`. Set the original title as an Obsidian `aliases`
entry so inbound `[[Original Title]]` links survive a filename change.

**Attachments.** Downloaded into a per-space `attachments/` folder, named by Confluence attachment id so
names are globally unique (required for `![[id.png]]` embeds, which also resolve by filename). Referenced
from notes as Obsidian embeds `![[ <attachment_id>.png ]]`.

---

## 3. Frontmatter (per note)

Frontmatter carries Obsidian-special keys plus the sync **identity + watermark**. Obsidian tolerates
arbitrary keys and hides them from the rendered note (they appear only in the Properties panel).

```yaml
---
aliases: ["Original Confluence Title"]   # keeps inbound [[links]] stable across renames
tags: [confluence/eng]                    # mirrors Confluence labels (optional)
confluence:
  id: "458211"                            # canonical page id — the ONLY durable identity
  url: "https://<site>.atlassian.net/wiki/spaces/ENG/pages/458211"
  space_key: "ENG"
  space_id: "262148"
  parent_id: "262260"                     # null only for the space root
  version: 7                              # remote version we last synced TO (authoritative remote signal)
  last_modified: "2026-06-01T10:22:00Z"
  format: adf                             # canonical remote format: adf (default) | storage
---
```

The **full last-synced base bodies** do NOT live in frontmatter (they'd bloat every file). They live in
the `.sync/` cache (§6). Frontmatter holds identity + version + a content hash only.

---

## 4. Canonical remote format: ADF

The Confluence REST API has **no Markdown** representation — all Markdown conversion is client-side. Of the
writable remote formats, we use **ADF (`atlas_doc_format`, JSON)** as canonical, with **`storage` (XHTML)**
as a fallback for non-Cloud instances.

Why ADF over storage:

- ADF is the modern editor's native model; storage is a derived serialization.
- ADF is a typed JSON tree — every construct (`panel`, `status`, `expand`, `extension`, `mention`,
  `emoji`, `inlineCard`) is an explicit node, so conversion branches on `node.type` instead of pattern-
  matching XHTML with custom `ac:`/`ri:` namespaces.
- ADF canonicalizes deterministically (sort keys, drop volatile ids); storage XHTML does not (attribute
  order, self-closing style, whitespace, and **volatile `ac:macro-id` UUIDs** all churn the bytes).

ADF bodies are sent on write as a **JSON-encoded string** in `value` (not a raw object).

---

## 5. Conversion (hybrid: deterministic converter + agent)

Per-page LLM hand-conversion of XHTML/ADF ↔ Markdown is non-deterministic, and that non-determinism is
exactly what produces false conflicts. So conversion is done by a **pinned, deterministic converter**; the
agent orchestrates (link rewriting, conflict merges, API calls) but does not hand-convert bodies.

**Converter interface** (whatever tool/script we vendor must expose two deterministic operations):

```
adf-to-md   <adf.json>  ->  <markdown>     # pull
md-to-adf   <markdown>  ->  <adf.json>      # push
```

Reference implementation candidate: **`marklas`** (Python, bidirectional ADF⇄MD, preserves ADF-only
constructs as namespaced HTML). Battle-tested one-way alternatives: `md2conf` / `mark` (push, storage).
The converter is swappable behind the two-operation interface above.

**Lossy-construct rule.** Anything with no CommonMark equivalent is **carried through, never dropped** —
encoded as a namespaced HTML comment or an `adf=`-attributed HTML element that survives Markdown editing.
A dropped macro is destroyed permanently on the next push, so this is non-negotiable.

| Confluence construct | On-disk Markdown representation |
| --- | --- |
| Info / Note / Tip / Warning / Error panel | Obsidian callout `> [!info]` / `[!note]` / `[!tip]` / `[!warning]` / `[!danger]` |
| Expand / collapse | Foldable callout `> [!note]-` (or `<details><summary>`) |
| Status lozenge | `<span adf="status" color="green">Done</span>` |
| Layout / columns | `<div adf="layout">…</div>` fences |
| Macro / extension (TOC, include, Jira, …) | `<!-- confluence:macro name="toc" params="…" -->` placeholder (regenerated on push) |
| User mention | `<span adf="mention" account-id="…">@Name</span>` (id preserved — bare `@name` breaks the mention) |
| Emoji | Unicode passes through; custom emoji keep `shortName` |
| Table with merged/colored cells | HTML `<table>` (GFM tables lose merges/colors) |
| Jira / smart link | `inlineCard` URL → `[KEY](url)`; re-promoted to a card on push if the URL matches |
| Tables / task lists / footnotes / Mermaid | Native Markdown — pass straight through |

---

## 6. Sync model: shadow base + three-way merge

Because conversion is lossy and asymmetric, **"hash the body and compare to remote" fires false
conflicts**. Instead we use the Unison/git **shadow-base** model: keep a copy of the *last-synced state*
and detect change by comparing **same-format to same-format**, never by re-converting inside the detection
path.

**State, per page** (`confluence/<SPACE>/.sync/<page_id>/`, gitignored):

```
.sync/<page_id>/
├─ base.md          # the Markdown as of the last successful sync
├─ base.adf.json    # the ADF (or base.storage.xml) as of the last successful sync — the 3-way base
└─ meta.json        # { version, parent_id, title, md_sha, adf_sha }
```

**Change detection** (no conversion in this path):

- `local_changed  = canon(current_md) != canon(base.md)`     ← pure same-format compare
- `remote_changed = remote.version != frontmatter.version`   ← exact, lossless, conversion-free

`version.number` is the authoritative remote signal: it is exact, monotonic, and free (fetched with the
page). Confluence enforces optimistic locking — a write must send `version + 1` or it 409s — so we also
**re-fetch and re-check the version immediately before every write**.

**Decision matrix:**

| local_changed | remote_changed | Action |
| --- | --- | --- |
| no  | no  | **SKIP** |
| yes | no  | **PUSH**: `md-to-adf(current_md)` → update with `version+1` → refresh `.sync` base from the write response |
| no  | yes | **PULL**: `adf-to-md(remote)` → write file → refresh `.sync` base |
| yes | yes | **3-WAY MERGE** (below) |

**Both changed → three-way merge.** Convert the remote to Markdown, then
`git merge-file(base.md, current_md, remote_md)`:

- **Clean merge** (edits didn't overlap) → accept, push the merged result, advance the base. This silently
  resolves the common "we edited different sections" case — the biggest UX win over the old design, which
  sent *every* both-changed page to manual conflict.
- **Overlapping edits** → stop, show the 3-way diff, offer **keep local / keep remote / edit markers by
  hand**. Never auto-pick a side.
- **False-conflict suppression** (Unison rule): if both changed but `canon(current_md) == canon(remote_md)`,
  the sides are already in agreement — just advance the base, no conflict.

Merge always happens on **Markdown** (line-oriented, merges well), never on ADF/XHTML.

---

## 7. Link rewriting (Obsidian ⇄ Confluence)

On disk links are always Obsidian-native; the Confluence link form exists only transiently during a push.

**The id↔path map.** Each run scans the tree and builds `page_id → { path, title }` from frontmatter. This
is the source of truth for rewriting both directions and survives renames (identity = id, not title).

- **Pull**: a Confluence page reference (`<ac:link><ri:page ri:content-id="123456"/></ac:link>`, or an ADF
  `inlineCard` URL) → look up `123456` → emit `[[Note Title]]` (path-qualified `[[dir/Note|Title]]` on a
  name collision). Attachment refs → `![[<attachment_id>.ext]]`. External links stay `[text](url)`.
- **Push**: parse each `[[wikilink]]`, resolve it the way Obsidian does (filename-first), read the target's
  `confluence.id`, and emit `<ac:link><ri:page ri:content-id="123456"/>…</ac:link>` (use **`content-id`**,
  not `content-title`, so links survive renames).

**Two-pass push for new pages.** If a wikilink targets a note with no id yet, push runs in two passes:
(1) create all new pages to allocate ids, writing each id back to its frontmatter; (2) re-resolve and
rewrite cross-links now that every target has an id.

---

## 8. Moves, renames, identity

Everything keys on **page id**, never path or title.

- **Remote move/rename**: page's `parent_id` or `title` differs from the `.sync` base → move/rename the
  local file (and its folder, for a folder note) to match; update frontmatter.
- **Local move**: a file's directory changed but its `confluence.id` is unchanged → push a re-parent.
  Confluence v2 `PUT` does not reliably re-parent; use the v1 **move endpoint**
  (`/wiki/rest/api/content/{id}/move/{position}/{targetId}`) for re-parenting, separate from the body
  update. Title rename is supported on the v2 update.
- Because identity is the id, a moved file is never mistaken for delete+create.

---

## 9. Confluence API specifics

- **Optimistic locking**: update must send `version.number == current + 1`, else **409**. Re-fetch +
  version-check immediately before each write; on 409, reclassify as a conflict and re-pull.
- **Version lag**: trust the version returned *in the update response* when writing the watermark, not an
  immediate re-GET (replication can briefly lag).
- **Discovering changes cheaply**: instead of walking the whole space every run, query CQL
  `space = <KEY> and lastModified > "<watermark>"` to get the changed-since set, then fetch bodies only for
  those.
- **Rate limits** (points-based, enforced from March 2026): honor `Retry-After` and `X-RateLimit-*`
  headers; exponential backoff with jitter on 429.
- **Body format on the wire**: read/write `atlas_doc_format`; ADF value is a JSON-encoded string. After a
  write, re-read and re-canonicalize before recording the base — Confluence re-normalizes on save.

---

## 10. Migration from the current scheme

The existing skill stores `storage` XHTML in the body with `index.md` parents and a `body_sha` watermark.
Migration, one space at a time:

1. Pull every page fresh as ADF; convert to Obsidian Markdown; write the new same-name folder-note tree.
2. Rename `index.md` → `<Folder Name>.md`; rewrite stored Confluence links to `[[wikilinks]]`.
3. Initialize `.sync/<id>/` bases from the freshly pulled state; drop `body_sha` from frontmatter (now in
   `.sync/meta.json`).
4. Treat the first run as PULL-authoritative for any page that lacks a `.sync` base.

---

## 11. Open items

- **Pick & pin the converter.** `marklas` is the closest bidirectional ADF⇄MD fit; validate fidelity on the
  real SD space (especially panels, tables, mentions, macros) before committing. Fallback: storage-based
  `md2conf`/`mark` for push + a dedicated ADF→MD path for pull.
- **Converter packaging.** Vendored script vs. a pinned pip install — decide how the skill invokes it via
  `Bash` reproducibly.
- **Attachment sync** (upload on push, download on pull, id stability) is sketched here but needs its own
  pass.
