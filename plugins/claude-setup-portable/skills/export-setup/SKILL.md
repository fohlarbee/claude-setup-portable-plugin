---
name: export-setup
description: Use when the user wants to save, export, or back up their Claude Code setup before switching machines — phrasing like "save my Claude Code setup", "export my config before I switch machines", "back up my plugins and skills", or "let me take my Claude setup to a new computer".
---

# export-setup

Writes `claude-setup.json` (plus, optionally, `claude-setup-skills.tar.gz`) to the current
directory, capturing what does NOT already survive a machine move on its own.

## What already survives a move — never touch this

Plugins enabled on the user's claude.ai account sync automatically to
`~/.claude/plugins/synced/` and load as `<name>@synced`, with no marketplace and no install
record (requires Claude Code v2.1.273+, signed in, `syncClaudeAiPlugins` on). Skills sync the
same way into `~/.claude/skills/synced/`. Both are already portable. Exclude them completely:

- Any plugin whose id ends `@synced` — do not put it in `plugins`; put it in `skipped` with
  reason `"synced from claude.ai account; already portable, no action needed"`.
- Any directory under `~/.claude/skills/synced/` or `~/.claude/commands/synced/` — never
  inventory these as if they were the user's own skills/commands.

## Steps

1. Run `claude plugin list --json`. For each entry:
   - If the id ends `@synced`, or its `installPath` sits under `plugins/synced/`, record it
     in `skipped` with reason `"synced from claude.ai account; already portable, no action
     needed"` and move on.
   - If the id ends `@skills-dir` (a loose plugin under `~/.claude/skills/<name>/`, not a
     marketplace install), do **not** put it in `plugins` — `"skills-dir"` isn't a real
     marketplace and `restore-setup` can't add it as one. Record it in `skipped` with reason
     `"skills-dir plugin; restored via the skillsDir entry of the same name instead"` and
     confirm step 3 below will pick it up by directory name.
   - Otherwise it's marketplace-installed: record `{name, marketplace, version, scope}`
     (split the id on `@`).
2. Read `~/.claude/plugins/known_marketplaces.json`. For each marketplace actually backing a
   plugin from step 1, record `{name, source, trust}`:
   - `"official"` for `anthropics/claude-plugins-official`
   - `"own"` when the source is a GitHub repo whose owner matches the exporting user's own
     GitHub login (`gh api user --jq .login`, best-effort — if `gh` isn't available or isn't
     authenticated, skip this check and fall through to `"third-party"` rather than guessing)
   - `"third-party"` for everything else — someone else's repo the user chose to trust on
     this machine
   Don't include marketplaces that back nothing installed — no point restoring a dead
   marketplace.
3. Inventory `~/.claude/skills/` and `~/.claude/commands/`, **excluding** any `synced/`
   subdirectory. For each entry:
   - Check for a `.git` directory inside it. If present, get the remote with
     `git -C <path> remote get-url origin` (may fail — that's fine, treat as untracked).
   - If it has a working origin remote: `{name, portability: "git", remote}`.
   - If not: ask the user (once, listing all untracked skills/commands together, not one
     prompt per item) whether to bundle them into `claude-setup-skills.tar.gz` next to the
     manifest. This reads file contents into an archive, so get explicit confirmation before
     doing it — do not bundle silently.
     - If yes: tar the untracked ones (`tar czf claude-setup-skills.tar.gz -C ~/.claude
       skills/<name> commands/<name> ...`), record `{name, portability: "bundled",
       bundlePath: "claude-setup-skills.tar.gz#skills/<name>"}` per entry.
     - If no: record `{name, portability: "manual"}` for those the user declined.
4. Check `~/.claude/CLAUDE.md`. If present, compute its sha256. Ask the user whether to
   include the actual file content in the manifest (default: no — hash only). If they say
   yes, inline the text under `claudeMd.content`; otherwise leave it `null`. Always record
   `present` and `sha256` regardless of their answer.
5. Read `~/.claude/settings.json`. Copy top-level keys whose value is a primitive (string,
   number, boolean) or a plain object of primitives, **excluding** any key matching (case
   insensitive) `token`, `key`, `secret`, `credential`, `auth`, and always excluding
   `enabledPlugins`, `extraKnownMarketplaces`, `pluginConfigs` — those three are
   reconstructed from `plugins`/`marketplaces` by real installs on restore, never copied
   directly. Never read `~/.claude/.credentials.json` or `~/.claude.json` at all — not even
   to check for keys to exclude.
6. Write `claude-setup.json` with `schemaVersion: 1`, `exportedAt` (ISO8601, now),
   `exportedFrom: {claudeCodeVersion, platform}` (`claude --version`, `process.platform` /
   `uname`), and the fields gathered above.
7. Write `claude-setup.md`, a short human-readable summary: what was captured (counts), what
   was skipped and why, what needs manual action (any `portability: "manual"` entries, and
   CLAUDE.md if hash-only).
8. Tell the user in chat: where the file(s) landed, the one-line takeaway (N plugins across M
   marketplaces, N skills — X git-backed, Y bundled, Z manual), and to carry `claude-setup.json`
   (and the tarball, if created) to the new machine however they like — it's plain text/a
   plain archive, safe to email to themselves once they've confirmed the settings section
   doesn't contain anything they'd rather not send that way.

## Never

- Never read or write `~/.claude/.credentials.json` or `~/.claude.json`.
- Never copy `~/.claude/plugins/cache/` or reference paths inside it in the manifest.
- Never include `projects/`, `history.jsonl`, `shell-snapshots/`, `todos/`, or `plans/`.
- Never guess at a settings key's safety by name pattern alone without applying the
  exclusion list in step 5 — when genuinely unsure whether a key is sensitive, leave it out
  and mention it was skipped in the summary, rather than including it.
