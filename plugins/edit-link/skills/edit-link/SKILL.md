---
name: edit-link
description: Let the user edit a local text file by hand in a browser, usually on their iPhone or iPad, by uploading it to their S3 bucket with an editor page behind a presigned link, then fetching what they save back into the file. Use when the user says /edit-link, wants to edit, fix or fill in a file themselves, manually or by hand, asks for a link to edit a file, or wants to change release notes, a note, config, Markdown, YAML, JSON or other text file from the phone while away from the Mac (as in a remote-controlled session), and when they ask whether such a file was edited, to fetch or pull in those edits, for the link again, to stop sharing it, or to set up or check edit-link.
allowed-tools: Bash(${CLAUDE_SKILL_DIR}/edit-link *), Bash(${CLAUDE_SKILL_DIR}/setup --check)
---

# edit-link

Upload a local text file to a private S3 bucket with an editor page next to it, show the
user a link to the page, and bring what they save there back into the file.

## Setup

Settings live in `~/.config/edit-link.env`; the keys are in the login Keychain. Any S3
service with presigned links works: AWS, or another one at the endpoint in the settings.
When the script says the settings are missing, or the user asks to set this up, tell them
to run this in their own Terminal, since it asks for the keys there:

```bash
${CLAUDE_SKILL_DIR}/setup
```

With deploy-ipa set up, it offers to use the same keys and bucket. Never ask for keys in
the conversation. To find out what is wrong with an existing setup, run
`${CLAUDE_SKILL_DIR}/setup --check`; it asks nothing and tests signed links, the change
checks saves rely on, listing and deleting.

## Steps

1. **Share the file:**

   ```bash
   ${CLAUDE_SKILL_DIR}/edit-link share <file>
   ```

   It ends with `KEY=value` lines. Put the page in the reply as a markdown link on its own
   line, followed by when it expires (`EXPIRES`):

   ```markdown
   [Edit <NAME>](<PAGE_URL>)
   ```

   Anyone holding the link can read and change the file until it expires, so it belongs in
   this conversation only. `edit-link status <file>` prints it again.

2. **Wait for the user to save**, with `run_in_background: true` and a Bash `timeout` of
   3660000:

   ```bash
   ${CLAUDE_SKILL_DIR}/edit-link fetch --wait <file>
   ```

   It checks every 3 seconds and exits as soon as the page saves, so wait for the
   completion notice rather than polling. After an hour without a save it exits with
   status 2; leave it at that unless the user is still editing. When the user says they
   are done before it fires, run `fetch` without `--wait`; a wait still running then keeps
   waiting for the next save.

3. **Apply the edits.** `fetch` prints a unified diff from the file as it is now to its
   edited version, then `CHANGED`, `CONFLICTS` and `INCOMING`, the whole edited version.
   With `CHANGED=1`, make that change to the file with Edit, or Write the content of
   `INCOMING` for a long diff. Don't copy `INCOMING` over the file with a shell command:
   the user follows every change to their files as a diff in the conversation. Then tell
   them in a line or two what they changed.

   - Apply it before anything else touches the file. Until something writes to the file,
     `share` refuses to upload it (exit 3), since that would undo the user's edits, and
     `fetch` repeats the same diff.
   - `CONFLICTS` above 0 means the file also changed here in the same lines: the diff
     has `<<<<<<< local`, `=======` and `>>>>>>> edited` blocks. Resolve them, asking the
     user when it isn't clear which side should win, then `share` so the page gets the
     result.
   - Typing on an iPhone can bring curly quotes, dashes and capitalized first words. In
     code or config, where those break things, point them out and ask before changing
     them back.
   - To keep following the user's edits, start step 2 again.

4. **Keep the page current.** When you change a shared file yourself, run `share` again: it
   uploads the new content and renews the link. An open page shows the new version within
   5 seconds, or merges it with the user's unsaved changes when they save.

5. **Stop** once the user is done with the file:

   ```bash
   ${CLAUDE_SKILL_DIR}/edit-link stop <file>
   ```

   This deletes the copy and the page from the bucket. It refuses while saved edits are
   not fetched and applied, unless given `--discard`, which is also how to drop edits the
   user wants undone. Shares that are not stopped are deleted by a later
   `share` once nothing was saved to them for the link lifetime.

`edit-link status` lists every shared file, whether edits are waiting, and its link.

## The editor page

- A text area with Save (Cmd-S on a keyboard), which warns before leaving with unsaved
  changes. It follows the system's dark mode and uses 16 px text so iOS doesn't zoom in.
- A save first asks whether the file changed since the page has it (a GET with
  `If-None-Match`, which every S3 service answers). If it did, the page merges both
  versions line by line, marking conflicts, and asks the user to review and save again,
  so neither side silently overwrites the other. Where the service takes `If-Match`, the
  save itself is conditional too, closing the moment between that check and the save.
- It only takes UTF-8 text up to 2 MB. A file with CRLF line endings is edited as LF
  and gets CRLF back on fetch.

## How it works

- The script signs every link itself with SigV4, using only Python's standard library,
  with the keys from the Keychain item the settings name (`EDIT_LINK_KEYCHAIN`). Requests
  go to AWS, or path-style to `EDIT_LINK_ENDPOINT` when set.
- Each share gets a folder `<prefix>/<date>-<random>/` holding the file and `edit.html`,
  with presigned PUT and GET links for that one file written into the page. Served from
  the same bucket, the page needs no CORS, and the links reach no other object. They last
  `EDIT_LINK_EXPIRES` (604800 s, the SigV4 maximum).
- In a versioned bucket, a save that was overwritten stays as an older version until the
  bucket's lifecycle rules remove it, also after `stop`.
- State is in `~/.local/state/edit-link/<hash of the path>/`: `share.json`, `base`, the
  last version both sides had and the base of 3-way merges with `git merge-file`, and
  `incoming`.

## When it fails

- **AWS errors:** run `${CLAUDE_SKILL_DIR}/setup --check` and pass on what fails.
- **"was saved from the editor page since the last fetch"**: run `fetch`, apply the
  edits, then `share`.
- **"the shared copy is gone"**: it was stopped or cleaned up; `share` again for a new link.
- **"not UTF-8 text" or "larger than 2 MB"**: the page can't edit it; say so.
- **The page says the link expired**: `share` again and give the user the new link.
