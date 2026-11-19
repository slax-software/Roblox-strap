# RobloxStrap

A bootstrapper for Roblox, built as a standalone Electron app inside the
project. It keeps its own Roblox installation, applies FastFlags and mods
right before launch, stores Roblox accounts encrypted, and adds a real server browser plus
Discord Rich Presence.

> Not affiliated with Roblox Corporation. Roblox is a trademark of Roblox Corporation.
> FastFlags and client mods are unsupported by Roblox. Use at your own risk.

## Features

- **Managed Roblox installs** — downloads the current player build straight from Roblox's
  deployment mirrors (`setup.rbxcdn.com` and friends), verifies package MD5s, extracts
  with Windows `tar.exe`, and keeps downloaded packages cached.
- **FastFlags editor** — curated performance/graphics/window/network flags, one-click
  presets (Unlock FPS, Low-end mode, High fidelity, Clean visuals, Show FPS), plus custom
  flags. Writes `ClientSettings/ClientAppSettings.json` into the selected version.
- **Mods** — drop replacement files into `%LOCALAPPDATA%\RobloxStrap\Mods` using the same
  folder layout as a Roblox version. They are copied into the version before every launch.
- **Account manager** — add accounts by signing in through a Roblox login window or by
  pasting a `.ROBLOSECURITY` cookie. Cookies are encrypted with Windows DPAPI via Electron
  `safeStorage` and never leave the machine.
- **Direct game join** — mints an authentication ticket from the account cookie (CSRF retry
  flow) and launches Roblox with a browser-equivalent `roblox-player:` command line.
- **Server browser** — resolves place → universe, loads live public servers with player
  counts, ping and FPS, paginates, and can join the best server in one click.
- **Discord Rich Presence** — local Discord IPC client, no bot token. Shows the experience
  and a Join button while Roblox is running.
- **Integrations** — launch external tools alongside Roblox and optionally close them when
  Roblox exits.
- **Protocol handling** — optionally register RobloxStrap as the handler for `roblox://`
  and `roblox-player://`, remembering the previous handlers so they can be restored.
- **Windows integration** — tray icon, minimize-to-tray, start with Windows, single
  instance (links are forwarded to the running app), and an activity log.

## Requirements

- Windows 10/11 (x64)
- Node.js 18+ and the project's Electron dependency (already installed in the repo root)
- Internet access for Roblox deployment and API requests

The app uses only Windows built-ins for system integration: `reg.exe`, `tasklist.exe`,
`taskkill.exe`, `tar.exe`, and Electron's DPAPI-backed `safeStorage`.

## Run it

From the repository root:

```powershell
npm run strap        # normal launch
npm run strap-smoke  # headless smoke test, exits with 0/1
```

Or from this folder:

```powershell
npm install          # not needed when the root node_modules is present
npm start
npm run smoke
```

## First use

1. Open **Play** and press **Install Roblox** (or **Settings → Install / update**). This
   downloads a managed build into `%LOCALAPPDATA%\RobloxStrap\Versions`.
2. Add an account in **Accounts**. Signing in through the Roblox website is recommended;
   the cookie is captured automatically and encrypted.
3. Optional: tune **FastFlags**, drop files into **Mods**, and enable Discord Rich Presence
   in **Integrations**.
4. Press **Launch Roblox**, or use **Servers** to join a specific public server.

If you prefer to keep Roblox's own install, switch **Settings → Installation** to
**Use existing Roblox install**. In that mode Roblox updates itself, and RobloxStrap only
applies flags/mods and launches the newest installed player.

## Data locations

| Path | Purpose |
| --- | --- |
| `%LOCALAPPDATA%\RobloxStrap\Versions` | Managed Roblox versions |
| `%LOCALAPPDATA%\RobloxStrap\Downloads` | Cached Roblox packages (keyed by MD5) |
| `%LOCALAPPDATA%\RobloxStrap\Mods` | Mod files copied before launch |
| `%LOCALAPPDATA%\RobloxStrap\Logs\app.log` | Activity log |
| `%LOCALAPPDATA%\RobloxStrap\config.json` | Settings, flags, integrations, recents |
| `%LOCALAPPDATA%\RobloxStrap\state.json` | Last launch / installed build metadata |
| `%LOCALAPPDATA%\RobloxStrap\accounts.json` | Accounts with DPAPI-encrypted cookies |

Roblox's own install, when used, is read from `%LOCALAPPDATA%\Roblox\Versions`.

## How launching works

1. Resolve the latest build for the selected channel through
   `clientsettingscdn.roblox.com/v2/client-version/WindowsPlayer`.
2. Download/extract the managed build if needed (or reuse the existing install).
3. Apply mods and `ClientAppSettings.json`.
4. For a direct join, request `https://auth.roblox.com/v1/authentication-ticket` with the
   account cookie, retry once with the returned `x-csrf-token`, and build a
   `roblox-player:1+launchmode:play+gameinfo:…+placelauncherurl:…` URI.
5. Spawn `RobloxPlayerBeta.exe` with that URI as a single argument and watch the process
   to clear presence and close auto-close integrations.

For an incoming `roblox-player://` link, RobloxStrap forwards the original URI to the
client unchanged, so browser-initiated launches keep working.

## Limitations

- Windows only; system integration is registry/process based.
- Roblox's APIs are rate-limited. Server browsing can return HTTP 429; the UI reports it
  and lets you retry.
- Roblox may re-register itself as the `roblox-player` handler after its own updates.
  RobloxStrap re-asserts the handler on startup when protocol handling is enabled.
- Mods replace client files. Roblox updates overwrite them — keep backups of originals.
- Account launching depends on Roblox's private authentication-ticket endpoint. If Roblox
  changes it, direct join will need updating; launching the Roblox home screen still works.
- Development builds register `electron.exe` plus the project path as the protocol handler.
  Package the app for a clean permanent registration.

## Packaging

```powershell
npm run strap-dist   # from the repo root
# or, inside roblox-strap/
npm run dist
```

Uses `electron-builder` with an NSIS target and the generated `build/icon.ico`. The build
includes `main.js`, `preload.js`, `lib/`, `renderer/`, `assets/` and `package.json`.

The result lands in `roblox-strap/dist/`:

- `RobloxStrap Setup 1.0.0.exe` — the installer (per-user, pick your install folder, creates
  Desktop and Start Menu shortcuts)
- `win-unpacked/` — the same build unpacked, useful for testing without installing

## Layout

```
roblox-strap/
  main.js              Electron main process, IPC, install/launch orchestration, smoke test
  preload.js           contextBridge API
  lib/
    paths.js           app data locations
    util.js            HTTP, downloads, zip extraction, process helpers
    deployment.js      Roblox version/manifest/install management
    launch.js          roblox-player URI parsing/building and spawning
    fastflags.js       flag catalog, presets, apply/read/reset
    accounts.js        encrypted account vault and ticket flow
    servers.js         place info, public server list, best-server picker
    mods.js            mods folder scan and copy
    discord.js         minimal Discord IPC rich-presence client
  renderer/            UI (vanilla HTML/CSS/JS, monochrome black & white theme)
  scripts/icon.js      icon generator
  assets/, build/      app and installer icons
```

## Smoke test

`npm run strap-smoke` runs without opening a window and checks:

- `roblox-player:` URI parse/build round trips and place-link parsing
- FastFlag apply/read/remove against a scratch directory and preset merging
- DPAPI cookie encrypt/decrypt round trip
- Roblox install detection, live latest-version and manifest fetches
- Place info lookup (network dependent)
- Protocol registry read and mods-folder scan

It exits non-zero if a check fails. Network checks may fail when Roblox rate-limits the
machine; rerun later in that case.
