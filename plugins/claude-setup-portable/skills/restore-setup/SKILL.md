---
name: restore-setup
description: Use when the user wants to restore or apply a previously exported Claude Code setup on a new or different machine — phrasing like "restore my Claude Code setup", "set up this new machine", "I have a claude-setup.json", or "apply my exported Claude config here".
---

# restore-setup

Applies a `claude-setup.json` (from `export-setup`) by running the real Claude Code CLI
commands, never by writing to Claude Code's internal state files directly.

## Steps

1. Locate `claude-setup.json` — `export-setup` writes it to `~/Desktop` by default, so check
   there first, then the current directory, then ask for a path if it's in neither. Parse
   it; refuse and explain if `schemaVersion` isn't `1`.
2. Check `claude --version` locally. If below `2.1.273`, tell the user claude.ai account
   plugin/skill sync isn't available on this install, so anything that was in the source
   export's `skipped` list (synced items) won't come over automatically either — they'd need
   to re-enable those manually on claude.ai once they're on a newer version, or accept not
   having them here.
3. **Build and show a plan before touching anything.** For each marketplace in the manifest,
   label it by its `trust` field: `(official)`, `(your own repo: <repo>)`, or `(third-party:
   <repo>)`. The person restoring may not be the person who exported, so make the
   distinction visible, not just the name — a third-party entry is someone else's repo the
   exporter chose to trust, which is a different risk than the exporter's own work or
   Anthropic's marketplace. List:
   - every marketplace to add, with its trust label
   - every plugin to install, grouped under its marketplace
   - every settings key that would be merged, showing current-machine value vs. incoming
     value where they differ
   - every skill/command to restore, and how (`git clone <remote>`, extract from the
     tarball, or "manual — no source captured, ask the user")
   - whether `CLAUDE.md` content is present in the manifest or hash-only (and if hash-only,
     that the user must copy that file over themselves)
   Get explicit confirmation on this plan before proceeding.
4. **Detect existing state before writing anything:**
   - A plugin id already in `claude plugin list --json` → skip it, note "already installed."
   - A settings key already set to a *different* value than the manifest → ask per key
     (or as a batch) rather than overwriting silently.
   - `~/.claude/CLAUDE.md` already exists and differs (by hash) from the manifest → **never
     overwrite silently.** Back up the existing file (e.g. `CLAUDE.md.bak-<timestamp>`) and
     either merge or ask the user how to reconcile.
   - Same backup-before-overwrite rule for `settings.json`.
5. For each marketplace not already known: run `claude plugin marketplace add <source>`.
6. For each plugin not already installed: **run `claude plugin install <name>@<marketplace>
   --json` directly** — verified against real installs, not assumed: an ordinary
   git/url/archive-sourced plugin installs successfully with no confirmation needed at all,
   even fully non-interactively, inside a Claude Code session. There is no general "can't
   install from inside a session" limitation — don't tell the user to run these themselves
   when the tool can just do it. Parse only the **last line** of stdout as JSON — a
   marketplace-declared command (see below) prints human-readable text above it, and that
   line is not part of the structured result.
   - **The one real exception**, and it's narrow: a plugin whose source is `command` or
     whose entry sets `headersHelper` triggers a confirmation gate that genuinely cannot be
     satisfied from inside a session (see below). Only for an install that actually comes
     back with `failureCode: "command_source_refused"` or `"entry_helper_unconfirmed"` —
     not preemptively, and not for every install — fall back to printing that one command
     for the user to run in their own terminal, then continue installing the rest directly.
   - A marketplace-declared command shows up two ways, both verified directly against real
     installs (not assumed from docs) — treat them the same:
     - A **command-source** plugin (the whole plugin comes from running a local command):
       `shownCommand.kind: "command_source"`, `failureCode: "command_source_refused"`.
     - An **archive-source `headersHelper`** (a normal URL download, but a local command
       mints the auth headers sent with it — distinct code path, only triggers when the
       user installs or updates that one plugin by itself, never for a batch install or a
       dependency pull): `shownCommand.kind: "entry_helper"`, `failureCode:
       "entry_helper_unconfirmed"`. Its `shownCommand` also carries `archiveUrl` — include
       that in what you show the user, not just the command.
     For either kind, tell the user about it in the printed instructions — **never tell
     them to pass `-y` or `--accept-command` blindly.** Verified directly for both kinds:
     inside a Claude Code session, `-y` and `--accept-command <sha256>` are refused
     categorically — even with the exact matching sha256 (`acceptCommandMatched: true` in
     the response), the command is not run and the response says to run it in your own
     terminal. This is a hard product boundary, not just a convention this skill is
     choosing to follow — there is no flag combination that lets an agent session accept
     either kind on the user's behalf. Print the full command from `shownCommand.command`
     (plus `archiveUrl` for the `entry_helper` kind) and the "run this in your own
     terminal" instruction verbatim so the user can review and accept it themselves; skip
     explaining this only if the user explicitly says they've already reviewed and trust
     it.
7. Restore skills/commands. Note: if a `~/.claude/skills/<name>/` entry contains its own
   `.claude-plugin/plugin.json`, restoring its files here is the *entire* restore for it —
   it auto-loads as `<name>@skills-dir` next session, with no separate `claude plugin
   install` step (the source export's `skipped` list will have excluded its `@skills-dir`
   plugin-list entry for exactly this reason — don't try to install it as a plugin too).
   - `portability: "git"` → `git clone <remote> ~/.claude/skills/<name>` (or `commands/`).
   - `portability: "bundled"` → extract the matching path from the tarball into place.
   - `portability: "manual"` → tell the user this one has no captured source; they need to
     bring it over themselves.
   - If a target directory already exists, don't overwrite — report the conflict and ask.
8. Merge allowlisted `settings` keys into `~/.claude/settings.json` (after backing it up),
   only for keys that were missing or that the user approved overwriting in step 4. **This
   must be read-existing-file, modify only the specific keys, write back — never write a
   fresh object built only from the manifest's `settings`.** Proven by getting this wrong in
   testing, not just a theoretical risk: writing `settings.json` from just `{model, theme}`
   silently deleted the `enabledPlugins` key that step 6's installs had just populated,
   leaving every plugin installed but disabled — exactly the #17832 failure mode this
   plugin's README exists to avoid, self-inflicted by skipping the merge step. Do the merge
   in one read → modify → write, in that order, after step 6 has finished installing, and
   never touch `enabledPlugins` here at all — step 6's real installs are what populate it
   correctly; this step only adds the unrelated allowlisted keys (`model`, `theme`, etc.)
   alongside whatever is already there.
9. If the manifest has `claudeMd.content`, write it to `~/.claude/CLAUDE.md` only if that
   file doesn't already exist, or after explicit confirmation if it does (see step 4). If
   only `claudeMd.sha256` is present, tell the user to copy that file over by hand and how to
   verify it landed correctly (`shasum -a 256 ~/.claude/CLAUDE.md`).
10. Verify: run `claude plugin list` and diff the result against the manifest's `plugins`
    list. Report per-plugin: installed & enabled / installed but disabled / missing, and for
    anything missing, the exact command to retry.
11. Tell the user to restart Claude Code (or run `/reload-plugins`) for everything to take
    effect.

## Never

- Never write directly to `installed_plugins.json`, `enabledPlugins` in settings, or
  anything under `plugins/cache/` — those are Claude Code's own state, rebuilt correctly
  only by the real install commands.
- Never claim you *can* auto-accept a marketplace-declared command on the user's behalf —
  neither a command-source install nor an archive-source `headersHelper`. Don't try `-y` or
  `--accept-command` as a shortcut on either; report it as blocked instead of quietly
  working around it. Verified for both kinds: Claude Code itself refuses `-y` and
  `--accept-command` inside a session, even with a correct sha256, so there's no workaround
  to reach for in the first place.
- Never overwrite `CLAUDE.md` or `settings.json` without a backup and explicit confirmation
  when they already exist and differ from the manifest.
- Never write `settings.json` as a fresh object containing only the manifest's `settings`
  keys — always read-modify-write the existing file. Confirmed by reproducing the bug: doing
  this wipes `enabledPlugins` and leaves every just-installed plugin disabled.
- Never assume `~` expands the same way cross-platform, or that `jq` is installed — do path
  joins and JSON parsing with the tools actually available in this environment.
