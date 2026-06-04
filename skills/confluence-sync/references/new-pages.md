# New pages

How to handle pages that exist on only one side. Loaded on demand from `SKILL.md` when a sync produces a
`CREATE_REMOTE` or `CREATE_LOCAL` case. All operations use the MCP with `contentFormat: "markdown"`.

## Local-only → CREATE_REMOTE (not yet in Confluence)

A new local file has `confluence.id: null`; `space_key` / `space_id` are inherited from its parent.

1. **Resolve the parent id** from the sibling layout:
   - A file `confluence/SD/<Foo>/<Child>.md` (inside a folder `<Foo>/`) → parent is the sibling
     `confluence/SD/<Foo>.md`.
   - A top-level file `confluence/SD/<Page>.md` → parent is the space home page (`parent_id` of the space
     root; for SD the home is id `262260`).
   - The space root itself can't get a new sibling above it — refuse.

   Read the resolved parent file, take `frontmatter.confluence.id` as `parent_id`.

2. **Prepare the body**: strip the frontmatter and the leading `# H1`; rewrite links (see below).

3. **Call `createConfluencePage`** with `spaceId` (from the parent), `parentId`, `title` (from frontmatter),
   `body` (prepared markdown), and `contentFormat: "markdown"`.

4. **Write the response back** into the local frontmatter: `id`, `url`, `parent_id`, `version`,
   `last_synced`, plus `body_sha = sha256(local_body)` (the on-disk body, H1 included). Preserve `title`,
   `space_key`, `space_id`.

### Two-pass push for cross-linked new pages

If new local pages link to each other, a link's target may not have an id yet. Push in two passes:

1. **Pass 1** — create every new page (steps 1–4) to allocate ids; write each id/url back to frontmatter.
   In this pass, drop any link whose target is still unpublished (keep the link text).
2. **Pass 2** — now that every target has a `confluence.url`, re-render each just-created page with links
   rewritten to absolute Confluence URLs and `updateConfluencePage` it.

## Remote-only → CREATE_LOCAL (in Confluence, no local mirror)

After the per-id loop, list pages with `getPagesInConfluenceSpace(spaceId)` and find ids not present in any
local file's frontmatter. For each:

1. **Find the parent location.** Take `parentId` from the page; find the local file whose
   `confluence.id == parentId`. The new page goes **beside** that parent file:
   - parent at `confluence/SD/<Foo>.md` → child goes in `confluence/SD/<Foo>/` (create the sibling folder if
     it doesn't exist yet — this is the leaf→parent promotion).
   - if the parent is the space home, the child goes at the space root `confluence/SD/`.
2. **Compute the filename:** the page title with filesystem-illegal characters escaped. On collision in the
   same folder, suffix `--<id>`. Keep these rules stable so existing local files keep matching their pages.
3. **Fetch the body** with `getConfluencePage(id, contentFormat=markdown)`; rewrite its links to relative
   `.md` form. Write the file: frontmatter (filled from the page) + the markdown body (H1 included).
4. **If the page has children** (check `getConfluencePageDescendants`): it becomes a sibling folder note —
   write `<Title>.md` and create `<Title>/`, then recurse for descendants. Otherwise write a leaf `<Title>.md`.
5. Set `body_sha = sha256(local_body)` and `version` from the page.
