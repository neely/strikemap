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
Geolocation prompts immediately on open — no user gesture required (on most browsers). This is intentional. You want the map centered before you need it. Compass is different — it waits for first compass tap. Don't merge these into a single permission flow.

### Attribution styling
CartoDB requires OSM + CARTO attribution. Styled small and dim (`font-size:7px`, `color:var(--dim)`, semi-transparent background) so it doesn't intrude. Don't remove it — it's a licensing requirement.

---

## Session 2 — Compass + Flash/Thunder

### Compass permission must come from a user gesture
`DeviceOrientationEvent.requestPermission()` on iOS 13+ will silently fail if called outside a user gesture handler. Originally wired to the first FLASH tap. Moved in Session 4b to the first compass ring tap — see Session 4b notes below for full rationale.

### `compassPermissionAsked` flag prevents double-prompting
Set to `true` on first call to `requestCompassPermission()`. Subsequent taps skip the permission call. Without this, every compass tap would re-trigger the iOS dialog.

### Android compass requires no permission dialog
On Android, `deviceorientation` fires immediately after `addEventListener` — no dialog, no async. Same `startCompass()` function handles both paths.

### Bearing captured at top of `onThunder()` — ordering is critical
`lockedHeading` is cleared as part of resetting state in `onThunder()`. The bearing must be read into a local variable **before** any state mutation. If you refactor `onThunder()`, keep this at the very top.

### `bearingArmed` flag controls when compass tap-to-lock is active
The compass ring is always visible and its bezel always rotates (once enabled), but tapping it does nothing unless `bearingArmed === true` or `lockedHeading !== null`. Set by `onFlash()`, cleared by `onThunder()`. Prevents accidental bearing locks outside a recording session.

### Circular mean for heading smoothing — not arithmetic mean
Arithmetic mean breaks near the 0°/360° boundary. Example: averaging 355° and 5° arithmetically gives 180°, which is completely wrong. Circular mean via `sin/cos` decomposition gives the correct 0°. Already in `pushHeading()` — don't simplify it.

### `bear: null` must be handled gracefully everywhere
Three places where null bearing appears:
1. **Log entry**: displays "--- NO HDG" in the meta line, cardinal shows "---"
2. **Map**: skips pizza crust + bearing line; renders annular donut instead
3. **Map strip DIRECTION**: shows "—" when closest strike has null bearing — correct, not a bug

### Trend uses newest-first array order
Strikes are stored newest-first (`arr.unshift()`). Trend comparison is `strikes[0].distMi - strikes[1].distMi` — negative means getting closer. Don't sort the array on read; insertion order is the sort.

### HEARD THUNDER must be inert until FLASH is tapped
The THUNDER button should do nothing if `flashTime` is null. The `armed` class is only added by the FLASH handler, not present in HTML at load. Verified in Session 4 — HTML has no `armed` class on the button.

---

## Session 3 — Pizza Crusts + Map Fixes

### Ellipses were geometrically wrong — replaced with annular sector
The original ellipse approach had two fundamental errors:
1. **Wrong origin.** Ellipses were centered on the strike point. Both error sources (bearing wobble, timing tap) fan outward from the *observer*, not from the strike.
2. **Wrong coordinate system.** Bearing and distance errors are polar. Fitting a cartesian ellipse loses the actual shape.

The correct shape is a **pizza crust** (annular sector) — see PLAN.md for full geometry.

### Timing error revised to ±500ms
Original plan used ±250ms. Revised to ±500ms as a more realistic assumption for human tap reaction, especially when startled by thunder. Gives `±0.107mi` depth on the pizza crust. Depth is still small relative to wedge width at any meaningful distance.

### Pizza crust collapses to wedge at very close range
If `dist < 0.107mi` (thunder under ~0.5 seconds), the inner arc clamps to zero via `Math.max(0, innerDist)` and the shape becomes a pure triangle/wedge. Edge case that almost never fires in practice.

### Age filter button bug was in text-content matching
The original active-state toggle used `btn.textContent.trim()` to identify which buttons to deactivate. This accidentally matched the CENTER button. Fixed by adding `data-age` attributes to the three filter buttons and scoping the toggle to `[data-age]` selector only.

### APPROACHING alert had hardcoded `display:flex` in HTML
Alert div started visible regardless of data. Fixed by setting `style="display:none;"` as baseline. JS in `updateMapStrip()` handles all show/hide.

### DISTANT notice — element + CSS were missing
Described in handoff but not in HTML. Added `#distant-notice` div inside `.map-wrap` and CSS positioning it top-center as absolute overlay. Styled in blue-grey to match old/far strike color.

---

## Session 4 — Three Tabs + Couch Mode

### Three-tab restructure is the biggest layout change
Split the original two-panel RECORD layout into RECORD (controls only) + TRENDS (log + chart). The two-column mockup layout is preserved for a future desktop version — don't try to merge them in the same responsive CSS.

### `showScreen()` wiring
- `showScreen('map')` → calls `map.invalidateSize()` then `renderMapStrikes()`
- `showScreen('trends')` → calls `renderLog()` (which calls `renderTrend()` internally)
- `showScreen('record')` → no data calls needed

### Couch mode is a UI path to `bear === null`
The localStorage schema always supported `bear: null`. Couch mode is just an intentional route to that state. The log, map, and trend code treat null bearing the same regardless of whether it came from couch mode or a missed compass lock.

### Couch mode does not persist across reload
`couchMode` is a JS runtime variable — resets to `false` on reload. Intentional for now; if needed, persist to localStorage in a future session.

### Map strip DIRECTION shows `—` for null-bearing closest strike
If the closest active strike has `card: null` (couch mode or no compass lock), DIRECTION shows `—`. This is correct behavior — the app doesn't know direction. May surprise users who expect to always see something there.

### Single `DOMContentLoaded` block — don't split it again
At one point during Session 4 there were two `DOMContentLoaded` blocks (init in one, age filter listeners in the other). Merged into one. Keep it that way — splitting them causes initialization order confusion.

### `setInterval(renderLog, 30000)` is unconditional
Runs regardless of which tab is active. `renderLog()` is a no-op if the TRENDS elements aren't visible (it guards with `if(!logEl) return`). This is intentional — timestamps need to age even when the user is on the RECORD or MAP tab. Session 5 task: extend this interval to also call `renderMapStrikes()` and `updateObsBadge()`.

---

## Session 4b — Compass UX + Expiry + Donut

### Compass permission moved from FLASH to compass ring tap
**Why the change:** Geolocation fires on open (location prompt appears immediately). Originally, compass permission fired on the first FLASH tap. This meant on iOS you got two system dialogs — location on open, compass on first FLASH tap — which felt like a double interrogation.

**New flow:** Compass ring shows `TAP TO START` with a cyan glow on open. First tap on the ring calls `requestCompassPermission()`. This separates the prompts in time and intent: location on open (makes sense — the map needs it), compass when you tap the thing that uses it (makes sense). On Android, the first tap silently starts the bezel, no dialog at all.

**Implementation detail:** `onCompassTap()` now has a pre-permission branch: if `!compassPermissionAsked`, call `requestCompassPermission()` and return. Subsequent taps fall through to the normal lock/unlock logic.

### `startCompass()` now calls `updateCompassDisplay()` after enabling
Without this, Android users would tap the ring, compass would start silently, but the display would still show `TAP TO START` until the next `deviceorientation` event. Calling `updateCompassDisplay()` immediately in `startCompass()` clears the pre-permission state right away.

### FLASH no longer calls `requestCompassPermission()`
Removed the `if(!couchMode && !compassPermissionAsked) requestCompassPermission()` line from `onFlash()`. If someone taps FLASH without ever tapping the compass ring first, the compass just won't have a heading — strike saves with `bear: null`. This is acceptable; the UI flow makes clear you should tap the compass first.

### Soft expiry — never hard-delete on load
Hard-deleting expired strikes on page load was considered and rejected. Reason: if you accidentally reload the page mid-storm, you lose the last hour of data. Soft expiry keeps everything in localStorage; expiry is applied at render time. CLEAR ALL is the only mechanism for wiping data.

### Expiry exclusion is layered: expired first, then age filter
In `renderMapStrikes()`: first filter by `!isExpired(ts)` to get active strikes, then apply the age filter (10 MIN / 30 MIN / ALL) to that active subset. The age filter buttons never surface expired strikes even when ALL is selected.

### Trend and OBS badge use active-only strikes
`renderTrend()` receives the active-only subset from `renderLog()` — it doesn't call `loadStrikes()` itself. `updateObsBadge()` filters independently. `updateMapStrip()` also filters independently for its trend calculation. All three are consistent.

### Annular donut replaces `L.circle` for null-bearing strikes
`L.circle` draws a filled disc — wrong geometry. The correct shape is the same annular band as the pizza crust, just swept 360° instead of `bearing ± BEARING_ERROR_DEG`. Built as an `L.polygon` via `makeDonut()` using 36 steps for smooth appearance. Dashed stroke (`dashArray: '4 3'`) distinguishes it from directional pizza crusts at a glance. Fill opacity is 0.15 (vs 0.20 for pizza crust) to reflect lower information content.

### `BEARING_ERROR_DEG` promoted to named constant, value raised to 10
Was embedded as a hardcoded `5` inside `makePizzaCrust()` in Session 3. Promoted to a top-level named constant in Session 4b alongside `TIMING_ERROR_S` and `EXPIRE_MIN`. Value raised from 5 to 10 based on realistic assessment of phone-pointing accuracy in the field (holding a phone and swinging it toward a distant flash, possibly in the dark, in rain). To be tuned after actual field testing.

### Map legend updated to reflect current values
Legend text now reads: `WEDGE = ±10° BEARING ERROR`, `ARC DEPTH = ±500ms TIMING`, `DONUT = NO BEARING (COUCH)`. Keep this in sync if constants change.

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
Strikes are never hard-deleted from localStorage on load. Expiry logic is applied at render time (`isExpired(ts)`). If you want to clean up old sessions, CLEAR ALL is the mechanism. `EXPIRE_MIN = 60` is a top-level constant — easy to adjust.

### `renderLog()` guards with `if(!logEl) return`
The TRENDS tab elements don't exist in the DOM when RECORD or MAP is active. `renderLog()` checks for the element before proceeding, so the 30s interval can fire unconditionally without crashing.
