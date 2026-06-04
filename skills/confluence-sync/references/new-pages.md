# New pages

How to handle pages that exist on only one side. Loaded on demand from `SKILL.md` when a sync surfaces a `CREATE_REMOTE` or `CREATE_LOCAL` case.

## Local-only (not yet in Confluence)

A new local file should have `confluence.id: null` and a `parent_id: null` placeholder; everything else inherited from the parent's frontmatter (`space_key`, `space_id`).

To push:

1. **Resolve the parent id** from the directory layout:
   - Leaf file `confluence/SD/<dir>/<slug>.md` → parent is `confluence/SD/<dir>/index.md`.
   - Branch index `confluence/SD/<dir>/index.md` → parent is `confluence/SD/<parent-dir>/index.md`, walking up.
   - The space root `confluence/SD/index.md` itself can't have a new sibling above it — refuse.

   Read the resolved parent `index.md`, take `frontmatter.confluence.id` as `parent_id`.

2. **Call `createConfluencePage`** with `spaceId` (from the parent), `parentId`, `title` (from the file's frontmatter), `body` (the file's body), and `body-format=storage`.

3. **Write the response back** into the local file's frontmatter: `id`, `url`, `parent_id`, `version`, `last_modified`, plus `body_sha = sha256(body that was sent)`. Preserve `title`, `space_key`, `space_id`.

## Remote-only (created in Confluence with no local mirror)

After the per-id loop, list pages in the space with `getPagesInConfluenceSpace(spaceKey)` and look for ids not present in any local file's frontmatter. For each:

1. Determine the parent — `parentId` on the response. Find the local file with `confluence.id === parentId`. That file's path is the parent location; the new page goes alongside it.
2. Compute the slug: lowercase the title, collapse non-alphanumeric runs to `-`, trim leading/trailing `-`. On collision in the same parent directory, suffix `--<id>`. Keep these rules stable so existing local slugs continue to match their remote pages.
3. **If the new page has children** (check via `getConfluencePageDescendants`): create a directory `<slug>/` and write its `index.md`. Then recurse for descendants.
   **If it's a leaf**: write `<slug>.md`.
4. Frontmatter is filled from the remote response; `body_sha = sha256(remote.body)`.
