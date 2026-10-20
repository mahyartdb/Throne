# Throne — Clean-Room Design

Source: clean-room discovery of the Throne desktop application (a desktop GUI managing a sibling proxy-core process). The original was inspected only on the dirty side; this document carries no original code, comments, internal names, or file structure. External contract strings (binaries, flags, paths, URLs, error text, wire fields) are reproduced exactly on purpose.
Version pinned: `d3fa5e33ad9b0fe4bd06fa0ad7f5bec5cbb49144` (branch `stable`), captured 2026-10-16.
Scope: the whole product — desktop GUI + sibling core process; Windows, Linux, macOS. Embedded proxy engines themselves are external dependencies (see Out of Scope).

Conventions: each area section has **Interface** (external contracts), **Behavior**, **Edge cases & errors**, **State & config** (defaults). `UNKNOWN` marks items that need runtime verification; they are not silent guesses. "Core" = the sibling `ThroneCore` process; "GUI" = the desktop application. Acceptance fixtures (FIX-*) are at the end.

---

## A. Application lifecycle & global infrastructure

### A.1 Interface

**Binaries and layout**
- GUI executable: `Throne` / `Throne.exe`. Distributed mainly as a portable ZIP; no installer step inside the app.
- Core executable: `ThroneCore` / `ThroneCore.exe`, expected next to the GUI binary; only needed when a profile is started.
- Update helper: `updater` / `updater.exe` next to the executable; optional. Its presence gates the "Check for update" menu action.
- Translations: `<exeDir>/translations/<locale>.qm` override embedded translations when present.
- Linux packaged builds add `<exeDir>/usr/plugins` to the library search path.

**GUI command line** — unknown flags are ignored silently (no usage output):

| Arg | Meaning |
|---|---|
| `-appdata` | Switch to per-user config dir. Default per-user dirs: Windows `%LOCALAPPDATA%\Throne`, Linux `~/.config/Throne`, macOS `~/Library/Preferences/Throne`. If the next token does not start with `-`, it is used as the config root instead (`-appdata D:\Foo` → data in `D:\Foo\config`). Only the first `-appdata` is honored. |
| `-many` | Legacy flag, stored but unused (no-op). |
| `-tray` | Start hidden to tray (same effect as the start-minimal setting). Removed from args on self-restart. |
| `-debug` | Legacy flag, stored but unused (no-op). Debug builds force it on. |
| `-flag_restart_tun_on` | Internal: set by the app when restarting itself to re-enable TUN after the core is up; stripped from args and re-appended by the restarter. |
| `-flag_restart_dns_set` | Internal: re-apply system DNS after a restart. |
| `throne://…` positional | Deeplink; the first matching arg wins, extra ones are ignored; delivered once the window exists. |
| file path / `file://` positional | Config import ("open with"). All such args are collected; relative paths resolve against the launch CWD (before CWD change); non-existent paths and directories are ignored; tokens starting with `-` are never treated as files. |

- Some release packages are built to always behave as if `-appdata` was given (macOS packaging does). In such builds the "Update" action is replaced by "Open in browser"/"Close".

**Single-instance arbitration**
- Endpoint name is derived deterministically from the absolute config-root path, so installs sharing a config root share one instance and different roots do not collide.
- Second instance: connects, waits up to 250 ms; on success sends any payload (one item per line: `throne://` URLs and/or percent-encoded `file://` URLs; no payload = plain wake-up), exits 0. The primary processes each line as deeplink/file import and raises its window.
- The listener is world-accessible so a non-elevated second instance can wake an elevated primary.
- Primary's in-app log is not touched by second-instance starts (the second instance exits before any log/DB work).

**Config root resolution** (verified order)
1. Default: the executable's directory ("portable").
2. Fall back to the per-user config dir when the install dir is not writable, or `<install>/config` is not writable, or `<install>/config/throne.db` exists but cannot be opened read-write. Writability is probed by creating/removing `<install>/config/.throne-write-test`. On fallback, an existing `<install>/config/throne.db` is copied recursively (existing targets skipped) into the user `config/`, and a marker `<userConfig>/config/.install-dir-unwritable` is written containing the install root path.
3. While the marker content equals the current install root, the user dir stays in use even if the install dir becomes writable again (prevents split DBs after an elevated run). If the install moved, the decision is re-evaluated (marker re-written on fallback).
4. `-appdata` forces the user config dir, or the explicit dir given (relative paths resolve against the install dir).
- Afterwards the process CWD becomes `<configRoot>/config`; relative paths land there.

**Created filesystem layout** (relative to config root):

| Path | Content / lifetime |
|---|---|
| `config/throne.db` (+ `-wal`, `-shm`) | Main SQLite DB: profiles, groups, routes, settings, ids. WAL mode, foreign keys on, synchronous=NORMAL, 64 MiB mmap, 5 s busy timeout. Created on first run; seeds a group named `Default` (localized) and a default route chain when those tables are empty. |
| `config/throne_stats.db` (+ `-wal`, `-shm`) | Separate traffic-statistics DB, same directory as the main DB. |
| `config/logs/throne.log` | Diagnostic log; truncated each session. |
| `config/logs/throne.log.1`…`.3` | Rotated logs (newest `.1`). |
| `config/logs/running.marker` | Present while running; content `pid=<pid> version=<version> started=<ISO-8601 local time>` + newline; removed on clean shutdown. |
| `config/logs/crashed-YYYYMMDD-HHmmss.log` | Previous session's log, preserved when the marker was found at startup. |
| `config/crashes/Throne_<version>_<arch>_<stamp>.dmp` / `.txt` | Windows crash dump + text sidecar (`arch` ∈ `amd64`,`arm64`,`386`,`unknown`). |
| `config/dashboard/index.html` | Only when `config/dashboard` did not exist and could be created: bundled notice page telling users to put clash dashboard files there. |
| `config/icons/` | Created if missing (custom icon PNGs; file names `Dns.png, Off.png, Proxy-Dns.png, Proxy.png, Throne.png, Tun.png`). |
| `config/temp/` | Deleted recursively at every startup. |
| `<basePath>/Throne.zip`, `geoip.dat`, `geosite.dat` | Download targets. Base path = executable dir in portable mode, per-user config dir in appdata mode. Note: with `-appdata <dir>`, downloads and geo files still go to the default per-user dir, not `<dir>`. `geoip.dat`/`geosite.dat` are exported to the core as `XRAY_LOCATION_ASSET`. |
| Executable dir extras | `updater.old` (transient during self-update; deleted on next start). |

- Directory permissions use the default umask (no explicit chmod). Root for the core is obtained only by making the core binary root-owned + setuid (Linux via `pkexec`; macOS `root:wheel` + `u+s` via sudo, and the setuid bit is checked before considering escalation satisfied). The GUI itself is never setuid.

**Hard-coded external constants**
- Update API: `https://api.github.com/repos/throneproj/Throne/releases`.
- Release asset naming: `Throne-<tag>-<platform>.zip` with platform ∈ {`windows64`, `windows-arm64`, `windows32`, `windowslegacy64`, `linux-amd64`, `linux-arm64`, `macos-amd64`, `macos-arm64`, `macoslegacy-amd64`}.
- Downloaded update file: `Throne.zip` in the base path.
- Default latency test URL `http://cp.cloudflare.com/`; default simple download URL `http://cachefly.cachefly.net/1mb.test`; country lookup `http://ip-api.com/json/`.
- DNS defaults in generated configs: remote `https://8.8.8.8/dns-query`, direct `localhost`.
- Geo asset URLs: `https://github.com/Loyalsoldier/v2ray-rules-dat/raw/release/geoip.dat` and `…/geosite.dat`.
- Adblock ruleset URL: `https://raw.githubusercontent.com/217heidai/adblockfilters/main/rules/adblocksingbox.srs`.
- Route-profile repo base: `https://raw.githubusercontent.com/throneproj/routeprofiles/…`.
- Mirror rewriting for `raw.githubusercontent.com`: `gcore|quantil|fastly|cdn|testingcf.jsdelivr.net/gh/<user>/<repo>@<branch>/…`; mirror setting values: GITHUB (passthrough), CLOUDFLARE (`testingcf…`, default), GCORE, QUANTIL, FASTLY, CDN.
- Dashboard help links: `https://github.com/MetaCubeX/Yacd-meta/archive/refs/heads/gh-pages.zip`, `http://yacd.metacubex.one/`.
- Windows version thresholds: 10240 (Tun stack/route defaults, 1507), 17763 (legacy64 assets, 1809), 22621 (emoji font, 11 22H2).
- Shared enums/values: no-profile id `-1919`; warp profile id `-2408`; domain strategies `{"", ipv4_only, ipv6_only, prefer_ipv4, prefer_ipv6}`; sniff protocols `http, tls, quic, stun, dns, bittorrent, dtls, ssh, rdp`; route action types `route, reject, hijack-dns, route-options, sniff, resolve`; reject methods `default, drop, reply`; speed-test modes `FULL, DL, UL, SIMPLEDL, COUNTRY`; mirrors `GITHUB, CLOUDFLARE, GCORE, QUANTIL, FASTLY, CDN`; VPN implementations `system, gvisor, mixed`; Xray log levels `debug, info, warning, error, none`; Xray VLESS preference strings `XHTTP Only, XHTTP And Reality, All VLESS`; route profile name prefix `route_profiles`.

**Diagnostic log format**
- Line: `yyyy-MM-dd HH:mm:ss.zzz [TAG] [<threadId hex>] <basename>:<line> | <message>`; UI-forwarded lines have no file part: `… [INF] [<tid>] | [ui] <message>`.
- Tags: `TRC DBG INF WRN ERR FTL` (`OFF` never emitted).
- Session header (written at init): `===== Throne session start =====`, then lines `version`, `built`, `qt` (runtime, built-against), `os` (kernel), `arch`, `pid`, `args` (`<joined by space>` or `(none)`), `log dir`, then `================================`.
- Clean shutdown appends `===== Throne session end (clean) =====`.
- Level parse (case-insensitive, trimmed): trace→Trace; debug→Debug; info→Info; warn/warning→Warn; error→Error; fatal/panic→Fatal; off/none→Off; anything else→Debug.

**Environment handed to the core**: `THRONE_CORE_SOCKET=throneIPC-<uuid>` (random per session), `GOTRACEBACK=crash`, optional `THRONE_CORE_DEBUG=1`, `XRAY_LOCATION_ASSET=<basePath>`.

**Update contract** (manual only; no startup version check)
- Menu action disabled at startup when no update helper binary is present.
- Flow: fetch the releases list (iterate newest-first, skip pre-releases unless beta updates allowed); pick the first asset whose name contains the platform token and has extension `zip`; compare its version (asset name minus `Throne-`, semver with `alpha < beta < rc`) to the build version.
- Messages: unsupported platform → "Not official support platform"; request error → `Requesting update error: <error>\n<body>`; no newer version → "No update"; concurrent download → "Last download request has not finished yet"; invalid version strings → log `Version strings seem to be invalid…`.
- Download to `Throne.zip` with progress; on success prompt "Update is ready, restart to install?"; Yes runs the normal-exit path then (Windows) copies the helper to `updater.old` and starts it, (other OS) starts `./updater`. Download failure: `Failed to download update assets: <errors>`. The helper's unpack/apply behavior is out of scope (external binary).

### A.2 Behavior

**Startup sequence** (ordered):
1. Install a framework log handler (all framework log output goes only to the log file/ring; nothing to console; pre-init messages buffered in memory up to 2000 lines, overflow counted; a 400-line ring is kept at all times).
2. Arm crash infrastructure: Windows unhandled-exception filter + terminate/new/pure-call/invalid-parameter/abort hooks with a 64 KiB stack guarantee; Linux raises the core-dump rlimit to infinity.
3. Configure app attributes: never use native dialogs; do not quit on last window closed.
4. Construct the application object (a missing platform plugin aborts here, before logging exists — no app-level handling).
5. macOS: install the file-open event filter before the event loop.
6. Newer framework versions, non-macOS: preload a bundled emoji font (Windows ≥ 22621 → notoEmoji, older → Twemoji; Linux → notoEmoji); failure logs `could not load emoji font!`.
7. Extract launch deeplink and files from argv (files against the shell CWD).
8. Set CWD to the executable dir; delete a stale `updater.old`.
9. Resolve the config root (see A.1); create `config/`; CWD := `config/`; purge `temp/` recursively.
10. Record the app-start epoch (uptime base for runtime stats).
11. Open/create+seed the DB; failure to open is uncaught → process terminates through the crash path (UNKNOWN: exact branch for "opens but queries fail" — see fixtures).
12. Apply the diagnostic log level setting.
13. Start traffic-stats maintenance: immediate catch-up rollup in a background thread, then every 10 minutes; minute-resolution window 48 h; hour-tier retention = traffic-stats retention setting (default 90 days).
14. Save argv; set flags (`-many`, `-tray`, `-debug`, restart flags); record appdata mode.
15. Linux: add the packaged plugin path.
16. Start core-dispatch and log threads.
17. Icon fallback search path; force theme name `breeze` if no theme is set.
18. Windows only: if the stored "set admin" preference is on, the process is not elevated, and the user has not disabled privilege requests → clear+save the preference, relaunch itself elevated **with no arguments**, exit 0. (Edge: CLI args such as `-appdata <dir>` are lost in this relaunch — verified.)
19. The start-minimal setting forces the tray-start flag.
20. Apply language: 1→English (no translator), 2→`zh_CN`, 3→`fa_IR`, 4→`ru_RU`, other→system locale; translator loaded from `<exeDir>/translations/<locale>.qm` else from embedded resources; only applied if loading succeeded.
21. Single-instance check (A.1). A second instance exits 0 before any config/log mutation.
22. Initialize logging on the config dir: create `logs/` and `crashes/`; if a running marker exists, rename the previous log to `crashed-<stamp>.log` (stamp = last-modified time) and keep the 5 newest crash logs; open the log truncated and write the session header; write the running marker; flush buffered pre-init lines (with overflow warning if any dropped); prune to 5 newest `*.dmp` and 5 newest `*.txt` crash artifacts. If the log cannot be opened: continue without disk logging (ring still works), and **no marker is written**.
23. Windows: latch the crash dir; register WER LocalDumps for both `Throne.exe` and `ThroneCore.exe` (machine registry; silently skipped without admin, with an INFO log line).
24. Create the single-instance listener; failure → warning, log shutdown, exit 1.
25. Wire quit handling: on quit, close/remove the listener and shut down the logger (session-end line, marker removal).
26. Wire termination: Linux SIGTERM/SIGINT via an async-safe self-pipe (single teardown; direct-handler fallback), Windows session-end/power-off filter (allows end, runs cleanup on actual end), macOS session-logout notification.
27. Create the core RPC client; construct the main window. The constructor additionally: repairs OS auto-run task, `throne://` scheme and config-file association (only when the stored install path changed), migrates the theme setting to `System` and applies the theme, restores window geometry/splitter, creates the in-app log view and starts the core-log reader thread, pre-warms device details on a worker thread, may randomize the inbound port, opens the privilege socket to the core (failure → requested quit), starts the core process immediately (with remembered-profile autostart), applies font settings, registers the periodic jobs.
28. Schedule deferred DB maintenance: 30 s delay, then WAL checkpoint + conditional VACUUM (only when free space ≥ 4 MiB and ≥ 50 % of pages are free; incremental vacuum when auto-vacuum=INCREMENTAL) on both `throne.db` and `throne_stats.db`, on a worker thread.
29. If the previous session left a marker: in-app warning `[Warn] Throne did not shut down cleanly last time. Diagnostics were saved to: <log dir>`.
30. Submit cold-start deeplink/files; replay previously pending ones (macOS file-open events can beat the window).
31. Run the event loop; exit code = loop result (0 on normal exit).

**Logging behavior**
- Levels low→high Trace, Debug, Info, Warn, Error, Fatal, Off; a line is written only if level ≥ configured. Filtering happens before the ring, so suppressed lines never appear in crash artifacts.
- Rotation when the file exceeds 4 MiB: drop `.3`, shift `.2→.3`, `.1→.2`, current→`.1`, open a fresh file; if reopen fails, disk logging silently stops for the session.
- Flush at most every 250 ms, except Warn and above flush immediately.
- UI-forwarded lines are split per line, trimmed, empty lines dropped, prefixed `[ui] `, written only at Info or more verbose.
- Windows crash dump: minidump (indirect memory, data segments, handle, thread-info, unloaded-module and process/thread data); last ≤400 log lines embedded as a custom stream and repeated in the `.txt` sidecar together with time/version/arch/build stamp, pid/tid, exception code+name, fault address, access-violation target, faulting module+offset, dump status; a message box reports version/arch/exception/address/log dir. Synthetic codes cover uncaught C++ exception, pure virtual call, CR invalid parameter, abort, and operator-new failure.
- The in-app log browser is fed separately (core stdout/stderr + UI messages) and is not a reader of `throne.log`.
- No redaction: subscription URLs (may contain tokens), imported file contents, QR text, and device identifiers can reach the file.

**HTTP helper** (one shared helper for: subscription fetch with optional HWID, mirror-based route-profile fetch, release listing, update download, geo asset downloads, settings-dialog URL downloads, country lookup)
- Uses the proxy when any of: net-use-proxy setting, system-proxy mode currently on, or caller requests it. Proxying with no started profile → immediate error `Request with proxy but no profile started.`
- Proxy endpoint: HTTP to the configured inbound address (any-address `::` mapped to `127.0.0.1`), mixed inbound port; optional inbound credentials when auth is on.
- Redirects: never downgrade secure→insecure; otherwise framework default. TLS: framework/OS defaults; insecure setting disables certificate verification and reports SSL errors as "(Ignored)".
- User agent (GET only): configured UA, else `Throne/<version>` where version = build version before the first `-`, with `1.0.0` fallback when no dot is present.
- GET: 10 s transfer timeout; blocking; no retries; no caching; full body buffered in memory; headers as raw pairs; error string = framework network error (empty on success).
- HWID headers when the caller requests (subscription updates do when enabled): `x-hwid`, `x-device-os`, `x-ver-os`, `x-device-model`, from cached device details, overridable per key by a `key=value` comma list (accepted keys case-insensitive: `hwid`, `os`, `osversion`, `model`; each value non-empty, <1000 chars, no CR/LF; invalid pairs ignored).
- Download: no transfer timeout (can hang on a stalled server); no custom UA, no HWID; progress to the UI; body written to `<basePath>/<fileName>` via `<file>.tmp` → remove old → rename. Error strings: `Download failed: server returned HTTP status <code>.`, `Download failed: the server returned an empty response.`, `Could not open file.`, `Could not write file.`, `Could not save downloaded file.`, or the framework error; success returns empty. A zero status is not treated as HTTP failure.

**Periodic runner**
- One timer, 60 s cadence, plus one catch-up tick 10 s after construction. A job runs when its interval > 0 and (never ran OR now − last ≥ interval minutes); the last-run stamp is written *before* running (prevents double-fire; survives restarts); a log line `Auto-update: running <id>` is emitted (ids localized: "subscriptions", "routing profiles").
- Jobs: subscriptions (interval `sub_auto_update` minutes when ≥ 30, else disabled; persistence stamp `sub_auto_update_last`; refreshes all groups quietly) and routing profiles (interval `route_auto_update`; refreshes all remote routing profiles). Both default disabled (stored negative: −30, −1440).
- A settings change triggers an immediate re-evaluation so newly enabled/shortened intervals apply without waiting.

**Device details** (collected once, cached, pre-warmed off the UI thread; shown as tooltip of the "send HWID" checkbox; empty fields display `N/A`)
- Windows: HWID = OS machine unique id, fallback `<hostname>-<productType>`; OS `Windows`; version `Major.Minor.Build`; model = WMI computer-system model and baseboard product joined with `/` when different (WMI failures → empty strings, no exception).
- Linux: HWID = `/etc/machine-id`, fallback `/var/lib/dbus/machine-id`; OS `Linux`; version = kernel version; model = pretty product name.
- macOS: HWID = machine unique id; OS `macOS`; version = OS product version; model = pretty product name.
- Unknown platforms: machine unique id, product type/version, pretty name.

**Country helper**
- Static in-binary name→ISO-3166 alpha-2 map (~240 entries; exact spelling as produced by the speed-test service, e.g. `Iran, Islamic Republic Of`→IR, `United States`→US, `Virgin Islands, U.S.`→VI; `Netherlands Antilles` appears twice, both AN). Matching is exact and case-sensitive; unknown → empty string.
- Flag: each character shifted by +0x1F1A5 to regional-indicator symbols (`IR` → 🇮🇷); no validation. Shown in profile list decoration, running profile info, connection info, runtime stats, speed-test results.

**Exit, crash, force-kill**
- Normal exit is idempotent. Order: remove system proxy if set → unset system DNS if set → unregister hidden-menu shortcuts/hotkeys → persist UI state (geometry/splitter/remembered profile/system-proxy/Tun/settings) → block further settings writes → stop the proxy → kill the core synchronously → hide window/tray → optionally spawn updater or restarted self → quit event loop → close/remove the single-instance listener → session-end line + remove marker.
- Self-restart: same executable with original argv minus `-tray`, `-flag_restart_tun_on`, `-flag_restart_dns_set`, plus the appropriate restart flag when restarting for Tun/DNS; elevated on Windows for Tun/DNS restarts.
- Window close: hidden to tray when the tray icon is visible (app keeps running); normal exit when tray is disabled.
- Signals/session end (Linux SIGTERM/SIGINT, Windows session end, macOS logout) all route through the same clean teardown; Linux guarantees exactly-once teardown on the main thread.
- Force kill/power loss: no teardown; the marker remains → next start preserves the old log as `crashed-*`, warns in the new log and in-app; system proxy / system DNS may remain set (product behavior; no start-time repair).

### A.3 Edge cases & errors
- Log cannot be opened → session continues without disk logging; no running marker → a later start will not treat it as a crash.
- Corrupt main DB → startup aborts via the crash path (no catch around DB open); a merely unreadable-table DB instead shows `DB error occurred … your database may be corrupted` and keeps running (UNKNOWN: exact branch classification — see FIX).
- Second instance when the primary is hung: connection may still succeed (accept backlog); second instance still exits 0 within ~250 ms; primary acts when its loop resumes. If connect fails within 250 ms and the listener name is taken → warning, log shutdown, exit 1.
- Stale listener after a crash: recovery depends on framework/OS semantics — UNKNOWN, verify at runtime.
- Windows admin relaunch happens before settings like language, and drops argv (verified).
- Update download failure without HTTP status is not an HTTP error.
- Geo asset download failures surface raw; usage sites decide messaging.

### A.4 State & config (defaults)
- Diagnostic log level: `debug`. Core log level: `info` (a `debug` value turns on the core debug env).
- In-app log burst cap 200 lines (counter reset every second); auto-scroll on; include/exclude filters off.
- Subscription auto-update: disabled; stored as −30 (magnitude = minutes; values < 30 are off); last-run stamp 0. Route auto-update: disabled; stored −1440; stamp 0.
- User agent: empty (default UA applies).
- Latency test URL `http://cp.cloudflare.com/`, timeout 3000 ms, concurrency 10; speed-test timeout 5000 ms; simple DL URL `http://cachefly.cachefly.net/1mb.test`.
- net-use-proxy off; net-insecure off; send-HWID off; custom HWID params empty.
- Language 0 (system); theme `"0"` at rest, migrated to `System` on first boot of this version; start-minimal off; disable-tray off.
- Ruleset mirror CLOUDFLARE (1); remember-last-profile off; remembered id `-1919` (none); remember system-proxy off; remember Tun off; Windows set-admin off; disable admin requests off.
- Inbound: address `127.0.0.1`, mixed port 2080, random port off, inbound auth off; clash API port −9090 (negative = disabled), listen `127.0.0.1`; DNS-in port 5533.
- Scheme mirror string empty (rewritten only when the install path changes).
- Runtime-only (never persisted): full argv, appdata/restart flags, core socket name, started profile id, core-running flag.

---

## B. GUI shell & interaction

### B.1 Interface

**Window composition**
- Single top-level window with a hidden application menu bar (menus live on top-bar buttons). Top to bottom:
  1. Top bar: five buttons `Program`, `Settings`, `Groups`, `Routing`, `Tools` (instant-popup menus, text under icon, 24 px icons), then the start/stop control, then three checkboxes `Tun Mode`, `System DNS`, `System Proxy`, then an expanding HTML status area (clickable links; no context menu, no scrollbars, no text selection except links).
  2. Vertical splitter: top = group tab bar (one movable tab per group, title = group name) with the profiles table in each tab; bottom = stats tabs `Logs`, `Connections`, `Traffic Graph`.
  3. Bottom row: three stretch labels — running status text, inbound text, speed text.
- Profiles table per tab: fixed columns `Type`, `Address`, `Name`, `Test Result`, `Traffic`; row selection; alternating row colors; no in-place editing; default row height 30 px; no header highlight; sort indicator suppressed (custom sort UI); drag-reorder enabled; per-pixel horizontal scroll. The vertical header shows `✓ ` for the running profile, otherwise the 1-based row number.

**Menus** (exhaustive; runtime additions noted):
- **Program**: Hide window (Ctrl+H) · — · New profile (Ctrl+N) · Add profile from clipboard (Ctrl+V) · Add profile from File(s) (Ctrl+O) · Scan QR Code (Ctrl+Shift+Q) · — · Start with system (check) · Remember last profile (check) · Allow other devices to connect (check) · — · Restart Core · Restart Program · Exit.
- **Settings** (internal title "Preferences"): Basic Settings · Routing Settings · Tun Settings · Hotkey Settings · Open Config Folder.
- **Server** (no visible button; profiles-table context menu + shortcut-only): Share ▸ {QR Code and link · Export Sing-box config (Ctrl+E) · Export Xray config · — · Copy links of selected (Ctrl+C) · Copy links of selected (Deep Links) (Ctrl+Alt+C) · Copy Test Result} · New profile · Add profile from clipboard · Add profile from File(s) · Scan QR Code · — · Start (Return) · Stop (Ctrl+S) · Share ▸ · Delete (Del) · — · Select All (Ctrl+A) · Clone (Ctrl+D) · — · Url Test Selected (Ctrl+Shift+S) · Speedtest Selected (Ctrl+Shift+P) · Resolve Selected Out IP · Resolve Selected Domain · Clear Test Result · Reset Traffic (Ctrl+R).
- **Hidden menu** (never shown; shortcuts only): Update subscription (Ctrl+U) · Resolve Domain for group (Ctrl+Shift+I) · Remove Invalid (Ctrl+Alt+I) · Url Test Group (Ctrl+Shift+G) · Remove Unavailable (Ctrl+Shift+R) · Clear Group test result (Ctrl+Shift+C) · Remove Duplicates (Ctrl+Shift+D) · Speedtest Group (Ctrl+Alt+P) · Resolve out IP for group · Refresh Column Widths · Operation Mode ▸ {System Proxy, Tun Mode} (+ `Disable` appended at runtime; checkable).
- **Routing** (rebuilt each open): Routing Settings · Enable AdBlock (toggle) · Enable Warp (toggle) · Download Profiles ▸ (country list) · — · one entry per routing profile (current marked).
- **Groups**: Manage Groups · Add new Group · Edit current Group · Delete current Group · — · Url Test Group · Speedtest Group · Resolve out IP for group · Resolve Domain for group · Clear Group test result · — · Update subscription · Update all subscriptions · Remove Duplicates · Remove Unavailable · Remove Invalid · Remove Insecure Configs · — · Refresh Column Widths.
- **Tools**: Auto Selector Stats (hidden until an auto-selector run exists) · Speedtest Current (F6) · — · Runtime Stats · Traffic Stats (hidden when traffic aggregation is disabled) · Check For Update.
- While a test runs, `Stop Testing` (Ctrl+.) is appended to **Server** and **Groups** and removed when idle.
- Declared-but-unwired actions (never reachable in this build): Debug Info, Copy Link, Tcp Ping, Url Test, Move, Full Test, Open Manager, plus invisible placeholder actions.

**Dialogs & popups**
- Right-click on the profiles table pops the **Server** menu (acts on selection; group-scoped invocations act on the whole group; an internal scope flag resets ~200 ms after close).
- Group tab-bar context menu: empty area → `Add new Group`; on a tab → selects the tab, then Add/Edit/Delete (+ subscription actions).
- Header right clicks: Type → sort key (type; Security only when the security suffix is folded in); Test Result → latency / download speed / upload speed / outbound IP; Traffic → total / downloaded / uploaded.
- Connections table headers: click-sort; right-click Traffic/Speed picks sub-field.
- Log view context menu: standard text actions + `Clear`.
- Other dialogs: warnings/infos/checks/scrollable bodies for import problems, QR capture failure, subscription/remote-route prompts, routing import confirmation, "Config copied", delete confirmations, restart prompts, elevation prompts, file pickers, hotkey capture, group edit/manage.

**Tray**
- Icon states: Off / Throne / Proxy / Proxy-Dns / Dns / Tun (built-in set; custom folder when "use custom icons").
- Menu: Show Window · — · Start with system · Remember last profile · Allow other devices to connect · — · Select Profile ▸ · Select Routing ▸ · Operation Mode ▸ {System Proxy, Tun Mode, Disable} · — · Restart Core · Restart Program · Exit. macOS composes its own operation-mode submenu.
- Tooltip = window title text with line breaks. Left click toggles show/hide (non-macOS). No balloon notifications anywhere.

**Start/stop control states**
- Disabled — tooltip `Select a profile to start`; Idle — label `Start`; Connecting — `Connecting…` (not clickable); Running — `Stop`; Disconnecting — `Stopping…` (not clickable).
- Ring color by effective mode: green core-only, blue system proxy, crimson Tun, dark gold system DNS, indigo proxy+DNS, amber for connecting/stopping/failure.

**Stats panes**
- Connections columns with exact tooltips: `Destination (Domain)` — "Click To Disable Sorting"; `Process` — "Click To Sort By Process"; `Protocol` — "Click To Sort By Protocol"; `Outbound` — "Click To Sort By Outbound"; `Traffic` — "Click to sort by traffic; right-click to choose total/down/up"; `Speed` — "Click to sort by speed; right-click to choose total/down/up". Clicking any cell copies its text and shows a `Copied!` tooltip ~1.5 s. No kill-connection action.
- Traffic Graph: container painted by the stats module.
- Logs: rich-text browser; context menu + `Clear`; include/exclude keyword and regex filters from settings (live); auto-scroll toggle; retained blocks capped by max-log-lines; messages > 20000 chars truncated to 1000 with `TRUNCATED LONG LOG: ` prefix; pending display queue capped at 1000; font = system fixed font at the app point size.

**Intake surface** (details in §I for formats)
- Deeplinks: `throne://add?<query>`, `throne://remoteroute/<payload>`, `throne://route/<base64>`, `throne://addsub/<base64-url>[#name]`; unknown command → log `Ignored deeplink with unknown command: <cmd>`; command matching case-insensitive.
- Files: multi-select with filters `All files (*)`, `Config files (*.json *.conf *.txt *.yaml *.yml *.ini)`, `QR code images (*.png *.jpg *.jpeg *.bmp *.gif *.webp)`. Per file: unopenable → `<name>: cannot be opened`; > 50 MB → `<name>: larger than 50 MB, skipped`; image with no QR → `<name>: no QR code found`; undecodable → `<name>: not a readable config file`. URL payloads take the subscription path; the rest import as a batch; all failed → `Nothing could be imported:` + reasons.
- Clipboard add and CLI/config-file intake funnel into the same paths (see §A/A.1).
- Global hotkeys: toggle window; open Manage Groups; open Routing Settings; pop Operation Mode at cursor; toggle System Proxy.

### B.2 Behavior

**Launch, window, exit**
1. First launch shows the main window; geometry, splitter, current group, tab order, per-group column widths/scroll/selection, last filter column, stats tab, theme restored before content; corrupt values fall back to defaults.
2. Second invocation forwards payload and exits (see A.1); no second window.
3. Window title: app name + version, `[Admin]` when elevated, `[Tun]`/`[System Proxy]` markers, routing profile name, and while running the profile with `@group` (+ subscription/country annotation). Tray tooltip = same with line breaks.
4. `Hide window` hides to tray; proxy keeps running.
5. `Exit` = graceful shutdown; `Restart Program` persists and relaunches; `Restart Core` relaunches only the core.
6. While a core restart is pending, captured remember-flags (system proxy/Tun) are re-applied when the core starts and a remembered profile autostarts.
7. Session-end/commit requests save state without prompt.
8. Close button hides when a tray icon is available (close ignored); with tray disabled, close exits. Minimizing does not stop the core.

**Start/stop**
9. Precondition: at least one selectable profile, else the control is disabled with tooltip `Select a profile to start`.
10. Start → Connecting; on core readiness → Running (tray icon reflects mode; remember-modes re-applied). Failure → back to Idle with a warning dialog carrying the core message.
11. Stop → Disconnecting → Idle; app-set system proxy/Tun/DNS are released as part of stop unless "reset proxy on disable" disagrees (partially inferred).
12. Core crash is reported to the shell and forces the stop flow back to Idle.
13. Deleting/editing the running profile is blocked unless "allow stopping active profile" is on; when on, the running profile stops first.
14. Effective mode precedence (single source for ring and tray): Tun > System Proxy + System DNS > System DNS > System Proxy > Core-only.

**Operation-mode toggles**
15. Checkboxes, their menu twins, and tray submenu stay in sync; effective mode drawn on the ring/tray.
16. Enabling Tun on Windows without elevation prompts for elevation (possibly relaunch); failed elevation reverts the toggle and mode.
17. `System DNS` hidden unless the show-system-dns setting is on; `Tun Mode` visibility is platform-dependent; `Allow other devices to connect` reflects the inbound address (checked when bound to `::` or `0.0.0.0`) and switching rewrites the inbound address.
18. Mode changes apply live where possible; otherwise `Settings changed, restart proxy?` (Yes restarts the core).
19. Any settings dialog change triggers immediate save, a periodic-runner check, visibility refreshes for stats/system-DNS/tray, and optionally a restart suggestion.

**Profiles table**
20. Rows show Type (security suffix when enabled), Address/host, Name, Test Result, Traffic; running row marked `✓ ` in the vertical header.
21. Row-based selection; Ctrl+A selects all rows unless the log view has focus.
22. Header-click sorting; per-group persisted sub-keys (Type: type or Security; Test Result: latency / DL speed / UL speed / outbound IP; Traffic: total / down / up), chosen via header right-click.
23. Filters for Type, Address, Name, Country. Address accepts plain text and port expressions `port=N`, `port=N:M`, `port=N:`, `port=:M`. A key shortcut toggles the filter row; a tab-corner button shows/hides filters; last-used filter column remembered.
24. Drag-reorder rearranges within the group and persists; disabled while any filter is active.
25. Right-click pops the Server menu (selection scope or whole-group scope).
26. Column widths are auto-measured per group and cached; `Refresh Column Widths` re-measures; toggling the security suffix invalidates the Type cache.
27. After any list refresh, selection/scroll/focus are restored; filter caret is never stolen mid-keystroke.

**Groups**
28. Tabs are movable; tab order persisted; switching swaps the table and restores that group's selection/scroll/focus.
29. `Add new Group` opens the editor; accepted → tab appears and is selected.
30. `Edit current Group` edits the tab's group; `Delete current Group` confirms and leaves a valid current group (or recreates a default).
31. Tab context menu adds quick actions.
32. Subscription updates diff old/new profile sets; summary logged; manual refreshes show a scrollable `Change of <group name>` dialog when the popup setting is on (body `Nothing` when empty); still-in-use profiles are kept and reported `Still in use, so kept instead of deleted:`.
33. `Update subscription` (Ctrl+U) refreshes the current group's URL only; no-op without a URL or while an update runs. `Update all subscriptions` iterates subscription groups.
34. On import completion: log `Imported %1 profile(s)` unless quiet, refresh table; new groups refresh the tab bar.

**Tray**
35. Visible unless disable-tray is on; toggling applies immediately.
36. Non-macOS tray click toggles visibility with a 350 ms recently-active guard so programmatic raise is not immediately re-hidden.
37. Select Profile / Select Routing list entries with the active checked; selection acts as the in-window action.
38. Tray Exit behaves as the Program exit. 39. No balloon notifications, ever.

**Hotkeys and shortcuts**
40. Every configurable action has a stable id; user overrides stored in a shortcuts map; defaults remain where no override exists.
41. Hidden-menu shortcuts are re-registered as window-level shortcuts (duplicates with visible-menu shortcuts skipped).
42. Global hotkeys: toggle window, Manage Groups, Routing Settings, Operation Mode at cursor, toggle System Proxy.
43. Global System-Proxy toggle calls the same action as the checkbox.
44. Default inventory: see B.1 menus (Ctrl+H, Ctrl+N, Ctrl+V, Ctrl+O, Ctrl+Shift+Q, Ctrl+E, Ctrl+C, Ctrl+Alt+C, Return, Ctrl+S, Del, Ctrl+A, Ctrl+D, Ctrl+Shift+S/G, Ctrl+Shift+P / Ctrl+Alt+P, F6, Ctrl+Shift+C, Ctrl+R, Ctrl+U, Ctrl+Shift+I, Ctrl+Shift+R, Ctrl+Shift+D, Ctrl+Alt+I, Ctrl+.).

**Import / export UX**
45. `New profile` (Ctrl+N) opens the editor in add mode.
46. Clipboard/file/QR funnel into one import path: deeplink → command handling; URL → subscription prompt; config text → batch import; failures logged per file, warning summary when nothing imported.
47. QR scan: hide window, 1 s sleep, grab primary screen, re-show, decode; capture failure → `Unable to capture screen`; decoded strings logged as `QR Code Result:` and fed into the import path.
48. `Export Sing-box config` (Ctrl+E, single selection): build, copy to clipboard, info dialog `Config copied` with the config body and buttons `Copy core config` / `Copy test config` / OK. `Export Xray config` analogous for Xray/chain profiles.
49. `QR Code and link` shows the profile QR image and link text.
50. `Copy links of selected` (Ctrl+C) copies plain links; Deep Links variant (Ctrl+Alt+C) copies `throne://`-style links.
51. `Copy Test Result` concatenates non-empty results of up to 1000 selected profiles; logs `%1 Test result(s) copied to clipboard!`.

**Test UI surface**
52. Url tests: selected (Ctrl+Shift+S) / group (Ctrl+Shift+G). Speedtests: current F6 (needs running profile, else silent no-op) / selected (Ctrl+Shift+P) / group (Ctrl+Alt+P). Outbound-IP resolve: selected / group.
53. While testing, `Stop Testing` (Ctrl+.) is appended to Server and Groups; a new test cannot start while one runs; completion refreshes the Test Result column per sub-key.
54. Status area priority: download > speedtest > latency test > auto-selector panel (only when nothing else active); 10-char ASCII progress bar. Results persisted to the Test Result column and copyable.
55. Test parameters from settings (defaults per A.4).
56. `Clear Test Result` / `Clear Group test result` clear stored results and reset the column-width cache.

**Connections view**
57. Rows refresh only while window visible, not minimized/hidden, tab selected; otherwise suspended and resumed on re-show.
58. Sorting per B.1; first column click disables sorting.
59. Cell click copies and shows `Copied!` ~1.5 s.
60. No per-connection kill; empty state is a plain empty table.

**Logs view**
61. GUI and core messages append to the same view; keyword/regex filters live.
62. `Clear` empties the view only (disk log unaffected).
63. Auto-scroll pins to bottom when enabled; retained blocks capped by max-log-lines.
64. > 20000 chars truncated to 1000 with `TRUNCATED LONG LOG: `; system fixed font; selectable.

**Deeplinks / files / CLI**
66. All deeplink paths raise the window before prompting.
67. `addsub` creates a group named from the fragment else the URL host; skip-auto-update = unchecked checkbox; refreshes tab bar and starts the update.
68. `remoteroute` fetches all profiles off-thread, saving each; failures log `Remote routing profile <url> failed: <err>`; summary `Added remote routing profiles: %1 of %2 fetched`.
69. Unknown command → log only.
70. Imported files become profiles in the current context via the batch importer; URL payloads use the subscription path.

**Theming, fonts, icons**
71. Themes applied at startup and live: `System` (default after migration; restores captured style/palette) plus built-in palettes/styles FlatGray, LightBlue, SoftPink, BlackSoft, QDarkStyle; other names resolved through a style-factory lookup.
72. Log syntax colors brightness-adjusted per theme.
73. Font family/size apply app-wide; empty = system default.
74. Icons from bundled resources; custom tray folder when enabled; emoji preloaded (newer framework only).

**Persistence & geometry**
75. On graceful exit and commit: window geometry (base64), size string (skipped when maximized), splitter state (base64), current group, tab order, per-group column widths + scroll-last-profile + selection, last filter column, stats tab, active route, theme/font/language, remember flags.
76. Restored before content; corrupt values fall back to defaults.

**Dialogs & error paths**
77. Restart-needed setting change → `Confirmation` / `Settings changed, restart proxy?` (Yes restarts core). Tun settings changed while running → `Tun Settings changed` / `Restart Tun to take effect.` Program restart needed → `Settings changed` / `Restart the program to take effect.` (Yes restarts).
78. Profile/group deletions confirm unless skip-confirmation is on.
79. Import failures per B.1; QR capture failure dialog.
80. Windows elevation prompts for Tun/admin; refusal reverts the toggle.
81. `Speedtest Current` without running profile and `Update subscription` with empty URL are silent no-ops.

### B.3 Edge cases & errors
- No profiles: start control disabled with the tooltip; Ctrl+N still reachable.
- Core binary missing: GUI opens; in-app log `start core error occurred: <error>`; profile never starts; no GUI crash.
- Restart prompts only when live-apply is impossible.
- Connection polling suspends when the view is not visible; no data loss beyond refresh.
- Import edge cases per B.1 (unopenable/large/no-QR/undecodable/URL-vs-batch).
- `Import %1 profile(s)`/diff popup interplay: popup only for manual refreshes (inferred).

### B.4 State & config (defaults)
| State | Default |
|---|---|
| Theme | `0` (migrates to `System`) |
| Language / font / font size | 0 / "" / 0 |
| Window size / geometry / splitter | "" (system), "" |
| Stats tab index | 0 |
| Last filter column | −1 |
| Log include/exclude filters (keyword+regex, enables) | empty / off |
| Log auto-scroll | on |
| Max log lines | 200 |
| Start minimal | off |
| Stats enabled | on |
| Traffic aggregation disabled | off |
| Traffic stats retention | 90 days |
| Speed-test mode / timeout / simple DL URL | FULL / 5000 ms / `http://cachefly.cachefly.net/1mb.test` |
| Latency URL / timeout / concurrency | `http://cp.cloudflare.com/` / 3000 ms / 10 |
| Show System DNS | off |
| Custom icons | off |
| Skip delete confirmation | off |
| Fold security into Type | off |
| Tray disabled | off (tray on) |
| Allow stopping active profile | off |
| Remember (enable/id/system-proxy/tun) | off / none / off / off |
| Subscription auto-update (+ popup) | −30 min (disabled; <30 off); change popup on |
| Route auto-update | −1440 (disabled; daily) |
| Deeplink scheme mirror string | "" |
| Shortcuts map / global hotkeys | empty |
| Active route id | 1 |
| Inbound address / port / mode | `127.0.0.1` / 2080 / mixed |
| Per-group UI state (column widths, scroll, selection, sort keys) | empty |

## C. Profiles, protocols & configuration semantics

### C.1 Interface

**Profile record** (persisted; "runtime config" = the engine JSON handed to the core on start; "test config" = the shared box built for latency tests):

| Field | Meaning / behavior |
|---|---|
| type | protocol key (list below). Unknown/empty type yields a base outbound flagged invalid. |
| name | user label; may be empty in stored data. |
| id | id; −1 = unsaved. |
| gid | owning group. |
| latency | ms; −1 = failed/unavailable, 0 = never measured, >0 = measured. |
| latency_at | unix seconds of measurement; 0 = unknown/never (legacy rows read as expired). |
| dl_speed / ul_speed | speed-test display strings ("N/A" filtered out). |
| test_country | ISO code; displayed as flag. |
| ip_out | exit IP string. |
| traffic_downlink / traffic_uplink | bytes (runtime counters). |
| outbound | protocol payload object (all protocol fields live here). |

- Setting latency to a positive value stamps the measurement time; setting 0 clears the stamp. Clearing test results wipes country, IP, latency (+stamp) and speeds.
- Test-result column: flag + "N ms", or literal "Unavailable" when latency < 0; speeds/IP appended only when the group's test-items setting allows. Traffic column shows "↑ up ↓ down" in readable sizes, empty at zero. Latency color: ≤100 green, ≤300 yellow, >300 red, failed dark gray.

**Identity & de-duplication**
- Same profile ⇔ deep-link JSON byte-equal; the exported tag (name) is stripped by default, so renaming does not create a duplicate.
- Update matching uses a second key: type + compact JSON of the identity object — base identity = server + server_port + protocol-specific identity: TLS identity = {enabled, disable_sni, server_name, uTLS fingerprint (if on), ECH enabled, Reality enabled}; transport identity = transport type only (empty for plain TCP); Xray stream identity = network, security, plus sni/fingerprint for tls or reality. Protocols without a server address (chain, auto selector) use the whole export JSON.
- Update matching consumes each source entry at most once, so N identical servers map to N distinct slots.

**Protocol type keys** (factory + type chooser): `socks`, `http`, `shadowsocks`, `chain`, `autoselector`, `vmess`, `trojan`, `vless`, `xrayvless`, `hysteria` (displayed v1/v2), `tuic`, `juicity`, `trusttunnel`, `anytls`, `mieru`, `shadowtls`, `wireguard`, `tailscale`, `ssh`, `custom`, `extracore`, `naive`, `direct`; `hysteria2` maps to the same protocol with version 2. Anything else → invalid base outbound (imported data can hold "unknown" profiles that fail validation).

**Editors** (per-type page + shared sub-editors):
- TLS page: enabled, disable SNI, server name, insecure, ALPN, min/max version, cipher suites, curve preferences, certificates (inline/path, public-key SHA-256, client cert/key inline/path), fragment tri-state + fallback delay, record fragment, TLS tricks tri-state, ECH, uTLS, Reality. uTLS page hidden for hysteria, tuic, juicity, naive.
- Transport page: type, host, path, method, headers, idle/ping timeouts, websocket max early data + early-data header name, gRPC service name.
- Multiplex page: tri-state + protocol, max connections/streams, min streams, padding, brutal (enabled, up/down Mbps).
- Xray stream page (Xray-backed profiles only): network, security, TLS/Reality sub-pages, xHTTP extras + xmux + download settings (own dialog), ws/httpupgrade/grpc pages.
- Dial-fields page: reuse_addr, connect_timeout, tcp_fast_open, tcp_multi_path, udp_fragment, bind_interface, inet4/inet6 bind address.

**Save-time validation**
- Name required for: chain, auto selector, custom, and the common dialog path ("Name cannot be empty.").
- Header lines must not contain `|`; name and values are trimmed.
- Auto selector additionally needs a selected, non-archived group and a name filter compiling as a regex (case-insensitive matching). On save it is normalized (clamps below) and its stored pool filtered to still-eligible members (stale ranking dropped, never carried).
- Chain editor: auto-selector hop refused immediately ("An auto selector cannot be a hop in a chain…"); chain-in-chain refused silently; at most one extra-core hop, and it must be top of the list ("…only be the final hop in the chain. Move it to the top of the list." / "Only one extra-core profile is allowed in a chain.").
- xHTTP: maxConnections and maxConcurrency are mutually exclusive ("XHTTP maxConnections cannot be specified together with maxConcurrency.").
- WireGuard: local addresses split on commas after stripping spaces; reserved parsed from comma-separated integers; the WARP auto-generation button fetches a key pair and a generated config and fills keys/addresses (mtu 1280, keepalive 30, reserved list), rewriting the outer dialog's server host/port.
- Extra core: socks port validated 1–65534; executable pickable from disk; appended to the global extra-core path list when new.

**Custom subtypes**: `outbound` (sing-box outbound JSON), `fullconfig` (entire sing-box config JSON), `xrayoutbound` (Xray outbound JSON), `xrayfullconfig` (entire Xray config JSON). Display type derived from the embedded JSON ("Custom <Type> Outbound", "Custom Config", "Custom Xray <Protocol> Outbound", "Custom Xray Config"). Only `outbound` can be an endpoint type when its embedded type is wireguard/tailscale. Only fullconfig/xrayfullconfig count as "full config" for chaining/selectors; `xrayoutbound` is treated as an Xray hop (a dummy sing-box socks outbound is added so engine validation passes; the real payload goes to Xray); `xrayfullconfig` uses a transient socks outbound to a reserved local bridge port.

**Shared option-group emission**
- Outbound base: runtime JSON contains `server`/`server_port` only when set (port > 0); the name is not emitted as tag by protocol builders (the chain/section builder attaches tags). Export/import JSON uses `tag` = name. Dial fields merge at the same level: `reuse_addr, connect_timeout, tcp_fast_open, tcp_multi_path, udp_fragment, bind_interface, inet4_bind_address, inet6_bind_address` (booleans only when true; strings only when non-empty).
- Custom TLS fragment: when TLS is on, fragment resolves on and the global fragment implementation is "custom", the builder emits dialer-level `tls_fragment {enabled, size, sleep}` and removes `tcp_fast_open` (engine rejects the combination). Sizes default "10-100"/"2-5". The built-in implementation emits `tls.fragment` inside the TLS object.
- TLS object: `enabled` always when on; else an empty object (protocols then usually omit `tls`). Optional keys: disable_sni, server_name, insecure, alpn[], min_version, max_version, cipher_suites[], curve_preferences[], certificate/certificate_path, certificate_public_key_sha256[], client_certificate(_path), client_key(_path); `fragment`(+fragment_fallback_delay) only when effectively on and built-in; `record_fragment`; `tls_tricks {mixedcase_sni: true}` when effectively on; `ech {enabled, config[], config_path, query_server_name}` when enabled; `utls {enabled, fingerprint}` (a configured global fingerprint is applied when not set — unless the protocol declares uTLS unsupported, where nothing is emitted); `reality {enabled, public_key, short_id}` — Reality on with no uTLS forces `utls {enabled: true, fingerprint: "random"}`.
- TLS link query keys: security (tls/reality/none normalized), sni/peer→server_name, alpn, allowInsecure/…, fp→uTLS fingerprint, pbk/sid→Reality, ech_enabled/ech_config/ech_config_path/ech_server_name. Export emits only enabled sub-objects (fp, pbk+sid, ech_*), TLS scalars/certificates, and fragment/tls-tricks only when not "Keep Default".
- Transport (sing-box): types `ws`, `http`, `httpupgrade`, `grpc` (empty = plain). Keys: path/host for ws+httpupgrade; method/host/path/headers for http; service_name for grpc; headers = JSON object. Websocket: max_early_data, early_data_header_name. The `?ed=N&eh=` query convention on ws/httpupgrade links is parsed into early-data fields on import/export, respecting port position. Empty type → no transport object.
- Multiplex (sing-box): tri-state Keep Default / On / Off. On → `multiplex {enabled: true,…}`; Off → nothing; Keep Default defers to the global mux-on setting (default off). Protocols smux/yamux/h2mux; empty protocol → global (default smux). If max_connections, min_streams and max_streams are all zero, max_streams falls back to global mux concurrency (default 8). Global padding (default off) turns `padding` on. Brutal: enabled + up_mbps/down_mbps.
- Xray stream settings: network ∈ {raw, xhttp, ws, httpupgrade, grpc} (default raw); security ∈ {none, tls, reality} (default none). Xray TLS keys: serverName, pinnedPeerCertSha256, verifyPeerCertByName, alpn, fingerprint (global fallback when empty). Reality: serverName, fingerprint, password, shortId, spiderX. xHTTP: host, path, mode (auto/packet-up/stream-up/stream-one; default auto), raw extra JSON passthrough, headers, xPaddingBytes, xPaddingObfsMode (+key/header/placement/method), uplinkHTTPMethod, session/seq placement+key+table+length, uplinkDataPlacement/Key/ChunkSize, noGRPCHeader, noSSEHeader, scMaxEachPostBytes, scMinPostsIntervalMs, scMaxBufferedPosts, scStreamUpServerSecs, serverMaxHeaderBytes, raw xmux passthrough + maxConcurrency/maxConnections/cMaxReuseTimes/hMaxRequestTimes/hMaxReusableSecs/hKeepAlivePeriod, downloadSettings (raw downstream stream-settings JSON). xs: path, host, ed, headers, heartbeatPeriod. httpupgrade: path, ed, host, headers. grpc: authority, serviceName, multiMode. UI constraints: download settings hidden/removed for stream-one; maxConnections ⊕ maxConcurrency; padding-obfuscation fields only when enabled. Documented defaults: xPaddingBytes 100-1000; serverMaxHeaderBytes 8192; scMaxEachPostBytes 1000000; scMinPostsIntervalMs 30; scMaxBufferedPosts 30; uplink method POST; chunk-size below 64 clamped; stream-up server seconds default 20-80; session/seq ids default to a random UUID when the table is empty; xmux defaults 1-1 / 600-900 / 1800-3000; hKeepAlivePeriod 0 = Xray default, negative disables.
- Xray multiplex: tri-state over `useDefault`; fields enabled, concurrency, xudpConcurrency (default 16); Keep Default defers to global xray-mux-on (default off) and global concurrency (8) when unset.

**Per-protocol catalog** (fields → JSON notes → share link; "no link" = empty string):
- **socks**: username, password, version (default 5; version 4 emitted as string "4"), uot. JSON type socks + username/password/uot. Link `socks4://`/`socks5://` (user:pass; fragment = name; dial fields in query). Display type includes version.
- **http**: username, password, path, headers (object), optional TLS (https scheme forces TLS on import). JSON type http + those + tls. Link `http://` / `https://`.
- **shadowsocks**: method (enum: `2022-blake3-aes-128-gcm, 2022-blake3-aes-256-gcm, 2022-blake3-chacha20-poly1305, none, aes-128-gcm, aes-192-gcm, aes-256-gcm, chacha20-ietf-poly1305, xchacha20-ietf-poly1305, aes-128-ctr, aes-192-ctr, aes-256-ctr, aes-128-cfb, aes-192-cfb, aes-256-cfb, rc4-md5, chacha20-ietf, xchacha20`), password, plugin, plugin_opts, uot (emitted `udp_over_tcp` as bool), multiplex. Link `ss://` (base64 userinfo form; SIP008 JSON import supported; simple-obfs normalized to obfs-local).
- **shadowtls**: version (default 1), password (only meaningful/emitted for version > 1), TLS (mandatory — forced on). Link `shadowtls://` (password in userinfo only when version > 1).
- **vmess**: uuid, security (default "auto"; enum auto/none/zero/aes-128-gcm/chacha20-poly1305; omitted from JSON when auto), alter_id, global_padding, authenticated_length, packet_encoding (default xudp; always emitted), TLS, transport, multiplex. Link `vmess://` + base64 JSON.
- **vless** (sing-box): uuid, flow (list includes xtls-rprx-vision), packet_encoding (default xudp), TLS, multiplex, transport. Link `vless://`.
- **xrayvless**: uuid, encryption (default "none"), flow (xtls-rprx-vision, xtls-rprx-vision-udp443), Xray stream settings, Xray multiplex. Emitted as an Xray outbound (protocol vless, settings vless array, streamSettings, mux); the sing-box side is a dummy socks outbound to 127.0.0.1 for validation only. Link `vless://` (query carries Xray stream pieces). Display "VLESS (Xray)".
- **trojan**: password, TLS, multiplex, transport. Link `trojan://`.
- **hysteria (v1/v2)**: common protocol_version ("1" default), server_ports (ranges normalized, emitted as string array), hop_interval, up_mbps, down_mbps, obfs. v1 only: auth_type (BASE64 → `auth`, STRING → `auth_str`), auth, recv_window_conn, recv_window, disable_mtu_discovery. v2 only: password; obfs emitted as object {type/…, password} (v1 obfs is a plain string). Links `hysteria://` (v1) / `hysteria2://` (v2). TLS mandatory; uTLS unsupported.
- **tuic**: uuid, password, congestion_control (cubic/new_reno/bbr), udp_relay_mode (native/quic), udp_over_stream, zero_rtt_handshake, heartbeat. JSON omits udp_relay_mode when udp_over_stream is set. TLS mandatory; uTLS unsupported. Link `tuic://`.
- **juicity**: uuid, password. TLS mandatory; uTLS unsupported. Link `juicity://`.
- **trusttunnel**: username, password, congestion_control, health_check, quic (with quic_congestion_control emitted when set). TLS mandatory. Link `tt://`.
- **naive**: username, password, congestion_control, quic, uot. TLS mandatory; uTLS unsupported. Links `naive+https://` / `naive+quic://`.
- **ssh**: user, password, private_key, private_key_path, private_key_passphrase, host_key[], host_key_algorithms[], client_version. No TLS. Link `ssh://`.
- **wireguard**: private_key; peer {address, port, public_key, pre_shared_key, reserved[], persistent_keepalive}; address[] (local CIDRs); mtu (default 1420); system (userspace vs system interface); worker_count; udp_timeout. Keepalive: plain seconds as integer; an AmneziaWG 3.0 range like "22-30" emitted as string; zero/absent omitted. AmneziaWG: enable_amnezia + jc/jmin/jmax/s1..s4 (ints when > 0), h1..h4, i1..i5 (strings), AWG3 header_protection_key, content_padding_addition, rekey_after_time, rekey_timeout, reject_after_time, keepalive_timeout, max_handshake_attempts (numeric ranges round-trip as strings). Emitted into the endpoints array (not outbounds). Link `wg://` (import sets peer address/port, local addresses, reserved, keepalive formatting preserved).
- **tailscale**: state_directory (default `$HOME/.tailscale`), auth_key, control_url (default `https://controlplane.tailscale.com`), ephemeral, hostname, accept_routes, exit_node, exit_node_allow_lan_access, advertise_routes[], advertise_exit_node, globalDNS. Endpoint type; link `ts://` (host/port not meaningful); skipped in test builds; refused as auto-selector member.
- **mieru**: transport (TCP/UDP, default TCP), username, password, multiplexing (empty/MULTIPLEXING_OFF/LOW/MIDDLE/HIGH), traffic_pattern, server_ports (comma list of ranges → array, normalized). Link `mierus://` simple form only; the standard base64-protobuf share form is rejected. No TLS group.
- **anytls**: password, idle_session_check_interval (30s), idle_session_timeout (30s), min_idle_session (5). TLS mandatory. Link `anytls://` (password in userinfo).
- **direct**: dial fields only; display address reflects bind options or "Empty"; direct outbound; no tag in build; no security verdict; no link.
- **extracore**: socks_address (default 127.0.0.1), socks_port, extra_core_path, extra_core_args, extra_core_conf, no_logs. Build yields a sing-box socks outbound to the configured local server; the runtime passes path/args/conf/no-logs to the core which launches the process (canonicalized path; OS-native separators for the direct rule). Display "ExtraCore"; no security; eligible as "extra core" for chain/selector rules; no link.
- **custom**: see subtype rules above. `fullconfig` → the whole JSON becomes the core config for a standalone start. `xrayoutbound` → dummy socks for sing-box + real outbound for Xray. `xrayfullconfig` → transient socks outbound to a reserved local bridge port; the user's Xray config gains a socks inbound (`throne-bridge`) with sniffing (http/tls/quic) authenticating the bridge; test/sole-hop modes drop the user's own inbounds. Xray full configs also expose all server domains found in outbounds (vnext/servers/address) for direct resolution.

**Meta-profile: chain**
- Model: `list` of profile ids "from the client outward": index 0 = the hop whose server is contacted directly (top of the editor); last index = hop nearest the destination. This is why an extra-core hop (socks server at 127.0.0.1) must be at the top.
- Editor rules: at most one extra-core hop at index 0; hops cannot be chains or auto selectors; missing/unknown ids dropped on load.
- Build: the list is reversed internally; hops emitted as outbounds `<prefix>-<index>`; the entry hop tagged `proxy` (main profile) or `warp-bypass` (under Warp); each outbound gets `detour` to the next, so traversal follows edition order. Tag prefixes: `config-` (main), `route-` (rule-required outbounds), `pool-` (selector members), `proxy-` (test box), `xrayfull-<id>` (test full configs).
- Constraints (user-visible error strings): chain-in-chain → "Chain in Chain is not allowed"; selector hop → "An auto selector cannot be used as a hop; it is not a fixed server"; sing-box full config → "Custom full config profiles cannot be used in a chain; only custom outbound profiles are chainable"; extra-core count → "Only one extra-core profile is allowed in a chain"; extra-core position → "Extra-core profiles can only be the final hop in a chain (top of the chain editor)"; multiple Xray full configs → "Only one custom Xray full config profile is allowed in a chain"; Xray full config + other Xray hops → "…cannot be combined with other Xray hops…"; Xray full config position → "…can only be the final hop…"; extra-core + Xray full config → "Extra-core and custom Xray full config profiles cannot be combined in a chain"; core transitions beyond 2 → "Too many core transitions, the valid sequence is: (optional sing-box chain)->(optional xray chain)->(optional sing-box chain)".
- Cross-core bridging: per bridge a free local port + random 32-char credentials are reserved. sing→Xray: sing-box gets a socks outbound hop; Xray gets a matching socks inbound tagged `<tag>-inbound` plus a routing field rule mapping that inbound to the chain's Xray entry tag; bridges emitted in Xray as `{"address","port","user","pass"}`. Xray→sing: sing-box gets a socks inbound named `bridge-<singIngressTag>`; Xray gets a socks outbound dialing it at the end of its chain. Count mismatch fails the build ("xray to sing-box bridges count does not match ingress tags count" / "xray ingress tags size does not match bridge count!"). Xray chain assembly: `log {loglevel, access}` (access "" when level info, else "none"), socks bridge inbounds, outbounds chained via proxySettings, `routing {domainStrategy: "AsIs", rules: field inbounds→outbound tags}`.
- Group landing/front proxies participate: landing prepended, front appended (front is therefore dialed directly).

**Meta-profile: auto selector** — defaults: gid −1; nameFilter/countryFilter empty; excludeUnavailable true; poolCap 1000 (max 3000); buildLimit 300 (max 500); resultValidityMins 1440; testURL/connectivityURL empty → global test URL; intervalSec 300; benchIntervalSec 600; watchIntervalSec 15; activeSize 8; sampling 10; toleranceMs 300; maxRTTms 0 (off); expected 3; dialRetries 2; interruptOnSwitch true; balance false; balanceMode "rotate" (other value "connection"); balanceIntervalSec 30. Persisted state: pool (ranked ids, best first), poolRankedAt, pinnedID (−1 = automatic), lastBuilt/lastBuiltAt, history (max 2000 entries: id/first/last/builds/fails/name, newest first, LRU-trimmed).
- Normalize clamps (after parse, before each build): poolCap 1..3000; buildLimit 1..500 and ≤ poolCap; resultValidityMins ≥ 0; intervalSec ≥ 10; benchIntervalSec ≥ intervalSec; watchIntervalSec 5..intervalSec; activeSize ≥ 1, ≥ expected, ≤ buildLimit; sampling 2..60; toleranceMs ≥ 0; maxRTTms ≥ 0; expected ≥ 1 and ≤ buildLimit; dialRetries 0..5; balanceIntervalSec ≥ 5; balanceMode forced to "rotate" unless exactly "connection".
- Planning (membership): starts from the tracked group's profile list excluding the selector itself. Skip reasons/messages: missing profile ("missing profile"); chains/selectors ("chain or auto selector"); Tailscale; extra-core; foreign full config ("full config profile"); malformed custom JSON ("config does not parse"); name filter (case-insensitive regex over display name); country filter (comma list, uppercased, matches test_country); fresh failure only ("last test failed"); Xray full config when the group has landing/front proxies ("Xray full config cannot be combined with the group's proxies"); >2 core transitions for landing→member→front ("needs too many core switches"). No group → error "Auto selector points at a group that no longer exists"; nothing eligible → "Auto selector has no usable members".
- Ordering/caps: members already in the persisted pool keep stored order; newcomers appended sorted by effective latency (measured ascending > never measured > failed). Truncate to poolCap; the first buildLimit are emitted. `needsRanking` is true only when the eligible set exceeds buildLimit and some chosen members are unmeasured. Ranking candidates = all currently eligible members; unmeasured/stale ones get tested; after a sweep the pool is re-sorted by latency and persisted. A result counts only inside the validity window; zero validity disables reuse.
- Emission (auto-selector outbound, tag `proxy` or `warp-bypass` under Warp): type, tag, outbounds (member tags), url (selector URL else global latency URL), interval/bench_interval/watch_interval (seconds strings), active_size, sampling, tolerance (ms number), expected, dial_retries, interrupt_exist_connections; `warm` (per member {tag, rtt (0 for known-bad), age} inside validity); `pinned` (tag of the pinned member's built hop, only if it made the cut); max_rtt ("<n>ms" only if >0); connectivity_url; balance + balance_mode + balance_interval (s) when on. Empty pool → error "Auto selector produced no usable members".
- Members are full chains themselves: [landing, member, front]; each built under prefix `pool-N`; Xray-backed members get their own bridge; Xray full-config members get their own instance, captured, slot cleared.
- On start, if ranking is needed the app ranks on a worker thread first, then rebuilds; selector-backed Xray sidecars receive `xray_lazy_start` and `xray_idle_seconds = max(120, intervalSec*2)` (full-config instances stay resident).
- Ranking summary shown in the editor: eligible count, "would run" count, skip breakdown, truncation note, reused-results count, and "the rest will be measured before the selector starts."
- Pinning is persisted and takes effect immediately when cleared (saved outside the dialog's accept, with a nudge to the running core).

**Core-config generation**
- Pipeline order: prerequisites scan (route profile's needed outbounds/rule-sets, DNS selectors, hijacked private ranges) → log → NTP → DNS → certificate → inbounds → outbounds (own chain, route chains, bridges, direct) → route → experimental → Xray assembly. Any stage error aborts: message box "BuildConfig return error" + log "Config build error:<text>". Successful builds go to the core over IPC as strings (sing-box config compact, Xray config compact, full configs indented) together with tun CIDR, disable-stats flag, Xray DNS wiring (loopback DNS-in address + domain strategy), and extra-process path/args/conf/no-logs. A standalone sing-box full-config custom profile returns the user's JSON directly as the core config (no injections).
- Reserved tags: `proxy`, `direct`, `warp-bypass`, `dns-remote`, `dns-direct`, `dns-local`, `dns-fake`, `dns-tailscale`, `dns-in`, `mixed-in`, `tun-in`, `hijack`, `hijack-dns`, `throne-bridge`, `throne-adblocksingbox`; prefixes `config-`, `route-`, `pool-`, `proxy-`, `xrayfull-`, `bridge-`. Warp sentinel profile id −2408; special outbound ids: −1 (the profile itself / proxy tag), −2 (direct), warp-bypass id.
- Small sections: log `{level: <global log level>}`; NTP (when enabled): enabled, server, server_port, interval, detour = `proxy` if the NTP-outbound setting says proxy and not a test build, else `direct`; certificate `{store: "mozilla"|"system"}`; experimental: `clash_api` only when clash port > 0 or stats enabled (external_controller = listen address + port, secret, external_ui "dashboard", default_mode ""), `cache_file {enabled:true, store_fakeip:true, store_rdrc:true}` always in runtime builds.
- DNS: a user DNS object passes through unchanged when the use-DNS-object setting is on; else synthesized servers remote/direct/local/fake/tailscale plus `dns-in` (reject rule served by the loopback DNS-in port). Remote DNS is upgraded UDP/QUIC→DoH whenever the proxy path involves Xray. Local address semantics: TUN + "resolved used" → type underlying; TUN on macOS → UDP to the configured underlying DNS; else type local. Tailscale adds MagicDNS rules; localhost predefined rules injected; fake-IP uses `198.18.0.0/15` and `fc00::/18`; final rule binds direct vs remote strategy per the default outbound; cache fields cache_capacity, disable_cache, disable_expire, reverse_mapping, independent_cache. macOS constraint: TUN + empty underlying DNS errors "Local DNS and Tun mode do not work together…". Test builds use only direct-domain rules for tested servers, no remote/fake DNS.
- Inbounds: `mixed-in` unless disabled (+optional username/password when auth on; address/port from settings); `tun-in` when TUN on (stack, MTU, strict_route, auto_redirect on Linux, addresses from tun CIDR, route_exclude_address(+_set) covering direct CIDRs and hijacked private ranges; name "throne-tun"; empty/omitted on macOS); DNS-server inbound (0.0.0.0 or 127.1.1.1) when enabled; `dns-in` at 127.0.0.1:<core DNS-in port>; `hijack` redirect inbound; user custom inbound passthrough; Xray→sing bridge socks inbounds appended after the outbounds section so tags line up.
- Outbounds: own profile single outbound or chain (optional Warp wrapper: Warp carries the entry tag and detours into the rest; for selectors the group takes `warp-bypass` and Warp takes `proxy` detouring to it). Endpoint types (wireguard, tailscale) go to the endpoints array; others to outbounds. Route-driven outbounds built next under `route-` with their own chains (detour linking only when a chain has >1 unique hop). Plain `direct` always appended last. Extra-core process-path rule forces the extra core's executable direct (OS-native separators on Windows).
- Route: with prevent-modifications the raw route object is emitted (outbound ids translated) and synthesis stops. Injected rules: sniff/resolve actions, dns-hijack, redirect-sniff, dns-in reject, extra-core process path direct, bridge-related rules, user rules translated. rule_set array: remote binary entries; `.srs` URLs used directly; other references via the mirror helper; adblock set `throne-adblocksingbox` when enabled. `final`: block id → direct + reject rule; warp-bypass id → `warp-bypass` (or `proxy` without Warp); else the mapped tag string. Optional find_process; default_domain_resolver = {server: dns-direct, strategy: global default}; auto_detect_interface when TUN is on.
- Xray assembly (only when Xray outbounds exist): `log`, `inbounds` (one socks bridge per collected ingress tag, `<xrayTag>-inbound`), `outbounds` (chain via proxySettings), `routing {domainStrategy:"AsIs", rules [{field, inboundTag, outboundTag}]}`. Outbound domain strategy mapping from the direct-DNS strategy: prefer_ipv4→UseIPv4v6, prefer_ipv6→UseIPv6v4, ipv4_only→ForceIPv4, ipv6_only→ForceIPv6, else UseIP.
- Test config differences: no inbounds (except Xray→sing bridge socks), no experimental, DNS limited to direct-domain rules for tested profiles, route = {auto_detect_interface: true, default_domain_resolver: {dns-direct, strategy}}, outbounds = chains (`proxy-<n>` tags, `xrayfull-<id>` for full configs) + direct, endpoints passed through, Xray sidecar from all Xray members with per-chain pre-probed bridge ports (2 reserved per chain).
- Test candidate classification: chains containing an extra-core or Xray-full-config hop are skipped ("Skipping chain with terminal … hop (cannot test)"); extra-core ("Skipping extra-core conf"), Tailscale ("Skipping Tailscale conf") and auto selectors ("Skipping auto selector conf (test its members instead)") skipped; invalid profiles skipped and stamped latency −1. Sing-box full-config customs are handed to the core as-is with inbounds cleared; Xray full configs each become their own Xray instance sharing one sing-box. Result carries per-tag → profile-id mapping for reporting.
- Validation used by start/test flows: sing-box profiles via the core's config check (with log level attached); Xray profiles by building the outbound and checking in Xray mode; Xray full configs only need parseable JSON; chains validate recursively; auto selector via its plan. Failures set latency −1 and log "Invalid …".

**Export surfaces**
- Share links (schemes): socks `socks4://`/`socks5://`; http `http://`/`https://`; shadowsocks `ss://`; shadowtls `shadowtls://`; vmess `vmess://` (base64 JSON); vless `vless://`; xrayvless `vless://`; trojan `trojan://`; hysteria `hysteria://` (v1) / `hysteria2://` (v2); tuic `tuic://`; juicity `juicity://`; trusttunnel `tt://`; naive `naive+https://`/`naive+quic://`; ssh `ssh://`; wireguard `wg://`; tailscale `ts://`; mieru `mierus://` (simple form only); anytls `anytls://`; direct/chain/autoselector/extracore/custom → none.
- Link query parameters mirror sub-objects: dial fields, TLS (security/sni/alpn/fp/pbk/sid/ech_*), fragment + tls-tricks when not default, transport (type/path/host/ed/eh/grpc authority/serviceName), mux, protocol extras. Name = URI fragment.
- Deep link: `throne://add/` + base64url (no padding) of the compact JSON of the protocol's export JSON; the import-side metadata `tag` can be stripped (default true for comparison).
- Config copy on a single profile: build the runtime config, clipboard defaults to compact JSON; buttons re-copy the core config or build+copy the test config (Xray full configs: the first captured Xray config). Errors "Build config error" / "Build Test config error". Output medium is clipboard/message box; no file is written.
- QR dialog: renders the QR bitmap (medium ECC, 2-module white border, black/white) sized to the window, a "Deep Link" checkbox switching between share link and `throne://add` deep link (opens on deep link when requested or when the protocol has no share link), and a read-only text box with the encoded string. Scanning grabs the screen (Wayland portal on Linux), decodes, and feeds each decoded text to the subscription updater.

### C.2 Behavior
See C.1 (emission rules are the behavior contract). Additional dynamics:
- Chain hop ordering is stable across reloads; persisted ids are resolved at build time; missing targets abort the routing section with log "The outbound described in the rule chain is missing, maybe your data is corrupted".
- Auto-selector seed/rerank runs before building the runtime config when ranking is needed.
- Saving a selector stores the normalized config and prunes stale ranking.
- Group landing/front proxies modify chain building as described.

### C.3 Edge cases & errors
- Unknown/empty protocol type → invalid outbound; validation fails with latency −1 and "Invalid …" log.
- Imported Xray full config with unusual inbounds still works via the bridge inbound; user inbounds dropped in test/sole-hop modes.
- Bridge count mismatches abort builds with the exact messages above.
- macOS TUN + empty underlying DNS aborts ("Local DNS and Tun mode do not work together…").
- All-zero mux maxima → global concurrency; maxConnections ⊕ maxConcurrency enforced in the editor.

### C.4 State & config (defaults)
- Global mux-on: off; global mux protocol smux; mux concurrency 8; mux padding off.
- Global fragment implementation ("built-in" vs "custom") and fragment-on default: setting-driven (see §G).
- Xray mux: off; concurrency 8 (global fallback).
- Xray domain strategy default → UseIP.
- DNS/cache/route defaults: see §D.4.
- Share-link/deep-link dedupe strips tag by default.

---

## D. Groups, subscriptions & routing

### D.1 Interface

**Group model** (persisted unless noted): id; archived flag (skip bulk auto-update); skip-auto-update; auto-clear-unavailable; name; subscription URL (empty = manual/basic group — the type is inferred from this); user-info header; last-update time; front-proxy and landing-proxy ids (−1 = none); ordered profile-id list; scroll anchor; sort keys (test/traffic/type); test-items-to-show; column widths (persisted) + computed widths (memory only).
- Groups are tabs; tab order persisted separately, drag-reorder saves immediately, current tab restored.
- Tab context menu: Add new Group / Edit selected Group / Delete selected Group (only when >1 group) / Update subscription (only with URL; guarded while an update runs).
- Edit Group dialog: **Common** (Name; Type combo `Basic`/`Subscription` — subscription section shown only for Subscription; Front Proxy and Landing Proxy editable combos: "None" = −1, entries `[Group] ProfileName` over every profile of every group in group order and profile order, auto-selectors excluded, max 200; checkbox Auto Clear Unavailable Profiles; the Type combo is disabled for existing groups), **Subscription** (URL; Skip automatic update), **Share** (only for a saved group with ≥1 profile: "Copy profile share links" → one share link per line; "(Deep Links)" variant → one `throne://add/<base64url>` per line; both show info box "Copied").
- Accept validation: emptying the URL of an existing URL group is rejected ("Please input URL"); an unknown typed front/landing value falls back to the previous value.
- Group card in the list: type line (Base/Subscription, `Archive ` prefix when archived, ` (n)` profile count), name, URL, sub-info line, buttons "Update Subscription", "Edit", "Remove". Basic groups hide URL, sub info, and the update button.
- Sub-info line: `Last update: <localized datetime>` when non-zero; when the stored user-info header is non-empty and contains `total=`: `Used: %1 Remain: %2 Expire: %3` (total=0 → Remain shown as ∞). Parsing regexes: `total=([0-9]+)`, `upload=([0-9]+)`, `download=([0-9]+)`, `expire=([0-9]+)`; used = upload + download; no `total=` → no traffic line.
- Remove: blocked when only one group exists; prompt `Remove <name>?`; if the running profile's group is removed the profile is stopped/complained first; deletion cascades to profiles and removes the tab-order row.
- First run (no groups): one group named translated `Default`, no URL.

**Subscription request shape**
- URL as typed (trimmed), GET, 10 s transfer timeout, no-less-safe redirects; UA = configured else `Throne/<version>` (build version before the first `-`, `1.0.0` fallback).
- Optional proxy when the app-proxy setting or system-proxy mode is on: HTTP to configured inbound address (`::`→`127.0.0.1`), mixed inbound port, inbound auth when enabled; with no running profile the update fails `Request with proxy but no profile started.`
- HWID headers when enabled: `x-hwid`, `x-device-os`, `x-ver-os`, `x-device-model`, overridable via the custom params list (`key=value`, accepted keys case-insensitive hwid/os/osversion/model, non-empty, <1000 chars, no newline).
- Log lines: `>>>>>>>> Requesting subscription: <name>`, `<<<<<<<< Subscription request fininshed: <name>` (the typo is the product string), error form `<<<<<<<< Requesting subscription <name> error: <error>\n<body>`.
- Response header `Subscription-UserInfo` (case-insensitive) is saved to the group; the group's last-update time is set **before** parsing (even if parsing later fails). Archived groups are skipped entirely.

**Accepted body formats** (first match wins; failing items skipped silently unless noted):
1. Whole-body **base64** (strict; empty result = not base64) → recurse on decoded text.
2. **JSON**: *Xray shapes* — object with `outbounds` containing ≥1 object with `protocol`; or object with `protocol`; or array whose first element has `protocol` or has `outbounds` with a `protocol` entry ("Xray JSON subscription", one profile per array element). *sing-box shapes* — object with `outbounds`/`endpoints`; object with `type`; array of objects with `type`.
   - Xray: outbound object → one profile (`freedom`/`blackhole`/`dns`/`loopback` dropped; `vless` normalized from settings.vnext[0].users[0] when needed; anything else → Custom Xray outbound from the raw JSON; tag as name). Subscription: each element's outbounds (inbounds discarded) becomes a full custom Xray config profile.
   - sing-box: each supported `type` imports as a native profile (socks, http, shadowsocks, vmess, vless, trojan, anytls, mieru, hysteria, hysteria2, tuic, juicity, trusttunnel, shadowtls, wireguard, ssh, naive); unknown skipped; a bare object with `type` → custom outbound; a whole object with `outbounds` → custom full config.
3. **SIP008** (text contains `version` and `servers`): one Shadowsocks per server (remarks → name; `simple-obfs` renamed `obfs-local`); invalid objects logged `invalid server object`.
4. **Clash YAML** (text contains `proxies:`).
5. **WireGuard INI** (contains `[Interface]` and `[Peer]`).
6. Multi-line: split into chunks — a line-initial `{` chunk is taken whole up to its matching brace (brace counting, no string awareness), otherwise to end of line; each chunk trimmed and re-parsed.
7. Comments (`//`, `#`) and strings shorter than 2 chars skipped.
8. `json://` link: fragment must be base64url JSON with `protocol` (→ xray<protocol>) or `type` (native; `hysteria2`→hysteria); invalid outbound dropped.
9. `throne://add/<base64>` (case-insensitive): path minus `/` decoded with standard base64 (not URL-safe); same JSON handling as `json://`.
10. `{...}` text: object with `outbounds` → custom full config; with `server` → custom outbound; else dropped.
11. Share-link prefixes in this fixed order (non-overlapping): `socks5://`, `socks4://`, `socks4a://`, `socks://`; `http://`, `https://`; `ss://`; `vmess://`; `vless://`; `trojan://`; `anytls://`; `mierus://`/`mieru://`; `hysteria://`, `hysteria2://`, `hy2://`; `tuic://`; `juicity://`; `tt://`; `shadowtls://`; `wg://`; `ssh://`; `naive+https://`, `naive+quic://`.

- Shared link base parsing: server = URL host, server_port = URL port (0 kept when port is invalid), name = URL fragment, dial fields from the query.
- `vless://`: UUID = username; default port 443 when absent; query `flow`, `packetEncoding` (default `xudp`; unknown value dropped); a TLS server_name from the query enables TLS; transport/multiplex from standard query names. The Xray implementation is chosen when: VLESS preference = all-VLESS, or `type=xhttp`, or (`security=reality` and preference = XHTTP+Reality), or non-empty `encryption` ≠ none, or non-empty `extra`; otherwise the sing-box VLESS profile.
- `vmess://`: first tries V2RayN-style base64 JSON (`id`, `add`, `port`, `ps`, `aid`, `net` (h2→http; `type=http` forces http), `host`, `path`, `scy`, `tls`/`sni`); else the standard URL form (uuid = username; port default 443; `encryption` default auto; alterId, globalPadding, authenticatedLength, packetEncoding with unknown values dropped).
- `ss://`: traditional form (username = base64url of `method:password`) and 2022 form (username = method, password = password); v2rayN form when the part before `#` has no `@` (base64url of the whole authority → full URL); query `plugin` (supports `simple-obfs;`→`obfs-local;`), `plugin-opts`, `uot`; invalid unless server and method and password are set.
- `trojan://`: password = username; TLS/transport from query; parses whenever the URL is valid.
- `socks*`, `http(s)`, `anytls`, `mieru`, `hysteria` family, `tuic`, `juicity`, `tt`, `shadowtls`, `wg`, `ssh`, `naive+…`: dispatched to their own parsers (per-parameter grammars not enumerated here). `mierus://` is the accepted simple mieru link; the standard base64 form is rejected.

**Clash YAML handling**: body sanitized first (NUL characters removed, leading BOM removed); permissive parse; any exception shows a warning box "YAML Exception" with the parser message (second fallback text "Failed to parse the Clash configuration."). Only `proxies` is read; `proxy-groups` and `rules` are ignored entirely. Supported `type`: `socks5`, `http`, `ss`, `vmess`, `vless`, `trojan`, `anytls`, `hysteria`, `hysteria2`, `tuic`, `ssh`; anything else skipped; per-proxy failures skip that proxy only. `vless` becomes the Xray profile when `network == xhttp` or encryption is non-empty ≠ none; else the sing-box profile. Accepted schema fields (superset): name, type, server, port, cipher, uuid, alterId, udp, tls, skip-cert-verify, servername, network (`ws`,`h2`,`http`,`grpc`,`xhttp`), ws-opts (path, headers, early data), h2-opts (host, path), http-opts, grpc-opts, xhttp-opts incl. download-settings, username, password, sni, alpn, plugin/plugin-opts (v2ray-plugin, obfs), fingerprint, obfs/protocol + params, flow, packet-encoding, reality-opts (public-key, short-id), ports, smux, udp-over-tcp, wireguard fields (ip, ipv6, public-key, pre-shared-key, private-key, reserved, peers, mtu), congestion-controller, udp-relay-mode, heartbeat-interval, obfs-password, idle-session fields, encryption. Field mapping was verified for ss (cipher→method, password, udp_over_tcp, plugin, multiplex), vmess (uuid, cipher→security, alterId, packet_encoding, tls/transport/multiplex), vless (uuid, flow, packet_encoding, tls/transport/multiplex), trojan (TLS always on); others delegate.

**Share/deep links**
- Profile deep link: `throne://add/` + base64url(compact outbound export JSON, no padding); the `tag` metadata is strip-able for dedupe.
- Subscription link: `throne://addsub/` + standard base64 of `URL[#name]`; name = fragment else URL host; prompt shows name and URL with an "Auto update" checkbox (default on; un-checking sets skip-auto-update); group added and fetched immediately.
- Route profile link: `throne://route/` + base64url(compact share object, no `=`); object: `{kind:"throne-route-profile", v:1, name, default_outbound: "proxy"|"direct"|"block"|"warp-bypass"|"unknown", rules:[...]}`; raw variant: `{raw:true, prevent_modifications, route:{...}, outbound_names:{"<sourceId>":"<profileName>"}}`. Import also accepts: raw JSON object, base64url text, standard base64 text, or a legacy bare JSON rule array.
- Remote route link: `throne://remoteroute/` + standard base64 (tolerant) of newline-separated `http(s)://` URLs; per URL: remoteURL = URL without fragment, name = fragment else host; empty/invalid entries skipped; error if none remain.
- Deeplink dispatch (host part, case-insensitive): `add` → treat whole URL as import/update payload; `remoteroute` → remote-route add; `route` → route profile import; `addsub` → subscription add; anything else logs `Ignored deeplink with unknown command: <cmd>`. The route deeplink and the route Import dialog share one parser; `remoteRoute` is recognized case-insensitively by both.

**Routes dialog** ("Routes"; tabs Common, Hijack, Warp, DNS, Route; OK validates and saves):
- Common: Resolve Domain Strategy (tooltip: resolve domains in requests per strategy); Default Domain Strategy (tooltip: default for outbound/dns-server domain addresses); Routing Profile combo; Remote Rule-set Mirror with items `GitHub`, `jsDelivr(Cloudflare)`, `jsDelivr(Gcore)`, `jsDelivr(Quantil)`, `jsDelivr(Fastly)`, `jsDelivr(CDN)`.
- Hijack: DNS Server box (Enable; "How does it work?" help; Allow Lan to Connect; Listen Port — hidden except Linux; IPv4 Response default `127.0.0.1`; IPv6 Response default `::1`; Rules multi-line with autocompletion offering `domain:`, `suffix:`, `regex:`, `ruleset:<name>`); Redirect Settings box (Enable; Listen Address default `127.0.0.1`; Listen Port default 443; fields enabled only when on).
- Warp: Enable Warp (tooltip: egress bypass for services restricting non-Warp VPNs); Endpoint; Private Key; Public Key; Interface Addresses (comma-separated); Reserved (comma-separated); "Generate Warp Config" (tooltip: fetches a new config from the Cloudflare API).
- DNS: **Simple DNS Settings** — Remote DNS (editable combo defaults `tls://8.8.8.8`, `tls://1.1.1.1`, `8.8.8.8`, `1.1.1.1`; tooltip examples `tls://`, `https://`, `tcp://host:port`, `dhcp://auto`, `h3://`, `quic://`), Direct DNS (defaults `localhost`, `223.5.5.5`, `119.29.29.29`, `178.22.122.100`, `77.88.8.8`; tooltip recommends localhost), query strategies for both, Local Override (underlying DNS), Default DNS server combo `remote`/`direct`, Enable FakeIP, Enable DNS Routing (domains destined direct are also resolved with direct DNS), Cache Capacity (default 65536), Disable Cache, Disable Expire, Reverse Mapping. **DNS Object Settings** — Use DNS Object check (toggles simple box off), Format button (pretty-print; invalid JSON → info "Invalid json"), Document button (opens the engine's DNS docs URL), JSON editor placeholder showing the empty-object skeleton.
- Route tab: Routing Profiles list; buttons New / Clone / Export / Import / Edit / Delete / Update. New offers "Structured profile", "Raw profile", "Remote profile" (structured/remote open the structured editor; remote adds a "Remote source" box with URL, auto-update check, Preview and Fetch). Double-click edits; Delete key deletes; Ctrl+C export / Ctrl+V import, scoped to the list. Clone appends ` clone` to the name, added unsaved. Delete refused when only one profile remains: "Routing Profiles cannot be empty, try adding another profile or editing this one". Export copies the selected profile's `throne://route/…` link and shows tooltip "Copied!". Import: clipboard fast path (asks "Import from clipboard?" naming either `a routing rule list` or `routing profile "<name>"`; a remoteRoute clipboard offers adding those remote profiles instead), then a paste dialog titled "Import routing profile" with placeholder "Paste a Throne route link, a remoteRoute link, a base64 blob, or a JSON rule array" (parse warnings shown as "Imported with warnings"; fatal errors as "Could not import this routing profile:" + reason). Update offers "Update selected" (remote only) and "Update all"; progress text "Updating..." / `Updating (%1 / %2)`; results "Update complete" / "Updated %1 remote routing profile(s)." or "Update finished with errors" (`Updated %1, failed %2:` + lines) or "Update cancelled" (`Cancelled: updated %1 of %2, %3 failed.`); nothing to update → "There are no remote routing profiles to update."

**Structured route editor**: Profile Name; default outbound combo populated with `proxy`, `direct`, `warp-bypass`, then `[Group] ProfileName` entries in group order; rules list; add rule (auto-named `rule_<n>`, counter seeded from existing random-numbered names), move up/down, delete; rule name field; action combo; per-attribute tab pages gated by editability, addable via a `+` tab menu (checking shows a tab, unchecking clears that attribute's value; tab selection remembered in-session, not persisted; hint "Check attributes to show as tabs; unchecking clears their values."); rule preview showing built rule JSON (outbound as profile display name); "How to use?" manual; simple-rules tab with four boxes (direct/bypass, block, proxy, warp-bypass), each validated when leaving the tab or saving (invalid lines reported "Some rules could not be added…"/"…fix them before saving"). A profile seeded empty gets one rule named `dns-hijack` (protocol `dns`, action `hijack-dns`) before display. Accept rules: name required ("Cannot create Route Profile with empty name"); remote additionally requires URL; simple-rule validation must pass; an empty structured profile is rejected ("No valid rules are in the profile") but a remote one may be empty; the default outbound is adopted from the combo.

**Raw route editor** ("Raw routing profile"): Name; "Prevent modifications" checkbox (tooltip: use the route object exactly as written; no plumbing added; advanced users only); JSON editor with brace/quote auto-editing and outbound-id autocompletion (popup inserts the numeric id inside `"outbound":`/`"final":` values); live validation label "Valid JSON" / "Invalid JSON: <reason> (offset N)"; "Format JSON" (non-object → "The route must be a valid JSON object"); default template when empty (one `domain_suffix` rule to −2, `final` −1); acceptance requires a name and an object JSON.

**Remote source box**: URL field; "Auto update" check; Preview (throws away: fetches to a copy and opens a read-only window "Remote routing profile preview" showing `<name> — <n> rule(s)`, warnings in red, and the pretty-printed share object); Fetch (asks "This will replace the current rules with the ones fetched from the URL. Continue?" when non-empty; then "Loaded %1 rule(s) from the remote URL." with warning variant); empty/invalid/non-http(s) → "Enter a valid http(s) URL first.".

**Route rule fields** (editor presentation and persistence): ip_version (""|4|6); network (""|tcp|udp|icmp); protocol (""+ sniff list http, tls, quic, stun, dns, bittorrent, dtls, ssh, rdp); inbound (multi-line); domain / domain_suffix / domain_keyword / domain_regex; source_ip_cidr; source_ip_is_private (false); ip_cidr; ip_is_private (false); source_port / source_port_range / port / port_range (ports emitted as numbers when set); process_name / process_path / process_path_regex; wifi_ssid / wifi_bssid (hidden on macOS); rule_set (catalog autocomplete; URLs mapped to stable names); invert (false); action (route, reject, hijack-dns, route-options, sniff, resolve; + `bypass` inserted at index 1 on Linux); outbound (proxy, direct, warp-bypass, then profiles); override_address / override_port (only meaningful for route/route-options/bypass); method (""|default|drop|reply; reject only); no_drop (false; reject only); override_destination (false; sniff only); strategy (""|ipv4_only|ipv6_only|prefer_ipv4|prefer_ipv6; resolve only).
- Emission per action: reject → `reject_method` (+`no_drop` when set); route/route-options/bypass → `override_address`/`override_port` when set and, for route/bypass, `outbound` (tag when building, display name when viewing, raw id if no tag); sniff → `override_destination` only; resolve → `strategy`. Outbound ids −3/−4 are fixed up at build (−3 → reject action, −4 → hijack-dns action). Empty simple-rule stubs dropped; a rule is "empty" if no fields remain (route/route-options/hijack-dns: ≤1 key; others: 0 keys).

**Remote route profiles**: model = isRemote, remoteURL, autoUpdate, remoteLastUpdate, structured rules underneath (raw not supported). Update contract: 10 s GET through the mirror mapping; the response must parse as a structured profile (kind object, base64, raw JSON, or legacy array); a raw response is rejected ("the remote content is a raw routing profile, which is not supported for remote profiles yet"); parsed rules and default outbound overwrite local ones; remoteLastUpdate = now; the remote name is adopted only if the local name is empty. Background job updates sequentially with per-profile logs and summary `Remote routing profiles: %1 updated, %2 failed`.
- Routing menu (rebuilt on open): Enable AdBlock (toggle + restarts a running profile), Enable Warp (same pattern), Download Profiles ▸ (`China`, `Iran`, `Russia` fetching `https://raw.githubusercontent.com/throneproj/routeprofiles/profile/Profile_<Country>` through the mirror mapping and feeding the remote-routes add flow), separator, one checkable entry per route profile (selecting sets the current route and restarts a running profile).

**Built-in defaults**: first run creates a route profile named `Default` with one rule (name "Route DNS", action `hijack-dns`, protocol `dns`); current route id 1.

### D.2 Behavior
- **Update lifecycle**: serial "update all" walks groups in tab order, skipping null/URL-less/archived groups (and, in only-allowed mode, skip-auto-update groups); each update runs on a background thread chained to the next; re-entry refused with log `The last subscription update has not exited.`; there is no cancel. Auto-update jobs: 60 s sweep, first check ~10 s after start; interval sign-encoded (negative disabled; magnitude < 30 min treated as off); last-run epoch persisted so missed sweeps fire on next launch; settings changes trigger an immediate re-check. Single-group update sets last-update and info first, then parses; a failed request logs and leaves existing profiles untouched. Manual single-group updates show the change popup when the setting is on; bulk/auto never do. Batch import: resets imported count, logs `>>>>>>>> Processing subscription data...`, one parser over all payloads, adds only newly-created profiles in order, logs `>>>>>>>> Process complete, applying...`. After an update, the list refreshes and logs `Imported %1 profile(s)` unless quiet.
- **Merge/diff semantics**: sticky set = auto-selector profiles of the group (excluded from the diff, positions remembered, never reported/deleted). `sub_clear`: all non-sticky profiles are deleted before parsing; the running profile is kept when the allow-stopping setting is off (silently dropped from the delete request); if any were kept → cleared=false and the diff continues with survivors; if all requested were deleted → treated as all-new. Diff keys: presence = outbound export deep-link JSON (name included); changed = type + compact identity JSON. Buckets: only-new (add), only-old (delete), common (keep), changed pairs (same identity, different presence — the old id is reused; its outbound and name are replaced; the id is added to a "disturbed" list for selectors). Order: the list is rebuilt in the exact remote order, then sticky ids are re-inserted at their original (clamped) positions. Profiles remaining from the old group but no longer listed are batch-deleted (failure: "Internal error" + "DB Error when deleting profiles, data may be corrupted"); a running profile kept by the delete routine is re-inserted at its previous position and reported `[=] <display name>`. Counters text (logged always; popped up for manual updates when enabled): `\nAdded %1 profiles:\n%2\nUpdated %3 profiles:\n%4\nDeleted %5 Profiles:\n%6` where the lists are `[+] `/`[~] `/`[-] ` + display type/name lines, summarized as `added N`/`updated N`/`deleted N` when a bucket has ≥1000 entries; kept line appended `\nStill in use, so kept instead of deleted:\n%1`; zero changes → `Nothing`. sub_clear full-clear branch: only `[+] ` lines under the same change header. Log header `<<<<<<<< Change of <group name>:`; popup title `Change of <group name>`. Completion signals carry the group id + disturbed profile ids so a running selector can rebuild.
- **WARP generation**: button "Generate Warp Config" → "Getting keypair..." → core key pair → "Generating config..." → HTTPS POST to `https://api.cloudflareclient.com/v0a737/reg` with fields `key` (public key), `install_id` "", `warp_enabled` true, `tos` = current UTC formatted `yyyy-MM-ddTHH:mm:ss.000+00:00`, `type` = OS string (`Windows`/`Linux`/`Darwin`/`Unknown`), `locale` `en_US`; headers User-Agent `WARP for Android`, Content-Type `application/json`; 10 s timeout; same optional proxy rule. Response: `config.client_id` base64 → reserved bytes; the first peer must exist or error `Received invalid response: <raw>`; `config.peers[0].public_key` required; endpoint host from `config.peers[0].endpoint.host`, fallback `engage.cloudflareclient.com:2408`; interface addresses `config.interface.addresses.v4`/`.v6` required. On success fields are filled (addresses `<v4>/32,<v6>/128`; reserved as comma-joined ints) and the button shows "Success!" for 2 s; failures show "Failed to get key pair"/"Failed to generate warp config". When WARP is enabled the proxy path is wrapped (proxy egress through the warp endpoint; warp-bypass maps to proxy when disabled).
- **Route dialog save/validation**: accept requires ≥1 profile ("Routing profile cannot be empty") and valid DNS hijack rules (every non-empty line starts with `domain:`, `suffix:`, `regex:`, or `ruleset:` — else "DNS Rules are not valid", dialog stays open). On accept the in-memory list replaces the DB set: profiles no longer listed are deleted; the current route id becomes the selected profile; settings (strategies, DNS simple/object, mirror, hijack, redirect, warp) are written; then a settings-changed message triggers a save and a "Settings changed, restart proxy?" prompt when a profile is running. Editing works on a copy applied on accept. Deleting a rule moves selection to the previous rule (or the first). Adding a rule names it `rule_<increment>` with the counter initialized from existing random-numbered names; unnamed rules get unique random `rule_<0..999>` names on first open. Simple rules: accepted per line `domain:`, `suffix:`, `keyword:`, `regex:`, `ruleset:`, `ip:` (address rules) and `processName:`, `processPath:` (process rules); trimmed, duplicates ignored, empty lines skipped; invalid → `invalid rule:<raw>` warning; grouped per box with fixed outbounds −1/−2/−3/−5 and actions route/route/reject/route.
- **Fetch behaviors**: route update errors are per-profile and non-fatal for a batch (UI and background both continue). Remote routes are fetched immediately on add (deeplink / Download Profiles), saved regardless, logging `Added remote routing profiles: %1 of %2 fetched` (per-URL failures logged). Remote updates inside the dialog persist only on accept. Preview never mutates; Fetch replaces rules/name only after confirmation.

### D.3 Edge cases & errors
- Invalid Clash body → modal "YAML Exception" warning per failing import call; other items/sections unaffected.
- Individual links/outbounds that fail to parse are skipped silently; a totally unparseable body produces 0 profiles and the normal "Imported 0 profile(s)" flow.
- Base64 decoding is strict; a valid-base64-but-not-intended body decodes and re-parses as text (may silently yield nothing).
- Importing a > 50 MB file is skipped with a log; unreadable files log "cannot be opened"/"not a readable config file"; QR images without a code log "no QR code found"; if nothing imported at all: warning "Nothing could be imported:" + problems.
- Group remove and route profile delete both refuse the last remaining item; group remove additionally refuses while it is the only group.
- Route rules referencing a missing profile: building logs "The outbound described in the rule chain is missing, maybe your data is corrupted" and aborts the routing section (empty result); importing a share with a missing outbound warns and substitutes proxy (`outbound "<name>" not found, using proxy`).
- Legacy rule array on import opens the editor instead of saving (nothing persisted until accepted).
- Invalid/empty remote URL is rejected in the editor and the updater.

### D.4 State & config
- Subscription/route settings (defaults): user agent "" (effective `Throne/<version>`); subscription auto-update disabled at −30 min; last-run 0; clear-first off; change popup on; send-HWID off; custom HWID params ""; allow stopping active off; app use-proxy off; app insecure TLS off; route auto-update disabled at −1440 min; last-run 0; current route 1.
- Route/DNS settings (defaults): remote DNS `https://8.8.8.8/dns-query`; remote strategy ""; direct DNS `localhost`; direct strategy ""; DNS cache capacity 65536; disable cache/expire off; reverse mapping off; DNS routing on; use DNS object off; DNS object ""; final DNS out `remote`; resolve-domain strategy ""; default-domain strategy ""; ruleset mirror = Cloudflare (mirror values: GitHub, Cloudflare, GCore, Quantil, Fastly, CDN); adblock off; fake DNS off; Warp off with empty key/endpoint/address/reserved fields; DNS-hijack server off (LAN off, port 53, v4 response `127.0.0.1`, v6 `::1`, rules empty); redirect off (`127.0.0.1`:443); underlying DNS "" (shown as Local Override); URL-scheme mirror string "".
- Interval encoding rule: sign = enabled flag (negative = disabled), magnitude = minutes; magnitudes < 30 are treated as off by the scheduler (both subscription and routing-profile jobs).
- Group defaults: archived off; skip-auto-update off; auto-clear off; URL/info ""; last update 0; front/landing −1; scroll anchor −1; sort keys: test latency (0), traffic total (0), type by-type (0); test items all (0); column-width list; tab order in a dedicated order table; profiles cascade-delete with the group.
- Route profile defaults: name; default outbound −1 (proxy); raw mode off; prevent modifications off; remote fields off/empty. Route rule defaults per row (ordered positionally): type custom (0); action `route`; outbound −2 (direct); reject method ""; no_drop off; sniff override off; strategy ""; list fields stored as JSON arrays; rules not referenced by any profile are deleted on dialog accept.
- Sort/display preferences (per group): sort method enum (Raw, ByType, ByAddress, ByName, ByTestResult, ById, ByTraffic, BySecurity), descending flag, test sort key (Latency / Download / Upload / IP Out), traffic key (Total / Downloaded / Uploaded), type key (Type / Security), include filter (Out IP / Speed).

---

## E. Core process & IPC protocol

### E.1 Interface

**Artifact and launch**
- One binary: `ThroneCore` (`ThroneCore.exe`), built from a Go module; proxy engines are embedded (internal engine behavior is an external dependency — see Out of Scope).
- The GUI locates it as `<exeDir>/ThroneCore[.exe]` and launches it with **no arguments**, capturing stdout/stderr into the in-app log.
- Environment set by the GUI: `THRONE_CORE_SOCKET=throneIPC-<uuid>` (required), `GOTRACEBACK=crash`, optional `THRONE_CORE_DEBUG=1`, `XRAY_LOCATION_ASSET=<base path>` (Xray geo assets).
- The GUI owns the local endpoint (unix domain socket / Windows named pipe), with access restricted to the current user; each side verifies the other's process identity and refuses mismatches (GUI-side rejection log: `[Warn] IPC connection from unexpected process rejected`).

**Startup stdout contract**
1. Print `sing-box: <version>` and `Xray-core: <version>`, then a blank line.
2. Require `THRONE_CORE_SOCKET`; missing → exit 1, `THRONE_CORE_SOCKET not set`.
3. Parent-identity check: the parent executable must be `Throne`/`Throne.exe` in the same directory as the core (case-insensitive on Windows; reparse points resolved); otherwise fatal `parent check failed: unexpected parent %q, selfPath is %q`, exit 1 (debug builds disable this check).
4. Parent-death watch: Windows waits on the parent process handle (`parent exited: ...` → fatal); unix polls the captured parent PID (signal 0) every 10 s.
5. Connect to the GUI endpoint: 10 attempts × 500 ms; failure → fatal `failed to connect to GUI socket after 10 attempts: <err>`, exit 1. Success → print `Core Has Successfully Connected to Throne!` and start serving.
6. If the GUI connection drops after startup: fatal `IPC connection dropped, exiting`, exit 1 — there is no reconnect.

**Process behavior**
- Recovered panic on the main goroutine: stderr `Core panicked: <value>` + stack, exit **2**. All `log.Fatal*` paths (socket missing, parent check, IPC failures, parent death): exit **1**.
- The GUI treats an unexpected exit as a crash and restarts the core, rate-limited: two exits within 10 s stop automatic restart (`Core exits too frequently, stop automatic restart this profile.`); manual restart = kill, wait 500 ms, start.
- Memory watchdog: 2 GiB process limit; live-heap sample every 2 s; at ≥1536 MiB live → forced GC with memory returned to the OS, plus a 30 s backoff; still above → write a heap profile `throne-core-heap-*.pprof` to a temp path and panic `Live heap reached <N> MiB after a forced GC, this is not normal` (exit 2).
- The binary contains a command-line subcommand tree (subcommands `run` and `version`, configuration flags, a no-color flag) that nothing ever invokes — unreachable dead code.

**Wire protocol** — one long-lived byte stream, exactly one connection for the core's lifetime; custom little-endian framing (not gRPC):
- Request: `u32 reqId, u16 methodLen, method UTF-8, u32 payloadLen, protobuf payload`.
- Response: `u32 reqId, u8 status (0 = OK, 1 = error), u32 dataLen, data`.
- Error `data` is a raw UTF-8 error string (no code enum on the wire). Unknown method → status 1, `unknown method: <name>`. Malformed protobuf → status 1 with the parser error text.
- Reader dispatches each request into its own goroutine; responses serialized by one write mutex; ordering between different ids is not guaranteed. All calls are unary — progress is poll-based (no streaming).
- Handler panic: recovered per request; logged `panic in <method>: ...` + stack; response status 1 `core panic in <method>: <panic value>`; the process stays up.

**Method table** (names case-sensitive, match the proto):

| Method | Request → Response |
|---|---|
| Start | load-config request → error response |
| Stop | empty → error response |
| CheckConfig | load-config request → error response |
| Test | test request → test response |
| StopTest | empty → empty response |
| QueryURLTest | empty → URL-test response |
| QueryStats | empty → stats response |
| QueryConnections | empty → connections response |
| SetSystemDNS | DNS request → empty response (Windows only) |
| IsPrivileged | empty → privilege response |
| GetDefaultInterface | empty → interface response |
| SpeedTest | speed request → speed response |
| QuerySpeedTest | empty → speed-query response |
| QueryCountryTest | empty → country-test response |
| IPTest | IP request → IP response |
| QueryIPTest | empty → IP-query response |
| GenWgKeyPair | empty → key-pair response |
| QueryAutoSelectors | empty → selectors response |
| AutoSelectorAction | action request → error response |

There is no close-connection RPC, no log-stream RPC, no metrics RPC, no restart RPC, and no version RPC.

**Load-config request fields** (Start/CheckConfig): `core_config`, `disable_stats` (accepted, ignored by the core), `need_extra_process`, `extra_process_path`, `extra_process_args`, `extra_process_conf`, `extra_no_out`, `need_xray`, `xray_config`, `tun_ipv4_cidr`, `xray_outbound_dns_address`, `xray_outbound_dns_strategy`, `xray_lazy_start`, `xray_idle_seconds`, `xray_full_configs` (repeated), `xray_full_idle_seconds`.

**Field-presence contract**: proto2-style scalars carry defaults, and the stock GUI serializes every scalar field on every call. The core dereferences several request fields directly (e.g. Start's extra-process flag, SpeedTest's mode flags, SetSystemDNS's clear flag), so a raw client that omits fields triggers the per-request panic path (`core panic in <method>: runtime error: invalid memory address or nil pointer dereference`, status 1) while the process survives. A reimplementation should either tolerate absent fields (preferred) or replicate this contract exactly. Message/repeated fields (`xray_full_configs`, `outbound_tags`, `tests`) may legitimately arrive empty; all response scalars are always set.

**Client-side call semantics** (GUI): default I/O timeout **30 000 ms** per call; `QueryStats` overrides to **500 ms**; success code 0; not-connected code `-1919`; error payload surfaced to callers, which inspect substrings (`Fwpm`, `configure tun interface`, `Instance is not running`).

### E.2 Behavior (per RPC)

- **Start(load-config)**: all under one lifecycle mutex. (1) A box already running → error `instance already started`. (2) Extra process when requested: shlex-split args (`Failed to parse args: <cause>`); if an extra-process config is given, securely create a temp config file (`Failed to create extra.conf: <cause>`) and substitute its path into the **first** arg containing `%s`; start the child. (3) Compute the auto-redirect mark from the core config (Linux tun setting `auto_redirect_output_mark`, engine default; 0 elsewhere). (4) Xray sidecar when requested: lazy → build a gate that takes over the original loopback inbound ports and starts the instance on first dial; eager → create, apply egress (default interface from the monitor), optionally wire an outbound DNS resolver (`xray_outbound_dns_address` + strategy; error `failed to create Xray outbound DNS resolver: <cause>`), start. (5) Each `xray_full_configs` entry becomes its own gate with `xray_full_idle_seconds`; the first config is validated alone (geo warm-up), the rest in parallel up to CPU count. (6) Build and start the sing-box box from `core_config` (error prefixes: `decode config at <raw config text>: `, `create service: `, `start service: `). (7) darwin only: with a non-empty tun CIDR, set system DNS to the next IP of the CIDR (parse errors `invalid tun_ipv4_cidr "<v>"` / `got invalid DNS IP from tun_ipv4_cidr: <ip>` tear the whole stack down and fail Start; DNS-set failure is logged only, flag set for unset-on-Stop). On any error the box pointer is cleared and the auto-redirect mark reset. `THRONE_CORE_DEBUG=1` logs full configs. Verified quirks: a duplicate Start not only returns `instance already started` but also clears the internal box pointer (the running box is orphaned; a later Stop becomes a no-op); a Start that fails after the extra process started does not stop that child on all paths, and the next Start overwrites the handle.
- **Stop(empty)**: no box → success no-op. darwin: unset DNS if Start had set it (failures logged). Unpublish the box first, close with a 2 s blocking timeout, stop the extra process, close the Xray sidecar and all gates, reset the mark. Always success; teardown problems are logged, not surfaced.
- **CheckConfig**: `need_xray` → validate the Xray config with the Xray parser and return its error text; else parse + construct + immediately close a box from `core_config` (`decode config at <config text>: `-style errors). Panics recovered locally (`CheckConfig panic: <value>`); never kills the core; does not require or disturb a running instance.
- **Test**: fields config, outbound_tags, use_default_outbound, url, test_current, max_concurrency, test_timeout_ms, need_xray, xray_config, xray_full_configs. `test_current` measures the running box; tags forced to `proxy` if present else the default outbound; nothing running → **success** with one result `{outbound_tag:"proxy", latency_ms:0, error:"Instance is not running"}`. Else a throwaway box (+ optional Xray sidecars) is built and URL probes run over the tags (or default outbound); non-current batches are probed **twice** and the second measurement reported (mux warm-up); timeout default 3 s when `test_timeout_ms<=0`; concurrency normalized (≤0 or ≥500 → 100); probes staggered 2 ms apart. Per-tag failures stay in `error`; cancellation marks pending tags `test aborted`.
- **StopTest**: cancels the in-flight probe session (aborting pending dials) and immediately re-arms a fresh one; no error.
- **QueryURLTest**: **drains** the accumulated URL-test buffer (one result per tag: outbound_tag, latency_ms, error); multiple consumers are not supported.
- **QueryStats**: empty maps with no box or no clash server; otherwise `ups`/`downs` keyed by every outbound and endpoint tag from the clash traffic manager; counters are **drain-on-read** (the GUI consumes per-poll deltas). Failures: `invalid clash server type`, `nil outbound manager`, `nil endpoint manager`.
- **QueryConnections**: no box → empty success; no clash server → error `no clash server found`; unexpected type → `invalid state, should not be here`. Returns `active` plus a `closed` ring (non-draining; consumers dedup by id). Per connection: id (UUID), created_at/closed_at (unix ms; 0 while live), cumulative upload/download, outbound, network, dest (host:port), protocol, domain, process (basename), process_path (full), chain[]. Ring size is engine-defined. There is no RPC to close a connection.
- **SetSystemDNS(Windows only)**: clear=false applies the local DNS mark and interface DNS; clear=true restores the previous state (DHCP vs manual distinguished by marker addresses). Errors: `No dns Manager, you may need to restart Throne`, `Default interface is nil!`. Off-Windows the not-implemented server method is invoked (error text inferred from generated gRPC stubs: `rpc error: code = Unimplemented desc = method SetSystemDNS not implemented`). Absent `clear` → nil-deref panic response.
- **IsPrivileged**: Windows always false (elevation tracked GUI-side); elsewhere euid == 0.
- **GetDefaultInterface**: from the always-on interface monitor (excludes TUN/loopback); error `no default interface` when none known yet.
- **SpeedTest**: fields config, outbound_tags, test_current, use_default_outbound, test_download, test_upload, simple_download, simple_download_addr, timeout_ms, only_country, country_concurrency, need_xray, xray_config, xray_full_configs. At least one of download/upload/simple/country must be true, else transport error `cannot run empty test`. `test_current` mirrors Test. Modes: concurrent download/upload (50 ms sampling, EWMA speeds, human-readable dl/ul strings); simple single-file download from the address (timeout ≤0 → 3 s); country-only server benchmarking (server-list fetch 8 s; concurrency ≤0 → 5). Results: dl_speed, ul_speed, latency, outbound_tag, error, server_name, server_country, cancelled, dl_bytes, ul_bytes. StopTest cancels the run and marks partial results `cancelled`.
- **QuerySpeedTest**: **snapshot** (non-draining) `{result, is_running}`; before any test the result is an empty/zero result with is_running=false; status 0.
- **QueryCountryTest**: **drains** the country-only result buffer (same result shape).
- **IPTest**: always builds a fresh box (no current variant); probes an IP-geolocation HTTP API; timeout ≤0 → 3 s; concurrency normalized as in Test. Result fields outbound_tag, ip, country_code, error; unknown tag → per-tag error `no outbound with tag <tag> found`.
- **QueryIPTest**: **drains** the IP-test buffer.
- **GenWgKeyPair**: WireGuard X25519 pair, base64 strings; failures are returned in the `error` field with status 0.
- **QueryAutoSelectors**: idempotent, non-draining snapshot of all running selector groups. Per group: `tag`, `phase` (`starting|probing|ready|suspended`), `selected`, `selected_udp`, `pinned` (empty = fully automatic), `balance`, `balance_mode`, `suspended` + `suspended_since_ms` (suspended = local network believed down; members must not be treated as broken), member counts (total/probed/alive/qualified/cooldown), `probes_in_flight`, `rounds_completed`, `last_round_ms`, `next_round_ms`, `last_switch_ms`, `last_switch_reason`, and `members[]`: tag, rank, state (`ok|degraded|untested|dead|cooldown`), selected, selected_udp, qualified, active (fast-probe tier), average_ms, deviation_ms, min_ms, max_ms, samples, failures, probes, dial_total, dial_fail, last_ok_ms, last_probe_ms, cooldown_until_ms, last_error. Empty list when no box runs.
- **AutoSelectorAction {tag, action, member}**: no selector → `no auto selector is running`; non-empty unmatched tag → `no auto selector with tag <tag>`; empty tag = apply to all groups. `recheck` → force a full sweep now; `select` → pin to `member` (empty member releases); unknown member → `no member <member> in group <group tag>` (first failing group wins); other action → `unknown auto selector action: <action>`. Ranking/cooldown policy internals are engine-side.

**Engine integration**
- **sing-box is embedded in-process**: each box is built from the JSON string in `core_config` with a fresh service-registry context (concurrent boxes must not share one). Tear-down order is fixed: services → endpoints → inbounds → outbounds → router → connection manager → dns-router → dns-transport → network → internal services → logger; Stop uses a 2 s blocking close timeout and logs (not surfaces) close problems; a late "sing-box did not closed properly" may be logged. Capability set comes from build tags (clash API, gvisor, QUIC, WireGuard, uTLS, DHCP, Tailscale, naive outbound, purego). Traffic stats and connection metadata exist only when the config includes the clash API service (the GUI writes it in).
- **Xray sidecar**: Xray-format configs are for Xray-shaped protocol stacks the GUI merges into a sidecar that sing-box dials through a socks outbound. Creation: parse + build + create + optional outbound-DNS resolver + egress + start. Lazy gate: requires inbounds with fixed ports; the gate takes over the original loopback ports and remaps the Xray inbounds to ephemeral same-host ports, starting the instance on first connection; sweep interval = `min(15 s, max(2 s, idle/2))`; `idle_seconds = 0` keeps the instance resident once started. Log lines: `Xray gate: started the instance on demand`, `Xray gate: the instance went idle, shutting it down`, `Xray gate: failed to bring up the instance: <e>`, `Xray gate: failed to reach the instance: <e>`; gate errors are prefixed `xray gate: ` (`cannot take over <addr>: <e>`, `parse config: <e>`, `parse inbounds: <e>`, `config has no inbounds to gate`, `inbound without a fixed port cannot be gated`, `no free port on <host>: <e>`, `closed`).
- `xray_full_configs` entries are opaque, each backed by its own gate/instance, never merged with the sidecar; separate idle settings. Geo asset loading uses `XRAY_LOCATION_ASSET`; asset-missing errors surface raw and are handled by GUI messaging.
- **Extra process (supervised child)**: args shlex-split; the first arg containing `%s` receives the path of a securely created temp config (unpredictable name, restrictive permissions; on elevated unix readable only by the target uid; Windows uses a per-user temp dir). Child environment = parent env minus every `THRONE*` variable. stdout/stderr forwarded line-by-line prefixed `Extra Core:` unless the no-output flag is set. De-privileging: a setuid-root unix core spawns the child as the real unprivileged uid (refusal `refusing to start extra process as root: no unprivileged user to drop to`); Windows uses `CreateProcessWithTokenW` with an unelevated user token (refusal `refusing to start extra process: cannot obtain an unprivileged user token: <e>`). Child exit is observed but non-fatal (`Extra process exited unexpectedly`); no automatic restart; the temp config is removed on Stop/exit; orphans are not hunted down.
- **No hot reload**: switching configs/profiles is Stop + Start driven by the GUI; there is no reload RPC. "Engine switching" is not a core concept — sing-box is always the routing engine; Xray is an optional sidecar per profile.
- **Egress**: the live sidecar and eager full instances get the default-route interface name + auto-redirect mark applied at start; the interface-monitor callback re-applies egress to all live Xray instances when the default route changes; test instances get a one-shot egress (no tracking).

**Platform services**
- **Default-interface monitor**: created at package init on every platform, always running, excluding tun/loopback interfaces; powers GetDefaultInterface and Xray egress tracking; failure to create/start is logged; unavailable → `no default interface`.
- **Windows system DNS**: managed with three marker addresses `127.1.1.1` (local resolver), `127.1.2.3` ("was DHCP"), `127.3.2.1` ("was manual"). Set installs the mark and applies DNS to the default interface; clear restores the prior server list, distinguishing DHCP vs manual by the marker; the resolver cache is flushed after changes; interface changes restore per interface (LUID). Registry-based reads of interface name servers with a command-line fallback. Errors listed above.
- **darwin system DNS**: only on Start and only with a non-empty tun CIDR: runs `/usr/sbin/networksetup -setdnsservers <hw port> <next IP of CIDR>` for the default interface; Stop unsets with `Empty`. Failures logged, never fatal; unset happens only if Start set it.
- **Linux**: no DNS writing; the only platform-specific Start behavior is reading the auto-redirect mark from the tun config, used for Xray/test egress exemptions; reset to 0 on Stop.
- **Privileges**: the core runs setuid-root on unix distributions and may run elevated on Windows; both extra-process launch paths deliberately de-privilege.
- **Supervision (process level)**: parent-identity gate, parent-death watch, bidirectional peer-PID verification, IPC-drop fatal, memory watchdog, GUI-side crash detection/restart limits.

### E.3 Edge cases & errors
- Bad configs: CheckConfig returns the parse error and never crashes; Start fails with the builder's prefixed error and unwinds extra process/Xray/box.
- Duplicate Start: `instance already started` **and** box-pointer cleared (orphaning the running box; later Stop is a no-op) — compatibility-relevant quirk.
- Stop when not running: success no-op; Stop failure paths logged only.
- Missing extra-process binary: Start fails with the OS exec error; temp file cleaned.
- Port conflicts: surfaced from box start or gate takeover (`xray gate: cannot take over ...`); never pre-detected.
- Engine crash: sing-box runs in-process; an unrecovered panic exits the process (main recover → exit 2) and the GUI restarts it, rate-limited. Xray failures are logged; gated instances retry on next dial. Extra-process exit is non-fatal.
- IPC drop → exit 1 (no reconnect). Client misuse (omitted scalar fields) → per-request panic response; process survives. Panics inside spawned test goroutines are outside the dispatcher's recover (risk mitigated by pre-checks).
- `disable_stats` is accepted and ignored. Tests use throwaway boxes and do not affect the running profile.
- UNKNOWN (checked, not resolvable): closed-connection ring size/eviction (engine-side); selector ranking/cooldown/probing policy internals (engine-side); exact off-Windows not-implemented string (generated stubs not committed); signal-induced exit codes (no signal handlers installed); whether the dead CLI ever shipped.

### E.4 State & config
- The core has no config directory, data directory, or log files of its own: configs arrive in RPC payloads; stdout/stderr are captured by the GUI; the working directory is never changed. It writes only temp files (extra-process config removed on Stop/exit; heap profile on watchdog trip) and OS DNS state (Windows registry/DNS, darwin networksetup).
- Defaults: URL/IP test timeout 3 s; test batch concurrency ≤0 or ≥500 → 100 (cap 100); probes staggered 2 ms; non-current URL batches probed twice; speed test: simple-DL timeout ≤0 → 3 s, country server-list fetch 8 s, country concurrency ≤0 → 5, 50 ms sampling; gate sweep `min(15 s, max(2 s, idle/2))`, idle 0 = resident; teardown close timeout 2 s; connect retry 10 × 500 ms; parent poll 10 s; heap sample 2 s; forced-GC backoff 30 s; memory limits 2 GiB / 1536 MiB.
- Exit codes: 1 (fatal paths above), 2 (recovered panic, watchdog panic).

---

## F. Client RPC, statistics, connections & testing

### F.1 Interface

**Channel obligations** (the GUI's side of the protocol in §E.1)
- One persistent full-duplex local-socket channel to the core; call/response with monotonically increasing request ids; framing exactly as §E.1 (little-endian).
- Default call timeout **30 000 ms**; `QueryStats` overrides to **500 ms** (polled at 1 Hz; must never stall the traffic loop).
- Not-connected failures carry code **-1919**; every such failure logs `IPC call failed (code %1)`. Core-side error text is logged as `[Core error] …`. Malformed/oversized frames are dropped with `[RPC] dropped malformed response: …` — observably the pending call then times out, not a distinct error.
- The GUI owns the endpoint: a local server named `throneIPC-<uuid>` per app run, handed to the core via `THRONE_CORE_SOCKET`; it verifies the connecting peer's process id per OS and rejects mismatches with `[Warn] IPC connection from unexpected process rejected`. Reconnection swaps the socket underneath a permanently-running I/O thread; the client object is never destroyed; pending calls fail on disconnect or timeout.

**Threading** — RPC I/O on its own thread; the statistics loops (traffic, connections, autoselector) are started once for the app lifetime (a latch prevents double start); UI updates are marshalled to the app thread; batch testing uses a shared pool capped at **10** concurrent core calls.

**Wire methods used** (exact names; full semantics in §E):

| Method | Client use | Cadence / timeout |
|---|---|---|
| `Start` / `Stop` | profile start/stop (serialized config payloads) | on demand / 30 s |
| `QueryStats` | cumulative per-tag byte counters | 1 Hz / **500 ms** |
| `QueryConnections` | connections view | 1 s visible, 5 s relaxed / 30 s |
| `QueryAutoSelectors` | autoselector monitor | 2 s / 30 s |
| `AutoSelectorAction` | pin / release / check-now buttons | on demand / 30 s |
| `Test` | test runner: per-profile full configs; aggregated URL/IP batches; speed requests; `test_current` egress probe | batch-driven / 30 s |
| `StopTest` | cancel a sweep / profile stop | on demand / 30 s |
| `CheckConfig`, `SetSystemDNS`, `IsPrivileged`, `GetDefaultInterface`, `SpeedTest`, `QuerySpeedTest`, `QueryCountryTest`, `IPTest`, `QueryIPTest`, `GenWgKeyPair` | settings flows, runtime dialog, WARP generation | on demand / 30 s |

Notes: `QueryAutoSelectors` is idempotent and safe to poll alongside `QueryStats`. The connections `closed` ring is **not draining** — the client deduplicates by connection id itself. The connection `chain` field is declared but consumed by no client UI.

**Cadences** (all GUI side):
- Traffic poll 1000 ms; traffic persist to profile rows every 10 ticks (≈10 s) + once on stop; minute-bucket flush on rollover; background downsample every 10 min; minute tier kept 48 h fixed; hour-tier retention = retention setting (min 1, default 90 days).
- Connections poll 1000 ms while visible / 5000 ms relaxed; minimum 500 ms between speed samples.
- Autoselector poll 2000 ms; exhaustion grace 20 s; rebuild backoff 60 s → ×2 → cap 600 s (reset once usable > 0); health persistence 60 s.
- Test runner: latency result poll 200 ms; speed result poll 100 ms; batches of 100 profiles; ≤10 concurrent core calls.
- Runtime stats refresh 1000 ms; egress probe at most every 30 s until first success, then only on config change.
- Profile-start prompt 10 000 ms; profile-stop prompt 5000 ms.
- Log UI flush at most one in flight; file flush 250 ms (immediate for Warn+); document block cap = max log lines (default 200); rotation 4 MB × 3; pre-init buffer 2000 lines.

**Settings governing this scope** (defaults): latency-test URL `http://cp.cloudflare.com/`; URL-test timeout 3000 ms; concurrent tests 10; speed-test timeout 5000 ms; simple-download URL `http://cachefly.cachefly.net/1mb.test`; speed mode `FULL` (enum order FULL, DL, UL, SIMPLEDL, COUNTRY); connection statistics **on** (adds process-name lookup + stats API to the generated config; live gate for the connections list); per-connection polling on; traffic aggregation on; max log lines 200; include/exclude log filters off with empty lists; auto-scroll on; disk log level `debug`; subscription auto-update −30; route auto-update −1440.

**Display formats** (client commits to these):

| Surface | Format |
|---|---|
| Status bar (running) | `Proxy: %1\nDirect: %2`, each `⏎<up>↑ <down>↓` |
| Status bar (stopped) | `STOP` and graph reset |
| Connection Traffic column | `<up>↑ <down>↓` (binary KiB/MiB/GiB, 2 decimals) |
| Connection Speed column | `<up>/s↑ <down>/s↓` |
| Connection Destination | `dest (domain)` when a distinct domain is present |
| Connection Protocol | `<network> (protocol)` when protocol non-empty, else network |
| Connection Process | process basename only |
| Profile test result | country flag + `Unavailable` (latency<0) or `<n> ms` (latency>0) + optional ` ↓dl` ` ↑ul` + optional ` 🌐ip` (per group display settings) |
| Profile traffic cell | empty at zero, else `⏎<up>↑ <down>↓` |
| Latency color | <0 dark gray; 0 no color; ≤100 dark green; ≤300 dark yellow; >300 red |
| CPU/RAM values | `"<pct>%"` and human size; unknown CPU shows `—` |
| Connections counter | `%1 active · %2 TCP · %3 UDP` |
| Chart y-axis | "nice" ceiling using 1/2/5×10^n steps |
| Stats-dashboard tooltip | `↓ … ↑ … Σ …` with local date/time |
| Update schedule | `Disabled` (interval < 30), `Due now`, `in <humanized>` |

### F.2 Behavior

**RPC mechanics** — calls are serialized on the single channel; a timeout or disconnect fails the call with code −1919 + the log line; no automatic retry for one-shot calls; the only non-core HTTP checks the client itself performs are the runtime egress probe and the autoselector connectivity fallback.

**Traffic loop** — every 1000 ms; when per-connection polling is off the iteration is skipped entirely (no rate computation, no UI updates; the UI is not zeroed by the loop in that state). With no profile running, each tick ensures a one-time `STOP` refresh and pushes zeroed graph values; enabling resumes silently. Rate = delta of cumulative bytes since the previous tick ÷ actual elapsed ms × 1000 (robust to poll jitter). At profile start the client seeds credit groups from the generated config's chain groups: each group = the watch tag the bytes land on after chain/warp transformation + the list of profiles to credit; `proxy` and `direct` aggregate entries are created up-front; `direct` bytes are never attributed to profiles. Every profile in a group receives the group delta; profiles appearing in multiple groups deduplicate at persist time. Persistence: every ~10 s only dirty groups are written (one batched transaction per profile); on stop a final persist + aggregation flush runs synchronously. UI: one batched refresh per tick for the unique credited profile ids, plus status-bar text and graph values. Historical metadata (group name, profile name) is snapshotted at seeding so history survives renames/removals.

**Aggregation database** — separate stats file next to the main DB; two tiers: minute (48 h fixed) and hour (retention setting). The partial minute bucket is flushed on rollover and on profile stop; a background downsample runs every 10 min; buckets align to the **local timezone**; dashboard periods are Last 24 h (hourly buckets), 7 d, 30 d, 90 d (daily buckets). Attribution is keyed by process basename (unknown → `Unknown`); the app's own speed-test traffic is credited to a synthetic `Speedtest` identity so it does not pollute normal stats. The dashboard shows the top 9 apps plus an `Other` row; download blue `#4F8AF7`, upload green `#34C98A` (down drawn under up); tooltips `↓ … ↑ … Σ …` with local date/times.

**Status bar & traffic graph** — two aggregate rates exist (proxy, direct); when stopped the status text is exactly `STOP` with one zeroed graph push per tick while the loop is idle. The graph keeps a 120-sample rolling window, auto-scales to a 1/2/5×10^n "nice" ceiling, uses two fixed pens (up/down), and has no tooltip by design.

**Connections view** — polling cadence per F.1; suspended while no profile runs; skipped entirely whenever connection statistics are off (checked every poll, so the gate is live for the list, while config-side process attribution only changes at the next profile start); cleared immediately on stop (full recreate with an empty list). Dedup: `active` entries keyed by connection id; entries in the non-draining `closed` ring are accounted exactly once so a finished connection is removed once and never re-added. Sorting: header click on column 0 = `Default` (reset to incremental mode); 1 = ByProcess; 2 = ByProtocol; 3 = ByOutbound; 4 = ByTraffic; 5 = BySpeed; right-click on a sorted Traffic/Speed column offers sub-keys (Total/Downloaded/Uploaded; Total/Download Speed/Upload Speed); re-clicking the same header flips asc/desc; selecting a new field starts descending; all modes tie-break by connection id. Row strategy: Default performs an incremental diff (add/update/remove); every explicit sort mode does a full recreate; Default is the only mode that preserves selection/scroll cheaply. Single click on any cell copies its rendered text and shows a `Copied!` tooltip for 1.5 s. No connection actions exist client-side (no close/close-all, no search/filter) — read/copy/sort only. A failed query logs `Failed to query connections: IPC error` and leaves the list as-is.

**Autoselector monitor** — exactly one selector per running config (the first encountered) is monitored. Poll 2000 ms; phase machine `starting → probing → ready`, `suspended` when the profile stops; member states `ok | degraded | untested | dead | cooldown`. Display: summary line + a member table with columns `#`, `Profile`, `Status`, `Latency`, `Jitter`, `Checks`, `Connects`, `Last OK`, `Notes`; unknown values sort last; a "only show profiles with problems" filter; buttons `Use this profile` (pin), `Back to automatic` (clear pin), `Check all now`. Exhaustion decision: `usable == 0 && membersProbed > 0 && !suspended && phase != starting` for ≥ 20 s → first an independent OS default-route check: with no default route the client logs `[Auto selector] Every profile is failing, but this machine has no network connection — keeping the current pool.` and does **not** rebuild; otherwise rebuild backoff begins at 60 s, doubling per consecutive failure up to 600 s, resetting as soon as `usable > 0`. Rebuild = mark the last built member set stale, re-rank candidates, restart the profile with the new pool; member disturbances (profiles built by the selector changing while selected) also trigger a rebuild. Health persistence every 60 s: for members with measurements, write average latency (or −1 when dead) into the member profile's latency field; skip unmeasured; write only when the value changed. Pinning stores the member id on the selector profile and sends the action; pool/build/history state is persisted across restarts.

**Test runner** — autoselector profiles are suppressed for URL/IP tests and for speed tests not on the current profile; only runnable profiles are included. Batching: URL/IP tests in batches of 100 profiles (one full-config call per profile plus one aggregate tagged call per batch, ≤10 concurrent); country-mode speed also batches 100, while full/DL/UL/simple modes step profile-by-profile. A session mutex blocks a second sweep (warning box). `StopTest` cancels; results already applied stay applied. Latency result mapping: empty error → latency = value (displays `<n> ms`); error containing `test aborted` or `context canceled` → latency = 0 (never measured; previous display state kept rather than showing failure); any other error → latency = −1 + log `[%1] test error: %2` (displays `Unavailable`). Measurement time is stamped unless 0. IP tests: success sets out-IP + country (flag shown); failure clears both. Speed tests: success sets the dl/ul display strings; failure sets them to `"N/A"` and latency to −1. Result application contract: −1 = failure, 0 = neutral (no update), >0 = value — so partial batch results never clobber good data. Progress panels (HTML) build while a sweep runs and clear on finish. Auto-clear-unavailable groups remove failed profiles after a URL test. Speed-test byte totals are credited to the profile counters and to the synthetic `Speedtest` bucket. A test failure caused by missing geo assets triggers the interactive download/retry prompt.

**Runtime stats dialog** — UI refresh 1 s; samples the GUI process and the core process (core pid from the main window); the first CPU sample per process is 0; CPU percent is normalized by logical CPU count; RAM human-formatted; uptime from the app-start epoch; database size = sum of the throne DB files in the working directory. Two mini-charts (CPU, RAM), 60 samples, niced axis, primary color `0x4F8AF7`, core color `0x34C98A`. Connections counter fetched off-thread once per refresh with a busy latch. Egress probe: `test_current` latency against the test URL, then a request to `http://ip-api.com/json/` to resolve the observed egress; runs once per config change and at most every 30 s until a success, then stops until the next config change. Update schedules shown as `Disabled` / `Due now` / `in …`.

**Log pipeline** — sources: core process stdout/stderr and UI-issued lines (UI lines also go to the file log with a `[ui]` prefix). Ingest queue cap 1000 (overflow incoming lines are dropped); any single line longer than 20 000 chars is truncated to its first 1000. Writer filters in order: include list → include regex; exclude list → exclude regex; empty/whitespace lines dropped; keywords split on newlines/trimmed; regexes validated before acceptance. Pending UI buffer capped at 2 MB with at most one flush in flight (backpressure, not unbounded memory). Document block cap = max log lines (default 200; old lines evicted as new arrive). Regex-based syntax highlighting for date/time, levels, accepted/rejected, `tcp:`/`udp:`, `IP:port`, `host:port` (dark/light variants). Auto-scroll anchors to the top visible block when enabled. Context menu = standard text actions + `Clear` (also clears the pending buffer). File logging per §A (rotation, flush cadence, marker, crash-log preservation). Compatibility quirk: a global counter permanently stops core-stdout logging once it exceeds max log lines in a session — a reimplementation that streams stdout forever differs observably in long sessions with verbose cores.

**Profile lifecycle surfaces** — Start shows Connecting; a 10 s timer opens "If there is no response for a long time, it is recommended to restart the software." (Yes → restart flow); the prompt is cancelled once start resolves. If the core is not up, the client logs `Try to start the config, but the core has not listened to the RPC port, so restart it...`, arms a start-when-core-up flag, and restarts the core instead. On success: chain groups seeded, traffic loop enabled, connections lister unsuspended, selector build recorded. Stop: guarded by a mutex; the view clears immediately; selector monitor cleared; traffic loop does a final update, persists, and flushes aggregation; a 5 s timer shows the same prompt during teardown; failure logs `Failed to stop, please restart the program.` while the button shows Disconnecting. Core exit: `[Fatal] Core exited, cleaning up...`, crash-mode profile stop, restart with rate limiting — two exits within 10 s → `[ERROR] Core exits too frequently, stop automatic restart this profile.`, otherwise `[Warn] Restarting the core …` after a 200 ms delay. Other error surfaces: `LoadConfig return error` + core text on config load failure; Tun prompts when the error contains `Fwpm` (offers disabling strict route, warning `Disabling Strict Route may cause DNS leaks.`) or `configure tun interface` (offers resetting the core process); system proxy/DNS actions surface `IPC error`.

### F.3 Edge cases & errors
- A garbage frame is observably a timeout (caller-visible), not a distinct error class.
- A foreign peer is rejected and logged; the connection stays down until the core dials again.
- `QueryStats` timeouts under load must never block the traffic loop (500 ms cap).
- Cancelled tests apply neutrality (0), not failure; previously applied results stay.
- Speed-test failure marks both speed cells `N/A` and latency −1.
- A selector with an empty pool after stop shows a neutral empty state, not an error.
- Log overflow drops lines silently at the queue; over-long lines truncate to 1000 chars.
- No per-connection actions, no tray balloons, and no consumption of the connection `chain` field exist anywhere in the client (verified) — must-keep compat facts.

### F.4 State & config
- Per-profile fields written by this scope: latency (>0 measured; 0 never measured; <0 unavailable/dead), measurement timestamp (cleared when latency is set to 0), download/upload speed display strings (`"N/A"` on failure), out-IP and country (cleared on failure), lifetime traffic counters (≈10 s cadence + stop), selector pinned id, selector build state + timestamp + accumulated history.
- Stats DB and log files as in F.1/F.2 and §A.
- Defaults recap: see the tables in F.1.

---

## G. Database, settings & persistence

### G.1 Interface

**Storage engine and files**
- SQLite; two databases are always created together on startup: the main DB `throne.db` in the working configuration directory, and the stats DB `throne_stats.db` in the same directory (its path is derived from the main file's absolute path).
- Connection pragmas (applied at open): incremental auto-vacuum — **stats DB only**, and only before the journal mode is set (otherwise it silently has no effect); foreign keys ON; WAL journal; synchronous NORMAL; 64 MiB mmap; 5 s busy timeout. A truncated WAL checkpoint runs after 10 000 accumulated writes per database and again during deferred maintenance.
- Write batching limits are observable in behavior: settings saves chunk at 750 key/value pairs (bind limit 1500); profile batches chunk at 115 rows (13 parameters per row).

**Data model** (names are implementation-private; these semantics are the contract)
- Profiles: monotonic id (never reused); kind; name; owning group (delete cascades); latency + measurement timestamp; download/upload speed strings; test country; out-IP; serialized outbound payload (includes per-profile option overrides and all auto-selector pool/pinning state); lifetime traffic counters; created/updated timestamps.
- Groups: archived flag; skip-auto-update; auto-clear-unavailable; name; subscription URL (empty = manual group); subscription user-info; last-update timestamp; front/landing proxy ids; column-width cache; **ordered profile-id list (this list — not the profile's group column — defines membership order)**; scroll anchor; sort keys (test/traffic/type); test-items display; timestamps.
- Tab order: a separate ordered list of group ids.
- Route profiles: name; default outbound; raw mode + raw route JSON; prevent-modifications; remote source URL / auto-update / last-update; timestamps.
- Route rules: ordered positionally per profile (wholesale delete-and-reinsert on save; cascade with the profile); every rule attribute from §D.1, with list fields stored as JSON arrays even for single values.
- Settings: a single key/value store; all values are text (see G.4).
- Id counters: one row holding per-entity last-ids; ids are allocated by increment and never reused after deletes; restores raise the counters.

**Id conventions referenced by rules and links**: none-profile −1919; outbound specials −1 proxy, −2 direct (also the rule default), −3 block, −5 warp-bypass; −4 reserved for the DNS-hijack action and must not be assigned; invalid marker −99999; direct/out-of-tunnel traffic has a reserved stat identity (not a real profile); groups default to id 0; chain pointers default −1.

**Stats database model** — time series in two tiers keyed by (bucket start, profile id) and (bucket start, process name), carrying up/down byte counters; plus reference metadata rows per profile (name, group name, kind, server address, first/last seen) and per process (last path, first/last seen) so the dashboard can render names after renames/deletes; special identities: direct/out-of-tunnel traffic, and a synthetic `Speedtest` process for the app's own tests.

**Backup archive format** — 4-byte magic `THRN` + format-version integer (current **2**) + JSON metadata + files map. Metadata: archive version, creation timestamp, platform, included parts. Files map: a `database` snapshot entry plus `icons/<name>` entries. Selectable parts: profiles (includes groups + tab order), routes, settings, icons.

### G.2 Behavior

**Startup** — resolve the config dir (§A.1); open the main DB (create/migrate), then the stats DB; first-run seeding is emptiness-based: no groups → create one named `Default`; no route profiles → create one named `Default` with a single rule (name "Route DNS", action `hijack-dns`, protocol `dns`) and default outbound proxy; then settings load once into memory and the diagnostic log level is applied; deferred maintenance runs ~30 s later (checkpoint; vacuum only when free space ≥ 4 MiB **and** free pages ≥ 50 % — an incremental 1024-page vacuum on the stats DB, a full VACUUM otherwise; then another checkpoint); traffic maintenance does one immediate catch-up rollup/prune then repeats every 10 min.

**Settings load/save**
- Load: absent keys keep defaults; unknown keys are ignored. Coercions: booleans accept only `"true"`/`"1"` (anything else false); integers are decimal-parsed and a **parse failure yields 0, not the default**; strings raw; string lists must be a JSON array or the stored value is discarded and the default kept; the two 64-bit sweep timestamps parse-fail to 0; the Xray VLESS preference is an int cast with no range check.
- Save: **all persisted keys are written on every save** (insert-or-replace, chunked); a global no-save gate suppresses all writes; the save result is reported unconditionally, which is why any settings change while a profile runs offers "settings changed, restart proxy?" (changes flagged as needing restart suppress it).
- Save triggers: settings-dialog accept (values are copied to memory first; invalid regex log filters are silently dropped; max log lines ≤ 0 clamps to 200; a blank clash-API port becomes 0); immediate saves without accept (font family/size and theme — also applied live; remember-toggles only after elevation checks succeed; every started-profile-id update); backup creation pre-saves settings when the settings part is included; graceful exit writes geometry/size/splitter + remembered profile/system-proxy/tun state, saves once, then sets the no-save gate so teardown cannot mutate state.
- Restoring a settings backup sets the no-save gate for the rest of the process and requires (and prompts for) a restart.

**UI state persistence** — window geometry (base64), size string (`WxH`), splitter state (base64), selected group, selected route, last filter column, stats tab index; persisted by the same save path (dialog accept / exit); theme and font save immediately.

**Entity lifecycle** — ids come from the counters (never reused). Profile save serializes the whole outbound object. Group delete cascades profiles (foreign key) and removes the tab-order row; deleting the active profile is governed by the allow-stopping setting. Tests update transient fields. Route-profile save deletes and reinserts its rules in list order; deleting cascades rules; raw mode stores the route JSON text instead of rules. Remote route profiles carry URL, prevent-modifications, and last-update/auto-update fields. Starting a profile writes the started id (and the remembered id when remembering is enabled); stopping keeps the remembered value.

**Subscription refresh (storage side)** — computes added/changed/removed sets; removed-but-still-in-use profiles are kept and reported; auto-selector profiles are sticky; with clear-first all non-sticky profiles of the group are deleted first ("Clearing servers...") and any delete failure aborts the refresh with a warning; the group row is updated (last update, user info); the diff summary is logged and, for manual refresh with the popup setting on, shown; deleted/changed ids are forwarded as "disturbed" for selectors. Bulk update-all walks groups in tab order, skipping URL-less, archived, and (in allowed-only mode) skip-auto-update groups; serialized, refuses re-entry.

**Backup/restore**
- Create: if the settings part is included, save settings first; produce a WAL-safe snapshot; wipe non-selected table groups from the snapshot and vacuum; always keep the id counters; delete the temp file on failure.
- Restore: archives from other versions/platforms are accepted; for each selected part only columns present in both schemas are copied, replacing all rows of the live table; runs as one immediate transaction with foreign keys temporarily off; id counters are raised to the maximum of current, restored max ids, and restored counters; failure rolls back and re-enables foreign keys; restoring profiles implies groups + order; counters bump whenever profiles or routes were restored; after a settings restore the no-save gate is set and a restart is required (prompted).

**Traffic statistics** — minute accumulators per profile and per process (current minute bucket); on rollover (or explicit flush at stop/exit) rows are written in one batched transaction per table; minute rows older than 48 h are aggregated into their hour bucket and deleted atomically; hour rows older than retention (clamped ≥ 1 day; default 90) are pruned; one pass at startup (catch-up) then every 10 min. Aggregation-off (the setting) makes all of this a no-op and hides the traffic-stats UI action; polling-off only stops the live per-connection feed. Per-profile lifetime counters are written every ~10 s in their own immediate transaction. Metadata is upserted at profile start (name/group/kind/server) and on first sighting per process (path). Series queries sum both tiers with bucket alignment shifted by the viewer's UTC offset (local day/hour boundaries); empty buckets are omitted.

**Maintenance & errors** — every single statement is guarded: a failure shows a warning naming a truncated query and stating the database may be corrupted, then the app continues; batch operations behave the same, are chunked, and count toward the checkpoint counter. A throwing variant is used inside backup/restore transactions so a failed statement aborts the unit. Migrations are purely additive: create-if-not-exists for tables/indexes plus per-column existence checks before adding columns; there is no schema-version table and no drop/rewrite path.

### G.3 Edge cases & errors
- Malformed stored values follow the coercion table above (notably: bad integers read as 0, not their defaults; bad lists keep defaults).
- Corruption: no integrity check, no auto-repair, and no automatic backup before risky operations (checked throughout the storage layer); whether any UI offers manual repair is UNKNOWN.
- Restore failure rolls back cleanly with foreign keys restored; no dangling references are possible because counters are raised inside the transaction.
- Group-ordering fallback when a group lacks an order row is not conclusively specified (UNKNOWN tie-breaking).
- `created_at`/`updated_at` refresh semantics on update were not fully traced (UNKNOWN which operations bump them).
- The stats DB is fully disposable — deleting/recreating it loses only usage history; its incremental auto-vacuum only applies if set before the journal mode.
- The WAL mode means `-wal`/`-shm` side files exist and must be copied together with the DB file for a consistent snapshot.

### G.4 State & config — settings catalog

**Catalog — 150 persisted keys.** Defaults apply when a key is absent. Encodings: booleans persist as `true`/`false`; integers as decimal text (signed); string lists as JSON arrays; the hotkey map as a JSON object; window geometry and splitter state as base64; secrets in plaintext. Effects below list only behavior not already stated elsewhere in this document.

**Booleans (54)**

| Setting | Default | Effect |
|---|---|---|
| Hide tray icon | off | |
| Random inbound port | off | ignore the configured inbound port; pick a free random port per start |
| Mux padding (new-profile default) | off | |
| Mux on (new-profile default) | off | |
| TLS fragmentation on (new-profile default) | off | |
| TLS tricks on (new-profile default) | off | |
| App requests use proxy | off | app HTTP (updates, subscriptions, downloads) follows the running proxy |
| Remember last state | off | remember profile / system-proxy / Tun state across restarts |
| Skip certificate verification (app) | off | |
| Allow insecure TLS (app) | off | |
| Fake DNS | off | feed fake-DNS into generated engine config |
| Disable per-connection polling | off | |
| Disable traffic aggregation | off | no-op the aggregation pipeline and hide its UI |
| Tun IPv6 | off | |
| Tun strict route | platform | true on Windows ≥ 10.0.1507, false elsewhere |
| Linux auto redirect | on | |
| Subscription clear-first | off | |
| Subscription change popup | on | show the diff popup after a manual refresh |
| Send HWID on subscriptions | off | adds the machine's HWID header to subscription requests |
| Start hidden to tray | off | |
| NTP module | off | enable NTP time in the generated config |
| DNS-hijack server | off | run the DNS-hijack inbound |
| DNS server on LAN | off | listen on all interfaces instead of loopback |
| Redirect inbound | off | enable the transparent redirect inbound |
| System DNS set (marker) | off | remembers the OS DNS override state |
| Windows admin recorded | off | remembers the granted-admin state (Windows) |
| Disable run as admin | off | never request elevation (Windows) |
| Connection statistics | on | per-connection tracking (process names + stats API) |
| Disable privilege requests | off | never ask for elevation |
| Route inside Tun | off | app's own traffic bypasses routing rules |
| Use bundled Mozilla CA list | off | |
| Allow beta updates | off | |
| AdBlock | off | apply the adblock ruleset |
| Show system DNS indicator | off | tray-menu indicator when system DNS is overridden |
| Use custom icons | off | |
| Xray mux on (new-profile default) | off | |
| Use raw DNS object | off | use the custom raw DNS JSON instead of generated DNS fields |
| Skip delete confirmation | off | |
| Show security suffix | off | append TLS/security marker in the profile Type display |
| Log include filter | off | |
| Log exclude filter | off | |
| Log auto-scroll | on | |
| WARP | off | enable WARP options |
| DNS routing | on | apply the DNS routing ruleset |
| Inbound authentication | off | require user/pass on the inbound |
| Allow stopping active profile | off | governs stopping/deleting the running profile |
| Disable mixed inbound | off | disable the mixed socks+http inbound |
| Remembered system proxy | off | persisted last system-proxy state |
| Remembered Tun mode | off | persisted last Tun state |
| Reset proxy when system proxy disabled | off | clear the OS proxy when the feature is switched off |
| DNS disable cache | off | |
| DNS disable expire | off | |
| DNS reverse mapping | off | |
| Disable private-range bypass | off | when on, private ranges also route through the proxy |

**Integers (26)**

| Setting | Default | Effect |
|---|---|---|
| Selected group id | 0 | |
| Last filter column | −1 | none |
| Mixed inbound port | 2080 | |
| Mux concurrency | 8 | |
| Concurrent tests | 10 | |
| Remembered profile id | none (−1919) | |
| Language index | 0 | 0 system, 1 English, 2 zh_CN, 3 fa_IR, 4 ru_RU |
| Font size | 0 | auto |
| Max log lines | 200 | ≤ 0 clamps to 200 at dialog accept |
| Stats tab index | 0 | |
| Stats retention (days) | 90 | clamped ≥ 1 when used |
| Subscription auto-update interval | −30 | signed seconds; values < 30 = disabled |
| Route auto-update interval | −1440 | signed seconds |
| Tun MTU | 1500 | Tun dialog validates the range |
| NTP port | 0 | 0 = standard |
| DNS server port | 53 | |
| Redirect port | 443 | |
| Clash API port | −9090 | only > 0 enables the external controller; blank input stores 0 |
| Speed-test mode | 0 | enum order FULL, DL, UL, SIMPLEDL, COUNTRY |
| Speed-test timeout (ms) | 5000 | |
| URL-test timeout (ms) | 3000 | |
| Xray mux concurrency | 8 | |
| Selected route profile id | 1 | |
| Ruleset mirror | 1 | 0 GitHub, 1 Cloudflare, 2 GCore, 3 Quantil, 4 Fastly, 5 CDN |
| Core DNS-in port | 5533 | |
| DNS cache capacity | 65536 | |

**Strings (55)** — config-generation feeds unless noted; most empty defaults mean "engine default".

| Setting | Default | Effect / notes |
|---|---|---|
| Subscription user agent | "" | empty = `Throne/<version>` |
| Latency test URL | `http://cp.cloudflare.com/` | |
| Inbound address | `127.0.0.1` | `::` = listen on all interfaces |
| Core log level | `info` | engine log level |
| Disk log level | `debug` | file-log threshold |
| Mux protocol | `smux` | |
| Fragmentation implementation | `built-in` | |
| Fragmentation size | `10-100` | |
| Fragmentation sleep | `2-5` | |
| Theme | `0` | theme index or name; applied live |
| Custom inbound JSON | `{"inbounds":[]}` | |
| Custom global route JSON | `{"rules":[]}` | |
| Font family | "" | |
| Hotkey: main window | "" | global shortcut |
| Hotkey: group list | "" | |
| Hotkey: route list | "" | |
| Hotkey: system-proxy tray menu | "" | |
| Hotkey: toggle menu | "" | |
| Active routing name | `Default` | display name of the active route profile |
| Main window size override | "" | |
| VPN implementation | platform | `system` on Windows/Linux, `gvisor` on macOS; Windows < 10.0.1507 forced to `gvisor` |
| Tun IPv4 CIDR | `172.19.0.1/24` | |
| Tun IPv6 CIDR | `fdfe:dcba:9876::1/96` | |
| Subscription custom HWID params | "" | appended to the HWID header |
| Splitter state | "" | base64 |
| uTLS fingerprint | "" | empty = default fingerprint |
| Clash API listen address | `127.0.0.1` | |
| Clash API secret | "" | |
| Underlying DNS (stats API) | "" | |
| NTP server address | "" | |
| NTP interval | "" | |
| NTP outbound | `direct` | |
| DNS v4 response | `127.0.0.1` | |
| DNS v6 response | `::1` | |
| Redirect listen address | `127.0.0.1` | |
| Proxy scheme template | `{ip}:{port}` | |
| Window geometry | "" | base64 |
| Xray log level | `warning` | |
| GeoIP asset URL | Loyalsoldier `v2ray-rules-dat` release URL | `https://github.com/Loyalsoldier/v2ray-rules-dat/raw/release/geoip.dat` |
| GeoSite asset URL | Loyalsoldier `v2ray-rules-dat` release URL | `…/release/geosite.dat` |
| Remote DNS | `https://8.8.8.8/dns-query` | |
| Remote DNS strategy | "" | |
| Direct DNS | `localhost` | |
| Direct DNS strategy | "" | |
| Custom DNS object | "" | raw DNS JSON override |
| Final DNS outbound | `remote` | |
| Resolve domain strategy | "" | |
| Outbound domain strategy | "" | |
| Simple-download URL | `http://cachefly.cachefly.net/1mb.test` | |
| WARP private key | "" | plaintext |
| WARP public key | "" | plaintext |
| WARP endpoint | "" | |
| Inbound user | "" | |
| Inbound pass | "" | plaintext |
| URL-scheme mirror | "" | alternate deeplink scheme host |

**String lists (11)**

| Setting | Default | Effect |
|---|---|---|
| DNS server rules | empty | DNS routing rule list |
| Extra core paths | empty | additional fallback locations |
| Log include keywords | empty | |
| Log include regexes | empty | validated at input |
| Log exclude keywords | empty | |
| Log exclude regexes | empty | |
| WARP interface addresses | empty | |
| WARP reserved | empty | |
| Proxy bind-interface history | empty | |
| Proxy bind IPv4 history | empty | |
| Proxy bind IPv6 history | empty | |

**Special (4)**

| Setting | Default | Effect |
|---|---|---|
| Hotkey map | `{}` | JSON object mapping the five hotkey actions |
| Xray VLESS preference | `1` | 0 = XHTTP only, 1 = XHTTP and Reality, 2 = all VLESS |
| Subscription auto-update last-run (64-bit epoch) | 0 | parse failure reads as 0 |
| Route auto-update last-run (64-bit epoch) | 0 | parse failure reads as 0 |

Cross-platform defaults are stamped into storage after the first load in packaged builds, so a later platform change keeps the previously chosen values.

## H. OS integration & system effects

### H.1 Interface

**Windows registry** (all per-user unless noted):

| Key | Values written | Purpose |
|---|---|---|
| `HKCU\Software\Classes\throne` | default `URL:Throne Protocol`; `URL Protocol` empty; `shell\open\command` default `"<exe>" "%1"` | `throne://` URL handler |
| `HKCU\Software\Classes\Throne.Config` | default `Throne profile`; `DefaultIcon` default `<exe>,0`; `shell\open\command` default `"<exe>" "%1"` | ProgID for config files |
| `HKCU\Software\Classes\.json` `.conf` `.yaml` `.yml` `.ini` `.txt` | subkey `OpenWithProgids\Throne.Config`, value name `Throne.Config`, empty data | "Open with" addition; default handler untouched |
| `HKCU\Software\Classes\Applications\Throne.exe` | `FriendlyAppName`=`Throne`; `shell\open\command`; `SupportedTypes\…` | "Open with → Choose another app" entry |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` (legacy) | value named after the exe base name, data `"<exe path>" -tray` | old autostart; read once, migrated to the scheduled task, then deleted |
| `HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\Throne.exe` (and `…\ThroneCore.exe`) | `DumpFolder`=`<config>\crashes`, `DumpCount`=5, `DumpType`=0 (custom), `CustomDumpFlags` | WER out-of-process dumps; silently skipped without admin |
| `HKCU\Software\Throne` | `TempSetupPath` (installer elevation stash; deleted on read) | per-machine install handoff |
| `HKCU\…\Uninstall\Throne` (HKLM for all-users) | `DisplayName`=Throne, `UninstallString`, `InstallLocation`, `NoModify`=1, `NoRepair`=1 | installer entry |
| WinINet per-connection proxy (LAN + every RAS connectoid) | via `InternetSetOption(INTERNET_OPTION_PER_CONNECTION_OPTION)` | system proxy |

**Windows files/dirs** — install: `%LOCALAPPDATA%\Throne` (per-user default) or `%PROGRAMFILES64%\Throne` (all-users, elevated with `runas` `/ELEVATED`); config (appdata) `%LOCALAPPDATA%\Throne`, else `<install dir>\config` with `throne.db` (+ side files), `logs\throne.log` + `.1`…`.3`, `logs\running.marker`, `logs\crashed-<timestamp>.log`, `crashes\Throne_<version>_<arch>_<yyyymmdd-hhmmss>.dmp` + matching `.txt`, marker `config\.install-dir-unwritable`; `%TEMP%\Throne_Task.xml` (UTF-16 BOM, transient).

**Windows programs/APIs** — `schtasks.exe /create /tn "Throne AutoRun <8 hex>" /xml "<%TEMP%\Throne_Task.xml>" /f` (plus `/query … /xml`, `/delete … /f`), re-run elevated via `runas` when the plain call fails; WinINet with `PROXY_TYPE_DIRECT|PROXY_TYPE_PROXY`, `INTERNET_PER_CONN_PROXY_SERVER`, then `INTERNET_OPTION_SETTINGS_CHANGED` + `INTERNET_OPTION_REFRESH`; `SHChangeNotify(SHCNE_ASSOCCHANGED)` after class registration; core-side DNS set/restore by interface LUID + `FlushResolverCache`; process-dump APIs; scheduled-task principal group `S-1-5-32-545`.

**Linux files** — installer `/opt/Throne` (0755 binaries, 0644 data) + `/opt/Throne/Version`; desktop launcher `/opt/Throne/Throne.desktop` copied to `/usr/share/applications/Throne.desktop` with `Exec=sh -c "PATH=/opt/Throne:$PATH /opt/Throne/Throne -appdata"`; autostart `~/.config/autostart/Throne.desktop`; URL handler `~/.local/share/applications/throne-url-handler.desktop` + icon `~/.local/share/Throne/throne.png`; KDE proxy `~/.config/kioslaverc` (`[Proxy Settings]`); GNOME state in dconf under `org.gnome.system.proxy*`.

**Linux commands** — GNOME path (always executed): `gsettings set org.gnome.system.proxy.<http|ftp|https> host 127.0.0.1`, `… port <http port>`, `org.gnome.system.proxy.socks host`/`port`, `org.gnome.system.proxy mode manual|none`. KDE/Trinity (exact `XDG_CURRENT_DESKTOP` match, `kwriteconfig5`/`kwriteconfig6`/`kwriteconfig` chosen by `KDE_SESSION_VERSION`): `--file ~/.config/kioslaverc --group "Proxy Settings" --key httpProxy|ftpProxy|httpsProxy "http://127.0.0.1 <port>"`, `--key socksProxy "socks://127.0.0.1 <port>"`, `--key ProxyType 1` (clear `0`); then `dbus-send --type=signal /KIO/Scheduler org.kde.KIO.Scheduler.reparseSlaveConfiguration string:''`. URL handler: `update-desktop-database <dir>`, `xdg-mime default throne-url-handler.desktop x-scheme-handler/throne`. TUN privilege: `pkexec <chown/chmod>` to `chown root:root` + `chmod u+s` the core. Lifetime: `SIGTERM`/`SIGINT` via self-pipe; core-dump rlimit raised to infinity.

**macOS files/programs** — bundle id `moe.Throne.macosx`; `Info.plist` declares scheme `throne` + document types (`public.json`, `public.text`, `public.data`, handler rank Alternate); LaunchServices re-index `lsregister -f <bundle>`; system proxy/DNS via `/usr/sbin/networksetup` (`-listallnetworkservices`, `-setwebproxy`, `-setsecurewebproxy`, `-setsocksfirewallproxy`, `-setwebproxystate`, `-setsecurewebproxystate`, `-setsocksfirewallproxystate`, `-setautoproxystate`, `-setdnsservers`, `-listallhardwareports`; output/stderr discarded by design); login item = session login-items list (item = the `.app` directory URL); privileged core via `osascript` Terminal command `sudo chown root:wheel '<core>' && sudo chmod u+s '<core>'` with administrator privileges; Dock process type toggled Foreground ↔ UIElement; config `~/Library/Application Support/Throne` or bundle-adjacent `config`.

**GUI↔core process interfaces** — single instance: a local server named `throne-` + Base64(MD5(absolute config-dir path)) with `+`→`0`, `/`→`1`, padding omitted, world-accessible; a second launch connects, sends `throne://` or `file://` URLs (one per line), exits, and the running instance raises its window. Core IPC: `throneIPC-<uuid>` (user-only, created per run); peers validated by expected core PID. Env exported to core: `THRONE_CORE_SOCKET`, `GOTRACEBACK=crash`, `XRAY_LOCATION_ASSET=<base path>`; plus `THRONE_CORE_DEBUG=1` when core log level is `debug`. CLI flags: `-appdata`, `-tray`, `-many`, `-debug`, `-flag_restart_tun_on`, `-flag_restart_dns_set`. Loopback ports (defaults): mixed inbound `127.0.0.1:2080` (optional random), core DNS-in `127.0.0.1:5533`, DNS server `53` (off), redirect `443` (off, Linux-only), stats API `-9090` disabled when negative.

### H.2 Behavior

**System proxy** — Windows: requires a valid HTTP port (>0, <65536) else no-op; proxy string via the scheme template (`http` → `http://{ip}:{port}`; `socks` → `socks={ip}:{port}`; default template → `127.0.0.1:2080`); applied to LAN and every RAS entry with DIRECT|PROXY flags; no bypass list is written; settings-changed/refresh broadcast; clear sets DIRECT only (leaves the proxy-server value). Linux: GNOME keys always written, then mode `manual`; clear sets mode `none` only (host/port values remain); KDE branch only when the desktop is exactly `KDE`/`Trinity`; other desktops get nothing. macOS: for every enabled service (skipping the header line and `*`-marked rows) web + secure-web proxy on at `127.0.0.1:<http port>` and SOCKS-firewall on at the SOCKS port; clear turns autoproxy + web + secure-web + SOCKS off; never elevates. Timing: applied when a profile starts successfully while the mode is on; cleared on profile stop, on disabling the mode, and during teardown (before DNS/core teardown); TUN + System Proxy may run simultaneously. With "reset proxy on disable" on, disabling the mode stops and restarts the profile. After a core auto-restart the proxy is re-applied only when both "remember profile" and "remember system proxy" were set. Force-kill leaves the OS pointing at a dead local port; nothing cleans it at next start. The app's own HTTP traffic never uses the OS proxy — it uses the running profile's inbound when "app requests use proxy" or the mode is on, else fails with `Request with proxy but no profile started.`

**TUN (VPN) mode** — generated inbound: interface `throne-tun` (empty/auto on macOS), `auto_route` always true, stack per setting (default `system` on Linux and Windows ≥ 10.0.1507, `gvisor` on macOS and Windows < 1507), strict route per setting (default true only on Windows ≥ 10.0.1507), MTU 1500, IPv4 `172.19.0.1/24`, IPv6 `fdfe:dcba:9876::1/96` (only when enabled), `auto_redirect` Linux-only (default true), route exclusions for loopback/private ranges (skipped when routing rules hijack them), DNS hijack via a port-53 rule to a hijack inbound with direct DNS from `127.0.0.1:5533`. Privileges: Windows non-admin → `Please run Throne as admin` (Yes relaunches self elevated with `-flag_restart_tun_on`, which auto-starts the intended mode); Linux → `Please give the core root privileges` (`Please install "pkexec" first.` when pkexec is missing), on accept runs the setuid helper and the user retries; macOS → same dialog, Terminal opens with the sudo command, then `Requesting permission` / `Please Enter your password in the opened terminal, then try again`. The privilege prompts are suppressed by "disable privilege requests"/"disable run as admin" paths. macOS TUN couples system DNS: with a non-empty IPv4 CIDR the core sets the default hardware port's DNS to the next address in the subnet (`172.19.0.1/24` → `172.19.0.2`) and restores with the literal argument `Empty` on stop. Error mapping: errors containing `configure tun interface` prompt a core reset; containing `Fwpm` offer to disable Strict Route with warning `Disabling Strict Route may cause DNS leaks.`

**System DNS toggle (separate from TUN)** — functional on **Windows only**; on Linux/macOS the core answers `not implemented for this OS`, the GUI logs `Failed to set system dns: …` and (settings dialog) reverts the checkbox; the flag is persisted regardless. Set (Windows): core picks the default interface (error when none) and sets its DNS servers to `127.1.1.1` plus a marker — `127.1.2.3` when the previous servers were DHCP-provided, else `127.3.2.1` — then flushes the resolver cache; clear (or default-interface change) restores the previous servers + flush. Teardown order clears system DNS first, then core. The GUI may persist the "set" state and re-apply it at next start/relaunch.

**Autostart ("Run at login / boot")** — Windows: per-user scheduled task `Throne AutoRun <first 8 hex of MD5(lowercased, cleaned exe path)>`; XML: author `DOMAIN\User`, logon trigger, principal group S-1-5-32-545, run level `HighestAvailable` when running as admin (and "run as admin" not disabled) else `LeastPrivilege`, action = quoted exe path **with no arguments**, ignore-new-instances on, allow-hard-terminate off, execution time limit `PT0S`, priority 5. Written to `%TEMP%\Throne_Task.xml` as UTF-16+BOM; creation/deletion falls back to an elevated `schtasks` run; stale run level/priority self-heals; a legacy Run value is detected and replaced. Linux: `~/.config/autostart/Throne.desktop` with `Exec=<APPIMAGE if set else app path> -tray [-appdata]`, `Type=Application`, `X-GNOME-Autostart-enabled=true`, `StartupNotify=false`; disable deletes the file. macOS: inserts/removes the `.app` directory URL in the session login items. Uninstallers do not remove any autostart entry.

**URL scheme, file associations, deeplinks (OS side)** — registration is self-healing: a stored mirror value (e.g. Windows `v2|"<exe>" "%1"`) is compared at startup, registration runs only on mismatch, then the mirror is saved. Windows writes the registry block above and broadcasts. Linux writes the desktop handler (`MimeType=x-scheme-handler/throne;application/json;application/yaml;text/yaml;text/plain;`), copies the icon, runs `update-desktop-database` + `xdg-mime`; under an AppImage, `Exec` targets `$APPIMAGE`. macOS re-indexes the bundle. OS-delivered URLs (Windows/Linux argv; macOS FileOpen) and second-instance hand-offs (one item per line) feed the same dispatcher; unknown verbs log `Ignored deeplink with unknown command: <cmd>`.

**Core process supervision** — spawn `<GUI dir>/ThroneCore[.exe]`, no arguments, env above; stdout/stderr pumped into the GUI log. A line containing `Extra process exited unexpectedly` triggers profile stop. Exit ≠ 0 is logged; a dead process stops the profile with `[Fatal] Core exited, cleaning up...`; automatic restart is attempted once per event, and a restart within 10 s of the previous one stops retrying with `Core exits too frequently, stop automatic restart this profile.`; a failed start is not retried. The core itself validates its parent executable/location and IPC peer PIDs, so orphans exit.

**Privilege helper flows** — Windows elevated relaunch (TUN as non-admin) → `runas` + `-flag_restart_tun_on`; installer relaunch uses `/ELEVATED` + `HKCU\Software\Throne\TempSetupPath`; Windows task creation falls back to elevated `schtasks`; Linux core setuid via `pkexec`; macOS core setuid via Terminal sudo; the Linux installer expects to be run with privileges (writes `/opt/Throne`, `/usr/share/applications`).

**Windows specifics** — crash handling: unhandled-exception + CRT handlers write `Throne_<version>_<arch>_<timestamp>.dmp` (custom flags) plus a `.txt` sidecar (recent log lines, 128 KB stream embedded), then a message box `A crash report was saved to: … Please attach the .txt file (and the .dmp if you can) to a bug report.`; WER LocalDumps registered for both exes (skipped without admin, logged). Retention: `throne.log` rotates 4 MB ×3; ≤5 `crashed-*.log`; ≤5 dmp + 5 txt. `WM_QUERYENDSESSION`/`WM_ENDSESSION` run the cleanup callback (system proxy/DNS/TUN teardown) on logoff/shutdown. Version gating: TUN `system` stack + strict-route default need build 1507+; update channel 1809+; `libcronet.dll` needs 1809+; Windows 11 22H2+ gets a different emoji font. Absent (verified): AppUserModelID, dark title bar, application manifest. Single instance keyed per config-dir path, so two installs coexist.

**macOS specifics** — showing the window switches to Foreground, closing to tray to UIElement; quit-on-last-window-close disabled; FileOpen events forwarded to the deeplink handler; TUN needs the setuid core; system proxy needs no elevation; ad-hoc signing at deploy time; no quarantine-attribute or App-Nap workaround code found.

**Linux specifics** — core-dump rlimit infinity (OS core_pattern applies); SIGTERM/SIGINT self-pipe → clean teardown on logout/systemd stop; desktop entries rely on XDG dirs; TUN requires the setuid-core step; a startup check detects `nameserver 127.0.0.53` in `/etc/resolv.conf` (systemd-resolved) with no traced consumer; AppImage-aware `$APPIMAGE` handling.

**Notifications** — none (no tray balloons or OS notification calls; only the Windows crash box, the privilege/TUN message boxes, and installer dialogs `This will uninstall Throne. Do you wish to continue?` / `Uninstalling Throne for all users requires Administrator privileges. Do you want to elevate?`).

**Installer OS effects (summary; artifact side in §J)** — Windows NSIS: refuses/aborts while `Throne.exe` runs, shortcuts, Uninstall key, optional data-deletion checkbox (its path targets `%APPDATA%\Throne` while runtime data lives in `%LOCALAPPDATA%\Throne` — likely stale mismatch, intent UNKNOWN). Linux script installs to `/opt/Throne` + desktop file (must already be elevated); uninstall removes both. No deploy/install step ships or installs `wintun.dll`.

### H.3 Edge cases & errors
- Force-kill/crash leaves OS proxy/DNS residue pointing at dead ports; no startup cleanup exists (INFERRED absent).
- Uninstallers remove no autostart entries, no URL registrations (Windows classes, Linux handler, macOS login item), no scheduled task — leftovers expected.
- Scheduled-task autostart launches with no arguments (the legacy registry mechanism used ` -tray`) — windowed startup at logon is the verified difference.
- Task name hashes the exe path: after moving the install, the old task lingers (self-heal covers priority/run level only, not path).
- The single-instance socket is world-accessible and parses `throne://`/`file://` lines — whether a foreign local user could drive the running app is UNKNOWN.
- macOS `networksetup` runs unprivileged with output discarded; whether proxy settings always stick when unelevated is UNKNOWN.
- The GUI writes WinINet per-connection state, not the legacy `HKCU\…\Internet Settings` values some tools read — divergence possible (UNKNOWN).
- Linux/macOS system-DNS toggle is a guaranteed soft failure (logged, checkbox reverts in the dialog path).
- `127.0.0.53` detection exists but no consumer was found (UNKNOWN).
- Some bundled third-party helper libraries carry no version metadata (versions UNKNOWN, see §J).

### H.4 State & config (OS-relevant settings; full catalog in §G.4)
- Proxy scheme template `{ip}:{port}`; inbound address `127.0.0.1`; inbound port 2080; random inbound port off; app requests use proxy off; reset proxy on disable off.
- Remember: system proxy off, last state off, Tun off, remembered profile none.
- Admin: disable run as admin off; Windows admin recorded off; disable privilege requests off.
- TUN: implementation platform (system Linux/Win≥1507, gvisor macOS/Win<1507); strict route true only Win≥1507; auto redirect true (Linux-only emitted); MTU 1500; IPv4 `172.19.0.1/24`; IPv6 `fdfe:dcba:9876::1/96`; disable private-range bypass off.
- System DNS set off; URL-scheme mirror empty; core DNS-in port 5533; max log lines 200; disk log level `debug`.
- Runtime-only launch flags: restart-tun-on, use-appdata, tray — derived from arguments, not user settings.
- Launcher-installed persisted files: scheduled task / autostart desktop file / login item; URL-handler registry keys / desktop file / LaunchServices index; WER LocalDumps keys (admin); installer Uninstall key; timestamped logs/dumps.

## I. Import/export formats, share links & deep links

### I.1 Interface

**Entry points that feed imported text**

| Entry point | Behavior |
|---|---|
| Paste box / clipboard import | text trimmed; `throne://` → deeplink dispatcher, else subscription updater |
| New / update subscription | HTTP GET; body → same pipeline; a `Subscription-UserInfo` response header is stored on the group |
| OS "Open with" / CLI file arguments | each existing file ≤ 50 MB: image sniff (content-based, not by extension) → QR decode; else BOM-aware text decode; trimmed payloads — URLs go the single-subscription path, everything else batched |
| OS URL scheme `throne://…` | single dispatcher |
| Drag & drop text onto the window | same as clipboard |
| Screen QR scan | screenshot → decode; each decoded text goes to the **subscription updater**, not the deeplink dispatcher (so a `throne://route/…` QR is ignored there) |
| Route manager import | plain text / clipboard → route profile parser |
| Remote route profiles | `throne://remoteroute/…` deep link, or a later fetch inside the route manager (JSdelivr-wrapped URL rules) |

**Payload model** — one trimmed string that is: a URL (`http/https` → download, or a scheme below), a base64 blob of such lines, a JSON/YAML config container, or a deep link. Container detection runs on the whole body first; multi-line bodies are split before per-line scheme parsing.

**Per-line dispatch** (case-sensitive prefix, first match wins):
1. base64 attempt on the whole body (once, before JSON/scheme detection): strict standard base64; on success the decoded bytes are re-run once with an "already decoded" flag (no infinite recursion on re-decode).
2. `json://<base64url-fragment>` → JSON object.
3. `throne://add/<base64>` (case-insensitive) → JSON object.
4. `{` raw JSON: with `outbounds` → custom full config; with `server` → custom outbound; else rejected.
5. `socks5://`, `socks4://`, `socks4a://`, `socks://`
6. `http://`, `https://`
7. `ss://`
8. `vmess://`
9. `vless://`
10. `trojan://`
11. `anytls://`
12. `mierus://` (and `mieru://`, rejected inside the parser)
13. `hysteria://`, `hysteria2://`, `hy2://`
14. `tuic://`
15. `juicity://`
16. `tt://`
17. `shadowtls://`
18. `wg://`
19. `ssh://`
20. `naive+https://`, `naive+quic://`
21. `ssr://`, `ts://`, `tailscale://`, `snell://`, `wireguard://` (long form), `hysteria3://` → no branch, silently discarded (`ssr://` has no support anywhere in the product).

Lines starting `//` or `#` and shorter than 2 chars are skipped; multi-line splitting is brace-aware so pretty-printed `{...}` objects stay whole.

**Common URL semantics** — fragment → name (percent-decoded fully; absent = empty name, and the UI displays `host:port` as a display fallback only); userinfo handled per scheme (percent-decoded once); a port that is the only URL-parsing complaint is not fatal — continue with 0, then the scheme default applies; query keys are matched case-sensitively, unknown keys ignored, last occurrence wins except repeated specials (mieru `port`/`protocol`); a link missing its scheme's required fields is dropped silently.

**Shared query layers**
- **TLS** (keys): `security` (`tls`/`1`/`true` on; `none`/`0`/empty off; `reality` parsed then downgraded here), `disable_sni` (`"1"`/`"true"`), `sni`/`peer` (later wins; non-empty server name forces TLS on), `allowInsecure`/`allow_insecure`/`insecure` (`"1"`/`"true"` → skip verification), `alpn` (comma), `fp`, `pbk`/`sid`/`spx` (Reality), ECH keys (`ech_enabled`, `ech_config`, `ech_config_path`, `ech_server_name`), `tls_min_version`/`tls_max_version`/`tls_cipher_suites`/`tls_curve_preferences` (comma lists), certificate keys (`tls_certificate`(comma), `tls_certificate_path`, `tls_certificate_public_key_sha256`(comma), `tls_client_certificate`(comma), `tls_client_certificate_path`, `tls_client_key`(comma), `tls_client_key_path`), trick keys (`tls_fragment`, `tls_fragment_fallback_delay`, `tls_record_fragment`, `tls_tricks` — booleans `"true"` only).
- **Transport**: `type` (`tcp`→`raw`; `ws`, `grpc`, `http`, `httpupgrade`, `xhttp`, `quic`; special: `type=tcp` with `headerType=http`, or `type=h2` → http with method `GET`), `host`, `path` (fully decoded), `method`, `headers` (split on `|`; odd counts clear the whole set with log `Failed to import invalid headers: …`), `idle_timeout`, `ping_timeout`, `max_early_data`, `early_data_header_name`, `serviceName`.
- **Multiplex** (sing-box): `mux` (`"true"` only), `mux_protocol`, `mux_max_connections`, `mux_min_streams`, `mux_max_streams`, `mux_padding` (`"true"`), Brutal (`brutal_enabled`, `brutal_up_mbps`, `brutal_down_mbps`). Xray transports use `mux`, `mux_concurrency`, `mux_xudp_concurrency`.
- **Dial fields**: `reuse_addr`, `connect_timeout`, `tcp_fast_open`, `tcp_multi_path`, `udp_fragment`, `bind_interface`, `inet4_bind_address`, `inet6_bind_address`.
- **Packet encoding** allowed values: empty, `packetaddr`, `xudp` — anything else is cleared.

**Per-scheme grammar (1/2)**

| Scheme | Acceptance (drop if not met) | Default port | Highlights |
|---|---|---|---|
| `socks5`,`socks4`,`socks4a`,`socks` | host non-empty | **1080** | version: 4 for socks4, 5 for socks5, unset for socks4a/socks (core default 5); `version` query int overrides; v2rayN compat: empty password + username that validly base64-decodes → split at first `:` (no colon → both become the decoded token); `uot` (`"true"` or int > 0 → UDP-over-TCP). Export: `socks5`/`socks4`, user/pass only when non-empty, `uot=1`, dial fields; `version=4` only in JSON form |
| `http`,`https` | valid URL (effectively nothing) | **443** | `https` (or `security=tls`) forces TLS; credentials: direct `user:pass@`; `base64(user:pass)@` (username slot decoded, split at first `:`; no colon → both = token); `:token@` → username := password := `token`; `path`, `headers` (comma list). Export: `https` if TLS else `http` |
| `ss` | host + method + password | none (0 stays 0) | two encodings: legacy (substring between `://` and first `#` is base64url; then `method:password@host:port` parsed with a synthetic `https://` prefix) and direct (contains `@`: empty password → username slot base64url-decoded into `method:password`; else `method=username`, `password=password` — the `2022-` style); `plugin` (decoded; `simple-obfs;`→`obfs-local;`; split at first `;` into name + opts; separate `plugin-opts` item overrides); `uot`. Export: `2022-` methods emit literal `method:password` userinfo, else base64url(`method:password`) |
| `vmess` | uuid + host | **443** | two encodings: V2RayN base64 JSON (`id`, `add`, `port` string-or-number, `ps`→name, `aid`, `net` (`h2`→`http`), `type == "http"` forces HTTP transport, `host`/`path`, `scy` non-empty only, `tls` only literal `"tls"` + `sni`; everything else ignored) and standard URL (`encryption` default `auto`; `alterId`; `globalPadding`/`authenticatedLength` only `"true"`; `packetEncoding` filtered). Export: always URL form; `encryption` only when ≠ `auto`; `alterId` only when > 0; `packetEncoding` always emitted (empty → `"none"`) |
| `vless` | uuid + host | **443** | **model split**: Xray VLESS when preference is `All VLESS`, or `type=xhttp`, or (`security=reality` and preference `XHTTP And Reality` — the default), or `encryption` present ≠ `none`, or `extra` non-empty; otherwise sing-box VLESS. Preference enum `XHTTP Only`(0) / `XHTTP And Reality`(1) / `All VLESS`(2). sing-box: `packetEncoding` default **`xudp`**; export emits `encryption=none` + `packetEncoding` always. Xray: `encryption` default `none`, Xray stream layer (`type` with `tcp`→`raw`, `h2`, `xhttp`, `grpc`, `ws`, `httpupgrade`, `quic`, `kcp`; `security` `tls`/`reality`/`none`; `sni`/`peer`/`server_name` last-wins; `alpn`, `fp`; `pbk`/`sid`/`spx`; `pcs`, `vcn`; transport keys `host`, `path`, `mode`, `headers`, `extra` (JSON), `x_padding_bytes`, gRPC knobs `no_grpc_header`, `sc_max_each_post_bytes`, `sc_min_posts_interval_ms`, `max_concurrency`, `max_connections`, `max_reuse_times`, `max_request_times`, `max_reusable_secs`, `keep_alive_period`, `heartbeat_period`, `ed`, `authority`, `serviceName`, `mode=multi`, `idle_timeout`; Xray mux keys); export emits `encryption` always + `flow` when non-empty |
| `trojan` | none (empty password allowed) | none | password = username slot; TLS only when `security`/`sni` says so. Export: TLS merged only when enabled |
| `anytls` | valid URL (none) | **443** | TLS forced on; `idle_session_check_interval`, `idle_session_timeout` (strings like `30s`), `min_idle_session` (int). Export: username = password, TLS always |
| `mierus` | `mieru://` rejected (parser requires host + non-empty query) | — | simple form `mierus://user:pass@host?port=…&protocol=…&profile=…`; `protocol` repeated → **first** wins, uppercased; `port` repeated → first bare port = `server_port`, further bare ports become `N-N`, ranges (`a-b`) kept verbatim, joined with `,` into `server_ports`; `multiplexing`, `traffic-pattern` stored as strings. Export: `profile=default`, one `port`+`protocol` pair per emitted port, then options |

**Per-scheme grammar (2/2)**

| Scheme | Acceptance (drop if not met) | Default port | Highlights |
|---|---|---|---|
| `hysteria`, `hysteria2`, `hy2` | always (valid URL is enough) | **443** | scheme selects v1 vs v2; ports: authority port, or the non-standard `:ports` set right after the authority (`host:20000-30000`, `host:443,8443` — parsed with a regex when URL parsing flags the port invalid; each token `a-b`/`a:b`/single normalized to `a:b`), or the `mport` comma list; TLS forced on. v1 only: `obfsParam` (decoded), `auth` with auth_type STRING, `recv_window_conn`, `recv_window`, `disable_mtu_discovery` (`"true"`). v2 only: password = `user`, or `user:pass` concatenated with `:`, plus `obfs-password`. Both: `upmbps`, `downmbps` ints, `hop_interval`. Export: `hysteria`/`hysteria2`; v1 emits `obfsParam` (percent-encoded), `auth` only when auth_type is STRING, recv windows, `disable_mtu_discovery`; v2 splits password at `:` into user/pass userinfo and emits `obfs-password`; port lists are re-inserted after the authority with `-` ranges joined by `,` |
| `tuic` | uuid + host | **443** | `uuid:password@`; TLS forced; `congestion_control`, `udp_relay_mode`, `udp_over_stream` (`"true"`), `zero_rtt_handshake` (`"true"`), `heartbeat` (string). Export: `udp_relay_mode` only when not in udp-over-stream mode; `udp_over_stream=true` and `zero_rtt_handshake=true` as literals; heartbeat verbatim |
| `juicity` | uuid + host | **443** | TLS forced; otherwise only the shared layers. Export: userinfo uuid[:password] + TLS |
| `tt` (TrustTunnel) | user + password + host (strict) | **443** | TLS forced; `health_check` (`"true"`); `congestion_control` presence forces `quic=true` and stores the value. Export: `tt`, `health_check=true` when set, `congestion_control` only when quic and non-empty |
| `shadowtls` | host non-empty | **443** | `version` query (int); optional password (password slot) used for versions > 1; TLS forced. Export: `version` always; password only when version > 1 and non-empty |
| `wg` | private_key + peer public key + host | none | `private_key` (required), `public_key`/`peer_public_key` (required; later wins), `pre_shared_key`, `reserved` (dash-separated ints, > 0 only), `persistent_keepalive_interval`, `local_address` (dash-separated list), `mtu` (int), `workers` (int), `use_system_interface` (`"true"`), `udp_timeout`; AmneziaWG keys `enable_amnezia` + `jc jmin jmax s1..s4 h1..h4 i1..i5 header_protection_key content_padding_addition rekey_after_time rekey_timeout reject_after_time keepalive_timeout max_handshake_attempts` (any present also sets enable_amnezia). Export: `wg`, peer host/port, `local_address` joined with `-`, `mtu` only when > 0 and ≠ 1420, keepalives, Amnezia keys only when set. **WireGuard config files** (`.conf` text or any payload containing both `[Interface]` and `[Peer]`): INI-like, case-sensitive keys `Address` (comma list), `PrivateKey`, `Jc Jmin Jmax S1..S4 H1..H4 I1..I5 HeaderProtectionKey ContentPaddingAddition RekeyAfterTime RekeyTimeout RejectAfterTime KeepaliveTimeout MaxHandshakeAttempts`; peer keys `PublicKey`, `PresharedKey`, `Endpoint` (`host:port`, IPv6 bracketed), `AllowedIPs` (peer endpoint taken), `PersistentKeepalive`; acceptance private + peer public key; imports one profile |
| `ssh` | host non-empty | none (0) | userinfo ignored in favor of the `user` query key; `password`, `private_key` (standard base64, trailing `=` optional), `private_key_path`, `private_key_passphrase`, `host_key`/`host_key_algorithms` (dash-separated base64 blobs), `client_version`; dial fields |
| `naive+https`, `naive+quic` | host non-empty | **443** | bare `naive://` has no branch; the quic form sets quic; `uot` (`"true"`/int>0), `congestion_control` (quic only); TLS forced. Export: scheme by quic, `uot=1`, congestion_control only under quic |
| `ts`/`tailscale` | — | — | **export-only**: a parser exists (`globalDNS`/`global_dns` aliases) but no import branch, so such links are not importable from subscriptions/clipboard (UI editing only). Export keys: `state_directory`, `auth_key`, `control_url`, `ephemeral`, `hostname`, `accept_routes`, `exit_node`, `exit_node_allow_lan_access`, `advertise_routes` (comma, percent-encoded), `advertise_exit_node`, `global_dns` |
| unsupported schemes | — | — | `ssr://`, `snell://`, `hysteria3://`, `wireguard://` long form, `vmess1://`: no branch anywhere; unknown schemes are ignored silently (no dialog; only unknown `throne://` verbs log) |

**Route share links** — `throne://route/` + base64url (no padding) of this compact JSON:

| Field | Value |
|---|---|
| `kind` | always `throne-route-profile` |
| `v` | 1 |
| `name` | profile name |
| `default_outbound` | `proxy` \| `direct` \| `block` \| `warp-bypass` (unknown → `proxy`) |
| `rules` | array of rule objects |
| `raw` | present + true for raw profiles |
| `prevent_modifications` | raw profiles only |
| `route` | raw profiles: the raw route object |
| `outbound_names` | raw profiles: map numeric profile id → server-profile name, for re-linking raw routes by name |

Rule objects carry every sing-box rule field the builder emits (`ip_version` int, `network`, `protocol`, `inbound`, `domain`, `domain_suffix`, `domain_keyword`, `domain_regex`, `source_ip_cidr`, `source_ip_is_private`, `ip_cidr`, `ip_is_private`, `source_port` nums, `source_port_range`, `port` nums, `port_range`, `process_name`, `process_path`, `process_path_regex`, `wifi_ssid`, `wifi_bssid`, `rule_set`, `invert`, `action`, `reject_method`, `no_drop`, `override_address`, `override_port`, `strategy`) plus share-only `name` and `type` token: `custom`, `simple_address_proxy|bypass|block|warp_bypass`, `simple_process_name_proxy|bypass|block|warp_bypass`, `simple_process_path_proxy|bypass|block|warp_bypass` (unknown token → `custom`). Rule parsing is tolerant: `outbound` may be a number (profile id; not found → warning `outbound id N not found, using proxy` + fallback) or a string (resolved by profile name, or `proxy`/`direct`/`block`/`warp-bypass`; unknown → same warning + fallback); other keys go through a generic setter (arrays → string list, string → single, bool → `"true"`/`"false"`). Outbound ids: proxy −1, direct −2, block −3, warp-bypass −5 (−4 reserved for DNS hijack). Accepted import inputs, in order: the deep link, raw JSON text, base64url blob, standard base64 blob. Failures: `Empty input`; `Deep link is invalid` / `Deep link has no data`; `Input is not valid JSON, base64, or a Throne route link`; `Unrecognized route object` (JSON without `kind`); a legacy bare array of rules is accepted (`wasOldArray`: no name/default imported; the rule editor opens instead of the name prompt); an empty rule array → "Input is not a valid json array".

**`throne://remoteroute/<b64>`** — base64 (strict standard) of newline-separated http(s) URLs, each optionally with `#name`. Each URL becomes an unsaved remote route profile (stored URL without fragment; name = fragment, else host); non-http(s) lines are skipped. Errors: `Deep link is invalid`, `Deep link has no data`, `Base64 is invalid.`, or "The link did not contain any valid http(s) routing profile URLs." A confirmation dialog lists the URLs with an **Auto update checkbox (default on)**; after adding, profiles are immediately fetched and saved; per-URL fetch failure is logged and leaves an empty updatable profile.

**Serialization**
- Per-scheme export: socks → `socks5`/`socks4`; http → `https` when TLS else `http`; ss → base64url userinfo unless method starts with `2022-`; vmess → always the URL form; vless sing-box → emits `encryption=none` + `packetEncoding` always; Xray vless → emits `encryption` always; trojan → TLS only when enabled; anytls → idle-session settings + TLS; hysteria/tuic/juicity/tt/shadowtls/wg/mierus/naive/tailscale → per their tables; custom / chain / autoselector / direct / extra-core profiles → **empty share link** (copy operations then fall back to the JSON deep link). All links are produced through a fully percent-encoded form with the profile name in the fragment.
- **JSON deep link** `throne://add/<payload>`: payload = URL-safe base64, **no trailing padding**, of the compact JSON of the outbound, optionally with the `tag` (name) field stripped. Used by: copy-link (NKR) for a selection, group copy-links (NKR), clone, and as the fallback whenever a profile has no share-link form (QR dialog included). Import accepts it from clipboard, the updater, QR payloads, and files; the decoded object must carry `type` (sing-box outbound; `hysteria2` maps to the hysteria model) or `protocol` (Xray outbound, profiled as `xray<protocol>`); an object with neither (e.g. `{"foo":1}`) is discarded.
- Copy / clone / group operations: copy selection = per-profile share link else JSON link, `\n`-joined; copy NKR = JSON links joined; clone = JSON links fed back as an import payload (confirm dialog, duplicates state); group copy links = members' share links (no fallback); group copy NKR = members' JSON links; "export config" copies the generated sing-box core JSON to the clipboard (not a link); "remove duplicates" keys on the export-link form and keeps the first occurrence.
- **QR export**: the selected profile encoded with UTF-8 byte mode, error correction MEDIUM, padding 2 modules, black on white; a `Deep Link` checkbox switches share link ↔ JSON deep link; when the share link is empty the dialog opens on the deep link; encoder-capacity overflow shows a warning with the encoder's error text. **QR import**: screen scan hides the window, waits 1 s, grabs the primary screen (Wayland: xdg-desktop-portal Screenshot interface with a UUID token; failure → "Unable to capture screen"), decodes **all** codes found, logs each as `QR Code Result:\n<text>`, and feeds each trimmed result to the subscription updater (not the deeplink dispatcher); no code → "QR Code not found"; image-file failure is logged as `<name>: no QR code found`.

**Dispatcher grammar** — `throne://<verb>[/<payload>][?…][#fragment]`; the verb is the URL host (URL hosts are lowercased → case-insensitive), the payload is the path minus its leading `/`; a verb without a path is ignored. Verbs: `add` (whole URL → updater, which recognizes `throne://add/<b64>` and imports the JSON profile; no extra confirmation), `route` (payload → route parser; failure → warning "Import routing profile / The link could not be parsed:<fatal>"; success → name prompt `Add this routing profile?\n\nName: %1` plus a "Note:" list of non-fatal warnings; blank name → `Imported profile`), `addsub` (payload base64 strict → URL; fragment → group name, else host; no decoded URL → "The link did not contain a subscription URL."; else prompt `Add this subscription?\n\nName: %1\nURL: %2` + Auto update checkbox default on; the fragment is removed from the stored URL), `remoteroute` (as above); any other verb logs `Ignored deeplink with unknown command: %1` and shows nothing. `addsub`/`route`/`remoteroute` proceed only when the path starts with `/`.

### I.2 Behavior

**Whole-body detection order** (per payload, before per-line parsing): 1) strict standard-base64 decode of the whole body succeeds → recurse once on the decoded text; 2) valid JSON → 5.2 sub-detection below; 3) contains `proxies:` → Clash YAML; 4) contains both `[Interface]` and `[Peer]` → WireGuard config; 5) contains a newline (top-level body only) → split into lines/braced-JSON chunks and re-run per chunk; 6) skip comment/short lines; 7) `json://` → base64url-decode the fragment → JSON profile; 8) `throne://add/` (case-insensitive) → base64-decode the path → JSON profile; 9) per-scheme branches.

**JSON sub-detection** (in order): **Xray** — an object with `outbounds` where any member has `protocol`, or an object with `protocol`, or an array whose first element has `protocol`, or an array whose first element has `outbounds` containing a `protocol` member. `vless` outbounds are normalized (`vnext`/`servers` shapes) into the Xray VLESS model; `freedom`/`blackhole`/`dns`/`loopback` are skipped; everything else becomes a Custom Xray outbound (name from `tag`); array-of-full-configs imports each as a Custom "Xray full config" with `inbounds` removed (name from `remarks`). **sing-box** — an object with `outbounds` or `endpoints` (per-outbound mapping below), an object with `type` (single Custom outbound), or arrays of outbounds. **SIP008** — a body containing both `version` and `servers`: each server object builds a Shadowsocks profile (`remarks`→name, `method`, `password`, `plugin` (`simple-obfs`→`obfs-local`), `plugin_opts`, `uot` (bool or `{enabled}`), `server`/`server_port`).

**Container field mappings (highlights)** — sing-box `type` → model: `socks`, `http`, `shadowsocks`, `vmess`, `vless` (Xray model when network `xhttp` or encryption non-empty/non-`none`), `trojan`, `anytls`, `hysteria`/`hysteria2` (protocol_version by type), `tuic`, `ssh`, `naive`, `wireguard`; unknown types skipped; common fields `tag`→name, `server`/`server_port`, dial fields, TLS (`enabled`, `server_name`, `insecure`, `alpn`, versions, cipher suites, certificates, `reality`), transport (incl. ws headers with `Host` promoted to `host`), multiplex, `udp_over_tcp` (bool or `{enabled}`), AmneziaWG for wireguard. Xray protocol → model: `vless`→xrayVless (`settings.address/port/id/flow/encryption`, `streamSettings` incl. reality/tls + transport options, `mux`), others → Custom Xray outbound. Clash YAML `proxies:` mapping highlights: `socks5` (server/port/username/password), `http` (TLS from `skip-cert-verify`/`sni`/`alpn`), `ss` (`cipher`→method, `udp-over-tcp`→uot; `v2ray-plugin` → plugin + `tls;host=…;path=…;mode=…;mux` opts; `obfs` → `obfs-local;obfs=…;obfs-host=…`), `vmess` (`uuid`, `cipher`→security, `alterId`, `packet-encoding`, ws/grpc/h2/http transports), `vless` (xhttp or non-`none` encryption → xrayVless; `servername`/`sni`, `client-fingerprint`, `reality-opts`, transports incl. `ws-opts` with early-data/upgrade booleans, `xhttp-opts`, `smux`), `trojan` (TLS forced), `anytls` (interval numbers → `Ns` strings), `hysteria`/`hysteria2` (`auth-str`→v1 STRING auth, `auth_str`→v1 BASE64, `obfs`→v1 obfs; v2 `password`/`obfs-password`; `up`/`down`, `ports`, `hop-interval`, recv windows, `disable-mtu-discovery`), `tuic` (`reduce-rtt`→zero_rtt, `heartbeat-interval`→`Nms`, `ip` overrides server, TLS forced), `ssh`; `wireguard` file form; types without a branch (`snell`, `mieru`, …) are skipped. Clash encoding tolerance: NULs stripped, leading BOM removed before YAML parsing, parse exceptions caught → "YAML Exception" warning box; discovery is purely textual (`proxies:`).

**Naming** — name = link fragment (fully decoded) or JSON `tag`/`ps`/`remarks`. If still empty the stored name stays empty and the UI displays `host:port` (display fallback only); there is no hostname-only fallback and no dedup renaming; batch report lines use `[DisplayType] host:port`. Legacy bare rule arrays get `imported rule #N`; tagged share objects get `rule_N` for nameless rules.

**Destination & prompts** — a payload typed as free text (not clipboard-import with an explicit target) that starts `http://`/`https://` opens a 3-way dialog: `Add profiles to this group` (default), `Create new subscription group` (also downloads, group named after the URL host), `Import HTTP proxy profile` (treats the URL text itself as an HTTP-proxy profile). Otherwise entries are imported into the current group (or the batch's target) via one batch insert, reported as `Imported %1 profile(s)`. File-import batches: URL payloads go through the single-subscription path (to get the prompt); all other payloads are batched without per-item prompts.

**Dedup / diff at import** — identity key = the JSON deep-link payload of the outbound (config equality, name often ignored); change classification adds `type + "|" + compact identity JSON` (server/port/TLS/transport, no name) to detect same-slot-different-server updates. Within a refresh: additions "＋", deletions "−", common entries paired one-to-one by export key (each entry claimed once; duplicates not collapsed); changed pairs update in place under the same id and are marked "disturbed" (running auto-selectors restart). Auto-selector profiles are sticky (kept, restored to position). Clear-first deletes non-sticky group profiles before import (running profile protected). Refresh rewrites order to match the remote; diff text elides with `added N`/`updated N`/`deleted N` when a bucket has ≥ 1000 entries. Manual dedup keeps the first occurrence by export-link key and removes later equal-config rows (positional duplicate slots are removed without deleting the profile).

### I.3 Edge cases & errors

| Input | Behavior |
|---|---|
| Bad/aborted base64 body | falls through to JSON/text detection; no error shown |
| Base64 of base64 | one recursive decode only, then parsed |
| Junk lines in a multi-line body | ignored silently (matches only silently too when malformed) |
| Unknown query parameter | ignored |
| Unknown scheme | ignored, no log |
| `vmess://` base64 that lacks `id`/`add` | dropped |
| `ss://` legacy with bad base64, or missing method/password/server | dropped |
| `mieru://` standard link | dropped (host/query requirement) |
| `tt://` without user or password | dropped |
| `wg://` without keys or host | dropped |
| `ssh://` without host | dropped |
| `hysteria://` with junk query only | **accepted** if the URL is valid (no required fields) |
| Any scheme where the port is the only URL-parsing complaint | port 0 → scheme default applies |
| `headers` with an odd number of `|` parts | whole header set cleared + log `Failed to import invalid headers: …` |
| `packetEncoding` outside {empty, `packetaddr`, `xudp`} | cleared |
| Route rule `type` token unknown | rule becomes `custom`, remaining fields still parsed |
| Route rule `outbound` unresolvable | warning + fallback to `proxy`; import still succeeds |
| Route JSON without `kind` | "Unrecognized route object" |
| Route bare array with non-object member | fails: "expected array of json objects but have member of type '…'" |
| `throne://route/` empty payload | "Deep link has no data" |
| `throne://remoteroute/` with only non-http lines | "The link did not contain any valid http(s) routing profile URLs." |
| Unknown `throne://` verb | log only |
| QR with no code | "QR Code not found" / per file `<name>: no QR code found` |
| Screen-scanned QR decoding to `throne://route/…` | silently ignored by the scan path (it goes to the updater, which has no route branch) |
| File > 50 MB | skipped: `<name>: larger than 50 MB, skipped` |
| File with NUL bytes or > 5 % control chars in the first 8192 bytes | rejected: `<name>: not a readable config file` |
| File with UTF-8/16/32 BOM | decoded accordingly, BOM dropped |
| Empty decoded file | rejected ("not a readable config file") |
| Batch where every file failed | warning "Nothing could be imported:" + list |
| `http(s)` URL as text update with fetch error | log `Requesting subscription %1 error: …`; no dialog |
| JSON parsing but matching no container | ignored silently |
| Clash YAML with hostile structure | "YAML Exception" warning box + message |
| Re-importing the same config into the same group | matched → no duplicate row; clones/duplicates pair one-to-one |

### I.4 State & config
- Fixed constants a reimplementation must keep: scheme prefixes and dispatch order (I.1); per-scheme default ports (1080 socks, 443 http/vmess/vless/anytls/hysteria/tuic/juicity/tt/shadowtls/naive; 0/none for ss/trojan/ssh/wg); display fallback `host:port`; route share `kind` = `throne-route-profile`, `v` = 1; outbound ids −1/−2/−3/−5 with −4 reserved; QR parameters (UTF-8, MEDIUM, padding 2); deep-link base64 alphabets (URL-safe, no padding for `add`/`route`; strict standard for `addsub`/`remoteroute` payloads).
- Imported entries land in the current group (or explicit batch target); subscription refresh behavior and group flags are in §D; JSON deep-link handling overlaps the copy/clone paths in §B.
- All error/dialog strings in I.2/I.3 are exact UI contract.

## J. Build, packaging, CI & artifacts

### J.1 Interface

**Build requirements**

| Item | Value |
|---|---|
| CMake | ≥ 3.20; CI uses the Ninja generator |
| C++ | C++20, standard required (no fallback) |
| Qt | Qt 6 with `Widgets`, `Network`, `LinguistTools` (+ `DBus` on UNIX-not-APPLE); CI builds against 6.2.x (Ubuntu apt), 6.2.13 (Windows legacy), 6.4.3 (macOS legacy), 6.11.1 (current); version-gated code: bundled emoji fonts need ≥ 6.9.0, the colour-scheme hook needs ≥ 6.5.0 |
| Threads | required |
| X11 | REQUIRED on Linux only (global hotkeys) |
| Go | `go 1.26` in go.mod; CI pins toolchain 1.26.5; legacy targets use a vendored `go.zip` toolchain |
| protoc (core) | pinned 31.1; `protoc-gen-go` v1.36.11; `protoc-gen-go-grpc` v1.6.2 |
| C++ proto codegen | vendored generator (no external protoc); optional clang-format of generated code |
| `srslist.h` | **external generated header, not in the repo, gitignored**; CI downloads it from `https://raw.githubusercontent.com/throneproj/routeprofiles/rule-set/srslist.h` before configure — the build fails without it |
| OpenSSL | provided via Qt; CI supplies `OPENSSL_ROOT_DIR` on Windows only |

**Configure-time options / behavior toggles**
- Packaging appdata-mode CMake options (generic + macOS variant) → compiled-in appdata mode (config under the user profile; in-app self-update button hidden even when the updater binary exists). CI turns it on only for macOS.
- `INPUT_VERSION` env → generated version header → the app version string (empty when unset); feeds the window title, logs/crash dumps, the default HTTP User-Agent, and update comparison. Also runtime: `-appdata [dir]`, `-tray`, `-many`, `-debug`, `-flag_restart_tun_on`, `-flag_restart_dns_set` (internal restart handoff; re-added on relaunch except `-tray`, which is removed).
- Debug configuration forces debug logging; Qt message log context is always kept in release builds.
- Static linkage everywhere (`BUILD_SHARED_LIBS OFF`); non-MSVC dead-code stripping (`-ffunction-sections`/`-fdata-sections` + `--gc-sections` / `-dead_strip`); MSVC `/OPT:REF /OPT:ICF /INCREMENTAL:NO` and **static CRT** (no MSVC redist DLLs in artifacts); RelWithDebInfo embeds debug info.
- Unity build ON for the GUI target with the SQLite amalgamation excluded.
- SQLite is compiled with a hardening define set (~19: no DQS, no UTF16, no JSON1, no load-extension, WAL synchronous default 1, etc.).
- Windows semantics: `UNICODE`, `_UNICODE`, `NOMINMAX`, `/utf-8`, `_WIN32_WINNT=0x600`.
- Emoji font resources are compiled in when NOT APPLE and Qt ≥ 6.9 (region-indicator flags rely on them); on Windows an extra Twemoji resource is added.

**GUI build steps** — configure Qt (consume `CMAKE_PREFIX_PATH`); include the platform file (Windows additionally generates the version resource: NAME/BUNDLE/COMPANY all "Throne", the Windows icon asset; note the generated `FILEVERSION`/`PRODUCTVERSION` stay **1.0.0.0** regardless of the release tag, and the resource declares a Russian language id — metadata oddities to preserve); generate C++ protobuf from the RPC proto definition into a static lib; build the vendored global-hotkey lib; create the GUI executable (`MACOSX_BUNDLE`, `WIN32_EXECUTABLE`, app icon, bundle id `moe.Throne.macosx`); translations: TS files hardcoded **zh_CN, fa_IR, ru_RU**, `lupdate` (`-locations none -no-obsolete`), `lrelease`, `.qm` files **embedded in the binary** (runtime first tries `<appdir>/translations/<locale>.qm` on disk, then the embedded copy); language selection: 0 system, 1 English, 2 zh_CN, 3 fa_IR, 4 ru_RU.

**Platform build specifics**

| Platform | Extra sources/frameworks | Notes |
|---|---|---|
| Windows | elevated-relaunch helper; crash-dump writer; session-end event handler; OS version info; autostart; URL-scheme registration | links `wininet wsock32 ws2_32 user32 rasapi32 iphlpapi ntdll wbemuuid psapi shell32`; GUI subsystem; version resource |
| macOS | macOS helpers, autostart, URL-scheme registration | `Security` framework; `.app` with `CFBundleURLSchemes = throne`, document types (json/text/data), `NSHighResolutionCapable` |
| Linux | elevation helper, core-dump handling, autostart, URL-scheme registration, environment checks | `dl` (+ DBus); runtime adds `<appdir>/usr/plugins` to Qt library paths |

**Go core build** — driven by a script with `GOOS`/`GOARCH`; a `GOOS` ending in `legacy` switches to the vendored legacy Go toolchain. Exact build tags: `with_clash_api,with_gvisor,with_quic,with_wireguard,with_utls,with_dhcp,with_tailscale,badlinkname,tfogo_checklinkname0` plus `with_purego,with_naive_outbound` (non-legacy Windows) / `with_naive_outbound` (darwin, linux). Env: Windows non-legacy `CGO_ENABLED=0` (+ libcronet payload); Windows legacy `CGO_ENABLED=0` (no naive, no libcronet); darwin `CGO_ENABLED=1` + weak-framework link flag; linux `CGO_ENABLED=1` with a Chromium/clang toolchain prepared by CI for the naive outbound. Generated protobuf stubs come from the RPC proto definition (stubs gitignored). Version stamping: resolve the sing-box fork module version, then link with `-trimpath` and stripped symbols (`-w -s`), pinning the engine fork's version string into the binary, forcing the multipath-TCP runtime default off, and leaving linkname checking disabled. Output binary `ThroneCore` (`.exe` on Windows), no packing.

**Payload files fetched at build time (Windows/Linux releases)** — `updater` (`updater.exe`) from `https://github.com/throneproj/updater/releases/latest/download/updater-windows-x<arch>.exe` / `updater-linux-<arch>` — **latest tag, no checksum, not pinned**; amd64 and arm64 both resolve to the x64 updater (arch bug to preserve). Windows non-legacy only: `libcronet.dll` from `https://github.com/SagerNet/cronet-go/releases/latest/download/libcronet-windows-<arch>.dll` (also unpinned). macOS releases fetch neither. The core binary's runtime contract: requires `THRONE_CORE_SOCKET` (else fatal), optional `THRONE_CORE_DEBUG=1`; exits when its parent dies; serves `throneIPC-<uuid>`; prints `sing-box: <ver>` and `Xray-core: <ver>` to stdout at startup (captured into the GUI log; there is **no version RPC**); 2 GiB memory limit with a 1.5 GiB watchdog panic (heap profile written to temp first).

**Runtime artifact layout**
- The GUI always looks for the core at `<applicationDirPath>/ThroneCore` (+`.exe`); the updater at `<applicationDirPath>/updater(.exe)`; the "Check for Update" action is disabled at startup when the updater binary is absent.
- **Windows portable ZIP** (`Throne-<tag>-windows64.zip`, `-windows-arm64.zip`, `-windows32.zip`, `-windowslegacy64.zip`): root folder `Throne/` with `Throne.exe`, `ThroneCore.exe`, `updater.exe` (optional), `libcronet.dll` (amd64/arm64 only, optional), PDBs stripped into a separate debug-symbols archive. Qt and the MSVC runtime are fully static (no DLL copying).
- **Linux portable ZIP** (`Throne-<tag>-linux-amd64.zip`, `-linux-arm64.zip`): root `Throne/` with `Throne`, `ThroneCore`, `updater`, `Throne.png`, `usr/lib/**` (Qt + ICU + xcb helper libs, rpath `$ORIGIN/usr/lib`, plugins rpath `$ORIGIN/../../lib`), `usr/plugins/{platforms/{libqxcb.so,libqwayland.so},platformthemes,imageformats,iconengines,wayland-shell-integration,wayland-decoration-client,tls}`; debug binary moved out.
- **Linux system-Qt deployment dir**: only `Throne`, `ThroneCore`, `Throne.png` — **no updater**, depends on host Qt 6.
- **Linux .deb** (`Throne-<tag>-debian-amd64.deb`, `-debian-arm64.deb`, `-debian-amd64-system-qt.deb`, `-debian-arm64-system-qt.deb`): package name `Throne`, Maintainer `Mahdi Mahdi.zrei@gmail.com`, payload `/opt/Throne/…`, `Depends: desktop-file-utils` (system-qt adds Qt 6 packages + `libxcb-cursor0`, `fonts-noto-color-emoji`); `postinst` writes `/usr/share/applications/Throne.desktop` with `Exec=sh -c "PATH=/opt/Throne:$PATH /opt/Throne/Throne -appdata"`, `Icon=/opt/Throne/Throne.png`, `Categories=Network;Application;` and runs `update-desktop-database`.
- **Linux CLI installer** (script run as root via `curl | sudo python3`): installs `/opt/Throne/{Version,Throne,ThroneCore,Throne.png,usr/**}` + the desktop file; **no updater** (CLI installs cannot self-update); uninstall removes `/opt/Throne` + desktop file.
- **macOS** (`Throne-<tag>-macos-arm64.zip`, `-macos-amd64.zip`, `-macoslegacy-amd64.zip`): root `Throne/` containing `Throne.app` (`Contents/MacOS/{Throne,ThroneCore}`), `macdeployqt` frameworks/plugins, **ad-hoc codesign only** (`codesign --force --deep --sign -`), stripped; no notarization; README documents the `xattr -d com.apple.quarantine` workaround and the requirement to run from `/Applications` before the first launch (setuid core escalation). All macOS builds compile in appdata mode.
- **Windows NSIS installer** (`ThroneSetup.exe`, released as `Throne-<tag>-windows-universal-installer.exe`): single install section, LZMA solid; `RequestExecutionLevel user` (re-launches itself with `/ELEVATED` for All Users); pages Welcome → Just Me/All Users → Directory → Install → Finish; default dirs `%LOCALAPPDATA%\Throne` / `%PROGRAMFILES64%\Throne` with a forced `Throne` subfolder; payload picked by CPU/OS at compile time (native amd64 for Win10 1809+, legacy amd64, arm64, legacy 386; abort otherwise); installs `Throne.exe`, `ThroneCore.exe`, `updater.exe`, plus `libcronet.dll` for amd64/arm64; Desktop + Start Menu shortcuts; writes `Software\Throne` `InstallPath` and the Uninstall key; the uninstaller offers a default-ON "Delete my profiles, settings and logs" checkbox (deletes `config/` only when `config/throne.db` exists, plus `%APPDATA%\Throne`); terminates running `Throne.exe` during install/uninstall; internal switches `/ELEVATED`, `/UINSTDIR=<path>`; standard `/S` silence. No RPM packaging exists in-repo (README links a third-party repo).

**Release asset names produced by CI (exact)** — `Throne-<tag>-windows-universal-installer.exe`; `-windows64.zip`; `-windows-arm64.zip`; `-windowslegacy64.zip`; `-windows32.zip`; `-linux-amd64.zip`; `-linux-arm64.zip`; `-debian-amd64.deb`; `-debian-arm64.deb`; `-debian-amd64-system-qt.deb`; `-debian-arm64-system-qt.deb`; `-macos-arm64.zip`; `-macos-amd64.zip`; `-macoslegacy-amd64.zip`; plus a workflow-only `debug-symbols.zip` (not a release asset). The README's deb naming (`…-debian-x64.deb`) is stale versus CI.

**Versioning & release contract**
- App version = CI input `tag` → `INPUT_VERSION` → a generated version constant. Tag convention: `x.y.z` with optional `-beta.N` (≈96 tags exist, `1.0.0` … `4.3.7`; no alpha/rc ever shipped, though the comparator supports them).
- Display: window title (windowed mode only), log header, Windows minidump metadata, and the default HTTP User-Agent `Throne/<version up to the first '-'>` (falls back to `1.0.0` if the string contains no dot).
- **In-app updater data contract**: `GET https://api.github.com/repos/throneproj/Throne/releases` (unauthenticated, API order); skip entries with `prerelease == true` unless "allow beta updates"; pick the **first** asset whose name contains the platform token **and** ends in `.zip`. Tokens: `windows-arm64`; `windows64` (x64 on build ≥ 10.1809); `windowslegacy64` (x64 older); `windows32`; `linux-amd64`; `linux-arm64`; `macos-amd64`; `macos-arm64` (so `macoslegacy-amd64.zip` matches nothing, and `windowslegacy64` matches through the `windows64` substring fallback). Version parse: strip the leading `Throne-` (7 chars), split on `-`: token 0 = `x.y.z`; token 1 is appended only if it contains `beta`/`alpha`/`rc`; token 4 (when 5 tokens) is the prerelease number. Numeric comparison; prerelease ordering alpha=1 < beta=2 < rc=3 < release.
- Download: the asset URL is saved as `Throne.zip` into the base path (portable: the app folder; appdata: the user config dir), honoring the app proxy settings and the insecure-TLS setting. Install: after confirmation the app exits with reason `RunUpdater`, sets cwd to the app dir, and on Windows copies `updater.exe` → `updater.old` and launches it (other OSes launch `./updater`). The updater is a separate third-party program that consumes `Throne.zip`; every release ZIP must contain a single top-level `Throne/` directory (layout contract relied upon by the updater and the Linux installer). A stale `updater.old` is deleted on the next GUI start.
- Missing updater binary → the menu action is disabled at startup (macOS, system-Qt deb, CLI installs, portable copies missing it); in appdata mode the update dialog offers only "Open in browser"/"Close".

**Release pipeline (CI)** — trigger `workflow_dispatch` only (no push/tag/PR triggers); inputs `tag` (required; doubles as the version) and `publish` (`r` normal release, `p` prerelease, empty = build only); three jobs:
- `build-go` — 9-entry matrix (`windows` amd64/arm64, `linux` amd64/arm64, `darwin` amd64/arm64, `darwinlegacy` amd64, `windowslegacy` amd64/386), `fail-fast: false`, output dirs `deployment/<os>-<arch>`; Go 1.26.5; build cache keyed on the build script + core sources (a hit skips build **and** upload); protoc 31.1 + pinned Go plugins; legacy jobs fetch the vendored Go zip; Linux jobs clone the cronet fork at the pinned commit and download the Chromium toolchain; uploads `Throne-<sha>-Common-<os>-<arch>` (tar of `./deployment`, compression level 0).
- `build-cpp` — 11-entry matrix (Windows x64 Qt 6.11.1; Windows arm64; Ubuntu amd64 6.11.1; Ubuntu amd64 system-Qt (apt 6.2.x); Ubuntu arm64 6.11.1; Ubuntu arm64 system-Qt; macOS amd64/arm64 6.11.1; macOS legacy 6.4.3; Windows legacy amd64 6.2.13; Windows legacy 386 6.2.13). Per-OS setup: MSVC via ms-dev-cmd (no MinGW job), Qt 7z + OpenSSL 7z from project releases, `install-qt-action` for non-system jobs, apt lists for system-Qt jobs, `-static-libgcc [-static-libstdc++]` link flags, macOS appdata packaging enabled + deployment target 10.15 + per-arch; each job downloads its Common artifact, downloads `srslist.h`, builds RelWithDebInfo with ccache/sccache, deploys, and uploads `Throne-<sha>-<dest_suffix>`. **No tests run anywhere** (no ctest, no go test), and there is **no signing/notarization** for any platform (macOS gets ad-hoc signing only).
- `publish` — merges all artifacts, installs NSIS, builds `ThroneSetup.exe` from the installer script and renames it, builds the four debs, renames each platform dir to `Throne`, zips with `zip -9` (macOS with `--symlinks`), collects debug symbols into `debug-symbols.zip`, uploads `Debug-<sha>` and `Deployment-<sha>`, then publishes with `ghr v0.17.0` (`-prerelease` for `p`; `-delete` replaces an existing same-tag release; release name/notes = the tag).

**Dev tooling** — a format script clang-formats tracked C/C++ files outside vendored trees (no root `.clang-format` exists — the effective config is UNKNOWN); a translate script runs the translation update target; there are no lint/static-analysis scripts.

### J.2 Behavior

**Portable vs packaged mode (the central artifact-behavior toggle)** — compile-time appdata mode via the packaging build options; runtime forced by `-appdata [dir]` (a following non-`-` argument overrides the location). Effect chain: config dir = user app-config location vs the app folder; the choice is persisted in settings and later read; the self-update action is allowed only outside appdata mode; the base path used for geo-asset downloads and the core's asset location env is the app folder in portable mode, the appdata dir otherwise.

**Unwritable portable fallback** — if the app folder/config dir is not writable (Program Files / read-only share), the app copies `config` (including the DB) to the user config dir, writes the marker file `config/.install-dir-unwritable` containing the install path (pinning the choice so a later elevated relaunch does not switch DBs), and behaves as appdata mode from then on.

**In-app update behavior** — per the contract above: check → compare → offer (beta-gated) → download `Throne.zip` → confirm → `RunUpdater` handoff to the sibling updater binary via the `updater.old` relay on Windows.

**Emoji/font behavior** — country flags and emoji render via regional-indicator emoji from the bundled fonts (Windows/Linux Qt ≥ 6.9); Windows uses Twemoji below Windows 11 22H2 and Noto otherwise; macOS and older Qt rely on system fonts.

**Other runtime flags that shape behavior** — `-tray` (start minimized to tray), `-many` (multi-instance allowance), `-debug` (debug logging), the two internal restart flags (re-added on relaunch; `-tray` removed), and URL/file launch arguments (forwarded to the primary instance, §H).

### J.3 Edge cases & errors
- Missing `srslist.h` → **compile failure** (included by route UI + config generation), never a runtime error.
- Build without `INPUT_VERSION` → empty version string: the update check logs "Version strings seem to be invalid" and reports no update; the window title shows a double space; the default UA becomes `Throne/1.0.0`; release build still works.
- Windows exe `FILEVERSION`/`PRODUCTVERSION` are 1.0.0.0 for every release (no version args are passed at configure time) — observable in file properties (INFERRED defaulting).
- macOS bundle version keys derive from the project default (0.1), which may disagree with the UI version (INFERRED; not overridden in-repo).
- Windows arm64 receives the **x64** updater binary (arch-suffix bug, INFERRED from string handling) — preserve for parity or fix deliberately.
- No checksum/signature verification exists for any downloaded binary (updater, libcronet, Qt/OpenSSL 7z, Go zip) — supply-chain property to be aware of.
- Whether the linuxdeploy `AppRun` survives into the Linux ZIP is UNKNOWN (cleanup removes `usr/bin`, `usr/share`, `apprun-hooks`, `usr/translations`, but not a root-level `AppRun`).
- MinGW: rc/font code has MinGW branches and CMake guards, but no CI job builds MinGW — support status UNKNOWN.
- RPM: no in-repo packaging; the README points to a third-party repository only.
- The updater program's internals are out of repo (only the observable contract above applies); the effective clang-format config is UNKNOWN (no root file).

### J.4 State & config
- Base/working dir: portable → app dir; appdata → the user app-config location (application name set to `Throne`); a `config` subdir is created and becomes the process CWD. Files: `config/throne.db` (seeded on first run), `config/logs/` (`throne.log` rotated + `running.marker` + `crashed-<stamp>.log`), `config/crashes/`, `config/temp/` (removed at startup), `config/dashboard/` (user-supplied dashboard files; an embedded notice page is shown when missing).
- Downloaded data: geo assets to the base path (with the asset-location env set for the core), `Throne.zip` for updates.
- Build/update-relevant settings: language index, allow beta updates, subscription user agent (default `Throne/<version>`), the two geo-asset URLs (defaults point at the Loyalsoldier route-data repo).
- Windows installer state (`Software\Throne` `InstallPath`, Uninstall key) is read back by the uninstaller to decide per-user vs all-users context.

## Acceptance Fixtures

Replay each row against a built app; "Expected" is the observable result. Kind = happy/edge/error + the section it proves. Strings in quotes are contractual.

| ID | Replay | Expected | Kind |
|---|---|---|---|
| FIX-001 | Fresh portable launch (no config dir) | `config/throne.db` + `config/logs/` + `config/crashes/` created; DB seeded with group `Default` and route profile `Default` (rule "Route DNS", action `hijack-dns`); window opens | happy / A,G |
| FIX-002 | Portable launch from a read-only folder | `config` copied to the user config dir; marker `config/.install-dir-unwritable` written with the install path; app behaves as appdata; elevated relaunch reuses the pinned dir | edge / A,J |
| FIX-003 | Start the app, then launch a second copy with `throne://addsub/<b64>` | second process exits promptly; first raises its window and runs the add-subscription flow; one tray icon | happy / H,I |
| FIX-004 | Kill the GUI while a profile runs (unclean), restart | prior log preserved as `logs/crashed-<stamp>.log`; `running.marker` removed only after a clean exit; startup log notes the unclean shutdown | edge / A |
| FIX-005 | Build with `INPUT_VERSION=4.3.7` | generated version header contains `4.3.7`; windowed title ends `Throne 4.3.7`; default UA `Throne/4.3.7` | happy / J |
| FIX-006 | Build without `INPUT_VERSION`; run update check | version string empty; log "Version strings seem to be invalid"; no update offered; UA `Throne/1.0.0` | edge / J |
| FIX-007 | Configure with the macOS appdata packaging option, run | the appdata build is reflected in the log; config under the user location without `-appdata`; update dialog has no Update button | happy / J |
| FIX-008 | Mock the release feed: assets `Throne-9.0.0-linux-amd64.zip`, `-linux-amd64.tar.gz`, `-linux-arm64.zip`; check on linux amd64 | only the `linux-amd64` `.zip` is selected | edge / J |
| FIX-009 | Mock feed: `Throne-9.0.0-beta.1-…-linux-amd64.zip`; check with beta allowed off/on | off → "No update"; on → update offered labeled "(Pre-release)" | edge / J |
| FIX-010 | Current `4.2.0` vs asset `4.2.0-beta.1`; and the reverse | first case: considered newer → offered; reverse: not newer | edge / J |
| FIX-011 | Run the NSIS installer on Win11 x64, Win8.1 x64, and 32-bit | installs the native amd64 / legacy-amd64 / legacy-386 payload; `libcronet.dll` only on the first; default dir `%LOCALAPPDATA%\Throne` (Just Me) vs `%PROGRAMFILES64%\Throne` (All Users) | happy / J |
| FIX-012 | Omit `updater` next to the binary; launch | "Check for Update" disabled at startup; `RunUpdater` unreachable | error / J |
| FIX-013 | Build without downloading `srslist.h` | compile failure on the missing header (not a runtime error) | error / J |
| FIX-014 | Configure Qt 6.2/6.4 (or any Qt on macOS) and Qt ≥ 6.9 (Win/Linux) | bundled emoji font resources exist only in the ≥ 6.9 non-Apple builds; startup logs "could not load emoji font!" when absent; Windows picks Twemoji below Win11 22H2 | edge / J |
| FIX-015 | Start any profile; observe config generation | engine config includes the mixed inbound on `127.0.0.1:2080` (or a random port when enabled), outbound chain per §C, DNS per settings, logging settings applied | happy / C |
| FIX-016 | Feed `vmess://eyJ2IjoiMiIsInBzIjoiRGVtbyIsImFkZCI6InZtZXNzLmV4YW1wbGUuY29tIiwicG9ydCI6Ijg0NDMiLCJpZCI6ImI4MzEzODFkLTYzMjQtNGQ1My1hZDRmLThjZGE0OGIzMDgxMSIsImFpZCI6IjAiLCJzY3kiOiJhdXRvIiwibmV0Ijoid3MiLCJob3N0IjoiY2RuLmV4YW1wbGUuY29tIiwicGF0aCI6Ii93cyIsInRscyI6InRscyIsInNuaSI6ImNkbi5leGFtcGxlLmNvbSJ9` | vmess profile "Demo", ws + TLS on, uuid/port as encoded; malformed base64 → dropped silently | happy/error / I |
| FIX-017 | Feed `ss://YWVzLTI1Ni1nY206VGVzdFBhc3MxMjNAc3MuZXhhbXBsZS5jb206ODM4OA#Demo` and the `?plugin=obfs-local%3Bobfs%3Dhttp%3Bobfs-host%3Dcdn.example.com` variant | ss legacy decodes to method/password/host:port; plugin becomes `obfs-local` + opts `obfs=http;obfs-host=cdn.example.com` | happy / I |
| FIX-018 | Feed `vless://<uuid>@h:443?type=ws&security=tls` and `vless://<uuid>@h:443?type=xhttp&security=reality&pbk=…&sid=…&fp=chrome` | first imports as sing-box VLESS; second as Xray VLESS (xhttp/reality); export shows `encryption=none` (sing-box) vs `encryption=<v>` + stream (xray) | happy / I |
| FIX-019 | Feed `hysteria2://user:pass@hy.example.com:8443?obfs-password=obfsPass&sni=hy.example.com#HY2` | hysteria v2: password `user:pass`, obfs set, TLS forced, port 8443 | happy / I |
| FIX-020 | Feed `mierus://user:pass@h?port=8964&port=2090-2099&protocol=TCP&protocol=UDP#M` | server_port 8964 + range `2090-2099`; protocol `TCP` (first wins); `mieru://` standard link dropped | happy/error / I |
| FIX-021 | Feed `socks5://RGVtb1VzZXI6RGVtb1Bhc3M@socks.example.com:1080` and `http://:tok3n@hp.example.com:8443` | socks: base64 username split into `DemoUser`/`DemoPass`; http: username = password = `tok3n`; default ports applied when absent (1080/443) | happy / I |
| FIX-022 | Feed the SIP008 JSON fixture (`version` + `servers[remarks,method,password,server,server_port]`) | one Shadowsocks profile named from `remarks` with the given method/password/endpoint | happy / I |
| FIX-023 | Feed a Clash `proxies:` list containing ss + trojan entries, then one with only `type: snell` | first: 2 profiles (trojan TLS forced); snell: zero profiles, no error; hostile YAML → "YAML Exception" box | happy/error / I |
| FIX-024 | Feed `throne://route/<b64 of the Fixture Route object>` | route profile "Fixture Route" created after the name prompt; 2 rules with tokens/rule_sets preserved; JSON without `kind` → "Unrecognized route object" | happy/error / I |
| FIX-025 | Feed `throne://remoteroute/<b64 of two url lines with #names>` | confirmation lists both URLs, Auto update on by default; profiles added and immediately fetched; non-http lines → "The link did not contain any valid http(s) routing profile URLs." | happy/error / I |
| FIX-026 | Feed `throne://add/<b64 of a vmess outbound JSON>`; and base64 of `{"foo":1}` | first imports a vmess profile; second is discarded (no `type`/`protocol`) | happy/error / I |
| FIX-027 | Feed `throne://addsub/<b64("https://sub.example.com/s#My Group")>` | prompt "Add this subscription?" with name `My Group`; group created, fragment stripped from the stored URL, fetch runs | happy / I |
| FIX-028 | Feed base64 of a 3-line body (vmess + trojan + `# comment`), then a UTF-8-BOM file with one trojan line | 2 profiles from the blob (comment skipped); BOM file imports 1 trojan profile; NUL-byte file rejected as not readable | happy/error / I |
| FIX-029 | Import a > 50 MB file; import a PNG QR containing a trojan link | file skipped `larger than 50 MB, skipped`; PNG imports the trojan profile; PNG without QR logs `no QR code found` | edge/error / I |
| FIX-030 | QR-export an SSH-key profile near encoder capacity; QR-import via screen scan on Wayland without portal | oversized payload → warning; portal failure → "Unable to capture screen"; a `throne://route/…` QR in a scan is ignored | edge/error / I |
| FIX-031 | Subscribe to a list; refresh with one added, one changed (same slot, new server), one removed | "＋/−" diff applied in place; changed id marked disturbed (running selector restarts); sticky selector profiles kept and repositioned | happy / D,I |
| FIX-032 | Refresh with clear-first on; with auto-clear-unavailable on and a failing profile | clear-first deletes non-sticky profiles first ("Clearing servers...", abort on delete failure); after a URL test the failed profile is removed | edge / D |
| FIX-033 | Run the route importer with a legacy bare rule array | accepted with `wasOldArray` semantics: no name/default imported; rule editor opens; empty array → "Input is not a valid json array" | edge / I |
| FIX-034 | Call `Start` with a serialized config, then `QueryStats` at 1 Hz | bytes attributed per watch tag; no drops under load; `QueryStats` never exceeds 500 ms timeout | happy / E,F |
| FIX-035 | Start a chain/selector config; watch the traffic callback math | credit groups distributed to member profiles per chain group; `direct` never attributed; multi-group profiles deduped; persist every ~10 s + on stop | happy / F |
| FIX-036 | Run a URL test with one success, one cancel, one error | results map to >0 (ms shown), 0 (neutral, display kept), −1 (`Unavailable` + `[%1] test error: %2`); speed failure → `N/A` + latency −1 | happy/edge / F |
| FIX-037 | Poll `QueryConnections` through a connection's full lifecycle | each connection removed exactly once (non-draining closed ring handled by the client); no duplicates; "Copied!" tooltip for 1.5 s on click | happy / F |
| FIX-038 | Selector: all members fail for > 20 s, with and without a default route | with no network: log "Every profile is failing, but this machine has no network connection — keeping the current pool." and no rebuild; with network: rebuild, backoff 60 s→600 s | edge / F |
| FIX-039 | Flood the log (line > 20 000 chars, queue overflow) | over-long line truncated to 1000; queue drops overflow silently; UI cap 200 lines; file log rotates 4 MB ×3; Warn+ flushes immediately | edge / F |
| FIX-040 | Write malformed values into settings storage (bad int, bad list, `"1"`/`"true"` bools) then load | bad int reads 0 (not default); bad list keeps default; `"true"`/`"1"` true; save rewrites all keys | edge / G |
| FIX-041 | Create a backup with profiles+routes+settings+icons; restore it into a fresh install; restore an old/other-platform archive | parts restored; only common columns copied; ids raised; no dangling refs; settings restore arms the no-save gate and prompts restart | happy/edge / G |
| FIX-042 | Run traffic for > 48 h virtual time; then beyond retention; disable aggregation | minute rows roll into hours after 48 h; hours pruned past retention (≥1 day); aggregation-off no-ops and hides the UI | edge / G |
| FIX-043 | Force-kill during writes; inspect DB files | WAL side files exist; a snapshot needs DB + `-wal` + `-shm`; checkpoint after 10 000 writes; vacuum only ≥4 MiB free & ≥50 % pages | edge / G |
| FIX-044 | Windows: enable system proxy with a running profile, then disable | Settings → Proxy shows manual `127.0.0.1:2080` on LAN + all RAS entries, no bypass; disable shows Direct; force-kill leaves stale proxy untouched at next start | happy/edge / H |
| FIX-045 | Linux GNOME/KDE enable + disable | GNOME `mode` → `manual` then `none` (host/port remain); KDE `~/.config/kioslaverc` gets `httpProxy=http://127.0.0.1 2080`, `socksProxy=socks://127.0.0.1 2080`, `ProxyType 1`→`0` + dbus signal | happy / H |
| FIX-046 | macOS enable + disable proxy | each enabled service gets web/secure-web/SOCKS proxies on `127.0.0.1`; `*`-marked services untouched; disable clears all four states | happy / H |
| FIX-047 | Windows TUN as non-admin; Linux/macOS TUN without setuid core | Windows: "Please run Throne as admin" → elevated relaunch with `-flag_restart_tun_on` → TUN starts; Linux: pkexec helper (`Please install "pkexec" first.` when missing); macOS: Terminal sudo chown/chmod + "Requesting permission" | happy/error / H |
| FIX-048 | macOS TUN with CIDR `172.19.0.1/24` start/stop | hardware port DNS becomes `172.19.0.2` while running; after stop `Empty` (DHCP) | happy / H |
| FIX-049 | Windows system DNS toggle set/clear | default interface DNS = `127.1.1.1` + marker (`127.1.2.3` DHCP / `127.3.2.1` static); cache flushed; clear restores previous servers | happy / H |
| FIX-050 | Linux/macOS system DNS toggle | log `Failed to set system dns:` + `not implemented for this OS`; dialog path reverts the checkbox; flag persists | error / H |
| FIX-051 | Enable autostart on Windows, Linux, macOS | Windows: task `Throne AutoRun <8hex>` exists (logon trigger, priority 5, run level per admin state, no args; legacy Run value replaced); Linux: `~/.config/autostart/Throne.desktop` with `-tray` (+`-appdata`); macOS: login item = bundle URL; disable removes each | happy / H |
| FIX-052 | Delete `HKCU\Software\Classes\throne`, restart | key recreated with `URL:Throne Protocol` + command `"<exe>" "%1"`; `.json` OpenWith entry present; second restart writes nothing (mirror) | happy / H |
| FIX-053 | Windows session end with proxy/TUN active; force a GUI crash | session-end teardown removes proxy/DNS/TUN before exit; crash writes `crashes\Throne_<ver>_<arch>_<stamp>.dmp` + `.txt` + message box; WER keys registered (admin) | edge / H |
| FIX-054 | Uninstall per-user with data deletion unchecked | Uninstall key + shortcuts removed; `%LOCALAPPDATA%\Throne` data left (the deletion path targets `%APPDATA%\Throne`) | edge / H |
| FIX-055 | Run the Linux CLI installer elevated | `/opt/Throne/Version` + `/usr/share/applications/Throne.desktop` (`Exec=/opt/Throne/Throne -appdata`); no `updater`; uninstall removes both | happy / H,J |
| FIX-056 | Start profile with stats off; enable stats; toggle system proxy mid-session | stats gate is live for the connections list but applies to config only at next start; proxy set/clear reacts immediately; "reset proxy on disable" restarts the profile | edge / F,H |
| FIX-057 | Feed a settings change while a profile runs | save writes all keys; "settings changed, restart proxy?" offered unless flagged no-restart; restoring settings counts as needing restart | happy / G |
| FIX-058 | Feed `throne://route/<b64>` with `"kind":"other"`; empty payload; only-ftp remoteroute | "Unrecognized route object"; "Deep link has no data"; "The link did not contain any valid http(s) routing profile URLs." | error / I |
| FIX-059 | Delete the stats DB and restart | usage history gone; app recreates it; profiles/settings untouched | edge / G |
| FIX-060 | Call `Invoke` methods with the wrong shape (unknown method, huge payload) | framing rejects oversized frames; unknown method errors do not crash either side; malformed responses drop with `[RPC] dropped malformed response: …` | error / E,F |

## Out of Scope

- **Embedded engines.** The bundled sing-box fork and the Xray core are external dependencies, not part of this mapping: protocol wire implementations, transport internals, the tun/UDP stacks, muxing, selector ranking/cooldown algorithms, and the stats-API server internals are engine territory. What this document fixes is everything observable around them — config shapes (§C, §D), RPC surface and lifecycle (§E), stats/listener payloads (§E, §F), and the generated-config gates the engines consume.
- **Third-party program internals.** The updater binary (separate project), the RPM ecosystem (third-party repo), and dashboard assets are referenced only by their external contracts.
- **Drivers.** No installer or deploy step provisions the Windows tun driver; driver-level behavior is out of scope.
- **Translation content.** The zh_CN/fa_IR/ru_RU translated strings are content, not contract; the English source strings quoted throughout are the contract.
- **CI infrastructure details** beyond the pipeline shape described in §J (secrets, runner images, cache backends).
- **Cryptography internals.** Wire-level crypto (Reality/XHTTP and obfuscation internals, key derivations) is engine scope; the configuration/query surface that selects it is in scope.
- **Known-UNKNOWN items.** Behaviors marked UNKNOWN in §§A–J were not established during discovery (e.g. the Linux systemd-resolved consumer, single-instance socket exposure risk, macOS arbitrary-file end-to-end handling, scheduled-task startup-args intent, the uninstaller data-path mismatch intent). A reimplementation may choose any behavior there but should record the choice; the fixtures avoid asserting them.
- **Performance tuning.** Memory budgets beyond the documented limits, thread counts, and cache sizing other than settings-exposed values are free to differ within the documented behavior.

