---
name: check-upstream
description: Check the upstream template repo (github.com/videlov/claude-knowledge-template) for updates to tooling, hooks, skills, and structure. Use when the user says "check upstream", "check for updates", "sync with upstream", or "/check-upstream".
---

# Check Upstream

Compare this knowledge repo's tooling against the upstream template and selectively apply updates, keeping local adaptations.

## How it works

`tooling/check-upstream` is a Python script that:
1. Downloads the upstream tarball (no auth needed — the template is public). The commit id comes from the tarball itself, so there is no second request and no API rate limit.
2. Works out a base — the upstream commit the local copy last matched — from `tooling/.upstream-base`, or on the first run from the copy's own git history (clone merge-base, or the newest upstream commit before a "Use this template" copy's first commit).
3. Classifies each trackable file against local, upstream, and base.
4. On `--apply`, runs a three-way `git merge-file` so local edits survive; files only the user changed are left alone.

If no base can be found, differing files are reported but never written — the script will not overwrite local edits it cannot tell apart from upstream.

## Steps

1. **Run the summary**

```bash
<repo>/tooling/check-upstream
```

Show the output. Files are grouped by status; both-sides files are marked `(merges clean)` or `(CONFLICT)`.

2. **Walk through the results with the user**

- **Changed upstream only**: safe to apply.
- **Changed on both sides**: a three-way merge. `(merges clean)` keeps both; `(CONFLICT)` writes conflict markers. Offer `--diff <path>` to preview the merged result first.
- **Changed upstream, deleted locally**: applying restores the file.
- **New upstream**: a file the copy never had. Apply or skip.
- **Deleted upstream / changed locally only / up to date / deleted locally / local only**: no action.

To preview one file's merged result without writing anything:

```bash
<repo>/tooling/check-upstream --diff <path>
```

3. **Apply**

Apply everything (and move the base forward):

```bash
<repo>/tooling/check-upstream --apply
```

Or apply only the files the user picked (the base does **not** move):

```bash
<repo>/tooling/check-upstream --apply <path> [<path> ...]
```

**Tell the user:** a full `--apply` moves the base to the current upstream, so any both-sides change they *didn't* take this time will not be offered again — it will read as "changed locally only" from now on. If they want to keep some of those offers open, apply the chosen files by path instead; the base stays put until a full apply.

4. **Handle conflicts**

For any `CONFLICT:` file, open it, resolve the `<<<<<<< local / ======= / >>>>>>> upstream` markers with the user, and remove the markers.

5. **Commit**

Stage only what the script changed, plus the base file — never `git add -A`, which would also sweep in a half-written note or other unrelated work:

```bash
git -C <repo> add <changed files> tooling/.upstream-base
git -C <repo> commit -m "chore(tooling): sync with upstream template"
```

Then remind the user to test: run `tooling/install` and verify the hooks still work.

## Safety

This skill pulls updates from the public upstream template. It never sends local content upstream.

### NEVER do:

- Any write (PUT, POST, PATCH, push) against the upstream repo
- Create PRs, issues, or forks on the upstream repo
- Expose local file contents to any external API

### Safe to do:

- Read upstream content (HTTPS download of the public tarball)
- Write to the local repo (with user approval)
- Commit and push to the user's own origin (normal — `/remember` does the same)

## Notes

- No `gh` CLI or GitHub auth required.
- The upstream repo is not added as a persistent git remote. The script does one `git fetch <url> main` to find the clone merge-base, and otherwise downloads tarballs to a temp dir.
- The script never deletes files. A file deleted locally stays deleted (unless upstream changed it since the base); a file deleted upstream is reported but kept.
- Applying writes to a temp file and moves it into place, so updating `tooling/check-upstream` itself while it runs is safe.
- `tooling/skills/check-upstream/` (this SKILL.md) is skipped by the script: it travels with the script and is updated together, not merged in.
- `findings/claude-code-token-usage.md` is tracked because it ships with the template and is cited by `tooling/instructions.md` and the README.
