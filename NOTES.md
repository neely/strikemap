# Strikemap — Implementation Notes
*Running log of decisions, gotchas, and things to remember across sessions*

---

## Session 1 — Map

### CartoDB Dark Matter chosen over standard OSM
Standard OSM tiles are light/colorful and fight the dark UI badly. CartoDB Dark Matter is free, no API key, loads from CDN (`https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png`). Confirmed working. If you ever want to swap:
- Light minimal: `https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png`
- Standard OSM: `https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`

### `map.invalidateSize()` is required when switching tabs
Leaflet measures its container on init. If the map `<div>` is hidden (display:none) when the page loads, Leaflet thinks it's 0×0. Calling `map.invalidateSize()` in `showScreen('map')` forces it to re-measure and re-render. Without this, you get a gray or broken map on first tab switch. This is already wired — don't remove it.

### `tap: false` in Leaflet options
Leaflet's built-in tap handler interferes with mobile touch on locked maps. Disabling it explicitly prevents phantom clicks and scroll jank. Already set.

### Fallback coordinates
James Island / Charleston, SC: `32.7357, -79.9956`. This is where the original mockup map screenshot was centered. Convenient for testing since the mockup SVG overlay was calibrated to it. Change to wherever makes sense after deploy if desired.

### `window.sm_map` convention
The Leaflet instance is assigned to `window.sm_map` so it's accessible across script scopes without module imports. Follow the same pattern for any other global state that needs cross-function access.

### Geolocation fires on page load
Per the plan, geolocation prompts immediately on open (no user gesture required for geo on most browsers). This is intentional and correct — you want it centered before you need it. Compass is different; it waits for user gesture. Don't merge these into a single permission flow.

### Attribution styling
CartoDB requires OSM + CARTO attribution. Styled it small and dim (`font-size:7px`, `color:var(--dim)`, semi-transparent background) so it doesn't visually intrude. Don't remove it — it's a licensing requirement.

---

## General Gotchas

### HTTPS required for compass + geolocation on mobile
Both `DeviceOrientationEvent` and `navigator.geolocation` are blocked on `file://` and plain `http://` on iOS Safari and Chrome Android. GitHub Pages serves over HTTPS automatically — that's the intended deploy target. For local testing on phone, use `npx serve` or similar and access via your machine's local IP over http (geolocation may still be blocked; compass will be).

### iOS vs Android compass APIs
- **iOS 13+**: `DeviceOrientationEvent.requestPermission()` must be called from a user gesture. Returns a promise; `granted` means compass is live. Use `event.webkitCompassHeading` for the heading value (0–360, true north, already corrected for device orientation).
- **Android / desktop Chrome**: No permission dialog. `deviceorientation` event fires automatically after `addEventListener`. Use `event.alpha` (0–360, but note: `alpha` is degrees *from* north measured clockwise when `event.absolute === true`; convert with `(360 - alpha) % 360` to get compass bearing).
- **Desktop**: `deviceorientation` typically doesn't fire at all (no sensor). Handle gracefully — `bear` stays `null`.

### `webkitCompassHeading` vs `alpha` conversion
```javascript
function getHeading(event) {
  if (typeof event.webkitCompassHeading === 'number') {
    return event.webkitCompassHeading; // iOS: already true north, 0–360
  }
  if (event.absolute && typeof event.alpha === 'number') {
    return (360 - event.alpha) % 360; // Android absolute: convert to compass bearing
  }
  if (typeof event.alpha === 'number') {
    return (360 - event.alpha) % 360; // Android non-absolute: best effort
  }
  return null;
}
```

### Rolling average for compass smoothing
Raw `deviceorientation` events fire ~60Hz and jump around. A simple circular mean over the last N samples is enough:
```javascript
const buf = [];
function pushHeading(h) {
  buf.push(h);
  if (buf.length > 8) buf.shift(); // 8-sample window
  let s = 0, c = 0;
  buf.forEach(a => { s += Math.sin(a * Math.PI/180); c += Math.cos(a * Math.PI/180); });
  return (Math.atan2(s, c) * 180/Math.PI + 360) % 360;
}
```
Use circular mean, not arithmetic mean — arithmetic mean breaks near 0°/360° (e.g. averaging 355° and 5° gives 180°, which is wrong; circular mean gives 0°).

### HEARD THUNDER must be inert until FLASH is tapped
The THUNDER button should do nothing if `flashTime` is null. The mockup had it pre-styled as `armed` (visual bug). In the real implementation the `armed` class should only be added by the FLASH handler, not present in HTML at load.

### localStorage — newest first
Strikes are stored newest-first (`array.unshift(newStrike)`). The log renders them in array order (newest at top). The map renders them in array order too — the first entry in the array is always the most recent strike, which gets the distance label. Don't sort on read; maintain order on write.

### Strike expiry is soft
Strikes are never hard-deleted from localStorage on load (it would be bad to lose data mid-storm just because you reloaded the page). Expiry logic is applied at render time. If you want to clean up old sessions, CLEAR ALL is the mechanism.
