# Tauri Plan: Agent Office als Tauri App

Branch: `TauriApp`

Ausgangsbasis: `origin/main` zum Zeitpunkt der ersten Planung (`7f7211e`).

Festgelegte Richtung: **Tauri 2 + bestehender Node-Server**, **geführtes Setup mit Auto-Install benötigter Tools nach Zustimmung**, **Windows/macOS/Linux**.

## Prüf-Ergebnis und Korrekturen

Die erste Fassung enthielt konkrete technische Annahmen, die vor der Umsetzung nicht belegt waren. Der Plan behandelt sie nun als Proof-of-Concept- bzw. Abnahmepunkte:

- **Windows ist bereits unterstützt.** `install.ps1` unterstützt natives Windows, `workers/process.ts` wählt `cmd.exe`, und `ptys.ts` nutzt dort direkte PTYs. Die getrennte PTY-Host-Persistenz wird in `ptys.ts` ausdrücklich nur auf Nicht-Windows gestartet. `install.sh` allein war keine ausreichende Basis für die Plattform-Aussage.
- **Nicht alle bestehenden Funktionen sind auf jedem OS gleichwertig.** Der Service-Port-Scanner hat Linux- (`ss`) und macOS- (`lsof`) Pfade, aber keinen Windows-Pfad. Der Windows-Support muss deshalb mit einer Feature-Matrix und konkreten Paritätsentscheidungen geplant werden.
- **Tauri bietet keine allgemeine `hardwareAcceleration`- oder `vsync`-Option in `tauri.conf.json`.** `transparent` ist eine gültige Window-Option, steuert aber Transparenz und nicht GPU-Beschleunigung. Die GPU-Ausführung kommt vom nativen WebView und hängt von Betriebssystem, Treiber und WebView-Version ab.
- **`pkg`/`nexe` plus `node-pty` ist kein gesicherter Packaging-Weg.** `node-pty` enthält native, zielabhängige Addons; Packaging, Ressourcenauflösung und Laufzeit-ABI müssen zuerst für alle Ziel-Triples bewiesen werden.
- **Die eingebettete Tauri-Origin passt nicht unverändert zum bestehenden Server.** HTTP-Authentifizierung und WebSocket-Upgrades prüfen Same-Origin. Eine Tauri-Origin mit API-Aufrufen an Loopback benötigt entweder eine eng begrenzte Origin/CORS-Anpassung samt CSP oder einen anderen UI-Transport. Beides ist ein früher Architektur-Spike.
- **Tauri-Capabilities sind keine Sandbox für den Sidecar.** Sie begrenzen Frontend-Zugriffe auf Tauri-Commands/Plugins. Der Node-Prozess läuft als angemeldeter OS-User und braucht für Git, Worktrees, Agenten und Dateien dessen normale Rechte.
- **Installationsbefehle und Paketnamen sind nicht universell.** Keine ungeprüften `curl | bash`- oder beliebigen Shell-Kommandos aus der UI; Agent-CLIs einzeln, opt-in und nur über unterstützte offizielle Wege behandeln.
- **Datenpfade und Links müssen bestehende Semantik erhalten.** Das Office nutzt `~/agent-office` bzw. `AGENT_OFFICE_HOME`, Projekt-Floors und `.agent-office`-Daten in den Checkouts. Der Start-Link ist aktuell `/login#key=…`; `/claim` und `/join#…` erfüllen andere Zwecke. Kein stilles Verschieben oder Umdeuten dieser Daten/Links.

## 1. Bestand und Ziel

Bestand laut `docs/how-it-works.md`, `docs/code-layout.md`, `install.sh`, `install.ps1` und Servercode:

- Frontend: `src/client`, Vite, three.js, plus `/lite`, Login-, Claim- und Join-Seiten.
- Backend: Node.js-HTTP- und WebSocket-Server auf Loopback; Static Assets werden bereits vom Server ausgeliefert. Worker starten über `@lydell/node-pty`; auf macOS/Linux kann ein getrennter `ptyhost` die Terminals über einen Server-Neustart retten, unter Windows derzeit nicht.
- Lokale Integrationen: Agent-CLIs, `git`, `gh`, SSH-Tunnel sowie betriebssystemspezifische Prozess-/Port-Erkennung. Worker dürfen absichtlich als aktueller OS-User Code ausführen.
- Funktionen umfassen lokale Einzelplatz-Nutzung und den bisherigen Browser-/Remote-Office-Betrieb. Die Desktop-App ersetzt die vorhandenen CLI-, Browser-, Server- und Deployment-Modi nicht.

Ziel: Tauri ist Desktop-Shell und native Integrationsschicht; bestehende Client- und Servermodule bleiben Feature-Quelle. Kein kompletter Rust-Rewrite und kein zweiter, abweichender Server.

## 2. Architektur-Gates vor dem Scaffold

### Gate A: Node-Sidecar mit nativen Abhängigkeiten

Tauri bindet ein ausführbares Sidecar über `bundle.externalBin` und target-spezifische Namen mit Rust-Target-Triple ein. Für diese App muss die Build-Variante zusätzlich `dist/server`, Produktionsabhängigkeiten, Frontend-Assets, gebündelten Node-Runtime-Code und das `node-pty`-Native-Addon korrekt finden.

Vor Festlegung der Packaging-Technik einen minimalen Prototyp bauen und aus einem installierten Test-Bundle starten:

- `@lydell/node-pty` lädt und startet mindestens eine Shell auf Windows, macOS Intel/Apple Silicon und Linux x64; weitere Architekturen erst nach nachgewiesenem Prebuild.
- Node-/Addon-ABI, `node_modules`-Auflösung, Pfade mit Leerzeichen/Unicode, Signalweitergabe, Logs, Shutdown und App-Update funktionieren.
- Monolithisches `pkg`/`nexe` ist nur eine Option, wenn dieser Test mit dem nativen PTY-Addon gelingt. Sonst Node-Runtime und App-Dateien als getrennte Sidecar-Ressourcen bündeln. Tauri dokumentiert beide Muster.
- Der Sidecar erhält nötige App-Argumente und Umgebungsvariablen nur aus Rust; kein beliebiges Kommando oder Argument an eine Frontend-Shell freigeben.

Referenz: [Tauri Sidecars](https://v2.tauri.app/develop/sidecar/), [Node.js als Sidecar](https://v2.tauri.app/learn/sidecar-nodejs/).

### Gate B: Frontend-Origin, HTTP, WebSocket und Auth

Ein Tauri-Build mit gebündeltem Frontend hat eine Tauri-eigene Origin; der Office-Server bindet aktuell Loopback, verwendet Session-Cookies und prüft bei WebSocket-Upgrades `sameOrigin`. Vor dem Scaffold mit bestehendem Login, WebSocket, Cookie, Voice-Signaling, Assets und Service-Relay einen durchgängigen Prototyp testen.

- Möglichkeit 1: gebündeltes Frontend behält Tauri-Origin; Server erlaubt ausschließlich die exakt benötigte Desktop-Origin, mit credentials, passender CSP `connect-src` für Loopback HTTP/WS und explizitem WS-Origin-Check. Keine `*`-CORS-Regel.
- Möglichkeit 2: Frontend kommt vom Office-Server auf Loopback. Dafür braucht es eine abgesicherte Server-Origin/Port-Startsequenz, sichere Tauri-IPC-Regeln und Schutz gegen fremde lokale Webseiten. Diese Variante nicht allein wegen einfacherer Same-Origin-Anfragen wählen.
- Tauri-`http`-Plugin ist kein Ersatz für Server-CORS, WebSocket-Origin-Prüfung oder CSP; nur aufnehmen, wenn die App tatsächlich dessen native Fetch-API benötigt.
- Bestätigen, dass die gewählte Origin als Secure Context für `getUserMedia`/`getDisplayMedia` funktioniert. Das bestehende `/login#key=…`-Verfahren bleibt der Ausgangspunkt für den einmaligen Desktop-Login.

Abnahmekriterium: Browser und Desktop benutzen dieselben Authentifizierungs-, HTTP-Routen und WebSocket-Protokolle; es entsteht keine alternative, schwächer geschützte Desktop-API.

## 3. Zielaufteilung

```text
Tauri 2 Desktop-Shell
  ├─ Rust: Sidecar-Lifecycle, Setup-Commands, Dateiauswahl, Deep Links, Tray/Updater nach Bedarf
  ├─ WebView: bestehendes Frontend und native WebView-GPU-Beschleunigung
  └─ Node-Sidecar: bestehender Server, Worker, PTYs, Hooks, Git/GitHub und Agent-CLIs
```

- `src-tauri/`: Tauri-Konfiguration, Fähigkeiten/Permissions, Rust-Lifecycle und eng begrenzte Setup-Commands.
- `bundle.externalBin`: Sidecar je unterstütztem Target-Triple. `frontendDist` oder Loopback-UI erst nach Gate B festlegen; nicht beide Transports ungeprüft vermischen.
- Vite-Devserver und bestehender Browser-/CLI-Start bleiben erhalten. Produktionskonfiguration verwendet keine Entwicklungs-Proxy-Annahmen.
- Nur echte Desktop-Integrationen kommen hinzu; Geschäftsfunktionen bleiben in ihren bisherigen Feature-Modulen und Server-Registries.
- Tauri-Plugins bewusst klein halten. Dialog, Updater, Single-Instance, Deep Link, Notification, Opener, Clipboard, Autostart und Window-State nur einbinden, wenn ein definierter Nutzerfluss sie braucht. Kein pauschales Plugin-Bündel.

## 4. GPU und Rendering

- three.js nutzt zunächst den bestehenden `WebGLRenderer`. In Tauri rendert die native Plattform-WebView; das ist GPU-beschleunigtes Web-Rendering, aber kein eigener Rust-`wgpu`-Renderer und keine Garantie, dass jeder Treiber Hardwarebeschleunigung liefert.
- Keine nicht existierenden Tauri-Window-Felder wie `hardwareAcceleration` oder `vsync` konfigurieren; `transparent` nur bei echtem UI-Bedarf setzen. Für jeden unterstützten WebView konkrete Version, WebGL2-Verfügbarkeit, Renderer und Software-Fallback testen.
- Linux ist besonders variabel: Tauri benötigt WebKitGTK 4.1-Entwicklungs-/Laufzeitabhängigkeiten; GPU/DMABUF-Verhalten richtet sich nach WebKitGTK, Treiber und Desktop-Stack. Mindestversion erst aus Tests festlegen, keine pauschale „2.42+“-Garantie.
- Diagnoseansicht protokolliert WebGL2/Renderer-Information und erklärt Software-Rendering. `/lite` bleibt erreichbarer Fallback, falls die 3D-Ansicht nicht startet.
- WebGPU oder ein Rust-`wgpu`-Renderer ist eine separate Machbarkeitsstudie; kein v1-Versprechen und kein direkter Drop-in für die bestehende three.js-Szene.
- GPU-Abnahme: identischer Szenen-/Interaktionstest auf Windows WebView2, macOS WKWebView, Linux WebKitGTK; headless Screenshot allein belegt keine GPU-Beschleunigung.

## 5. Tauri-Sicherheit und Permissions

Tauri v2 verwendet `src-tauri/capabilities/*.json` und plugin-spezifische ACL-Permissions. Die exakten Permission-Identifier und Scopes sind gegen die generierten Schemas der ausgewählten Plugin-Versionen zu validieren; Capability-Namen sind nicht frei erfundene globale Rechte.

- Frontend bekommt nur benötigte Core-/Plugin-Commands. Rust-Commands sind typisiert und prüfen Inputs, Pfade, Deep Links und Zustände selbst.
- Sidecar-Start erfolgt aus Rust über `ShellExt::sidecar` bzw. eine gleichwertige Backend-Lifecycle-Integration. Die Capability erlaubt kein generisches Ausführen von `git`, `gh`, `claude`, `npm`, Paketmanagern oder beliebiger Shell-Kommandos aus JavaScript.
- Keine `shell:allow-execute`-Freigabe für System-Shell oder beliebige Arguments. Installer-Aufrufe laufen in Rust mit festem Executable und getrennt validierten Argumenten; Elevation wird nie stillschweigend angefordert.
- `fs`-Plugin nur für konkrete Frontend-Dateizugriffe und eng begrenzte Scopes. Der Node-Sidecar wird dadurch nicht eingeschränkt. Rust-/Node-Dateizugriffe prüfen zusätzlich kanonische Pfade, Symlinks und gewünschte Schreiborte.
- Sidecar bindet ausschließlich Loopback, nutzt nach Möglichkeit einen nicht vorhersagbaren Port bzw. authentisierte lokale Verbindung und erhält einen klaren Shutdown. Origin-/CSRF-Schutz bleibt aktiv. Ports, Cookies, claim tokens und Logs dürfen keine Geheimnisse offenlegen.
- Sidecar ist ein mächtiger lokaler Prozess: ein Office-Mitglied kann über Agenten Code als OS-User starten. Die bestehende Vertrauensgrenze und README-Sicherheitswarnung bleiben bestehen; Tauri-ACLs ändern diese Semantik nicht.
- Keine fest angenommene Tauri-`http`-Scope-Liste für normale WebView-`fetch`-Aufrufe: Plugin-Scopes kontrollieren Plugin-Requests, nicht CSP oder Server-CORS.
- Mikrophon/Kamera/Bildschirmfreigabe werden separat auf Plattform- und WebView-Ebene geprüft. macOS benötigt passende Usage-Descriptions/Privacy-Freigaben; Screen Recording/`getDisplayMedia`, Windows Capture und Linux PipeWire/Portal-Verhalten sind eigene Abnahmepunkte, nicht bloß Tauri-Capabilities.
- CSP bleibt restriktiv, erlaubt nur tatsächlich verwendete Loopback- und Dienst-Origins. Tauri IPC nicht für entfernte Inhalte oder nicht vertrauenswürdige Fenster aktivieren.

Referenzen: [Tauri Capabilities](https://v2.tauri.app/security/capabilities/), [Sidecars](https://v2.tauri.app/develop/sidecar/), [Updater](https://v2.tauri.app/plugin/updater/).

## 6. Onboarding und Setup

Ein geführter UI-Flow macht Voraussetzungen verständlich, kann sie prüfen und unterstützt Nutzer bei der Installation. „Automatisch“ bedeutet nicht unbeaufsichtigt, mit Adminrechten oder dass alle Agenten installiert werden.

1. **Willkommen und Speicherorte:** vorhandene Office-Daten sowie Projekt-Checkouts erkennen; vorhandene `AGENT_OFFICE_HOME`-/`~/agent-office`-Semantik standardmäßig beibehalten. Optional einen Workspace über Native Dialog wählen. Vor Migration Backup/Bestätigung, nie bestehende Floordaten ungefragt verschieben.
2. **Systemcheck:** Sidecar startet; `git`, `gh` und Agent-CLIs werden einzeln erkannt; Authentifizierung/Versionen und OS-Kompatibilität werden angezeigt. Gebündeltes Node bedeutet, dass Node für die App selbst nicht vorausgesetzt wird.
3. **Fehlende Werkzeuge:** Nutzer wählt mindestens einen unterstützten Agenten. Git und GitHub CLI sowie genau diese Agent-CLI optional installieren. Erst Paketmanager/Installationsmethode erkennen, Anbieterquelle und Version anzeigen, explizit bestätigen lassen, Ausgabe/Fehler zeigen und abbrechen/erneut versuchen erlauben.
4. **Sichere Installationswege:** bevorzugt OS-Paketmanager oder dokumentierte Anbieter-Installer mit Signatur/Hash-Prüfung. Kein `curl | bash`, PowerShell-`iex`, unvalidiertes `npm install -g` oder unaufgefordertes `sudo`. Wenn Elevation/System-Pakete erforderlich sind, klare manuelle Schritte oder OS-Installer statt stiller Privileg-Ausweitung.
5. **Plattformmatrix statt Einheitsbefehlen:** Homebrew (macOS), WinGet (Windows), erkannte Linux-Distribution/Paketmanager. Keine Distribution oder kein Manager? Links/Anleitung anbieten, Installation überspringbar lassen. Nicht alle Provider/Agenten versprechen native Unterstützung auf allen Systemen.
6. **GitHub-Login:** vorhandenen Gerätecode-/Sign-in-Flow wiederverwenden; Authentifizierung bleibt beim `gh`-Tool. Tokens nicht in Tauri-Logs, Setup-Events oder UI-State persistieren.
7. **Erstes Projekt:** bestehende `gh`-Repo-Auswahl und Clone-Logik wiederverwenden, Clonefortschritt anzeigen; vorhandene Floors/Checkouts werden wiedererkannt.
8. **Fertig:** Office über bestehenden sicheren Einmal-Login öffnen. Der aktuelle Start-Link ist `/login#key=…`; Account-Join-Links nutzen `/join#…`. Deep Link `agent-office://` ist optional und separat zu testen, inklusive Validierung nicht vertrauenswürdiger Eingaben und Single-Instance-Weiterleitung.

Kein Tauri-Shell-Paketmanagerrecht pauschal vergeben. Falls Rust Installer-Commands implementiert, nur OS-erkannte Executables und Argumente erlauben, Ausgabe begrenzen und User-Confirm/Cancel vorsehen. Setup bleibt überspringbar und vorhandenes Terminal-/CLI-Setup bleibt verfügbar.

## 7. Plattform- und Feature-Matrix

Vor breiter Umsetzung pro Feature (`voll`, `eingeschränkt`, `nicht unterstützt`) und OS abnehmen:

| Bereich | Linux | macOS | Windows |
| --- | --- | --- | --- |
| Tauri/WebView/GPU | WebKitGTK 4.1+, distro-/treiberabhängig | WKWebView | WebView2 Runtime |
| PTY | `node-pty`; Host-Persistenz separat testen | `node-pty`; Host-Persistenz separat testen | `node-pty`/ConPTY; derzeit kein persistenter `ptyhost` |
| GitHub/Agent-CLIs | je Anbieter/Distribution prüfen | je Anbieter prüfen | je Anbieter/CLI prüfen; `cmd.exe`-Argumente/PATHEXT testen |
| Services-Erkennung | bestehender `ss`-Pfad | bestehender `lsof`-Pfad | noch kein bestehender Scanner; Windows-Implementierung oder ausdrücklich eingeschränkte Funktion |
| Voice/Kamera/Screen Share | WebKitGTK + PipeWire/X11/Compositor testen | WKWebView + OS-Datenschutzfreigaben testen | WebView2 + Windows-Aufnahmefluss testen |
| Paketformate | AppImage + deb; rpm nur bei getesteter Pipeline | dmg/app | NSIS exe + MSI |

- Erster Release unterstützt nur getestete Architekturen. `node-pty` muss für jedes Target-Triple, insbesondere macOS arm64 und ggf. Linux arm64, verfügbar sein.
- Linux AppImage beseitigt nicht alle Laufzeitvoraussetzungen: WebKitGTK 4.1 und weitere native Bibliotheken/Mindest-GLIBC dokumentieren. Build-Baseline an ältester unterstützter Umgebung ausrichten. `.deb` allein deckt nicht Fedora/openSUSE ab; RPM erst mit CI und Runtime-Tests zusagen.
- Windows-WebView2 Installationsmodus/Offline-Szenario und macOS Code-Signing/Notarisierung vor Release festlegen. Tauri-Updater benötigt signierte Artefakte, Public Key und sichere Verwaltung des privaten Update-Schlüssels.
- Bestehende `install.sh`, `install.ps1`, Browserbetrieb, CLI und Cloud-Deployment bleiben unterstützt, bis explizite Migration und Parität nachgewiesen sind.
- Beibehaltene Daten: keine pauschale Verschiebung in `appDataDir`. Desktop-Konfiguration, Office-Home, Floors und Projekt-Checkouts haben getrennte Rollen; Pfadkonzept samt Upgrade/Migration testen.

## 8. Funktionserhalt und Regressionen

Für jeden Punkt existiert vor Freigabe ein Regressionstest oder ein dokumentierter manueller Abnahmeschritt:

- 3D Office und `/lite`; alle Maps, Floors, Elevator und lokale Einstellungen.
- Worker aller unterstützten Provider, PTY-Interaktion/Resize/ANSI, Resume, Status-/Hooks und Sign-ins.
- Projekte, GitHub Issues/PRs, Queue, Meetings, Worktrees/Workspace, Diff/Commit/PR, Clonefortschritt und Cleanup.
- Accounts/Rollen/Invites, einmaliger Login, Credentials-Isolation im bestehenden Umfang, HTTP- und WebSocket-Auth.
- Chat, Voice, Screen Sharing/TV, Excalidraw-Whiteboard, Docs/Bookshelf, Bildproxy und Service-Relay/Tunnel.
- Jukebox, Rooftop, Hund, Golf/Arcade, Castle/Station-Verhalten, Usage/Budget, Notifications und Sound.
- Browser-/Server-/Cloud-Deployment und CLI-Funktionen bleiben unangetastet und werden nicht vom Tauri-Prozess vorausgesetzt.
- Unterschiede für Windows (PTY-Restart-Persistenz, Service-Erkennung) werden ausdrücklich entschieden; „Funktionserhalt“ nicht mit nicht vorhandener heutiger Plattformparität gleichsetzen.

## 9. Umsetzungsphasen und Gates

1. **Feasibility-Spike:** Tauri-Window + echter bestehender Server + Login/HTTP/WS + ein `node-pty`-Worker + GPU/WebRTC-Test auf je einem Runner pro OS. Architecture Gate A/B schließen; Sidecar und UI-Origin erst danach festlegen.
2. **Tauri-Shell:** minimale App, Sidecar-Start/Healthcheck/Shutdown, Ressourcenpfade, Logging, Single-Instance. Keine pauschalen Plugins. Browser-/CLI-Run bleibt unverändert.
3. **Desktop-Onboarding:** Feature-Modul nach `docs/code-layout.md`, typisierte Rust-Commands oder bestehende Server-Endpunkte, Tool-Check, Workspace-Auswahl und optionaler, nachvollziehbarer Installationsflow. README und passende Setup-Doku im selben PR aktualisieren.
4. **Plattform-Parität:** Windows-Servicescanner entscheiden/implementieren; PTY-Verhalten und Worker-Survival dokumentieren und testen; OS-spezifische Pfade, Prozesssignale, Login und Native Dialoge prüfen.
5. **Packaging:** reproduzierbare CI-Matrix für getestete Triples, NSIS/MSI, dmg, AppImage/deb (RPM später falls gewünscht), Signierung, macOS-Notarisierung und WebView-Runtime-Voraussetzungen.
6. **Updater/Hardening:** erst mit Release-/Versionsstrategie; signierte Update-Artefakte und Schlüsselrotation/Backup, restriktive CSP/Capabilities, Deep-Link-Validierung, gespeicherte Datenmigration und Recovery.

Pro PR gemäß `AGENTS.md`: `npm run typecheck`, `npm test`, `npm run build`, `tests/size.test.ts` (in `npm test`), dazu `cargo check`/Tauri Build und betroffene Plattformtests. UI-Änderungen mit reproduzierbarem WebView-Screenshot/Smoke-Test; ein Headless-WebGL-Screenshot ist kein GPU-Nachweis.

## 10. Tauri-Konfiguration und Rechte konkretisieren

Während der Umsetzung anhand der generierten Schemas festlegen, nicht vorab als erfundene Permission-Liste festschreiben:

- Capability pro benötigtem Window und Plattform; nur verwendete Commands freigeben.
- Sidecar-Spawn: wenn Rust den Lebenszyklus kontrolliert, nur Backend-seitig starten. Wenn Frontend-Spawn zwingend ist, Shell-Scope exakt auf Sidecar und erforderliche Argumente beschränken; kein `args: true`.
- Dialog-/Opener-/Clipboard-/Notification-/Deep-Link-/Updater-Rechte jeweils für konkrete UI-Aktion.
- Kein `fs:default` ohne Bedarf; Workspace-Picker gibt Rust einen User-ausgewählten Pfad, der serverseitig validiert wird.
- CSP-`connect-src`, Server-CORS, Cookies, WebSocket-Origin, Loopback-Port und Secure Context gemeinsam testen; keine vermeintliche Tauri-HTTP-Allowlist als Ersatz verwenden.
- macOS `Info.plist`-Privacy-Strings und reale Promptflüsse, Windows WebView2, Linux WebKitGTK-/Portal-Abhängigkeiten in plattformspezifischen Konfigurationen dokumentieren.
