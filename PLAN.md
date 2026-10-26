# frame-launcher — Apple TV-style home screen for the Samsung Frame

Fork of [helalaou/AeroTizen](https://github.com/helalaou/AeroTizen) (MIT). Goal: restyle it into an Apple TV-like launcher for the living-room Frame, make it reliable on the 2021 Frame, and automate Art Mode. Hobby project, evenings/weekends.

- Repo: `X:\Projects\frame-launcher` → `origin` = github.com/XanderDeLaet/frame-launcher, `upstream` = helalaou/AeroTizen.
- Keep in sync: `git fetch upstream && git merge upstream/main`.

## Facts (verified 2026-10-26)
| Item | Value |
|---|---|
| TV | Samsung The Frame 65" `QE65LS03AAUXXN` (2021 → Tizen 6.0; confirm with `aerotizen status`) |
| IP / MAC | `192.168.0.197` / `04:B9:E3:3D:72:14` (reserve the IP in the router) |
| Open APIs | REST `:8001/api/v2/`, WebSocket remote + art `:8002`, DLNA `:9197`, DIAL `:7678`; token auth on |
| Dev PC | XanderDesktop, Wi-Fi `192.168.0.249`; node 20+, pnpm, python, docker present; no Tizen Studio (not needed: AeroTizen ships its own CLI) |
| Pi | `192.168.0.167`, idle, on the LAN (only needed for Phase 3) |
| Upstream status on this model | LS03A (2021): "runs, but doesn't return on Home or start at power-on yet" |

## How AeroTizen is built
- `launcher/` is the TV app: `index.html`, `css/style.css` (228 lines), `js/*.js` (home-screen, apps, app-catalog, backgrounds, clock, weather, inputs, settings-ui), plus a Preact settings screen in `launcher/src/settings/`.
- `launcher/service/index.js` is a Tizen background service that polls port 8001 and relaunches the launcher when Samsung Home covers it. `service/artmode.js` is a read-only Art Mode check so the launcher stays away while art is showing.
- `aerotizen` CLI (Python, prebuilt Windows exe in Releases) does discover, developer-mode status, certificates, sign, install, debloat. No Tizen Studio.
- Build: `npm ci && npm run build` → `dist/launcher`, then `aerotizen install ./dist/launcher`.
- `config.xml`: app id `DclHome001.Home`, privileges `application.launch`, `application.info`, `tv.window`, `tv.inputdevice`, `internet`.

## Phases

### Phase 0 — Stock AeroTizen on the TV (one evening)
1. Download the Windows x64 **CLI** release from the AeroTizen Releases page, extract, `.\aerotizen.exe discover` → should list 192.168.0.197.
2. TV: Apps → type `12345` → Developer mode ON → Host PC IP `192.168.0.249` → **unplug the TV for a minute** (remote off is not enough). `.\aerotizen.exe status` → `developerMode=1`, note the Tizen version.
3. `.\aerotizen.exe certs` (Samsung sign-in in the browser, free profile is fine for a 2021 set).
4. `.\aerotizen.exe install launcher` (stock release) → open it from the Apps row.
5. Write down what works on the LS03A: starts at power-on? returns on Home? Art Mode guard? This is the baseline for Phase 2.
**Done when:** stock AeroTizen runs on the Frame and the findings are in `docs/ls03a-notes.md`.

### Phase 1 — Apple TV restyle (one weekend)
- `npm ci && npm run build` locally first (not run yet), then `aerotizen install ./dist/launcher`.
- Layout (1920×1080, drawn at 1080p and upscaled): top shelf hero (today's art / a landscape / weather), one row of large rounded app tiles with parallax-ish focus (scale 1.08 + shadow, 150 ms ease-out), second row for inputs (HDMI 1 Telenet box, HDMI 2, Art Mode).
- Files to touch: `launcher/index.html`, `launcher/css/style.css`, `launcher/js/home-screen.js`, `launcher/js/backgrounds.js` (drop the 31 animated backgrounds or keep one), `launcher/js/brands.js` (tile artwork).
- Keep: `apps.js`/`app-catalog.js` (app discovery + launch), `inputs.js`, settings screen, the background service.
- Tizen 6 gotchas (Chromium 76): no flexbox `gap`, no `inset`, no `:is()`/`:where()`, no optional chaining in plain `js/` files unless the build transpiles them; use `transform`/`opacity` only for animation, no big `filter: blur`.
**Done when:** TV boots into the restyled launcher and Netflix, YouTube and the Telenet box open and come back with Return.

### Phase 2 — Make it work properly on the 2021 Frame (a few evenings)
- Debug `launcher/service/index.js` on the LS03A: why no power-on launch and no Home return. Likely the 8001 app-visibility reporting or the Art Mode guard waiting forever on this firmware.
- Add logging the CLI can read, fix, and open a PR upstream with a model report.
**Done when:** Home returns to the launcher from Samsung Home and Netflix, and cold boot lands on the launcher (or in Art Mode when art was on).

### Phase 3 — Art Mode automation on the Pi (optional, one weekend)
- AeroTizen only *reads* Art Mode. Uploading or selecting art needs the `:8002` art channel from an always-on LAN device → small Python service on the Pi with `samsungtvws` (pair once, "Allow" on the TV).
- Daily rotation at 07:00 (own OneDrive photos, Unsplash, or generated), Art Mode on in the morning, TV off at night, Wake-on-LAN.
- Skip this phase if "new picture every morning" is not wanted.

### Phase 4 — Later (new house)
- Home Assistant + KNX scenes drive TV, lights, blinds; Art Mode follows presence/daylight.

## Risks
- Firmware updates may close the sideload route; keep the `.wgt` and certs backed up.
- Samsung developer certificates expire and need re-signing.
- Only one 2021 Frame has been reported upstream and it is the model with issues: Phase 2 may take longer than planned.
- Autorun Samsung TV Plus / Autorun Last App must be off or they open before the launcher.

## Decisions before Phase 1
1. Strict Apple TV clone or calmer gallery-first look that fits a Frame?
2. Keep AeroTizen's weather/clock header or go minimal?
3. Art source for Phase 3 (OneDrive photos, Unsplash, generated) — only if Phase 3 happens.

## Next action
Phase 0, step 1: download the AeroTizen CLI and run `discover`.
