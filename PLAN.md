# Strikemap — Development Plan
*Created from design conversations, July 2026*

---

## Timeline

- **c. 2007** — Original concept: point at a flash, tap at thunder, get distance and bearing. No means to build it at the time.
- **~April 2026** — Idea revisited and sketched out in design conversation form (UI mockup, geometry approach).
- **Early May 2026** — Sessions 1–5 built: real map, compass + recording loop, corrected uncertainty geometry (pizza crust), three-tab layout with couch mode, compass UX rework, live refresh, desktop companion app, and app icon. `strikemap.html` and `strikemap-desktop.html` reached final form. Confirmed via git history (2026-05-08).
- **July 2026** — Repo split out from the `apps` monorepo into its own project; retroactive plan/notes documentation captured across this and the preceding sessions.

---

## What We're Building

A single-file static HTML app for real-time lightning tracking. Point your phone at a flash, tap a button, tap again at thunder. The app calculates distance and bearing, logs each strike with age-based color coding, and plots uncertainty regions on a live map so you can watch a storm approach or recede.

Hosted on GitHub Pages. No server, no API keys, no accounts. Works for multiple users sharing the same URL — each person tracks independently on their own device.

---

## Decisions Made (and Why)

### No backend / no Google Sheets
Early idea was to log to a Google Sheet for persistence. Dropped it — localStorage is sufficient for the duration of a storm session. You're not trying to archive this data across days, just track a 30–45 minute event. CLEAR button wipes it and you start fresh next storm.

### No external lightning data (for now)
Blitzortung.org was identified as the best free community option for real strike confirmation data. Xweather (formerly AerisWeather) has a usable free dev tier for queried JSON. Decision: not worth the complexity for v1. A stub function `fetchExternalStrikes()` returning `[]` is in the code so the hook exists. Add it later if desired.

### No Google Maps, no Mapbox
Both require API keys. Keys in client-side code on a public GitHub repo is bad practice, and adding referrer restrictions is friction for a personal tool shared with a few people. OpenStreetMap + Leaflet is genuinely free, no key, CDN-loaded, and works everywhere. Easy call.

### Static map — no pinch to zoom (mobile)
Pinch-to-zoom gets out of whack quickly on a phone, then you need a recenter button, and the UX complexity grows. The whole point of the map is spatial awareness of storm progression, not navigation. A locked static view at zoom 12 (~10–12 mile wide area) is cleaner and more useful. One RE-CENTER button in the controls strip is the only escape hatch.

The **desktop version** (`strikemap-desktop.html`) enables full drag + scroll-wheel zoom since a mouse-driven map is natural and doesn't suffer the same UX problems. Manual `+ ZOOM` / `− ZOOM` buttons in the toolbar supplement the scroll wheel.

### Zoom level 12 as default
This shows roughly a 10–12 mile wide area, which means ~5–6 mile radius from center. This aligns almost exactly with the 30-second thunder delay rule (30s × 343 m/s ≈ 6.3 miles). Strikes beyond that get logged with a DISTANT flag but aren't plotted — the map view doesn't need to stretch for them.

### Tile provider: CartoDB Dark Matter
Standard OSM tiles are light and colorful — they fight the dark UI hard. CartoDB Dark Matter is free, no API key required, CDN-loaded, and matches the `--bg:#080c10` aesthetic naturally.

### Pizza crust, not ellipse — this is the final correct geometry
Early mockup and early implementation used ellipses centered on the strike point. This was geometrically wrong in two ways:

1. **Wrong shape origin.** Both error sources fan outward from the *observer*, not from the strike. An ellipse centered on the strike implies uncertainty spreads equally in all directions from that point, which doesn't match the physics.

2. **Wrong error model.** The uncertainty region is polar (defined in bearing + distance from observer), not cartesian. Projecting it into cartesian space and fitting an ellipse loses the actual shape.

The correct shape is an **annular sector** ("pizza crust"):
- **Bearing error (±10°):** Two rays from the observer at `bearing ± 10°`. Wedge width grows linearly with distance. Started at ±5°, revised to ±10° as more realistic for field pointing accuracy — tune after testing.
- **Timing error (±500ms):** Inner arc at `dist − 0.107mi`, outer arc at `dist + 0.107mi`. Constant depth regardless of distance.

Built as an `L.polygon` by projecting points via `strikeLatLon()`. Center dot plotted at the best-estimate point. The ±250ms timing assumption from the original plan was revised upward to ±500ms (more realistic for human tap reaction, especially startled by thunder).

### Bearing lines kept as faint dashes
A faint dashed line from user to each strike center helps read the bearing visually even though the crust is the precise uncertainty region. Kept at low opacity so it doesn't clutter.

### Age color scheme
Three tiers for storm timescales:
- < 5 min → Yellow `#f5c842` (active concern)
- 5–15 min → Orange `#f08030` (context)
- > 15 min → Blue-grey `#4a7aaa` (historical, fading)
- > 60 min → Expired (soft — kept in log at 45% opacity, hidden from map)

Applied consistently to log entries, pizza crusts, bearing lines, and trend bars.

### Strike soft-expiry at 60 minutes
Strikes older than `EXPIRE_MIN` (60 min) are soft-expired: kept in localStorage (safe to reload mid-storm), shown in the TRENDS log with dimmed style and `EXPIRED` tag, but excluded from map, trend calculation, approaching alert, and OBS badge. Hard-delete on load was explicitly rejected — losing data on accidental reload during a storm is unacceptable.

### iOS compass permission UX — tap compass ring to enable
`DeviceOrientationEvent.requestPermission()` on iOS 13+ must be triggered by a user gesture. Originally wired to FLASH tap, but this caused a double-prompt on open (location fires immediately, compass would fire on first FLASH). Changed in Session 4b: compass shows `TAP TO START` on open, first tap on the compass ring fires the permission request. On Android/desktop no dialog is needed — first tap silently starts the bezel. This keeps the two prompts separated in time and intent.

### Compass UX: tap-to-lock, after FLASH
1. Open app — compass shows `TAP TO START`
2. Tap compass ring — permission fires, bezel goes live
3. Tap FLASH → starts timer, compass turns yellow (armed)
4. Swing phone toward where lightning was, hold still
5. Tap compass ring to lock — display snaps to green "✓ NNE"
6. Set phone down, wait for thunder
7. Tap HEARD THUNDER

Tapping compass again while locked will unlock it, allowing re-aim. If user never taps compass before HEARD THUNDER, strike logs with `bear: null`.

### Compass display: rotating bezel, no needle
A spinning needle pointing at a fixed angle on screen is confusing. Rotating bezel (N/S/E/W ring rotates opposite to heading so N always faces north, like a real compass) plus fixed yellow lubber line at 12 o'clock. Center shows degrees + cardinal text. N label is red per standard compass convention.

### Three-tab layout: RECORD / TRENDS / MAP
The original two-panel RECORD layout (compass + log side by side) is too cramped on a phone. Separating into three tabs gives each pane room to breathe:
- RECORD: full-width single column — compass, couch toggle, buttons, calc box
- TRENDS: trend chart + full log with delete controls
- MAP: map only

The mockup's two-column layout is used in the desktop version (separate file, completed Session 5).

### Couch mode
For recording distance-only from inside (no line of sight to flash). Toggle on RECORD screen disables compass entirely — no permission request, no lock UI, `bear` saves as `null`. On the map, null-bearing strikes render as an **annular donut** at `distMi` radius (not a filled circle — same timing uncertainty band as the pizza crust, just swept 360°, dashed stroke, reduced fill opacity). No bearing line. Communicates "definitely this far away, direction unknown."

Couch mode is **on by default in the desktop version** — desktop machines have no compass. The toggle still exists if a user wants to disable it (e.g. on a tablet with a sensor).

### Clear / delete UX
- **CLEAR ALL** lives on TRENDS tab — confirm dialog required, then wipes localStorage
- **Per-entry ✕** on each log row — no confirm needed for a single strike, just delete by `ts`
- No CLEAR on MAP tab — data management lives in one place

### Single HTML file (per version)
No build step, no npm, no bundler. Leaflet from CDN, fonts from Google Fonts CDN, everything else vanilla JS. Drop files in a GitHub repo, enable Pages, done. Two files:
- `index.html` (renamed from `strikemap.html`) — mobile version, primary
- `strikemap-desktop.html` — desktop version

### Trend bar height floor
`TREND_BAR_MIN_PCT = 20`. When a storm approaches very close, the newest bars (short distance = short bar in a descending storm) would get crushed tiny relative to earlier distant strikes. The 20% floor keeps all bars legible while preserving relative proportions above that threshold.

### App icon
SVG master at 512×512 (`strikemap-icon.svg`): vertical lightning bolt (yellow with white-hot core, glow filter) striking the center of a bullseye on a black background. Bullseye rings use app palette colors — outermost blue-grey, then orange, two yellow, red bull. Faint crosshair lines add targeting feel. Also rendered as 180×180 PNG (`strikemap-icon-180.png`) for Apple touch icon.

---

## Development Sessions

### Session 1 — Real Map ✅ COMPLETE
**Goal:** Leaflet + OSM rendering correctly centered on user location.

**Completed:**
- Leaflet 1.9.4 CSS + JS loaded from CDN
- CartoDB Dark Matter tiles (no key, fits dark UI)
- All interaction disabled: `dragging`, `touchZoom`, `scrollWheelZoom`, `doubleClickZoom`, `boxZoom`, `keyboard`, `tap`
- `navigator.geolocation.getCurrentPosition()` fires on load; centers map on result
- Fallback center: James Island / Charleston, SC (`32.7357, -79.9956`)
- "ACQUIRING LOCATION…" overlay while waiting; hides on success
- Cyan pulsing "YOU ARE HERE" dot via `L.divIcon` with CSS `pulse-ring` animation
- `⊕ CENTER` button calls `map.setView([userLat, userLon], 12)`
- `showScreen('map')` calls `map.invalidateSize()` — required because Leaflet container is hidden on initial load
- `window.sm_map` exposes the map instance for cross-function access

---

### Session 2 — Compass + Flash/Thunder → localStorage ✅ COMPLETE
**Goal:** Core recording loop works end to end.

**Completed:**
- `DeviceOrientationEvent` / `webkitCompassHeading` wired for iOS and Android
- 8-sample circular mean heading buffer for jitter smoothing
- Rotating bezel compass — bezel rotates opposite to heading, N always faces north
- Fixed yellow lubber line at 12 o'clock marks phone's forward direction
- Compass ring is a `<button>` — tap to lock heading (green "✓ NNE"), tap again to unlock
- SAW FLASH → starts 100ms timer interval, arms THUNDER button, sets `bearingArmed = true`
- HEARD THUNDER → reads `lockedHeading` first, then clears state, calculates distance, saves strike
- Strike saved as `{ ts, delay, distMi, bear, card }` to localStorage newest-first
- Calc box updates live during flash-to-thunder interval

---

### Session 3 — Pizza Crusts + Map Strike Rendering ✅ COMPLETE
**Goal:** Strikes plotted as correct uncertainty regions on the live map.

**Completed:**
- `strikeLatLon()` — great-circle projection from observer to strike given bearing + distance
- `makePizzaCrust()` — annular sector polygon (16-step outer arc + 16-step inner arc)
- Center dot (`L.circleMarker`) + faint dashed bearing line at each strike
- Distance label on most recent strike (quadrant offset to avoid overlapping the dot)
- `clearStrikeLayers()` + `strikeLayers[]` array — full redraw on each render call
- Age filter buttons (ALL / 10 MIN / 30 MIN) — `data-age` attribute, active state toggle
- DISTANT notice banner for strikes beyond `MAX_DIST_MI`
- APPROACHING alert — starts hidden, shown when active trend is negative
- Map strip: OBS, CLOSEST, DIRECTION, TREND — all driven by `updateMapStrip()`
- Fixed ellipse geometry bug (ellipses were wrong shape and wrong origin)
- Fixed age filter toggle bug (was matching CENTER button text)
- Fixed APPROACHING alert visibility (was hardcoded visible in HTML)

---

### Session 4 — Three Tabs + Couch Mode ✅ COMPLETE
**Goal:** Full phone-ready layout, couch mode, all initialization bugs fixed, clear/delete UX.

**Completed:**
- Three-tab nav: RECORD / TRENDS / MAP
- RECORD restructured as single full-width column
- TRENDS screen: trend bar chart + full log + CLEAR ALL + per-entry ✕ delete
- Couch mode toggle with pip switch UI — hides compass, shows status label, `bear` saves as `null`
- MAP: null-bearing strikes render as annular donut (not filled circle, not `L.circle`)
- HEARD THUNDER starts inert — no `armed` class in HTML
- All calc box fields initialized to `—` in HTML
- Trend bars initialize empty (`NO DATA`) on load
- Map strip stats initialize to `—` in HTML
- Timestamp `setInterval` is unconditional — runs regardless of active tab
- CLEAR ALL wired to TRENDS tab only, with `confirm()` dialog
- Per-entry ✕ delete — no confirm, re-renders log + map

---

### Session 4b — Compass UX + Expiry + Donut Geometry ✅ COMPLETE
**Goal:** Compass tap-to-enable (no double prompt), soft strike expiry, correct donut shape.

**Completed:**
- Compass permission moved from FLASH tap to first compass ring tap
- Compass shows `TAP TO START` on open; `updateCompassDisplay()` has a pre-permission state
- `startCompass()` calls `updateCompassDisplay()` after permission granted so bezel starts moving immediately
- FLASH no longer calls `requestCompassPermission()`
- Soft expiry: `EXPIRE_MIN = 60` constant, `isExpired(ts)` helper
- Expired strikes: shown in TRENDS log at 45% opacity with `EXPIRED` tag; excluded from map, trend, OBS badge
- Log count shows `N STRIKES · M ACTIVE` when some are expired
- OBS badge counts active (non-expired) only
- Annular donut (`makeDonut()`) replaces `L.circle` / `makeBullseye()` for null-bearing strikes — same `TIMING_ERROR_S` band, 36 steps, dashed stroke, `fillOpacity: 0.15`
- `makePizzaCrust()` now references named constants (`BEARING_ERROR_DEG`, `TIMING_ERROR_S`) instead of hardcoded values
- `BEARING_ERROR_DEG = 10` (was embedded as `5` in Session 3)
- All tunable values as named top-level constants
- Single `DOMContentLoaded` block — init, clear-all listener, age filter listeners unified
- Map legend text updated to reflect ±10° and donut

---

### Session 5 — Live Refresh + Desktop + Icon ✅ COMPLETE
**Goal:** Age colors stay live without user interaction; desktop companion app; app icon.

**Completed:**

**Mobile (`strikemap.html` → rename to `index.html`):**
- Live age color refresh: 30s interval now calls `renderMapStrikes()` and `updateObsBadge()` in addition to `renderLog()` — age colors on map, OBS badge, and map strip all update without user interaction
- Trend bar height floor: `TREND_BAR_MIN_PCT = 20` constant — bars never shorter than 20% even when close strikes are tiny relative to earlier distant ones
- Timer interval variable renamed `timerInterval` (was inconsistently named `timerTick`)

**Desktop (`strikemap-desktop.html`):**
- Two-column layout: 340px left panel (tabbed RECORD / TRENDS / LOCATION) + full-height interactive map
- Couch mode on by default (`couchMode = true` at declaration); compass section hidden; IIFE applies visual state before first paint
- Map is fully interactive: drag, scroll-wheel zoom, keyboard; manual `+ ZOOM` / `− ZOOM` buttons in toolbar
- LOCATION tab with three entry methods:
  - **Address search** via Nominatim geocoder (OSM, no API key) — up to 4 clickable results, selecting jumps to RECORD tab
  - **Manual lat/lon** — two number fields + SET button with validation and OK ✓ / ERR flash feedback
  - **Use Device Location** button — explicit user gesture, calls `getCurrentPosition()`
- Silent background geolocation attempt on page load (no prompt or error if it fails — user sets via LOCATION tab)
- `geo-status` overlay reads "SET LOCATION IN ← LOCATION TAB" on desktop instead of "ACQUIRING…"
- All constants, geometry, strike schema, and localStorage key identical to mobile — same `sm_strikes` key, data shared if both open in same browser

**Icon:**
- `strikemap-icon.svg` — 512×512 SVG master: vertical lightning bolt (yellow + white-hot core, glow filter) striking bullseye center; rings use app palette (blue-grey outermost → orange → yellow → yellow → red bull); faint crosshair lines; warm radial glow on black background; rounded-square crop via `border-radius:22%`
- `strikemap-icon-180.png` — 180×180 PNG rendered via cairosvg for Apple touch icon

**Current constants (both files):**
```javascript
const SOUND_MPS          = 343;
const MI_PER_M           = 0.000621371;
const MAX_DIST_MI        = 6.2;
const BEARING_ERROR_DEG  = 10;
const TIMING_ERROR_S     = 0.5;
const EXPIRE_MIN         = 60;
const TREND_BAR_MIN_PCT  = 20;
const LS_KEY             = 'sm_strikes';
```

---

## Deploy Checklist

```
strikemap.html          → rename to index.html
strikemap-desktop.html  → keep filename
strikemap-icon.svg      → repo root
strikemap-icon-180.png  → repo root

In <head> of both HTML files:
  <link rel="icon" href="strikemap-icon.svg">
  <link rel="apple-touch-icon" href="strikemap-icon-180.png">
  <meta name="theme-color" content="#080c10">

GitHub: Settings → Pages → Deploy from branch → main → / root
Wait ~60s → https://yourusername.github.io/reponame/
```

---

## Field Test Checklist (requires HTTPS)

**iOS (priority):**
- [ ] Location prompt appears on open
- [ ] Compass shows `TAP TO START` with cyan glow — no compass prompt yet
- [ ] Tap compass ring → iOS permission dialog
- [ ] After permission: bezel rotates live, degrees update
- [ ] Tap SAW FLASH → thunder button arms orange; compass turns yellow
- [ ] Tap compass ring → locks green `✓ NNE`
- [ ] Tap HEARD THUNDER → strike saved; TRENDS log shows it; MAP plots pizza crust
- [ ] Wait 30s without touching — map colors and OBS badge update on their own
- [ ] Couch mode: compass hidden; HEARD THUNDER saves `bear: null`; MAP shows donut ring
- [ ] Expiry test: set `EXPIRE_MIN = 1`, wait, verify map clears + `EXPIRED` tag + badge drops

**Android:**
- [ ] Chrome: `deviceorientation` fires without dialog — bezel live on first tap
- [ ] Firefox: may need `about:config` → `device.sensors.enabled` = true

**Desktop:**
- [ ] Map drag + scroll-wheel zoom works
- [ ] LOCATION tab: address search returns results, selecting one centers map
- [ ] LOCATION tab: manual lat/lon SET validates and re-centers
- [ ] Couch mode on by default
- [ ] Strikes recorded on mobile appear on desktop if same browser / same `sm_strikes` key

---

## Tunable Constants (top of script block)

```javascript
const SOUND_MPS          = 343;    // m/s speed of sound
const MI_PER_M           = 0.000621371;
const MAX_DIST_MI        = 6.2;    // beyond this → DISTANT, not plotted
const BEARING_ERROR_DEG  = 10;     // ±° pointing error — tune after field testing
const TIMING_ERROR_S     = 0.5;    // ±s human tap reaction time
const EXPIRE_MIN         = 60;     // minutes before soft-expiry
const TREND_BAR_MIN_PCT  = 20;     // minimum bar height % in trend chart
// Age thresholds: hardcoded in ageClass() — <5min recent, <15min mid, else old
```

---

## After v1 — Possible Future Work

- **Blitzortung overlay**: wire `fetchExternalStrikes()`, show confirmed strikes as distinct markers for comparison with your observations. Good calibration tool.
- **Bearing-error tuning**: ±10° is a field estimate. After testing, compare logged bearings against known landmarks or Blitzortung data and adjust the constant.
- **Multiple observers**: if two people at known locations both record the same strike, triangulation becomes possible. Out of scope for now.
- **Tile variants**: CartoDB Positron (light) or standard OSM if dark tiles feel wrong in daylight use.
- **Couch mode persistence**: currently resets to off on reload (mobile). Could persist to localStorage with a single flag if desired.

---

## Files

| File | Description |
|---|---|
| `PLAN.md` | This document |
| `NOTES.md` | Implementation decisions and gotchas discovered during development |
| `README.md` | Project overview, live URLs, geometry explanation |
| `strikemap-mockup.html` | Original UI mockup — historical artifact, not the live app |
| `index.html` | **Mobile live file** |
| `desktop.html` | **Desktop live file** |
| `strikemap-icon.svg` | App icon master (512×512 SVG) |
| `strikemap-icon-180.png` | Apple touch icon (180×180 PNG, rendered from the SVG) |
| `img/` | README imagery — icon render, screenshots |
| `LICENSE.md` | PolyForm Noncommercial 1.0.0 |

---

## Post-Deploy Cleanup (July 2026)

A fresh-eyes pass after the initial deploy, checked against the [App Patterns field guide](https://neely.github.io/patterns/) (Strikemap is a clean fit for **Pattern 1** — static single-file, `localStorage`-only state, reset-is-the-design; not yet listed as an example there):

- **File organization** — renamed `icons/` to `img/` since that folder only ever held README imagery (never an app-functional icon), and it's about to hold screenshots too.
- **Icon background fix** — the SVG only filled a circle inscribed in the 512×512 canvas, leaving the four corners transparent. iOS fills transparent home-screen icon corners with white, which looked wrong. Added an opaque square background rect (matching the app's `--bg` color, `#080c10`) behind the circular glow, then re-rendered both PNGs. Also fixes inconsistent appearance in light-mode GitHub rendering.
- **Open Graph / meta description tags** — added to both `index.html` and `desktop.html` so shared links get a proper title/description/preview image instead of nothing.
- **Accessibility** — the per-entry delete button (`✕`) only had a `title` attribute; added `aria-label="Delete strike"` for screen readers. Everything else already had visible text labels.
- **Constants audit** — confirmed `SOUND_MPS`, `MAX_DIST_MI`, `BEARING_ERROR_DEG`, `TIMING_ERROR_S`, `EXPIRE_MIN`, `TREND_BAR_MIN_PCT`, and `LS_KEY` are identical between `index.html` and `desktop.html` — no drift between the two files.
