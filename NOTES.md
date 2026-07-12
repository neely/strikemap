# Strikemap — Implementation Notes
*Running log of decisions, gotchas, and things to remember across sessions*

**Timeline:** idea originated c. 2007; design sketched ~April 2026; Sessions 1–3 built early May 2026 (confirmed via git history); Session 4 onward in July 2026. See PLAN.md for full timeline.

---

## Session 1 — Map

### CartoDB Dark Matter chosen over standard OSM
Standard OSM tiles are light/colorful and fight the dark UI badly. CartoDB Dark Matter is free, no API key, loads from CDN (`https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png`). Confirmed working. If you ever want to swap:
- Light minimal: `https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png`
- Standard OSM: `https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`

### `map.invalidateSize()` is required when switching tabs
Leaflet measures its container on init. If the map `<div>` is hidden (display:none) when the page loads, Leaflet thinks it's 0×0. Calling `map.invalidateSize()` in `showScreen('map')` forces it to re-measure and re-render. Without this, you get a gray or broken map on first tab switch. Already wired — don't remove it.

### `tap: false` in Leaflet options
Leaflet's built-in tap handler interferes with mobile touch on locked maps. Disabling it explicitly prevents phantom clicks and scroll jank. Already set.

### Fallback coordinates
James Island / Charleston, SC: `32.7357, -79.9956`. This is where the original mockup map screenshot was centered. Change to wherever makes sense after deploy if desired.

### `window.sm_map` convention
The Leaflet instance is assigned to `window.sm_map` so it's accessible across script scopes without module imports. Follow the same pattern for any other global state that needs cross-function access.

### Geolocation fires on page load
Geolocation prompts immediately on open — no user gesture required (on most browsers). This is intentional. You want the map centered before you need it. Compass is different — it waits for user gesture. Don't merge these into a single permission flow.

### Attribution styling
CartoDB requires OSM + CARTO attribution. Styled small and dim (`font-size:7px`, `color:var(--dim)`, semi-transparent background) so it doesn't intrude. Don't remove it — it's a licensing requirement.

---

## Session 2 — Compass + Flash/Thunder

### Compass permission must come from a user gesture
`DeviceOrientationEvent.requestPermission()` on iOS 13+ will silently fail if called outside a user gesture handler. Wired to the first FLASH tap — natural moment, satisfies the requirement. Do not move it to page load or DOMContentLoaded.

### `compassPermissionAsked` flag prevents double-prompting
Set to `true` on the first FLASH tap. Subsequent FLASH taps skip the permission call. Without this, every FLASH tap would re-trigger the iOS dialog.

### Android compass requires no permission dialog
On Android, `deviceorientation` fires immediately after `addEventListener` — no dialog, no async. Same `startCompass()` function handles both paths.

### Bearing captured at top of `onThunder()` — ordering is critical
`lockedHeading` is cleared as part of resetting state in `onThunder()`. The bearing must be read into a local variable **before** any state mutation. If you refactor `onThunder()`, keep this at the very top.

### `bearingArmed` flag controls when compass tap-to-lock is active
The compass ring is always visible and its bezel always rotates, but tapping it does nothing unless `bearingArmed === true`. Set by `onFlash()`, cleared by `onThunder()`. Prevents accidental bearing locks outside a recording session.

### Circular mean for heading smoothing — not arithmetic mean
Arithmetic mean breaks near the 0°/360° boundary. Example: averaging 355° and 5° arithmetically gives 180°, which is completely wrong. Circular mean via `sin/cos` decomposition gives the correct 0°. Already in `pushHeading()` — don't simplify it.

### `bear: null` must be handled gracefully everywhere
Three places where null bearing appears:
1. **Log entry**: displays "--- NO HDG" in the meta line, cardinal shows "---"
2. **Map**: skips pizza crust + bearing line; renders bullseye ring instead (Session 4)
3. **Map strip DIRECTION**: shows "—" when closest strike has null bearing — correct, not a bug

### Trend uses newest-first array order
Strikes are stored newest-first (`arr.unshift()`). Trend comparison is `strikes[0].distMi - strikes[1].distMi` — negative means getting closer. Don't sort the array on read; insertion order is the sort.

### HEARD THUNDER must be inert until FLASH is tapped
The THUNDER button should do nothing if `flashTime` is null. The `armed` class is only added by the FLASH handler, not present in HTML at load. (This was a bug that lingered into Session 3 — make sure the HTML has no `armed` class on the button.)

---

## Session 3 — Pizza Crusts + Map Fixes

### Ellipses were geometrically wrong — replaced with annular sector
The original ellipse approach had two fundamental errors:
1. **Wrong origin.** Ellipses were centered on the strike point. Both error sources (bearing wobble, timing tap) fan outward from the *observer*, not from the strike. The correct shape must be referenced from the user's position.
2. **Wrong coordinate system.** Bearing and distance errors are polar. Projecting them into cartesian space and fitting an ellipse loses the actual shape — it implies equal uncertainty in all directions from the strike, which isn't true.

The correct shape is a **pizza crust** (annular sector):
- Bearing error defines a wedge (two rays at `bearing ± 5°` from observer). Wedge fans out with distance.
- Timing error defines an annular band (inner/outer arcs at `dist ± 0.107mi`). Constant depth.
- Intersection is the uncertainty region. Built as an `L.polygon` using ~26 projected points.

### Timing error revised to ±500ms
Original plan used ±250ms. Revised to ±500ms as a more realistic assumption for human tap reaction, especially when startled by thunder. This gives `±0.107mi` depth on the pizza crust (was `±0.053mi`). Depth is still small relative to the wedge width at any meaningful distance.

### Pizza crust collapses to wedge at very close range
If `dist < 0.107mi` (thunder under ~0.5 seconds, strike under ~560 feet), the inner arc clamps to zero via `Math.max(0, innerDist)` and the shape becomes a pure triangle/wedge. This edge case almost never fires in practice.

### `TIMING_ERR_S` and `BEARING_ERR_DEG` constants removed
These were top-level constants in Session 2. After the geometry rewrite, the values are embedded directly in `makePizzaCrust()` with inline comments for clarity. If you need to tune them, they're in that function.

### Age filter button bug was in text-content matching
The original active-state toggle used `btn.textContent.trim()` to identify which buttons to deactivate. This accidentally matched the CENTER button if its text ever changed. Fixed by adding `data-age` attributes to the three filter buttons and scoping the toggle to `[data-age]` selector only.

### APPROACHING alert had hardcoded `display:flex` in HTML
The alert div started visible regardless of strike data. Fixed by setting `style="display:none;"` as the baseline HTML state. The `updateMapStrip()` JS function already had correct show/hide logic — it just needed a clean starting state so CLEAR ALL works correctly.

### DISTANT notice — element + CSS were missing
The handoff doc described a DISTANT banner but it wasn't in the HTML. Added `#distant-notice` div inside `.map-wrap` and CSS positioning it top-center of the map as an absolute overlay. Styled in blue-grey (`var(--old)`) to visually match old/far strikes. `updateMapStrip()` shows/hides it and sets the count text.

### `fetchExternalStrikes()` stub added
One-liner async function returning `[]`, placed after constants with a comment marking it as the Blitzortung hook. It's async so future implementations can `await fetch(...)` without touching call sites.

---

## Session 4 — Three Tabs + Couch Mode (UPCOMING)

### Three-tab restructure is the biggest change in Session 4
The tab split touches HTML structure, CSS layout rules, and nav wiring simultaneously. Do it first, get it rendering correctly, then add couch mode and bug fixes on top.

### Couch mode and `bear: null` are already the same code path
The localStorage schema already supports `bear: null`. Couch mode is just an intentional UI path to that state — the map, log, and trend code already handle null bearing. The only new map rendering needed is the bullseye ring for null-bearing strikes (currently those strikes are silently skipped on the map).

### Bullseye ring for null-bearing strikes
Use `L.circle` centered on the user at `distMi * 1609.34` meters radius. Dashed stroke, age color, reduced fill opacity (`0.10`). Center dot still plotted. No label (direction unknown, so labeling the ring would be misleading). This communicates "definitely this far away, direction unknown" — visually a bullseye target ring.

### Map legend may need to be collapsible
Current position is `bottom:60px right:12px`. On a 390px wide phone (iPhone 14 standard), this can overlap pizza crusts in the NE quadrant at close distances. If it's a problem in field testing, add a tap-to-toggle with default collapsed state.

---

## General Gotchas

### HTTPS required for compass + geolocation on mobile
Both `DeviceOrientationEvent` and `navigator.geolocation` are blocked on `file://` and plain `http://` on iOS Safari and Chrome Android. GitHub Pages serves over HTTPS automatically. For local testing on phone, use `npx serve` and access via local IP — geolocation may still block; compass usually works.

### iOS vs Android compass APIs
- **iOS 13+**: `DeviceOrientationEvent.requestPermission()` required from user gesture. Use `event.webkitCompassHeading` (0–360, true north, already corrected for device orientation).
- **Android / desktop Chrome**: No permission dialog. Use `event.alpha` with `(360 - alpha) % 360` conversion when `event.absolute === true`.
- **Desktop**: `deviceorientation` typically doesn't fire at all. `bear` stays `null`. Handle gracefully.

### `webkitCompassHeading` vs `alpha` conversion
```javascript
function getHeading(event) {
  if (typeof event.webkitCompassHeading === 'number') {
    return event.webkitCompassHeading; // iOS: already true north, 0–360
  }
  if (event.absolute && typeof event.alpha === 'number') {
    return (360 - event.alpha) % 360; // Android absolute
  }
  if (typeof event.alpha === 'number') {
    return (360 - event.alpha) % 360; // Android non-absolute: best effort
  }
  return null;
}
```

### Circular mean for compass smoothing
```javascript
const buf = [];
function pushHeading(h) {
  buf.push(h);
  if (buf.length > 8) buf.shift();
  let s = 0, c = 0;
  buf.forEach(a => { s += Math.sin(a * Math.PI/180); c += Math.cos(a * Math.PI/180); });
  return (Math.atan2(s, c) * 180/Math.PI + 360) % 360;
}
```
Use circular mean, not arithmetic mean — arithmetic mean breaks near 0°/360°.

### localStorage — newest first
Strikes are stored newest-first (`array.unshift(newStrike)`). The log renders them in array order (newest at top). The map uses array order too — index 0 is always the most recent strike and gets the distance label. Don't sort on read; maintain order on write.

### Strike expiry is soft
Strikes are never hard-deleted from localStorage on load. Expiry logic is applied at render time. If you want to clean up old sessions, CLEAR ALL is the mechanism.
