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

Option 2 is cleaner UX — you only need the compass when you're about to record a strike, so requesting permission at that moment is natural. If permission is denied or unavailable, bear saves as `null`, the strike still logs with distance only, and the map skips the ellipse for that entry (or shows a distance-only ring as a future enhancement).

### Single HTML file
No build step, no npm, no bundler. Leaflet from CDN, fonts from Google Fonts CDN, everything else vanilla JS. Drop the file in a GitHub repo, enable Pages, done.

---

## Development Sessions

### Session 1 — Real Map ✅ COMPLETE
**Goal:** Leaflet + OSM rendering correctly centered on user location.

**Completed:**
- Leaflet 1.9.4 CSS + JS loaded from CDN
- CartoDB Dark Matter tiles (decided during this session — no key, fits dark UI)
- All interaction disabled: `dragging`, `touchZoom`, `scrollWheelZoom`, `doubleClickZoom`, `boxZoom`, `keyboard`, `tap`
- `navigator.geolocation.getCurrentPosition()` fires on load; centers map on result
- Fallback center: James Island / Charleston, SC (`32.7357, -79.9956`)
- "ACQUIRING LOCATION…" overlay while waiting; hides on success; "LOCATION UNAVAILABLE" notice on failure (auto-hides after 3s)
- Cyan pulsing "YOU ARE HERE" dot via `L.divIcon` with CSS `pulse-ring` animation
- `⊕ CENTER` button calls `map.setView([userLat, userLon], 12)`
- `showScreen('map')` calls `map.invalidateSize()` — required because Leaflet's container is hidden on initial load
- `window.sm_map` exposes the map instance for cross-function access
- Static `<img>` screenshot and SVG overlay fully removed

**State of file:** `strikemap.html` is the live file. Mock flash/thunder timing buttons still present on RECORD tab — they will be replaced in Session 2.

---

### Session 2 — Compass + Flash/Thunder → localStorage ← START HERE
**Goal:** Core recording loop works end to end.

**Tasks:**
- Wire `DeviceOrientationEvent` / `webkitCompassHeading` for iOS
- Trigger permission request on first FLASH tap (not on page load)
- Smooth heading with short rolling average to reduce jitter
- Lock heading on FLASH tap, display locked value
- FLASH starts timer, THUNDER stops it, calculates delay + distance
- Write strike object to localStorage on THUNDER tap
- Render strike in log immediately
- Handle no-compass case gracefully: `bear: null`, `card: null`, log entry shows "NO BEARING"

**localStorage schema:**
```javascript
{
  ts: Date.now(),   // timestamp at thunder tap
  delay: 2.3,       // seconds (1 decimal)
  distMi: 1.42,     // miles (2 decimal)
  bear: 43,         // integer degrees, null if unavailable
  card: "NNE"       // cardinal string, null if unavailable
}
```

**Handoff notes for Session 2:**
- `window.sm_map` = Leaflet map instance
- `userLat` / `userLon` = live coordinates (set by geo success, fallback to James Island)
- `placeUserMarker(lat, lon)` = moves the cyan dot
- Mock flash/thunder functions (`mockFlash`, `mockThunder`) are in the current file — replace them entirely
- `DeviceOrientationEvent.requestPermission()` must come from a user gesture; the FLASH tap satisfies this
- Android/desktop: `deviceorientation` fires without a permission dialog — handle both paths
- Compass requires HTTPS; `file://` won't work on mobile

Deliverable: Full tap → log flow working on real phone. Data persists through page reload.

---

### Session 3 — Ellipses on Real Map
**Goal:** Computed strikes appear as correctly-shaped ellipses on the Leaflet map.

**Tasks:**
- Implement destination point formula (spherical) to compute strike lat/lon from user position + bearing + distance
- Build ellipse polygon from parametric points, rotated to bearing:
  - Radial half-axis: `TIMING_ERR_S * SOUND_MPS` meters (short axis)
  - Tangential half-axis: `sin(BEARING_ERR_DEG * π/180) * distMi * 1609` meters (long axis)
- Render as `L.polygon` with correct color per age class
- Add faint dashed `L.polyline` from user to strike center
- Add small `L.circleMarker` at strike center
- Label most recent strike with distance
- Handle `bear === null` strikes: skip ellipse, optionally show a thin distance ring (full circle at distMi radius)
- Handle DISTANT strikes (>6.3mi): log entry gets DISTANT tag, no map plot, show notice banner on map tab
- Wire age filter buttons (ALL / 10 MIN / 30 MIN) to re-render visible strikes
- Confirm ellipse shape is correct: wide perpendicular to bearing, shallow along bearing

Deliverable: Strikes appear on real map as correctly oriented ellipses. Progression visible as storm moves.

---

### Session 4 — Polish + Deploy
**Goal:** Everything live-updating and deployed.

**Tasks:**
- Age colors update without user interaction (setInterval refresh every 30s)
- Trend bar chart renders from active strikes, updates on new entry
- Trend indicator: approaching / receding / steady based on last two strikes
- Map bottom strip: OBS count, closest distance, direction, trend arrow, APPROACHING alert
- APPROACHING alert animates when storm is getting closer
- CLEAR button with confirmation dialog
- Add `fetchExternalStrikes()` stub returning `[]` with a console note
- Test on iOS: geolocation prompt, compass permission flow, rendering
- Test on Android: compass typically works without permission dialog
- Deploy to GitHub Pages (rename to `index.html`, push, enable Pages)
- Confirm all features work over HTTPS on real device

Deliverable: Shareable URL, works on any phone.

---

## Tunable Constants (top of script block)

```javascript
const SOUND_MPS        = 343;     // m/s speed of sound
const MI_PER_M         = 0.000621371;
const MAP_ZOOM         = 12;      // ~10-12mi view
const FAR_MI           = 6.2;    // beyond this → DISTANT, not plotted
const BEARING_ERR_DEG  = 5;      // ±° compass wobble — tune after field testing
const TIMING_ERR_S     = 0.25;   // ±s human reaction time on both taps
const AGE_RECENT_MIN   = 5;      // yellow threshold
const AGE_MID_MIN      = 15;     // orange threshold (older = blue-grey)
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
