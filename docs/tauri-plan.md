# Tauri Plan: Agent Office als Tauri App

Branch: `TauriApp` — Stand: `origin/main@7f7211e`
Entscheidungen: **Node-Sidecar**, **Auto-Install Alles**, **Standard-Installer** (.msi/.exe, .dmg, .AppImage/.deb).

## 1. Ist

- `src/client` (Vite + three.js, Entries: main/lite/login/claim/join) + `src/server/server.ts` (Node 20+, http + ws auf 127.0.0.1:4600, Loopback Hook-Server, Unix-Socket `ptyhost` via `@lydell/node-pty`).
- Externe CLIs: `claude/codex/opencode/grok/muse/dsh/pi/cursor-agent`, dazu `git`, `gh`, `ss`/`lsof`, `ssh`.
- `install.sh/install.ps1` heute: macOS/Linux only (Windows nur WSL), alles manuell: Node 20+, git, gh, Agent-CLI.
- Architektur siehe `docs/how-it-works.md`, Modulregeln siehe `docs/code-layout.md`, Größenwächter `tests/size.test.ts`.

## 2. Ziel-Architektur (Sidecar, kein Rewrite)

```text
Tauri Window (Webview: dist/public) ──http/ws──▶ Node-Sidecar `agent-office-sidecar` (127.0.0.1:4600)
  ├─ Rust: Fenster, Tray, Updater, Autolaunch, Dialoge, Healthcheck, Single-Instance
  └─ Node-Sidecar: heutiger Server unverändert (server.ts + ptyhost.js + node-pty prebuilds)
```

- `src-tauri/tauri.conf.json`: `frontendDist: ../dist/public`, `devUrl: http://localhost:5173`, `sidecar bins/agent-office-sidecar`, `singleInstance`, Deep-Link `agent-office://join#…`.
- `src-tauri/Cargo.toml`: `tauri 2.x` + Plugins `shell http fs dialog notification opener clipboard-manager process os updater autolaunch single-instance window-state`.
- Pipeline: `build:client` → `build:server` → `node sidecar-pack.mjs` (Sidecar-Binary je Target-Triple nach `src-tauri/bins/`) → `tauri build`.
- `vite.config.ts` bleibt (proxy für `npm run dev`); prod ohne Hardcode auf `:4600` ausser via `TAURI_ENV`.
- Neue Features nur als eigene Module nach `docs/code-layout.md` (kein Ausbau von `main.ts`/`server.ts`/Store/`protocol.ts`).

## 3. Native GPU Power

- three.js bleibt WebGL, läuft in der Webview hardwarebeschleunigt.
- `windows.hardwareAcceleration: true`, `transparent: false`, `vsync: true`.
- Windows WebView2 (Chromium) → WebGL2 default. macOS WKWebView → WebGL2 default. Linux WebKitGTK 2.42+ → Standard, kein `WEBKIT_DISABLE_DMABUF_RENDERER=0` setzen.
- Kein WebGPU/wgpu in v1. Phase 2 optional: `three` `WebGPURenderer` mit WebGL-Fallback per Feature-Detect.
- Guard: `src/client/lab/*` Screenshot-Test pro PR.

## 4. Tauri Permissions (least privilege)

`src-tauri/capabilities/default.json`:

- `core:default`, `shell:allow-spawn-sidecar` (nur Sidecar), `shell:allow-open` (Join-Links).
- `http`: nur `http://127.0.0.1:4600/*`, `ws://127.0.0.1:4600/*`, `https://open-meteo.com/*`, `https://github.com/*`, `https://api.github.com/*`.
- `fs`: `appdata/document`, Scope `$APPDATA/agent-office/*` + `$HOME/agent-office/*` (floors.json, workers.json, scrollback).
- Externe CLIs laufen im Sidecar, nicht via Tauri-shell → keine breite `shell:execute`. Nur Onboarding-Installer braucht gescopte `shell:allow-run` für `winget/brew/apt/dnf/pacman/powershell`.
- `notification` (Beacon/Ding), `dialog` (Workspace-Picker), `clipboard-manager:write` (Invite-Links), `process:restart`, `os:os-type`, `updater`, `autolaunch`.
- Mic/Camera: Webview `getUserMedia`, zusätzlich macOS `NSMicrophoneUsageDescription` + `NSCameraUsageDescription`.
- Loopback bleibt: Hook-Server + PTY-Socket nur `127.0.0.1`/Unix-Socket (Win: Named Pipe, isolierter Patch in `ptys.ts`).

## 5. Easy Onboarding (ersetzt Terminal-Walkthrough)

Neues Modul `src/client/features/onboarding/` + `src-tauri/src/setup.rs`:

1. Willkommen → Workspace wählen (Tauri `dialog`, Default `~/agent-office`, Migration `~/.local/share/agent-office` erkennen).
2. Systemcheck (Rust `sidecar health` + `GET /api/health`): Sidecar, git, gh, gebundelte Node-Runtime, mind. 1 Agent-CLI.
3. Auto-Fix mit Confirm + Log:
   - macOS: `brew install git gh` + `brew install --cask claude-code` (Fallback `npm i -g opencode-ai`).
   - Windows: `winget install Git.Git GitHub.cli Anthropic.ClaudeCode` (kein WSL mehr).
   - Linux: `apt/dnf/pacman git gh` + `curl -fsSL claude.ai/install.sh|bash`; Hinweis WebKitGTK-Deps (`libwebkit2gtk-4.1`, `libayatana-appindicator3-1`).
4. `gh auth login --web` im eingebetteten Terminal (Sidecar-PTY, gestreamt).
5. Erstes Projekt: `gh repo list` → Pick → Clone-Fortschritt aus `.agent-office/clones/` wiederverwenden.
6. Fertig → intern `…/claim#…` öffnen, kein Copy-Paste. Alles überspringbar (wie Enter heute).

## 6. MultiOS

- CI `.github/workflows/tauri.yml`: `ubuntu-22.04 / windows-latest / macos-14-arm64 + macos-13-x64` → `.AppImage+.deb`, `.msi+nsis.exe` (WebView2-Bootstrapper), `.dmg+.app` (Notarize via `APPLE_*`).
- `node-pty` Prebuilds je Triple im Sidecar-Pack mitliefern, im Onboarding verifizieren.
- `AGENT_OFFICE_HOME` → Tauri `appDataDir` (Win `%APPDATA%/agent-office`, Mac `~/Library/Application Support/agent-office`, Linux `~/agent-office` Default).
- Deploy-Skripte (`deploy/*`, Tunnel, `provision.sh`) bleiben für Remote-Modus.

## 7. Funktionserhalt (Checkliste)

Worker/PTYs, Hooks/Status, Boards (Issues/PRs), Queue, Meetings, Changes/Diff/Commit/PR, Worktrees/Workspaces, Voice/WebRTC, Chat, Screenshare/TV, Whiteboard (Excalidraw), Jukebox/Rooftop, Golf/Spiele, Hund, Maps (Office/Castle/Station), `/lite`, Tunnel/Services-Relay, Accounts/Sign-ins, Tunnel-Header-Auth, Budget/Usage, Docs/Bookshelf, Jail/Airlock.

## 8. Phasen (kleine PRs je AGENTS.md)

1. Scaffold: `src-tauri/` + Pack-Skript + CI, kein App-Change. Verify `tauri build --debug` alle 3 OS.
2. Onboarding-UI: `features/onboarding/` + `setup.rs` (`check_tool`, `install_tool`, `pick_dir`), Doku hier + README.
3. Win-Pipe + Pfade: `ptys.ts` OS-Switch, `config.ts` Home-Dir.
4. Installer/Updater: Signatur, `tauri-plugin-updater` gegen GitHub Releases; `install.sh/ps1` als Thin-Wrapper.
5. Hardening: Capabilities final, CSP `tauri://`, Screenshot-Tests.

Verify je PR: `npm run typecheck`, `npm test`, `npm run build` (+ `cargo check`, Screenshot bei UI).

## 9. Risiken

- Alte Linux-Distros ohne GPU-WebKit → Fallback `/lite` automatisch anbieten.
- Sidecar-Größe 80–120 MB → via Updater differentiell ok.
- Win Named-Pipe-Patch + `size.test.ts` (neue Dateien <600 Zeilen) einplanen.
