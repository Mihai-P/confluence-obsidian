# Conflict resolution

How to handle a `CONFLICT` — local and remote both moved since the last sync (`local_changed` and
`remote_changed` both true). Loaded on demand from `SKILL.md`. **Never auto-merge. Never silently choose a
side.**

When you hit a CONFLICT:

1. Print:

   ```text
   CONFLICT  <path>
     local  v<N>  body_sha=<hex8>
     remote v<M>
   ```

2. **Show the diff.** Both sides are Markdown now, so a line diff is meaningful. Write the local body and
   the remote markdown body to two temp files under `/tmp/confluence-sync/` and run `diff -u` on them (or
   render inline if short). Compare the **bodies** (frontmatter excluded, H1 included) so the diff isn't
   polluted by the watermark.

3. **Ask the user to pick:**
   - **keep local** → PUSH (run the push guard, then strip frontmatter + H1, rewrite links → absolute,
     `updateConfluencePage`).
   - **keep remote** → PULL (overwrite local body with the remote markdown, rewrite links → relative `.md`).
   - **edit** → open the local file for a manual merge, then re-run sync.

4. Whichever side wins, after applying it the frontmatter must reflect the final remote state — `version`
   (from the write response on PUSH, or the fetched page on PULL), `last_synced`, and
   `body_sha = sha256(local_body)`.

Note: because `remote_changed` is detected by `version` (not by hashing), a Markdown round-trip that only
reformats bytes never *causes* a conflict on its own — a CONFLICT means the remote `version` genuinely
advanced while local edits were also made.
