# `claude-setup.json` schema

`schemaVersion: 1`. Both skills read/write this shape; bump the version and branch on it in
both skills before changing the shape, so an old manifest doesn't get silently misread.

```jsonc
{
  "schemaVersion": 1,
  "exportedAt": "2026-09-23T03:00:34.000Z",     // ISO8601, export time
  "exportedFrom": {
    "claudeCodeVersion": "2.1.278",              // from `claude --version`
    "platform": "darwin"                         // darwin | win32 | linux
  },

  "marketplaces": [
    {
      "name": "backend-architecture-tools",
      "source": { "source": "github", "repo": "fohlarbee/backend-architecture-plugin" },
      "trust": "own"                             // "official" | "own" | "third-party"
    }
  ],
  // Only marketplaces that back at least one entry in `plugins` below. "own" means the repo
  // owner matched the exporting user's GitHub login at export time (best-effort via `gh`);
  // "official" is reserved for anthropics/claude-plugins-official; everything else is
  // "third-party" — someone else's repo the exporter chose to trust.

  "plugins": [
    { "name": "backend-architecture", "marketplace": "backend-architecture-tools",
      "version": "1.0.0", "scope": "user" }
  ],
  // Marketplace-installed plugins only. Never includes @synced or @skills-dir plugins —
  // those are handled by `skipped` and `skillsDir` respectively.

  "skipped": [
    { "id": "some-plugin@synced",
      "reason": "synced from claude.ai account; already portable, no action needed" },
    { "id": "claude-setup-portable@skills-dir",
      "reason": "skills-dir plugin; restored via the skillsDir entry of the same name instead" }
  ],

  "skillsDir": [
    { "name": "foo", "portability": "git", "remote": "https://github.com/you/foo.git" },
    { "name": "bar", "portability": "bundled", "bundlePath": "claude-setup-skills.tar.gz#skills/bar" },
    { "name": "baz", "portability": "manual" }
  ],
  "commandsDir": [
    // same three-shape entries, sourced from ~/.claude/commands/ instead
  ],
  // Never includes anything under a `synced/` subdirectory — that's claude.ai's sync
  // channel, not the user's own files, and it's already portable on its own.

  "claudeMd": {
    "present": true,
    "sha256": "…",
    "content": null                              // string only if the user opted in at export; else null
  },

  "settings": {
    // Shallow copy of ~/.claude/settings.json top-level keys, filtered:
    // - excluded if the key name (case-insensitive) contains: token, key, secret,
    //   credential, auth
    // - always excluded: enabledPlugins, extraKnownMarketplaces, pluginConfigs
    //   (these are reconstructed by real installs on restore, never copied)
    // - included only if the value is a primitive, or an object of only primitives
    "model": "opus",
    "theme": "dark"
  },

  "notes": [ "free-text strings for anything export-setup wants to flag" ]
}
```

## Never captured, by design

- `~/.claude/.credentials.json`, `~/.claude.json`, OAuth tokens, or any `sensitive: true`
  `userConfig` value.
- `~/.claude/plugins/cache/` — regenerates from a real install; also holds
  machine-specific absolute paths and auto-installed `node_modules`.
- `projects/`, `history.jsonl`, `shell-snapshots/`, `todos/`, `plans/` — per-device session
  state, not configuration.

## Compatibility

A `restore-setup` built against schema version N must refuse (with a clear message, not a
crash) a manifest whose `schemaVersion` is greater than N. It may choose to support reading
older manifests if the shape is compatible, but must say which version it's reading either
way.
