# claude-setup-portable

A Claude Code plugin for moving your setup — marketplace-installed plugins, custom skills
and commands, `CLAUDE.md`, and a safe subset of `settings.json` — to a new machine.

## The problem

Claude Code stores plugins, skills, and settings locally under `~/.claude/` (Windows:
`C:\Users\<name>\.claude\`). Move to a new machine and all of it is gone. There is no
official sync for marketplace-installed plugins or custom skills: the feature request for
account-level settings sync,
[anthropics/claude-code#22648](https://github.com/anthropics/claude-code/issues/22648), is
still open with nothing shipped, and the more direct ask —
[#36693](https://github.com/anthropics/claude-code/issues/36693), "no native way to sync
Claude Code config and skills across machines" — was closed **not planned**.

### What already syncs on its own — this plugin does not touch it

Plugins enabled on your claude.ai account sync automatically to
`~/.claude/plugins/synced/` and load as `<name>@synced`, with no marketplace and no install
record (requires Claude Code v2.1.273+, signed in to claude.ai, `syncClaudeAiPlugins` on).
Skills sync the same way into `~/.claude/skills/synced/`. Both are already portable —
`export-setup` detects and excludes them, and reports why in its summary. Including them in
a manifest would produce install commands that fail, since `claude plugin
install`/`update`/`uninstall` don't apply to synced items.

### The actual gap: four things do not survive a move

1. Marketplace-installed plugins (`claude plugin install <name>@<marketplace>`)
2. Custom skills in `~/.claude/skills/` and commands in `~/.claude/commands/`
3. `~/.claude/CLAUDE.md`
4. A safe subset of `~/.claude/settings.json`

## Why the obvious fix — copy the files — doesn't work

Claude Code keeps plugin state in four locations that routinely drift apart:

- `~/.claude/plugins/installed_plugins.json`
- `enabledPlugins` in `~/.claude/settings.json`
- `enabledPlugins` in `~/.claude/settings.local.json`
- `~/.claude/plugins/cache/`

This isn't a hypothetical concern — it's been reported against Claude Code itself
(issue numbers checked against the repo before publishing this; noted here with their
actual resolution, not just their titles):

- **[#17832](https://github.com/anthropics/claude-code/issues/17832)** ("Directory
  marketplace plugins not auto-enabled in settings.json") — an install can land in
  `installed_plugins.json` without a matching entry in `enabledPlugins`, so the plugin looks
  installed but its skills never load. Closed **not planned**, i.e. acknowledged, not fixed.
  **[#20661](https://github.com/anthropics/claude-code/issues/20661)** reported the same
  failure independently and was closed as a duplicate of it.
- **[#52456](https://github.com/anthropics/claude-code/issues/52456)** ("`/plugin` TUI
  cannot reliably uninstall plugins; only CLI `claude plugin uninstall` works end-to-end") —
  a documented instance of the same class of bug: an uninstall path that doesn't clean up
  every location it touched. Closed **completed**, with no maintainer comment confirming
  what shipped, so treat it as a real historical failure mode rather than a currently
  reproducible one — the point stands regardless: the four locations *can* go out of sync
  through normal use, by construction, not just hypothetically.
- **[#32606](https://github.com/anthropics/claude-code/issues/32606)** ("`extraKnownMarketplaces`
  + `enabledPlugins` in project settings never prompts user to install") — declaring those
  keys directly in a project's `.claude/settings.json` does **not** trigger installation,
  contrary to what the docs on trusted-repo marketplace prompts imply. Closed **not
  planned**. This is precisely why `restore-setup` never writes `enabledPlugins` or
  `extraKnownMarketplaces` into `settings.json` and hopes Claude Code notices — it always
  runs the actual `claude plugin marketplace add` / `claude plugin install` commands
  instead.

Copy that state to a new machine and you don't get a working copy of the setup — you get
whatever already-diverged state the source machine happened to be in, reproduced exactly,
including the parts that were silently broken. A plugin can appear installed in
`installed_plugins.json` and do nothing at all.

`~/.claude/plugins/cache/` compounds this: it holds version-pinned plugin copies with
machine-specific absolute paths baked in, plus any auto-installed `node_modules` for
plugins with their own dependencies. Copying it across machines (especially across OSes) is
unsafe on top of being unnecessary — it's a cache, not a source of truth.

**Therefore: this plugin never syncs state.** It reads the real, current state on the
source machine, writes a plain manifest describing it, and on the destination machine
**regenerates that state by running the actual `claude plugin marketplace add` and `claude
plugin install` commands** — the same commands a person would type themselves — and lets
Claude Code write its own internal state. Nothing here writes directly to
`installed_plugins.json`, `enabledPlugins`, or `plugins/cache/`.

## Why not the existing community sync tools

There are two camps of community tool already solving a version of this (checked against
their actual repos, not assumed from the name):

- **Cloud-storage sync**, e.g. [`claude-sync`](https://github.com/tawanorg/claude-sync) —
  genuinely does what it sounds like: continuous, encrypted (age), bidirectional sync to
  Cloudflare R2, AWS S3, GCS, or WebDAV. That closes the gap, at the cost of a storage
  provider account to set up and pay for, a passphrase/key to generate and not lose, and
  care that its selective-sync scoping keeps `.credentials.json` and OAuth tokens out of
  what gets uploaded.
- **Git-repo sync**, e.g. the `claude-sync-kit`-style tools — sync settings, skills, and
  plugin/MCP enablement across machines via a private GitHub repo instead of cloud storage.
  Lighter setup than the cloud-storage camp, but it's still **syncing state** — the same
  `~/.claude/` files this README opened by explaining as unreliable to move directly — and
  it puts that state in a repo, which means git-history discipline becomes the thing
  standing between you and a leaked credential, instead of a storage-bucket exclude list.

Either camp genuinely works and is a reasonable choice if you're already comfortable with
its setup. Both inherit the same underlying risk, though: they **sync state**, so a
corrupted or half-installed plugin (see the four-location drift above) propagates to every
other machine just as faithfully as a working one would — encryption and git history don't
distinguish a healthy `installed_plugins.json` from a broken one.

Most people move to a new machine once or twice a year, not continuously. Standing cloud
infrastructure, key management, and an ongoing security surface is a lot of machinery for
an occasional, one-directional task.

**The differentiator here: no cloud account, no encryption, no key management, no secrets
leave the machine unless you explicitly opt in.** `export-setup` writes a plain-text JSON
manifest (plus an optional plain `.tar.gz` for skills with no git remote) that you carry
over however you already move personal files — AirDrop, USB, emailing it to yourself. It
never touches `.credentials.json` or `.claude.json`, never includes tokens/keys/secrets by
name-pattern, and asks explicitly before including anything borderline (like the actual
text of `CLAUDE.md`, rather than just its hash).

## Install

```
/plugin marketplace add fohlarbee/claude-setup-portable-plugin
/plugin install claude-setup-portable@claude-setup-portable-tools
```

## Usage

**On the machine you're leaving:**
> "Export my Claude Code setup" / "back up my plugins and skills before I switch machines"

Writes `claude-setup.json` (and `claude-setup.md`, a plain-English summary of what it
captured) to `~/Desktop`, falling back to your home directory if `Desktop` doesn't exist —
either way, Claude tells you exactly where it landed.

**On the new machine:**
> "Restore my Claude Code setup" / "set up this new machine" / "I have a claude-setup.json"

`restore-setup` always shows the full plan first — every marketplace (labeled official /
your own / third-party), every plugin, every settings key that would change — before
touching anything, and never overwrites `CLAUDE.md` or `settings.json` without a backup and
your explicit confirmation.

**A note on installs:** `claude plugin install` can't run unattended inside a Claude Code
session (`-y` has no effect there, and it's refused when stdin isn't a TTY). `restore-setup`
prints the exact commands and has you run them from your own terminal — this is also what
keeps a marketplace-declared command's confirmation prompt intact instead of routing around
it.

See [`plugins/claude-setup-portable/SCHEMA.md`](./plugins/claude-setup-portable/SCHEMA.md)
for the full `claude-setup.json` format.

## License

MIT
