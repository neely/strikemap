# Strikemap — External Lightning Data Plan
*Session 6 planning, August 2026 — not yet implemented*

This document covers adding **real network-detected lightning strikes** to the map alongside your own flash-to-thunder observations. It is a companion to [`PLAN.md`](PLAN.md); read that first for the base app design.

**Status: planning complete, implementation not started.** No code in the repo has changed as a result of this document.

**Build order:** Phase 1 → Phase 2 → spike G3.1 → decide on Phase 3. Phases 1 and 2 are committed work. Phase 3 is deliberately deferred to a decision point informed by real latency and quota numbers from Phase 2, with only the cheap spike done up front.

---

## The Goal

Right now `fetchExternalStrikes()` is a stub returning `[]`. The goal is to fill it in: overlay strikes detected by a real lightning network onto the map, so you can compare your own observations against ground truth and see the whole storm, not just the flashes you personally caught.

**Priority: latency, bounded by cost and reliability.** The point of this app is watching a storm move in real time; a 10-minute-old overlay is nearly useless. But the investigation that produced this plan found the lowest-latency source unreachable from a browser, so the shipped design accepts a ~15s worst case in exchange for a supported vendor and a quota that survives being shared. Sub-5s remains possible (Phase 3) but is no longer the driving constraint.

**Non-goals:**
- Not replacing your own observations. Your recorded strikes stay the primary data, with full uncertainty geometry. Network strikes are a supplementary layer.
- Not archiving. Same session-scoped philosophy as the rest of the app.
- Not triangulating. Still a single-observer app.

---

## Source Evaluation (with test results)

Four candidates were considered. Three were ruled out, two of them empirically rather than on paper.

### Blitzortung.org — community VLF network

Free, no key, ~1,800 volunteer stations, good North America coverage. The obvious philosophical fit for a no-backend app. Three access paths, all tested:

**Raw websocket firehose (`wss://ws1|ws2|ws7|ws8.blitzortung.org:3000`)** — what most hobby GitHub clients use. Send `{"a": 111}` to begin streaming; payload is LZW-obfuscated.

Tested from Safari on 2026-08-27:

```
CLOSE — code 1006, reason "(none given)", wasClean=false   [ws1]
CLOSE — code 1006, reason "(none given)", wasClean=false   [ws7]
CLOSE — code 1006, reason "(none given)", wasClean=false   [ws2]
```

Code 1006 = abnormal closure, no close frame, `onopen` never fired. Control test against `wss://ws.postman-echo.com/raw` in the same session succeeded cleanly (connected 224ms, echo round trip 274ms, close code 1000), proving general WebSocket connectivity was fine. **Conclusion: the connection is refused before handshake completion, consistently, from a browser.** Consistent with their published policy — "Do not access our websocket servers from highly frequented websites or via apps. Demonstration of live data on commercial websites or via apps is not permitted."

**Official MQTT broker with geohash topics (`mqtt.blitzortung.org`)** — this is the *right* way to do area filtering: 10-bit Equal Area Space geohash topics (`strikes/lzw_core/b1/.../b10`) let the server filter to your region, with MQTT wildcards to aggregate adjacent cells. Requires username/password. Credentials are only issued to **registered station operators** — people running physical VLF receiver hardware and contributing data back. It is not a developer API key you can request. Ruled out unless a detector is ever built (see Future Work).

**Community relay (`blitzortung.ha.sed.pl:1883`)** — run by the author of the Home Assistant Blitzortung integration, specifically so downstream apps don't each need their own station. Maintains one compliant upstream connection and rebroadcasts geohash-filtered data.

Probed six candidate WebSocket ports from Safari on 2026-08-27 (`wss` and `ws` on 8084, 8083, 9001, 8080, with `/mqtt` and `/` paths): **all six timed out with no response whatsoever** — not even a rejection. Consistent with a bare Mosquitto instance listening only on 1883 raw TCP, which is all the Python-based HA client ever needed.

**Net: Blitzortung is unreachable from a browser by any path.** Requires a server-side bridge (see Phase 3).

### NOAA GOES GLM — satellite, via AWS Open Data

Free, official, no key. Level 2 products land in a public S3 bucket with 30–60s latency from observation, new files roughly every 20s — better than initially assumed. Ruled out anyway:

- **NetCDF4 binary format.** No browser parses this natively; would need a heavy JS library not designed for this use.
- **Poll, not push**, so our own interval stacks on top of the 30–60s satellite latency.
- **CORS unverified** for direct browser `fetch()` against NOAA buckets.
- **~10km energy-weighted flash centroids**, noticeably coarser than ground-based fixes.

Combined latency (satellite processing + poll interval) is worse than Xweather with none of the format simplicity. Explicitly declined.

### NWS / NLDN direct

**Does not exist as a public feed.** NWS operates no lightning network of its own — it licenses Vaisala's NLDN and GLD360 data for internal forecaster use. The NCEI archive states raw data is "available only to government and military users." "NWS lightning data" and "Xweather lightning data" are the same underlying network; Xweather's free tier is the accessible front door to it, not an alternative.

### Xweather (formerly AerisWeather) — Vaisala NLDN/GLD360

Commercial-grade, clean JSON REST endpoint, 15,000 calls/month free tier, no credit card. NLDN has 84m median accuracy across CONUS. `/lightning` returns up to 1,000 events from the past 5 minutes within a radius. Needs a client ID/secret, which must not ship in a public static file — hence a proxy.

**Selected as the Phase 1/2 source.**

---

## Budget Math (Xweather free tier)

Because polling happens **in the Worker, not the browser**, multiple people watching **the same area** share one upstream call. But the cache key is location+radius, so **distinct regions multiply usage**:

| Simultaneous distinct regions | Calls/hour @15s | Sustainable hrs/month |
|---|---|---|
| 1 | 240 | 62.5 |
| 2 | 480 | 31.2 |
| 3 | 720 | 20.8 |
| 5 | 1,200 | 12.5 |

An earlier draft of this plan claimed visitor count was irrelevant to budget. That was wrong — it's true only for co-located viewers. If Strikemap gets shared with people in different cities who use it during the same storm systems, the quota drains several times faster than the single-region math suggests.

**Related exposure:** a public endpoint accepting arbitrary lat/lon can be drained by anyone polling random coordinates — malicious or just a buggy client in a loop. Region-limiting and rate-limiting are therefore budget protections, not just hygiene. See G1.9/G1.10.

### Geographic restriction — the primary budget control

The cleanest fix for multi-region drain is to **serve external data only within a configured bounding box** (initially coastal SC / Charleston metro). Outside it, the Worker returns `outOfRegion: true` and makes no upstream call.

This collapses the table above back to the single-region row: **~62 sustainable hours/month**, regardless of how widely the link gets shared. Someone in Atlanta still gets the full core app — flash-to-thunder recording, geometry, log, trends — they just don't get the network overlay. That's consistent with the modularity principle: the core works everywhere, the add-on is regional.

Requirements:
- **Enforced server-side.** A client-side check is trivially bypassed and protects nothing.
- **Configurable, not hardcoded.** A list of allowed boxes, so a second region can be added deliberately (including travel) rather than requiring a code change.
- **Explained clearly in the UI.** "Network data is Charleston-area only" is a fine answer; a silently empty overlay is not.
- **No new privacy exposure.** The Worker already receives coordinates to run the query; a bbox test adds no data it didn't already have. Coordinates still must never be logged.

Keep the distinct-region cap (G1.9) as a backstop *within* the box — it's cheap and covers the case where the box is later widened.

15,000 calls/month, single region, consumed only while actively polling:

| Poll interval | Calls/hour | Sustainable hrs/month | Avg latency | Worst-case latency |
|---|---|---|---|---|
| 5s | 720 | 20.8 | 2.5s | 5s |
| 10s | 360 | 41.7 | 5s | 10s |
| **15s** | **240** | **62.5** | **7.5s** | **15s** |
| 20s | 180 | 83.3 | 10s | 20s |
| 30s | 120 | 125 | 15s | 30s |

Charleston sees roughly 20–25 thunderstorm days in a peak summer month. At 15–20 active sessions of 1–2 hours each, expect **20–40 active hours/month**.

**Chosen default: 15s.** Lands comfortably inside budget with real headroom for an unusually active season.

Critical nuance: because the endpoint returns everything from the past 5 minutes, **a shorter interval never prevents missing a strike** — it only reduces how long you wait to learn about one. At 30s you would still see every strike, just up to 30s late. So the interval is a pure latency-vs-budget dial with no completeness risk. Do not over-tune it.

**With the geographic restriction in place, exhaustion becomes unlikely again.** ~62 sustainable hours against 20–40 expected is roughly 2× headroom, and the Phase 2 visibility pause stretches it further. Sharing the link no longer threatens the budget, because out-of-region visitors cost nothing upstream.

Degradation stays graceful and bounded either way: the overlay goes dark, a status line says so plainly, and every core function keeps working.

---

## Architecture

Three components, built in order, each independently useful.

```
┌─────────────────────────────────────────────────────────┐
│  index.html / desktop.html                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │  external-strikes module (new)                    │  │
│  │  - transport-agnostic: accepts strikes, renders    │  │
│  │  - source order: Xweather poll → BZ bridge → off   │  │
│  │  - switches to BZ only when quota is exhausted     │  │
│  │  - status indicator: which source is live          │  │
│  └───────────────────────────────────────────────────┘  │
└──────────────┬────────────────────────┬─────────────────┘
               │ HTTP poll (15s)        │ WebSocket
               │ PRIMARY                │ BACKSTOP (Phase 3)
               ▼                        ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│ Worker: xweather-proxy    │  │ Durable Object: bz-bridge│
│ - holds API credentials   │  │ - TCP → blitzortung relay│
│ - caches response         │  │ - hand-rolled MQTT sub   │
│ - KV usage counter        │  │ - LZW decode server-side │
│ - PHASE 1                 │  │ - PHASE 3 (only if needed)│
└──────────────────────────┘  └──────────────────────────┘
```

**Why Xweather is primary despite being the metered one:** it's better data (84m accuracy vs. community-network fixes), on real infrastructure with an SLA, behind a proxy we fully control. Blitzortung is free and unmetered but rides on one volunteer's relay with no guarantees. The metered-but-reliable source should carry normal operation; the free one is insurance against the meter running out.

**Until Phase 3 exists, quota exhaustion simply means no network overlay until the monthly reset.** That is an acceptable state — the app's core function is unaffected — and it's the honest near-term reality. The Blitzortung backstop is not a small config switch; it is the entire Phase 3 build. Don't plan around having it before it's built.

### Design principles

**1. The core app never depends on this module.**

Flash-to-thunder recording, distance/bearing calculation, the strike log, trends, pizza-crust geometry, and localStorage persistence must all work exactly as they do today with the network layer absent, disabled, broken, rate-limited, or offline. This feature is strictly **additive**.

The test for this is literal, not aspirational: **if you deleted the external-strikes module entirely, the app should still run without touching anything else.** No shared functions, no shared state, no core code path that calls into it and expects a result. If satisfying that test requires editing core code, the boundary is in the wrong place.

Practically this means the module:
- Owns its own render layer (its own Leaflet layer group, cleared and redrawn independently of `strikeLayers`)
- Never writes to `sm_strikes` or any localStorage key other than its own on/off preference
- Never participates in the trend, approaching/receding, OBS badge, or expiry logic
- Is wrapped so an exception inside it cannot break the map render or the recording loop
- Defaults **off** — a fresh visitor gets exactly today's app until they opt in

**2. The client does not know or care where strikes come from.**

Both transports normalize to one internal strike shape before rendering. This is what makes Phase 3 a drop-in addition rather than a rewrite, and what lets source handover work without touching render code.

**3. Your observations and network data never merge.**

They are different kinds of measurement with different error characteristics — yours is a polar estimate with real uncertainty geometry, theirs is a reported point fix. Rendering them differently isn't just a UI nicety; conflating them would misrepresent both. They stay in separate arrays, separate layers, separate visual language.

### Normalized strike shape

Both sources emit this. Deliberately distinct from the `sm_strikes` localStorage schema — external strikes are **never** written to localStorage and never mix with your own observations.

```javascript
{
  lat:    32.7412,      // decimal degrees
  lon:    -79.9231,
  ts:     1787876736860, // ms epoch, strike time (not receive time)
  source: 'xweather',   // 'xweather' | 'blitzortung'
  distMi: 3.42          // computed client-side from user position
}
```

---

## Repo Layout & Division of Labor

The app is deliberately single-file and no-build. A Worker breaks that constraint, so it must be quarantined rather than allowed to spread.

```
strikemap/
├── index.html              ← unchanged constraint: single file, no build
├── desktop.html            ← unchanged constraint: single file, no build
├── worker/                 ← NEW, self-contained, its own toolchain
│   ├── src/index.js
│   ├── wrangler.toml       ← config only; NO secrets committed
│   ├── package.json
│   └── README.md           ← deploy instructions
└── ...
```

The `worker/` directory is the only place a build step, `node_modules`, or config file may exist. Nothing in it is ever loaded by the HTML files — they only see its deployed URL. The root app remains exactly as portable as it is today: copy `index.html` anywhere and it works, minus the optional overlay.

Add `worker/node_modules/` and `.dev.vars` to `.gitignore` before the first Worker commit.

### What Claude does vs. what Benjamin does

Claude can write and test all Worker and client code, and verify logic locally. Claude **cannot** deploy or hold credentials.

**Benjamin's tasks (blocking, cannot be delegated):**
1. Create the Xweather free-tier account; obtain client ID and secret
2. Set them as Worker secrets directly (`wrangler secret put XWEATHER_CLIENT_ID`, same for the secret) — entered in his own terminal, never pasted into chat, never written to a repo file
3. Run `wrangler deploy` for the initial deploy and confirm the resulting URL
4. Create the KV namespace and bind it
5. Set a Cloudflare billing alert before any Phase 3 work

**Claude's tasks:** all code, all local verification, all documentation, plus a deploy checklist precise enough that the above is mechanical.

**Implication for sequencing:** Phase 1 code can be written and unit-tested before credentials exist, using a mock upstream. Gates G1.1 and G1.4–G1.7 require a real deploy and therefore Benjamin. Don't treat Phase 1 as blocked on credentials — treat *verification* as blocked.

---

## Phase 1 — Xweather Proxy Worker

**Deliverable:** a deployed Worker at a URL that returns normalized strike JSON, with credentials hidden and usage tracked.

### Scope
- Single Worker, no Durable Object (stays on Cloudflare free tier, 100k req/day)
- Holds `XWEATHER_CLIENT_ID` / `XWEATHER_CLIENT_SECRET` as Worker secrets — never in the repo, never in client JS
- Endpoint: `GET /lightning?lat=&lon=&radius=`
- Validates and clamps inputs (see gates below)
- Caches upstream response for the poll interval so multiple clients cost one upstream call
- KV counter tracking monthly upstream call count
- Returns normalized array, plus metadata: `{ strikes: [], budget: { used, limit, resetsAt }, cachedAt }`

### Gates — must all pass before Phase 2

- [ ] **G1.1** Worker deploys and responds to a manual `curl` with valid JSON
- [ ] **G1.2** Credentials confirmed absent from the repo and from any client-visible response (`git grep` for the key; inspect response body)
- [ ] **G1.3** Input validation rejects malformed lat/lon and clamps radius to a max (prevents a hostile or buggy client burning budget on a giant query)
- [ ] **G1.4** Two rapid requests within the cache window produce exactly **one** upstream call (verify via KV counter not incrementing twice)
- [ ] **G1.5** KV counter increments correctly and survives Worker restarts
- [ ] **G1.6** Budget ceiling enforced: when counter ≥ 15,000, Worker returns cached/empty with a clear `budgetExhausted: true` flag rather than calling upstream
- [ ] **G1.7** Upstream failure (bad key, Xweather 5xx, timeout) returns a structured error, never a 500 with a stack trace
- [ ] **G1.8** CORS headers permit `strikemap.benneely.com` only — not `*`
- [ ] **G1.8b** Bounding-box check enforced server-side: coordinates outside the configured region return `outOfRegion: true` with **zero** upstream calls — verified by watching the KV counter stay flat during out-of-region requests
- [ ] **G1.8c** Allowed regions are configuration, not hardcoded constants — a second box can be added without editing logic
- [ ] **G1.9** Distinct-region cap: the Worker tracks how many unique location+radius cache keys are active and refuses new ones beyond a set ceiling, so an unexpected spread of users can't silently drain the month
- [ ] **G1.10** Coordinates snapped to a coarse grid (~0.1°) before forming the cache key, so near-identical locations share one upstream call instead of each creating their own
- [ ] **G1.11** Per-IP rate limit, so a single misbehaving or hostile client can't burn quota in a loop

### Testing
- Manual `curl` against known coordinates during an actual storm (verify non-empty results) **and** clear weather (verify empty array, not an error)
- Force budget exhaustion by temporarily setting the limit to 2 and confirming G1.6 behavior
- Temporarily break the API key and confirm G1.7

---

## Phase 2 — Client Integration

**Deliverable:** network strikes visible on the map in both HTML files, cleanly distinguishable from your own.

### Scope
- New `external-strikes` module inside each HTML file (keeping the single-file, no-build constraint)
- Replace the `fetchExternalStrikes()` stub with a real implementation
- Poll only while the MAP tab is visible **and** the document is focused — pause on tab switch and on backgrounding (`visibilitychange`), which materially extends budget
- Render network strikes as a **small dot/crosshair, not a pizza-crust wedge** — the network gives an actual lat/lon fix, so drawing our bearing/timing uncertainty geometry on it would be actively misleading

**Rendering specifics (decided, not left to implementation):**
- Marker: 3px `L.circleMarker`, no fill, 1px stroke — visually subordinate to your own strikes, which keep the filled dot + wedge + label treatment
- **Display cap: 300 markers.** The endpoint can return up to 1,000 events; rendering all of them on a phone during an active cell would tank frame rate. Keep the 300 most recent, drop the rest, and note truncation in the status line
- **Request radius: 25 miles**, independent of `MAX_DIST_MI` (6.2). That constant governs which of *your* observations are plottable given thunder audibility; network strikes have no such limit and showing the wider storm context is the point. Do not reuse the constant
- Markers cleared and redrawn wholesale each poll — no diffing, no animation. Simpler and the data is fully replaced anyway

**Age coloring — a real inconsistency to resolve:** the Xweather endpoint only returns the past 5 minutes, so every network strike would land in the `recent` bucket and the yellow/orange/blue-grey scheme conveys nothing. Options: (a) drop age coloring for network strikes and use one neutral color, or (b) recolor across a 0–5min gradient with its own thresholds. **Decision: (a), single neutral color (`--dim` family, distinct from all three age colors).** It's honest — we genuinely don't have age spread to show — and it reinforces the visual separation from your own strikes. Revisit only if a source with a longer window is ever added.
- Toggle button in the map controls bar next to AGE: `NETWORK ON/OFF`, defaults **off** on first visit (opt-in, no surprise network calls)
- Status line: source, last-updated time, and budget state — staleness must always be visible, never silent
- New legend rows explaining the marker distinction
- Fails soft: if the Worker is unreachable, the app behaves exactly as it does today

**Response handling — explicit behavior per case:**

| Worker response | Client behavior |
|---|---|
| `strikes: [...]` | Render; update last-updated timestamp |
| `strikes: []` | Render empty; status "no strikes in range" — **not** an error state |
| `outOfRegion: true` | **Stop polling for the session.** Show the region explanation, auto-disable the toggle. Retrying is pointless and wastes requests |
| `budgetExhausted: true` | Stop polling until next reset (use `resetsAt`). Status says so plainly. Phase 3: hand over to bridge instead |
| HTTP error / timeout | Retry with backoff (30s, 60s, 120s, then every 5min). After 3 consecutive failures, status shows "network data unavailable" |
| Malformed JSON | Treat as HTTP error; never let a parse throw escape the module |

Backoff state resets on a successful response or when the user toggles the layer off and on.

### Gates — must all pass before considering Phase 3

- [ ] **G2.1** Your own strikes and network strikes are visually unambiguous at a glance on a 390px screen
- [ ] **G2.2** Network layer fully off = byte-identical behavior to current app; zero network calls made
- [ ] **G2.2b** **Deletion test:** commenting out the entire external-strikes module leaves a fully working app with no errors and no dangling references — verified by actually doing it, not by inspection
- [ ] **G2.2c** An exception thrown inside the module cannot break map render or the flash/thunder recording loop — verified by deliberately throwing one
- [ ] **G2.3** Polling verifiably stops on tab switch away from MAP and on document blur
- [ ] **G2.4** Worker unreachable / returns error → app still fully functional, no console errors, clear "network data unavailable" state
- [ ] **G2.5** External strikes never appear in the TRENDS log, never enter `sm_strikes`, never affect the approaching/receding calculation
- [ ] **G2.6** CLEAR ALL does not touch external strikes (they're not ours to clear); they refresh on next poll
- [ ] **G2.7** Constants stay identical between `index.html` and `desktop.html` — no drift (per the existing constants-audit practice)
- [ ] **G2.8** Field test during a real storm: network strikes plausibly correlate with your own recorded observations
- [ ] **G2.9** Budget state is visible to you without digging — remaining quota surfaced in the status line or a dashboard check, since this number is what decides whether Phase 3 ever gets built
- [ ] **G2.10** Out-of-region users see a clear explanation ("network data is Charleston-area only"), not an empty overlay or an error — and the core app is visibly unaffected

### Testing
- **Bearing calibration** — the payoff. Compare your logged bearings against network-confirmed positions for the same strike. This is the empirical data needed to tune `BEARING_ERROR_DEG` from its current ±10° field estimate.
- Clear weather test: empty overlay renders cleanly, no error state
- Airplane-mode test: fails soft per G2.4
- Both mobile and desktop files

**Gate to Phase 3:** Phase 2 deployed and field-tested across at least one storm, **and** the Phase 3 spike (G3.1) has passed.

**This gate has moved twice; here's the honest reasoning.** It was originally conditional on quota exhaustion. It was then promoted to planned work because multi-region sharing looked likely to drain the budget. The geographic restriction largely removes that pressure — with a bbox in place, ~2× headroom returns and exhaustion is unlikely again.

So the multi-region argument for building Phase 3 no longer holds. What remains:

- **Latency.** 1–3s push vs. a 15s poll floor. Real, but 15s may well be fine in practice — that's an empirical question Phase 2 answers.
- **Insurance.** An unmetered second source means no single-vendor dependency, and no dead overlay if the free tier changes terms.
- **Build-when-calm.** Still true that a fallback written under pressure is worse than one written deliberately.

**Recommendation: do the G3.1 spike regardless** — it's an afternoon, it settles whether this is even possible, and the answer is worth having on record either way. Decide on the full build *after* Phase 2 has run through a real storm and produced actual latency and quota numbers. If 15s feels fine and the counter sits at 20%, Phase 3 is optional polish, not insurance.

---

## Phase 3 — Blitzortung Bridge (deferred decision, spike first)

**Deliverable:** an unmetered second source that takes over automatically when the Xweather quota is exhausted, and reverts when it resets. Side benefit: true push-based strikes at 1–3s latency instead of a 15s poll floor.

**Build it while things are calm, not when the quota is already dead.**

### The cold-fallback problem

A fallback that sits dormant for months and then activates unattended, having never run in anger, will fail exactly when it's needed. This is the main risk of building it early, and it has to be designed against rather than hoped away:

- **Exercise it regularly.** A scheduled health check (Cron Trigger, a few times a week) connects the bridge, confirms it receives real strikes, and logs the result. Cheap, and it means the fallback is known-good rather than assumed-good.
- **Make handover manually triggerable** so it can be tested on demand, not only when the quota dies.
- **Alert on health-check failure** so a broken bridge is discovered in advance, not during a storm.

### Scope
- Cloudflare **Durable Object** (free tier, SQLite-backed) using the `connect()` TCP Socket API
- Opens raw TCP to `blitzortung.ha.sed.pl:1883`, speaks MQTT directly
- **Hand-rolled minimal MQTT client** — existing npm clients assume Node's `net` module, not the Workers Streams API. Only need CONNECT / SUBSCRIBE / PUBLISH-receive / PINGREQ for a read-only subscriber; the wire format is simple enough to implement safely.
- Subscribes to geohash topics covering the client's area
- LZW decode server-side (algorithm already validated in the Session 6 test tool)
- Fans out to browser clients over WebSocket

### Critical constraint — hibernation and cost

Durable Objects bill for wall-clock duration while active and unable to hibernate. A DO holding a TCP socket open 24/7 would consume roughly **10,800 of the 13,000 free daily GB-seconds doing nothing**.

**Therefore: the bridge must open the upstream connection only while at least one client is connected, and shut down completely when the last one disconnects.** This fits the app's occasional-use nature naturally. Non-negotiable — getting this wrong turns a free service into a billed one.

### Risks — acknowledge honestly before starting

| Risk | Severity | Mitigation |
|---|---|---|
| Relay is one person's hobby infrastructure, no SLA | **High** | Phase 2 fallback always remains wired and functional |
| Hand-rolled MQTT has subtle bugs | Medium | Read-only subscriber only; extensive logging; never publishes |
| Relay could block us or go away | Medium | Fallback chain; no user-facing breakage if it does |
| DO hibernation misconfigured → unexpected bill | **High** | G3.4 below; set a billing alert before first deploy |
| Adds real complexity to a deliberately simple app | Medium | This is the reason Phase 3 is gated, not automatic |

### Gates

- [ ] **G3.1** MQTT CONNECT/SUBSCRIBE handshake succeeds against the relay from a Worker (spike test first, before building anything real)
- [ ] **G3.2** LZW decode produces valid strike JSON server-side
- [ ] **G3.3** Geohash topic calculation verified against known coordinates — subscribing to the correct cells for a given lat/lon
- [ ] **G3.4** **Cost gate:** DO confirmed idle/hibernating with zero clients connected. Verify in the Cloudflare dashboard over 24h that duration charges are effectively zero when unused. Billing alert configured.
- [ ] **G3.5** Last client disconnecting closes the upstream TCP socket
- [ ] **G3.6** Automatic handover: when the Worker reports `budgetExhausted: true`, the client switches to the bridge without a reload and without user action
- [ ] **G3.7** Handover reverses cleanly at monthly quota reset — back to Xweather as primary, bridge disconnects and DO hibernates
- [ ] **G3.8** Bridge failure while it is the active source degrades to "no network data," not a broken map — verified by killing the bridge mid-session
- [ ] **G3.9** Client-visible indicator correctly shows which source is live, including during handover
- [ ] **G3.10** Scheduled health check runs, verifies real strikes are received, and logs pass/fail without keeping the DO awake between runs
- [ ] **G3.11** Handover can be triggered manually (config flag or test endpoint) so the path is exercisable on demand, not only at quota death
- [ ] **G3.12** Full dress rehearsal: force `budgetExhausted`, confirm handover, run on the bridge through a real storm, then revert — all before relying on it

**Spike first:** G3.1 alone should be validated in a throwaway Worker before committing to the full build. If the relay won't accept a Workers TCP connection, Phase 3 dies there and costs an afternoon instead of a week.

---

## Licensing & Attribution

- **Xweather** free tier — review their terms for attribution requirements before shipping; add credit to the map legend or footer as required.
- **Blitzortung** — noncommercial only. Strikemap's PolyForm Noncommercial 1.0.0 license aligns, but their data policy is separate from our code license and must be honored independently. Attribution required if displayed.
- Neither source's data is stored or redistributed — displayed live, then discarded. This is deliberate and keeps us clear of archival/redistribution terms.
- Add attribution alongside the existing OSM/CARTO credit, which is already a licensing requirement — don't remove or crowd that.

---

## Explicitly Out of Scope

- Storing external strikes in localStorage or any persistence
- Merging external strikes into the trend/approaching calculation
- Historical strike playback
- Alerts or notifications based on network data
- Any paid tier of any provider — if the free tiers can't do it, that's a finding, not a budget request

---

## Rollback

Every phase must be revertable without drama, because this ships to a live URL used during actual storms.

- **Client (Phase 2):** `git revert` the commit and redeploy. Cloudflare Pages serves the previous build within a minute. Because the module is isolated per the deletion test, reverting cannot orphan core code.
- **Worker (Phase 1):** the client already treats an unreachable Worker as a soft failure, so deleting or disabling the Worker degrades cleanly with no client change required.
- **Kill switch:** the Worker can return `outOfRegion: true` unconditionally via a config flag, disabling the overlay for everyone without a client deploy. Useful if the quota is unexpectedly draining or the upstream misbehaves mid-storm.
- Never revert Phase 1 and Phase 2 in the same step — take the client down first, then the Worker, so no one is left polling a dead endpoint.

---

## Open Questions

- Does Xweather's free tier require attribution text? (check terms at signup — blocks shipping Phase 2 if yes and unaddressed)
- What's Xweather's actual observed latency from strike to API availability? Measure it in Phase 1 rather than trusting marketing copy. This number materially affects whether Phase 3 is worth building.
- Is the community relay still actively maintained in 2026? Last confirmed activity predates this planning session. **Check before the G3.1 spike, not after.**

**Resolved:** the network layer defaults **off** on first visit, and the choice persists in localStorage under its own key (`sm_ext_enabled`). Opt-in once, remembered thereafter. This is the only localStorage key the module may write.

---

## Session 6 Test Artifacts

A standalone diagnostic tool (`blitzortung-test.html`) was built during this session to empirically test the Blitzortung websocket rather than assume. It measures message cadence, per-strike latency, and client-side radius filtering, and includes a control test against a known-good echo server plus a relay port prober.

It is **not part of the app** and is intentionally not committed to this repo — it connects directly to Blitzortung's firehose, which their policy asks apps not to do. It was a one-off personal diagnostic. Its findings are recorded above; recreate it if similar testing is needed again.
