# Claude knowledge repo (template)

Long-term memory for Claude Code: one private git repo of Markdown notes that every `claude` session can search, plus a `/remember` skill that saves what a session learned. This is a reference setup for a single `claude` installation; copy it and adapt it to your environment.

## How it works

```text
every session ── loads ──▶ tooling/instructions.md + your MEMORY.md index   (≈0.6–1k tokens, always)
     │
     ├─ recall ─────▶ grep findings/ and projects/, read the 1–2 notes that match   (≈1k tokens, on demand)
     └─ /remember ──▶ write or update one note ─▶ pre-commit hook ─▶ commit ─▶ push
```

- `findings/`: one durable fact per note (a finding, decision, how-to or reference).
- `projects/`: the state of ongoing efforts, rewritten as things change.
- `INDEX.md`: one line per note, rebuilt on every commit.
- `tooling/`: the instructions Claude gets, the `/remember` skill, the git hook, the installer, `claude-usage`, which shows what each session costs in tokens, `claude-context`, which shows how big the current session's context is, and `check-upstream`, which compares your tooling against the upstream template and offers to apply updates.

The notes are plain Markdown in git, so every change can be reviewed with `git log -p`, nothing is locked into a tool, and the folder opens as an Obsidian vault. There is no database, embedding model or MCP server: Claude uses the search and read tools it already has.

## Quick start

1. **Make your own private copy.** Use this template on GitHub ("Use this template"), or clone it and push it to a new private repository. If your notes will contain company knowledge, keep the repository on your company's GitHub.
2. **Clone it** to a path without spaces, for example `git clone <your repo URL> ~/knowledge`.
3. **Install the helpers** (recommended): `brew install gitleaks jq`. gitleaks scans every commit for secrets; jq lets the installer edit Claude's `settings.json`.
4. **Run the installer:** `~/knowledge/tooling/install`.
5. **Start a new `claude` session** and try it: *"remember that …"*. A session that was already running may not see the new skill.

To check that Claude registered the skill:

```sh
claude -p ok --output-format stream-json --verbose | jq -c 'select(.subtype=="init") | .skills'
```

## What the installer changes

In your Claude config folder (`$CLAUDE_CONFIG_DIR`, default `~/.claude`):

| File | Change |
|---|---|
| `CLAUDE.md` | A marked block with the repo path and `@<repo>/tooling/instructions.md`, so every session knows the repo and how to use it. Your own content stays. |
| `skills/remember` | A symlink to `tooling/skills/remember`. |
| `skills/check-upstream` | A symlink to `tooling/skills/check-upstream`. |
| `settings.json` | Adds the repo to `permissions.additionalDirectories`, so Claude can read it from any folder, and sets `autoMemoryDirectory` to `~/.claude-memory`, so all your folders share one auto memory. `KNOWLEDGE_MEMORY_DIR= tooling/install` leaves auto memory as it is. |

In the repo it sets `core.hooksPath` to `tooling/githooks`. Running it again is safe. `tooling/uninstall` removes all of it and leaves your notes and the memory folder alone.

## Day-to-day use

| You want to | Say or do |
|---|---|
| Keep what a session found | "remember this", or `/remember` (without an argument it suggests up to five candidates) |
| Keep something specific | `/remember <what>`, ideally with the reason: "remember that X, because Y" |
| Use what is already known | Just work: Claude checks the repo first and says which notes it used. Or ask "what do we know about X?" |
| Track a multi-session effort | "Start a project note for X" |
| Stop: a chunk of work is done, or a break of more than an hour is coming | Claude hands off on its own: it updates the project note, asks you to `/clear` with the context size (tokens and % of the window) and puts a prompt for the fresh session on your clipboard; the fresh session starts from the note instead of re-reading a long conversation ([Cost in tokens](#cost-in-tokens)). Say "hand off" to trigger it yourself |
| Change how Claude works everywhere | "From now on …" (this goes into the shared auto memory, not the repo) |
| See what was saved | `git -C ~/knowledge log -p` |
| Check for template updates | `/check-upstream` — compares tooling against the upstream template and offers to apply changes |
| Browse | `INDEX.md`, or open the folder in Obsidian |

## Note format

```markdown
---
title: "One-line claim, specific enough to act on"
type: finding            # finding | decision | howto | reference | project
tags: [topic, system]
as_of: 2026-09-28
status: current          # current | superseded (then add superseded_by: <slug>)
sources:
  - a URL, file path or command that shows it
---

The fact in one to three sentences.

**Evidence:** how it was verified

**How to apply:** when it matters and what to do

Related: [[other-slug]]
```

The file name is the slug (`findings/<slug>.md`), so `[[slug]]` links resolve in Obsidian. `projects/example-project.md` shows a project note; delete it once you have your own. The three notes in `findings/` are real Claude Code findings and worth keeping.

## Cost in tokens

Measured on a working setup, estimating tokens as bytes ÷ 4:

- **Every session:** the instructions (≈650 tokens), the skill description (≈100) and your auto-memory index: about 0.7k to start and around 1k once the index has grown. It is the same text every turn, which suits prompt caching.
- **A recall:** a grep plus one median note, about 1k tokens, and only when a topic comes up.
- **A save:** the `/remember` instructions, about 1k tokens, and only when you save.

For comparison, loading all 60 notes of that setup into every session would cost about 76k tokens, and the notes were about 220× smaller than the raw session transcripts they came from. Claude Code deletes transcripts after 30 days by default; the notes stay.

The bigger cost is the session itself, and the repo is what lets you keep sessions short. Every call re-sends the whole conversation as a cheaper cache read, and the cache lasts an hour: the first call after a longer break writes all of it to the cache again. A fresh session starts at about 37k tokens (system prompt, tools, CLAUDE.md, auto memory); one left open for days re-sends up to 0.9M on every call. On one measured day of three long-lived sessions on Claude Opus 5.5, at its API prices, cache writes were about half of the cost (almost all full re-caches after hours idle), cache reads over a third and output about a ninth. Updating a project note and running `/clear` starts the next session at about 40k instead of resuming at 900k. Details in `findings/claude-code-token-usage.md`; measure your own day with `tooling/claude-usage [YYYY-MM-DD]`.

## Limits, and how to keep it cheap

| Watch out for | Keep it healthy |
|---|---|
| Long sessions: every call re-sends the whole conversation, and after an hour idle all of it is written to the cache again | One session per chunk of work: at a hand-off Claude updates the project note, tells you how big the context is and copies a prompt for the fresh session to the clipboard; run `/clear` and paste it. `/remember` alone saves nothing; the `/clear` after it does. If a session must go on, `/compact` early (~150–200k; `tooling/claude-context` shows the size) |
| Very long notes: costly to read and harder to search | One fact per note, 3 KB or less; project notes hold state, not a diary |
| Word-based search: synonyms can slip through as notes pile up | Claude tries synonyms; add a semantic index only when lookups miss, built from the Markdown and never a second copy |
| Stale notes | Check `as_of`, verify, and mark old notes `status: superseded` |
| A growing auto-memory index, which loads into every session | Keep `MEMORY.md` to preferences and one-line pointers |

## Adapting it

- **Content that must never be committed:** add one regular expression per line to `tooling/blocked-patterns.txt`, for example internal hostnames if the repo lives outside your company. The hook rejects notes that match.
- **Separate work and personal notes:** this template keeps to one repo. Two repos need a routing rule ("company-internal → work repo") in `tooling/instructions.md`, one `/remember` skill that knows both repos, and an installer that wires both.
- **A shared team repo:** possible, but first agree on who reviews notes (pull requests) and how stale notes get retired.

## Requirements and troubleshooting

- macOS or Linux with bash and git; on Windows use WSL or Git Bash. gitleaks and jq are recommended.
- A current Claude Code with user skills and `autoMemoryDirectory`. On older versions auto memory stays per folder; the rest works.
- If your organization's managed Claude Code settings block user skills, `/remember` has to ship as a plugin instead. The hook is a plain git hook and is not affected.
- **`/remember` is missing:** start a new session and check the skill list with the command above.
- **A push fails:** check VPN and git credentials; the commit stays local until the next push.
- **A commit is blocked:** remove the secret or blocked detail from the note; don't bypass the hook with `--no-verify`.
