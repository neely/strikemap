# Strikemap — Development Plan
*Created from design conversations, July 2026*

---

## Timeline

- **c. 2007** — Original concept: point at a flash, tap at thunder, get distance and bearing. No means to build it at the time.
- **~April 2026** — Idea revisited and sketched out in design conversation form (UI mockup, geometry approach).
- **Early May 2026** — Sessions 1–3 built: real map, compass + recording loop, corrected uncertainty geometry (pizza crust). Confirmed via git history (2026-05-08).
- **July 2026** — Repo split out from the `apps` monorepo into its own project; Session 4 (three-tab layout, couch mode) in progress.

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

### Static map — no pinch to zoom
Pinch-to-zoom gets out of whack quickly on a phone, then you need a recenter button, and the UX complexity grows. The whole point of the map is spatial awareness of storm progression, not navigation. A locked static view at zoom 12 (~10–12 mile wide area) is cleaner and more useful. One RE-CENTER button in the controls strip is the only escape hatch.

### Zoom level 12 as default
This shows roughly a 10–12 mile wide area, which means ~5–6 mile radius from center. This aligns almost exactly with the 30-second thunder delay rule (30s × 343 m/s ≈ 6.3 miles). Strikes beyond that get logged with a DISTANT flag but aren't plotted — the map view doesn't need to stretch for them.

### Tile provider: CartoDB Dark Matter
Standard OSM tiles are light and colorful — they fight the dark UI hard. CartoDB Dark Matter is free, no API key required, CDN-loaded, and matches the `--bg:#080c10` aesthetic naturally.

### Pizza crust, not ellipse — this is the final correct geometry
Early mockup and early implementation used ellipses centered on the strike point. This was geometrically wrong in two ways:

1. **Wrong shape origin.** Both error sources fan outward from the *observer*, not from the strike. An ellipse centered on the strike implies uncertainty spreads equally in all directions from that point, which doesn't match the physics.

2. **Wrong error model.** The uncertainty region is polar (defined in bearing + distance from observer), not cartesian. Projecting it into cartesian space and fitting an ellipse loses the actual shape.

The correct shape is an **annular sector** ("pizza crust"):
- **Bearing error (±5°):** Two rays from the observer at `bearing ± 5°`. Wedge width grows linearly with distance.
- **Timing error (±500ms):** Inner arc at `dist − 0.107mi`, outer arc at `dist + 0.107mi`. Constant depth regardless of distance.

Built as an `L.polygon` by projecting ~26 points via `strikeLatLon()`. See NOTES.md for full implementation. Center dot still plotted at the best-estimate point. The ±250ms timing assumption from the original plan was revised upward to ±500ms (more realistic for human tap reaction, especially startled by thunder).

### Bearing lines kept as faint dashes
A faint dashed line from user to each strike center helps read the bearing visually even though the crust is the precise uncertainty region. Kept at low opacity so it doesn't clutter.

### Age color scheme
Three tiers for storm timescales:
- < 5 min → Yellow `#f5c842` (active concern)
- 5–15 min → Orange `#f08030` (context)
- > 15 min → Blue-grey `#4a7aaa` (historical, fading)

Applied consistently to log entries, pizza crusts, bearing lines, and trend bars.

### iOS compass permission UX
`DeviceOrientationEvent.requestPermission()` on iOS 13+ must be triggered by a user gesture. Requested on first FLASH tap — natural moment, satisfies the requirement. If denied or unavailable, `bear` saves as `null`, strike logs with distance only, map skips the crust.

### Compass UX: tap-to-lock
1. Tap FLASH → starts timer, compass goes live (bezel rotating freely)
2. Swing phone toward where lightning was, hold still
3. Tap compass ring to lock — display snaps to green "✓ NNE"
4. Set phone down, wait for thunder
5. Tap HEARD THUNDER

Tapping compass again while locked will unlock it, allowing re-aim. If user never taps compass before HEARD THUNDER, strike logs with `bear: null`.

### Compass display: rotating bezel, no needle
A spinning needle pointing at a fixed angle on screen is confusing. Rotating bezel (N/S/E/W ring rotates opposite to heading so N always faces north, like a real compass) plus fixed yellow lubber line at 12 o'clock. Center shows degrees + cardinal text only. N label is red per standard compass convention.

### Three-tab layout: RECORD / TRENDS / MAP
The original two-panel RECORD layout (compass + log side by side) is too cramped on a phone. Separating into three tabs gives each pane room to breathe:
- RECORD: full-width single column — compass, couch toggle, buttons, calc box
- TRENDS: trend chart + full log with delete controls
- MAP: map only

### Couch mode
For recording distance-only from inside (no line of sight to flash). Toggle disables compass entirely — no permission request, no lock UI. `bear` saves as `null`. On the map, null-bearing strikes render as a full-circle bullseye ring at `distMi` radius (dashed, age color) instead of a pizza crust. Communicates "definitely this far away, direction unknown."

### Clear / delete UX
- **CLEAR ALL** lives on TRENDS tab — confirm dialog required, then wipes localStorage
- **Per-entry ✕** on each log row — no confirm needed for a single strike, just delete by `ts`
- No CLEAR on MAP tab — data management lives in one place

### Single HTML file
No build step, no npm, no bundler. Leaflet from CDN, fonts from Google Fonts CDN, everything else vanilla JS. Drop the file in a GitHub repo, enable Pages, done.

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
- Compass permission requested on first FLASH tap
- 8-sample circular mean heading buffer for jitter smoothing
- Rotating bezel compass — bezel rotates opposite to heading, N always faces north
- Fixed yellow lubber line at 12 o'clock marks phone's forward direction
- Compass ring is a `<button>` — tap to lock heading (green "✓ NNE"), tap again to unlock
- `bearingArmed` flag: compass only accepts lock taps after FLASH is pressed
- FLASH: starts timer, sets `bearingArmed = true`, compass goes live
- THUNDER: captures locked bearing (or null), computes delay + distance, writes to localStorage, renders log
- Bearing captured at top of `onThunder()` before state is cleared
- `bear: null` handled gracefully — logs as "--- NO HDG", skipped on map
- Trend bar chart from up to 8 most recent strikes, colored by age
- Approaching / receding / steady indicator from last two strikes
- CLEAR button with `confirm()` dialog, wipes localStorage
- Timestamps refresh every 30s via `setInterval`

---

### Session 3 — Pizza Crusts + Map Fixes ✅ COMPLETE
**Goal:** Correct uncertainty geometry on map; fix initialization bugs and map wiring.

**Completed:**
- Replaced ellipse geometry with correct annular sector ("pizza crust") — `makePizzaCrust()` projects points directly from observer via `strikeLatLon()`, no cartesian ellipse math
- Timing error revised to ±500ms (was ±250ms) — more realistic for human tap reaction
- `TIMING_ERR_S` and `BEARING_ERR_DEG` constants removed; values embedded in `makePizzaCrust()` with comments
- Age filter buttons fixed — now use `data-age` attributes to scope active-state toggle; CENTER button is no longer accidentally affected
- APPROACHING alert — removed hardcoded `display:flex` from HTML; now starts hidden, JS-controlled entirely
- DISTANT notice banner — added `#distant-notice` element and CSS; shows count of strikes >6.2mi not plotted
- `fetchExternalStrikes()` stub added — async, returns `[]`, ready for Blitzortung hook
- Map legend present, positioned `bottom:60px right:12px` as absolute overlay

**Current constants in file:**
```javascript
const SOUND_MPS   = 343;
const MI_PER_M    = 0.000621371;
const MAX_DIST_MI = 6.2;
const LS_KEY      = 'sm_strikes';
```

---

### Session 4 — Three Tabs + Couch Mode + Polish ← START HERE
**Goal:** Full phone-ready layout, couch mode, all initialization bugs fixed, clear/delete UX complete.

**Tasks — do all in one pass, in this order:**

1. **Three-tab nav (RECORD / TRENDS / MAP)**
   - Add TRENDS button between RECORD and MAP in header nav
   - Add `#screen-trends` section
   - Move trend bar chart + strike log into TRENDS screen
   - RECORD becomes single full-width column: compass + couch toggle + buttons + calc box only
   - `showScreen('trends')` calls `renderLog()` and `renderTrend()` on activation

2. **Couch mode**
   - Toggle button on RECORD screen, between compass and SAW FLASH
   - When active: hide compass ring, show "COUCH MODE — NO BEARING" label, set `couchMode = true`
   - SAW FLASH in couch mode: skip compass permission, skip bearing lock, `bear` saves as `null`
   - Calc box bearing row shows "COUCH MODE"
   - Map: `bear === null` strikes render as `L.circle` ring at `distMi * 1609.34` meters radius — dashed, age color, `fillOpacity: 0.10`, no bearing line, center dot still plotted

3. **UI initialization bugs**
   - HEARD THUNDER button — remove `armed` class from HTML; starts inert
   - Calc box bearing — add `id="c-bearing"` directly in HTML, remove DOM-query patch
   - Trend bars — initialize empty on load; call `renderTrend([])` in DOMContentLoaded
   - Map strip — set all stat values to `—` in HTML
   - Timestamp interval — remove screen check; run 30s refresh regardless of active tab
   - Strike label clipping — use bearing quadrant to set `iconAnchor` so label stays on-screen

4. **CLEAR / delete UX**
   - CLEAR ALL on TRENDS tab only — `confirm()` dialog, wipe localStorage, re-render log + map
   - Per-entry ✕ on each log row — no confirm, delete by `ts`, re-render
   - Remove CLEAR from RECORD screen
   - No CLEAR on MAP tab

5. **Map legend**
   - Check visibility on 390px wide phone screen
   - If it occludes strike zones, make it collapsible: tap to toggle, default collapsed

**Deliverable:** Full phone-ready three-tab app, couch mode working, all stale values fixed.

---

### Session 5 — Field Test + Deploy
**Goal:** Everything live-updating and deployed to real HTTPS URL.

**Tasks:**
- Age colors update without user interaction (extend 30s setInterval to call `renderMapStrikes()` and `updateObsBadge()`)
- Field test on iOS: geolocation prompt, compass permission flow, bezel rotation, tap-to-lock, couch mode
- Field test on Android: compass fires without dialog, verify bearing accuracy
- Deploy to GitHub Pages (rename to `index.html`, push, enable Pages)
- Confirm all features work over HTTPS on real device
- Tune `BEARING_ERROR_DEG` and `TIMING_ERROR_S` constants based on field observations

**Deliverable:** Shareable URL, works on any phone in a real storm.

---

## Tunable Constants (top of script block)

```javascript
const SOUND_MPS   = 343;     // m/s speed of sound
const MI_PER_M    = 0.000621371;
const MAX_DIST_MI = 6.2;     // beyond this → DISTANT, not plotted
// Inside makePizzaCrust():
//   DELTA_D_MI = 0.5 * SOUND_MPS * MI_PER_M  (~0.107mi, ±500ms timing error)
//   DELTA_B    = 5                             (±5° bearing error)
// Age thresholds: hardcoded in ageClass() — <5min recent, <15min mid, else old
```

---

## After v1 — Possible Future Work

- **Blitzortung overlay**: wire `fetchExternalStrikes()`, show confirmed strikes as distinct markers for comparison with your observations. Good calibration tool.
- **Desktop layout**: two-column view (controls left, log/trend right). Couch mode on by default since no device compass. Manual lat/lon entry since IP geolocation is imprecise on desktop.
- **Multiple observers**: if two people at known locations both record the same strike, triangulation becomes possible. Out of scope for now.
- **Tile variants**: CartoDB Positron (light) or standard OSM if dark tiles feel wrong in daylight use.
- **Bearing-error tuning**: ±5° is a guess. After field testing, compare logged bearings against known strike locations (if Blitzortung data is available) and adjust the constant.

---

## Files

| File | Description |
|---|---|
| `PLAN.md` | This document |
| `NOTES.md` | Implementation decisions and gotchas discovered during development |
| `STRIKEMAP_HANDOFF.md` | Full geometry, schema, and session-by-session task reference |
| `strikemap-mockup.html` | Full UI mockup — visual reference only, do not edit |
| `strikemap.html` | **Live file — source of truth** |
