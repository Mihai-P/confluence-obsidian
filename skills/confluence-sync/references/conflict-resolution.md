# Conflict resolution

How to handle a `CONFLICT` (local and remote both moved since the last sync). Loaded on demand from `SKILL.md` when the decision matrix lands in the conflict bucket. **Never auto-merge. Never silently choose a side.**

When you hit a CONFLICT, do this:

1. Print:

   ```text
   CONFLICT  <path>
     local  v<N>  body_sha=<hex8>
     remote v<M>  body_sha=<hex8>
   ```

2. Show the diff. Easiest: write `local.body` and `remote.body` to two temp files under `/tmp/confluence-sync/` and run `diff -u` on them, or render the diff inline if it's short.
3. Ask the user to pick: **keep local** (push), **keep remote** (pull), **edit** (open the local file for the user to merge by hand, then re-run sync).
4. Whichever side wins, after applying it the frontmatter must end up reflecting the final remote state — `version`, `last_modified`, `body_sha`.
