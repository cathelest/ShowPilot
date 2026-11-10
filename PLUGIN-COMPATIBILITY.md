# ShowPilot ↔ FPP plugin compatibility guide

For the maintainer and for Claude. Read this before any change where the
ShowPilot server and the FPP plugin (`ShowPilotFPP/ShowPilot-plugin`) have to
agree on something: a new field the plugin sends, a new answer the server
gives, or a feature that only works with a newer plugin.

## Why this matters

The server and the plugin are released separately, and owners update them
separately:

| Part | Where it lives | How owners get it | Version |
|---|---|---|---|
| ShowPilot server | this repo | Updates page (Stable / Master / Beta), Docker | `package.json` |
| FPP plugin | `ShowPilot-plugin` repo, `main` branch | FPP's Plugin Manager | `$PLUGIN_VERSION` in `version.php` (and `pluginInfo.json`) |
| Other show players | their own projects | their own updaters | `player-<version>` in the heartbeat |

So at any moment a server can be talking to an **older** plugin (owner hasn't
updated FPP, or the plugin change isn't released yet) or a **newer** one
(owner updated the plugin, server is on Stable). Both have to keep working.

What went wrong once, as the example to avoid: the FPP Monitor (server, Master
v0.33.278-master.15+) started reading `remoteStatus`, which only plugin
0.14.27 sends. 0.14.27 wasn't released yet, so every install had 0.14.26. The
server fell back to FPP's MultiSync `lastSeen`, which FPP only refreshes now
and then, and warned "*remote* (MultiSync remote) not seen recently" for
remotes that were working fine. No update banner appeared either, because
0.14.26 *was* the newest released plugin. The owner had no way to fix it.
Fixed in v0.33.280-master.2: unchecked remotes show as unknown, and the
update note waits until the plugin is released.

## What the server knows about the plugin

- `POST /api/plugin/heartbeat` carries `pluginVersion`; it's kept in
  `config.plugin_version` (`routes/plugin.js`) and passed to the FPP Monitor
  as `opts.pluginVersion`.
- `lib/show-player-update.js`:
  - `latestPlugin()`: newest **released** plugin, read from the plugin repo's
    `main/version.php` every 6 hours, never older than `BUNDLED_PLUGIN_LATEST`.
  - `PLUGIN_REQUIRED`: oldest plugin that works properly at all. Below it,
    the Dashboard and FPP show a "please update" banner.
  - `cmp(a, b)` compares versions.

## Rules

### 1. Newer data is optional, never assumed
Every field a newer plugin adds is optional on the server. Missing means "this
plugin doesn't send it", not "something is broken". Same the other way: the
plugin must ignore answer fields it doesn't know, and must not fail when a
field it expects from a newer server is missing.

### 2. Don't warn from a guess
If a feature needs data an older plugin doesn't send, show **no state** for
it (blank, "unknown", or the row without the extra stats), not a guessed one.
Only warn, alert or show "offline" when the data actually says so. A fallback
that is known to be unreliable (like MultiSync `lastSeen`) must not raise
warnings.

### 3. Tell the owner what an update would give them, but only if they can get it
When a feature needs plugin X and the owner has an older one:
- If X is released (`cmp(latestPlugin(), X) >= 0`): show one calm note where
  the feature is, e.g. "Update the ShowPilot plugin in FPP to vX or newer to
  see each remote's status." Not a red warning; the show is fine.
- If X isn't released yet: say nothing. Never point people at an update that
  FPP's Plugin Manager can't install.

### 4. Write down which plugin each feature needs
Next to the code, in the comment: `(plugin 0.14.27+, v0.33.278-master.15+)`.
Use the plugin version to decide (`cmp(opts.pluginVersion, '0.14.27') >= 0`)
together with whether the data is present, so a newer plugin that legitimately
sends nothing (e.g. no remotes) isn't mistaken for an old one.

### 5. Release order: plugin first
1. Release the plugin change (merged to the plugin repo's `main`).
2. Then the server change that uses it, as a Master update.
3. A server **release** (Stable) must not depend on an unreleased plugin.

If the server change has to ship first, rule 2 applies: with older plugins the
feature stays quiet.

### 6. Raise `PLUGIN_REQUIRED` only for real breakage
Raising it puts an update banner on every install below it, so only do it when
older plugins actually misbehave (wrong songs played, requests lost), never
just to unlock a new feature. Release the plugin first. Set
`PLUGIN_REQUIRED_WHY` to a plain-English reason.

### 7. Keep `BUNDLED_PLUGIN_LATEST` current
It's the fallback when the server can't reach GitHub. When a server change
depends on a newer plugin, raise it to the newest released plugin.

### 8. ShowPilot-Lite
Lite uses the same plugin and much of the same code; the same rules apply
when mirroring.

## Checklist for a change that touches both sides

- [ ] Server works with the **previous released** plugin: no errors, no false
      warnings, feature quiet or showing an update note (rules 2 and 3).
- [ ] Server works with the **new** plugin.
- [ ] New plugin works with the **current Stable** server (extra fields
      ignored, nothing it needs is missing).
- [ ] Code comment names the plugin version the feature needs (rule 4).
- [ ] Plugin released before the server change, or the server change is
      quiet without it (rule 5).
- [ ] PR body has a line: `Needs plugin vX.Y.Z+ for <feature>; with older
      plugins: <what the owner sees>.`
- [ ] `PRIMER.md` row mentions the plugin version needed.
