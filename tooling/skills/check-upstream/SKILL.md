---
name: check-upstream
description: Check the upstream template repo (github.com/videlov/claude-knowledge-template) for updates to tooling, hooks, skills, and structure. Use when the user says "check upstream", "check for updates", "sync with upstream", or "/skill:check-upstream".
---

# Check Upstream

Compare this knowledge repo's tooling against the upstream template at `https://github.com/videlov/claude-knowledge-template` and report what changed.

## Steps

1. **Fetch the upstream default branch listing**

```bash
gh api repos/videlov/claude-knowledge-template/git/trees/main?recursive=1 --hostname github.com -q '.tree[] | "\(.path)\t\(.sha)"'
```

If main fails (404), try master.

2. **Identify trackable paths**

Only compare paths that are part of the template scaffolding — not user content:

- `tooling/` (all files)
- `INDEX.md` structure (ignore content)
- `README.md`
- `.gitignore`

Skip `findings/`, `projects/` — those are user data.

3. **For each upstream file, diff against local**

For every file in the trackable set, fetch its content and diff:

```bash
gh api repos/videlov/claude-knowledge-template/contents/<path> --hostname github.com -q '.content' | base64 -d > /tmp/upstream-<sanitized-name>
diff /tmp/upstream-<sanitized-name> <local-repo>/<path>
```

4. **Report**

Print a summary table:

| File | Status |
|------|--------|
| `tooling/install` | changed (show key diff) |
| `tooling/skills/remember/SKILL.md` | up to date |
| `tooling/reindex` | new upstream (not present locally) |

For changed files, show a short diff summary (not the full diff unless the user asks).

5. **Offer actions**

For each changed or new file, ask:
- **Apply** — overwrite local with upstream version
- **Skip** — keep local version
- **Show diff** — display full diff before deciding

Do NOT apply any changes without user approval.

## Safety: One-Way Pull Only

This repo may contain internal/company-specific data in `findings/`, `projects/`, and customized tooling. This skill is strictly pull-only.

### NEVER do any of the following:

- `git push` to any remote (including origin)
- `git remote add` the upstream repo
- `gh api` write operations (PUT, POST, PATCH) against the upstream repo
- Create PRs, issues, or forks on the upstream repo
- Expose local file contents to any external API beyond diffing locally

### ONLY do:

- Read upstream file contents via `gh api` GET requests
- Download to `/tmp/` for local comparison
- Write to the local repo (with user approval) to apply upstream changes
- Commit locally (with user approval) after applying changes

If the user asks to push, contribute back, or sync upstream, refuse and explain that the local repo may contain internal data.

## Notes

- This skill requires `gh` CLI authenticated against github.com (not the enterprise instance).
- The upstream repo is not a git remote. This skill uses the GitHub API to avoid polluting the local git config.
- All upstream content is fetched read-only and compared locally. No local content is ever sent upstream.
- After applying updates, remind the user to test the tooling (run `tooling/install`, verify hooks).
