# Strikemap — Development Plan
*Created from design conversations, July 2026*

---

## What We're Building

A single-file static HTML app for real-time lightning tracking. Point your phone at a flash, tap a button, tap again at thunder. The app calculates distance and bearing, logs each strike with age-based color coding, and plots uncertainty ellipses on a live map so you can watch a storm approach or recede.

Hosted on GitHub Pages. No server, no API keys, no accounts. Works for multiple users sharing the same URL — each person tracks independently on their own device.

---

## Decisions Made (and Why)

### No backend / no Google Sheets
Early idea was to log to a Google Sheet for persistence. Dropped it — localStorage is sufficient for the duration of a storm session. You're not trying to archive this data across days, just track a 30–45 minute event. CLEAR button wipes it and you start fresh next storm.

### No external lightning data (for now)
Blitzortung.org was identified as the best free community option for real strike confirmation data. Xweather (formerly AerisWeather) has a usable free dev tier for queried JSON. Decision: not worth the complexity for v1. A stub function `fetchExternalStrikes()` returning `[]` will be in the code so the hook exists. Add it later if desired.

### No Google Maps, no Mapbox
Both require API keys. Keys in client-side code on a public GitHub repo is bad practice, and adding referrer restrictions is friction for a personal tool shared with a few people. OpenStreetMap + Leaflet is genuinely free, no key, CDN-loaded, and works everywhere in the world. Easy call.

### Static map — no pinch to zoom
Pinch-to-zoom gets out of whack quickly on a phone, then you need a recenter button, and the UX complexity grows. The whole point of the map is spatial awareness of storm progression, not navigation. A locked static view at zoom 12 (~10–12 mile wide area) is cleaner and more useful for this purpose. One RE-CENTER button in the controls strip is the only escape hatch.

### Zoom level 12 as default
This shows roughly a 10–12 mile wide area, which means ~5–6 mile radius from center. This aligns almost exactly with the 30-second thunder delay rule (30s × 343 m/s ≈ 6.3 miles). Strikes beyond that get logged with a DISTANT flag but aren't plotted — the map view doesn't need to stretch for them.

### Tile provider: CartoDB Dark Matter
Standard OSM tiles are light and colorful — they fight the dark UI hard. CartoDB Dark Matter is free, no API key required, CDN-loaded, and matches the `--bg:#080c10` aesthetic naturally. Decided during Session 1; can swap easily by changing the tile URL string.

### Ellipses centered on the strike point, not wedges from the observer
Early mockup had comma/wedge shapes emanating from the user's position. This was wrong. The correct geometry is:
- Compute the strike's actual position (bearing + distance from user)
- Place an ellipse **centered on that point**
- The ellipse represents measurement uncertainty *around* the strike, not a probability zone between user and strike

The two error sources are:
- **Radial (depth):** ±250ms timing error × 343 m/s = ±86m. This is tiny and becomes the *short* axis.
- **Tangential (width):** ±5° compass wobble, grows with distance. At 1.4mi ≈ ±215m, at 3.1mi ≈ ±475m. This is the *long* axis.

Ellipse is rotated to bearing so the long axis is always perpendicular to the line of sight from the user.

### Bearing lines kept as faint dashes
Even though the ellipse is centered on the strike, a faint dashed line from the user to each strike center helps read the bearing visually. Kept at low opacity so it doesn't clutter the map.

### Age color scheme
Three tiers felt right for storm timescales:
- < 5 min → Yellow `#f5c842` (active concern)
- 5–15 min → Orange `#f08030` (context)
- > 15 min → Blue-grey `#4a7aaa` (historical, fading)

Applied consistently to log entries, ellipses, bearing lines, and trend bars.

### iOS compass permission UX
`DeviceOrientationEvent.requestPermission()` on iOS 13+ must be triggered by a user gesture — the browser will silently ignore it if called on page load. Options considered:
1. Dedicated "Enable Compass" button on first open
2. Trigger permission on first FLASH tap

Option 2 is cleaner UX — you only need the compass when you're about to record a strike, so requesting permission at that moment is natural. If permission is denied or unavailable, bear saves as `null`, the strike still logs with distance only, and the map skips the ellipse for that entry.

### Compass UX: tap-to-lock instead of lock-on-flash
Early design locked the heading at the moment the FLASH button was tapped. Problem: the user sees the flash and taps reflexively — they haven't yet pointed the phone. The correct flow is:

1. Tap FLASH → starts timer, compass goes live (bezel rotating freely)
2. Swing phone toward where lightning was, hold still
3. Tap compass ring to lock — display snaps to green "✓ NNE"
4. Set phone down, wait for thunder
5. Tap HEARD THUNDER

Tapping compass again while locked will unlock it, allowing re-aim. If user never taps compass before HEARD THUNDER, strike logs with `bear: null` (distance-only).

### Compass display: rotating bezel, no needle
A spinning needle pointing at a fixed angle on screen is confusing — it implies a direction relative to the phone that doesn't match real-world bearing. Replaced with a rotating bezel (N/S/E/W ring rotates opposite to heading so N always faces north, like a real compass) plus a fixed yellow lubber line at 12 o'clock marking where the phone is pointing. Center shows only degrees + cardinal text (e.g. `043° ENE`). N label is red per standard compass convention.

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
- "ACQUIRING LOCATION…" overlay while waiting; hides on success; "LOCATION UNAVAILABLE" notice on failure (auto-hides after 3s)
- Cyan pulsing "YOU ARE HERE" dot via `L.divIcon` with CSS `pulse-ring` animation
- `⊕ CENTER` button calls `map.setView([userLat, userLon], 12)`
- `showScreen('map')` calls `map.invalidateSize()` — required because Leaflet's container is hidden on initial load
- `window.sm_map` exposes the map instance for cross-function access
- Static `<img>` screenshot and SVG overlay fully removed

---

### Session 2 — Compass + Flash/Thunder → localStorage ✅ COMPLETE
**Goal:** Core recording loop works end to end.

**Completed:**
- `DeviceOrientationEvent` / `webkitCompassHeading` wired for iOS and Android
- Compass permission requested on first FLASH tap (user gesture satisfies iOS requirement)
- 8-sample circular mean heading buffer for jitter smoothing
- Rotating bezel compass (no needle) — bezel rotates opposite to heading, N always faces north
- Fixed yellow lubber line at 12 o'clock marks phone's forward direction
- Compass ring is a `<button>` — tap to lock heading (green "✓ NNE"), tap again to unlock
- `bearingArmed` flag: compass only accepts lock taps after FLASH is pressed
- FLASH: starts timer, sets `bearingArmed = true`, compass goes live
- THUNDER: captures locked bearing (or null), computes delay + distance, writes to localStorage, renders log
- Bearing captured at top of `onThunder()` before any state is cleared (important ordering)
- `bear: null` handled gracefully throughout — logs as "--- NO HDG", skipped on map
- Trend bar chart from up to 8 most recent strikes, colored by age
- Approaching / receding / steady indicator from last two strikes
- CLEAR button with `confirm()` dialog, wipes localStorage
- Timestamps refresh every 30s via `setInterval`
- `renderMapStrikes()` called on MAP tab switch

**localStorage schema (unchanged from plan):**
```javascript
{
  ts: Date.now(),   // timestamp at thunder tap
  delay: 2.3,       // seconds (1 decimal)
  distMi: 1.42,     // miles (2 decimal)
  bear: 43,         // integer degrees, null if unavailable
  card: "NNE"       // cardinal string, null if unavailable
}
```

**Current constants in file:**
```javascript
const SOUND_MPS       = 343;
const MI_PER_M        = 0.000621371;
const MAX_DIST_MI     = 6.2;
const TIMING_ERR_S    = 0.25;    // ±250ms
const BEARING_ERR_DEG = 5;       // ±5°
const LS_KEY          = 'sm_strikes';
```

---

### Session 3 — Ellipses on Real Map ← START HERE
**Goal:** Computed strikes appear as correctly-shaped ellipses on the Leaflet map.

**Status:** Core geometry is already implemented in `strikemap.html`:
- `strikeLatLon(lat, lon, bearingDeg, distMi)` — spherical destination point formula ✓
- `makeEllipsePolygon(lat, lon, bearingDeg, distMi, color)` — parametric ellipse, tangential/radial axes, rotated to bearing ✓
- `renderMapStrikes()` — reads localStorage, filters by age, draws lines + ellipses + dots ✓

**Remaining tasks:**
- Fix age filter button active-state toggle (currently matches on text content, fragile)
- Wire APPROACHING alert visibility — doesn't reset on CLEAR ALL
- Add DISTANT notice banner on map tab for strikes > `MAX_DIST_MI`
- Add `fetchExternalStrikes()` stub returning `[]`
- Verify ellipse shape in field: wide perpendicular to bearing, shallow along bearing
- Handle `bear === null` strikes on map — currently skipped entirely; consider thin distance ring (full circle at distMi)
- Age colors on map currently only update on tab switch or new strike — extend 30s setInterval to also call `renderMapStrikes()` and update OBS badge

**Deliverable:** Strikes appear on real map as correctly oriented ellipses. Progression visible as storm moves.

---

### Session 4 — Polish + Deploy
**Goal:** Everything live-updating and deployed.

**Tasks:**
- Age colors update without user interaction (extend existing 30s setInterval)
- Map bottom strip all fields live
- APPROACHING alert animates and resets correctly
- Test on iOS: geolocation prompt, compass permission flow, bezel rotation, tap-to-lock
- Test on Android: compass works without permission dialog
- Deploy to GitHub Pages (rename to `index.html`, push, enable Pages)
- Confirm all features work over HTTPS on real device

**Deliverable:** Shareable URL, works on any phone.

---

## Tunable Constants (top of script block)

```javascript
const SOUND_MPS       = 343;          // m/s speed of sound
const MI_PER_M        = 0.000621371;
const MAX_DIST_MI     = 6.2;          // beyond this → DISTANT, not plotted
const BEARING_ERR_DEG = 5;            // ±° compass wobble — tune after field testing
const TIMING_ERR_S    = 0.25;         // ±s human reaction time on both taps
// Age thresholds are hardcoded in ageClass() — < 5min = recent, < 15min = mid, else old
```

---

## After v1 — Possible Future Work

- **Blitzortung overlay**: wire `fetchExternalStrikes()`, show confirmed strikes as distinct markers for comparison with your observations. Good calibration tool.
- **Bearing-null ring**: for strikes recorded without compass, draw a thin full-circle arc at the computed distance so they appear on the map as "somewhere on this ring."
- **Desktop layout**: two-column view (controls left, log/trend right). Couch mode on by default since no device compass on laptop. Manual lat/lon entry since IP geolocation is imprecise on desktop.
- **Multiple observers**: if two people at known locations both record the same strike, triangulation becomes possible. Out of scope for now.
- **Tile variants**: CartoDB Positron (light) or standard OSM if dark tiles ever feel wrong in daylight use.

---

## Files

| File | Description |
|---|---|
| `PLAN.md` | This document |
| `NOTES.md` | Implementation decisions and gotchas discovered during development |
| `STRIKEMAP_HANDOFF.md` | Geometry, schema, tech stack reference |
| `strikemap-mockup.html` | Full UI mockup — visual reference, do not edit |
| `strikemap.html` | **Live file — source of truth** |
