---
name: confluence-sync
description: Two-way sync between local markdown mirror under confluence/<SPACE>/ and the Atlassian Confluence space, using the Atlassian MCP server directly. Walks the local tree, reads version + body_sha from each file's YAML frontmatter, fetches the live remote page via getConfluencePage, and decides per-page whether to skip, push, pull, or surface a conflict. Pushes via updateConfluencePage; creates new pages via createConfluencePage with the parent inferred from directory layout. Triggers on phrases like "sync confluence", "pull confluence", "push the spec", "publish to confluence", "is the local mirror up to date", "what changed in confluence".
---

# confluence-sync

Two-way sync between `confluence/<SPACE>/` and the Atlassian Confluence space, driven entirely by Atlassian MCP calls plus `Read` / `Edit` / `Write` on the local files. No helper scripts, no snapshot file, no batch importer — every operation hits the live Atlassian API through the MCP server.

## Required reading

- `confluence/README.md` — the round-trip contract: which frontmatter fields exist, what `confluence.id` is for.
- The frontmatter at the top of any existing `.md` file under `confluence/SD/` — same shape applies to every page. This skill adds **one new field, `body_sha`**, on top of that shape.

## State stored in frontmatter

Every page's frontmatter carries the sync watermark:

```yaml
---
title: "..."
confluence:
  id: "262260"               # canonical id; null only for not-yet-pushed local pages
  url: "https://xstep.atlassian.net/wiki/spaces/SD/pages/262260"
  space_key: "SD"
  space_id: "262148"
  parent_id: "..." | null    # null only for the space root
  version: 5                 # remote version we last synced TO
  last_modified: "..."       # remote last-modified at last sync
  body_sha: "<sha256>"       # sha256 of the body bytes that matched `version`
---
```

`version` + `body_sha` together describe the last known agreement between local and remote. They are how we tell who edited what since.

Files predating this skill won't have `body_sha`. Treat a missing `body_sha` as "uninitialised" — on first sync, take the remote body as authoritative for that page (PULL), then write `body_sha` so subsequent runs can detect local edits.

## Decision matrix

For each local `.md` file with a `confluence.id`, fetch the remote via `getConfluencePage(pageId, body-format=storage)` and compute three booleans:

- `local_changed = sha256(local.body) !== local.frontmatter.body_sha`
- `remote_changed = remote.version !== local.frontmatter.version`
- `bodies_match = local.body === remote.body`

Action:

| local_changed | remote_changed | Action | What to do |
| --- | --- | --- | --- |
| no | no | **SKIP** | Nothing to do. |
| yes | no | **PUSH** | `updateConfluencePage`; on success rewrite frontmatter `version`, `last_modified`, `body_sha`. |
| no | yes | **PULL** | Overwrite local body with `remote.body`; rewrite frontmatter `version`, `last_modified`, `body_sha`. |
| yes | yes | **CONFLICT** | Stop. Show the diff. Let the user pick a side; then PUSH or PULL accordingly. Don't auto-merge. |

Also possible:

- `local.id === null` → **CREATE_REMOTE** (see "New pages" below).
- Remote page exists in space but no local file holds its id → **CREATE_LOCAL** (see "New pages").

If `body_sha` is missing on the local file, set `local_changed = false` for the first sync only and apply a PULL; that initialises tracking.

## Computing the local body sha

Read the file with `Read`, then split off the body (everything after the second `---` line). Hash with sha256.

In Bash, the inline pipeline is:

```bash
awk 'fence==2 {print} /^---$/ && fence<2 {fence++}' <file> | sha256sum | awk '{print $1}'
```

Use that exact incantation when you need a sha and don't have one in hand — the trailing newline behaviour of `awk` matches what `Edit`/`Write` produce, so a sha taken right after writing the file will match a sha taken before the next sync.

## Body format

Use `body-format: storage` for both `getConfluencePage` and `updateConfluencePage`. Reasons:

- `storage` is the canonical Confluence XHTML — it round-trips losslessly.
- `view` is rendered HTML — not pushable.
- `atlas_doc_format` is JSON ADF — also round-trippable but more verbose; we don't need it.

Pick one and stick with it. If a page was pulled with `storage`, push it back with `storage`. Existing files under `confluence/SD/` already hold storage XHTML in the body section, so `storage` is the only safe choice for round-tripping current files.

If you discover a page in a different format (rare), pull it fresh first with `storage` before doing any other operation on it.

## Workflow — full sync (default)

Inputs: a target directory, default `confluence/SD/`.

1. **Walk locally.** Use `Glob` for `confluence/<SPACE>/**/*.md`. For each file: `Read` it, parse the `confluence:` block from the frontmatter, capture `id`, `version`, `body_sha`. Compute the current body sha with the awk pipeline above.
2. **For each file with an id:** call `getConfluencePage` with `body-format=storage`. Capture `remote.version`, `remote.body`, `remote.lastModified`, `remote.title`.
3. **Apply the decision matrix.** Group results into PUSH / PULL / CONFLICT / SKIP / CREATE_LOCAL / CREATE_REMOTE buckets. Don't apply anything yet.
4. **Report the plan** to the user before mutating anything:

   ```text
   confluence-sync plan:
     SKIP:  N pages
     PUSH:  M pages
       - confluence/SD/.../foo.md  (local v5 → remote v6)
       - ...
     PULL:  K pages
       - ...
     CONFLICT: J pages (will stop on first)
       - confluence/SD/.../bar.md  (local v5, remote v7, both edited)
   ```

5. **Apply, in this order, halting on the first error:**
   - **PULL** first (cheap, can't lose data — the local body is replaced but the source of truth is remote).
   - **PUSH** second.
   - **CONFLICT** last — for each, show a diff (`diff` via Bash on two temp files, or render inline) and ask the user to keep local / keep remote / open in editor.
6. **After each PUSH**, take the new `version` and `last_modified` returned by `updateConfluencePage` and write them back into the file's frontmatter together with the new `body_sha = sha256(body just pushed)`.
7. **After each PULL**, write the new file (frontmatter + remote body) and set `body_sha = sha256(remote.body)`.
8. **Report a final tally** in the output contract format below.

## Workflow — pull-only

Same as full sync, but in step 5, skip PUSH and treat CONFLICT as PULL (remote wins). Use this when you've made out-of-band edits in Confluence and want to pull everything down without examining each conflict.

Don't make this the default — silent overwrite of local edits is a data-loss risk.

## Workflow — push-only

Same as full sync, but skip PULL and treat CONFLICT as PUSH (local wins). Use this when you've made local edits and need them up; you accept overwriting any concurrent Confluence edits.

Same caveat: don't make it the default.

## New pages

### Local-only (not yet in Confluence)

A new local file should have `confluence.id: null` and a `parent_id: null` placeholder; everything else inherited from the parent's frontmatter (`space_key`, `space_id`).

To push:

1. **Resolve the parent id** from the directory layout:
   - Leaf file `confluence/SD/<dir>/<slug>.md` → parent is `confluence/SD/<dir>/index.md`.
   - Branch index `confluence/SD/<dir>/index.md` → parent is `confluence/SD/<parent-dir>/index.md`, walking up.
   - The space root `confluence/SD/index.md` itself can't have a new sibling above it — refuse.

   Read the resolved parent `index.md`, take `frontmatter.confluence.id` as `parent_id`.

2. **Call `createConfluencePage`** with `spaceId` (from the parent), `parentId`, `title` (from the file's frontmatter), `body` (the file's body), and `body-format=storage`.

3. **Write the response back** into the local file's frontmatter: `id`, `url`, `parent_id`, `version`, `last_modified`, plus `body_sha = sha256(body that was sent)`. Preserve `title`, `space_key`, `space_id`.

### Remote-only (created in Confluence with no local mirror)

After the per-id loop, list pages in the space with `getPagesInConfluenceSpace(spaceKey)` and look for ids not present in any local file's frontmatter. For each:

1. Determine the parent — `parentId` on the response. Find the local file with `confluence.id === parentId`. That file's path is the parent location; the new page goes alongside it.
2. Compute the slug: lowercase the title, collapse non-alphanumeric runs to `-`, trim leading/trailing `-`. On collision in the same parent directory, suffix `--<id>`. Keep these rules stable so existing local slugs continue to match their remote pages.
3. **If the new page has children** (check via `getConfluencePageDescendants`): create a directory `<slug>/` and write its `index.md`. Then recurse for descendants.
   **If it's a leaf**: write `<slug>.md`.
4. Frontmatter is filled from the remote response; `body_sha = sha256(remote.body)`.

## Conflict resolution

When you hit a CONFLICT (local and remote both moved), do this:

1. Print:

   ```text
   CONFLICT  <path>
     local  v<N>  body_sha=<hex8>
     remote v<M>  body_sha=<hex8>
   ```

2. Show the diff. Easiest: write `local.body` and `remote.body` to two temp files under `/tmp/confluence-sync/` and run `diff -u` on them, or render the diff inline if it's short.
3. Ask the user to pick: **keep local** (push), **keep remote** (pull), **edit** (open the local file for the user to merge by hand, then re-run sync).
4. Whichever side wins, after applying it the frontmatter must end up reflecting the final remote state — `version`, `last_modified`, `body_sha`.

Never auto-merge. Never silently choose a side.

## Don't

- Don't update the local frontmatter `version` until the corresponding MCP write has succeeded — otherwise a transient API failure leaves you out of sync.
- Don't trust `local.version` alone to decide actions — always fetch `getConfluencePage` for the current remote version, even if it feels redundant. The whole point of sync is that remote may have moved since last run.
- Don't push a page whose body is in a format other than `storage` — pull it fresh with `storage` first.
- Don't create remote pages whose `parent_id` you can't resolve from the local tree. If you can't find a parent, stop and ask.

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

`status: CONFLICT` if anything went into the conflict bucket and wasn't resolved. `status: ERROR` if any MCP call or write failed. Otherwise `status: SUCCESS`.
