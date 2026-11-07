# OSRM commute times (table API, cached per pair, best-effort fill)

- **category**: maps-geo
- **provider**: OpenStreetMap OSRM (public demo servers, e.g. routing.openstreetmap.de, or self-hosted OSRM)
- **reusable**: yes. Keyless; the cache-key scheme and the failure containment apply to any "minutes from A to many Bs" feature.
- **docs**: https://project-osrm.org/docs/v5.24.0/api/#table-service

Computes travel minutes from a few fixed anchors (work, family, a pin) to many records at once, using OSRM's one-to-many `table` call, and caches the result per (anchor, record) pair. Walk, bike and car use real profiles; "transit" is approximated from car time because OSRM has no transit routing.

## Playbook

### Prerequisites
- Records with `lat`/`lng`; anchors with `{ id, mode, coordinates }`.
- A store for the cache (one doc per team/user holding `recordId -> anchorId -> { minutes, key }`).

### Setup steps
1. Pick the host and per-mode profiles (`routed-foot`, `routed-bike`, `routed-car` on the OSM-hosted server; plain `/table/v1/driving` on a self-hosted one).
2. Call `table` with the anchor as source index 0 and records as destinations, `?sources=0&annotations=duration`, in chunks of about 90 (public servers cap the table size).
3. Take `durations[0].slice(1)`, convert seconds to minutes, store with a **pair key**.
4. Compute only **missing or stale** pairs (see pair key below).
5. Wrap every call in retry with backoff and a hard cap on how long any user-facing request waits for it.

### Core pattern
```ts
// Key covers BOTH ends. Anchor-only keys leave stale minutes after a record's pin is fixed.
const anchorKey = (a) => `${a.mode}:${a.coordinates.lat.toFixed(5)},${a.coordinates.lng.toFixed(5)}`;
const pairKey = (a, rec) => `${anchorKey(a)}>${rec.lat.toFixed(5)},${rec.lng.toFixed(5)}`;

// Only recompute when the key differs from what was stored.
const stale = (recs, anchor, map) =>
  recs.filter((r) => r.coordinates && map[r.id]?.[anchor.id]?.key !== pairKey(anchor, r.coordinates)).map((r) => r.id);

// SSRF-safe: host is a constant, only numeric coordinates are interpolated.
const coord = (c) => {
  if (!Number.isFinite(c.lat) || !Number.isFinite(c.lng)) throw new Error("invalid coordinate");
  return `${c.lng.toFixed(6)},${c.lat.toFixed(6)}`; // lng,lat order
};

async function fetchWithRetry(url: string) {
  let res;
  for (let i = 0; i <= 2; i++) {
    res = await fetch(url, { signal: AbortSignal.timeout(12_000), cache: "no-store", headers: { "User-Agent": "my-app/1.0" } });
    if (res.status !== 429 && res.status < 500) return res;
    if (i === 2) break;
    const ra = Number(res.headers.get("retry-after"));
    await sleep(ra > 0 ? Math.min(ra * 1000, 5000) : 1500 * (i + 1));
  }
  return res!;
}

const url = `${HOST}/${PROFILE[mode]}/table/v1/driving/${[anchor.coordinates, ...pts].map(coord).join(";")}?sources=0&annotations=duration`;

// No transit routing in OSRM: estimate and flag it so the UI shows "~".
const transitMinutes = (carSeconds) => Math.round((carSeconds / 60) * 1.7 + 8);
```

Best-effort fill right after a record is created, so a flaky router can never block or slow the create:
```ts
let cooldownUntil = 0;
export async function fillAfterCreate(records) {
  if (Date.now() < cooldownUntil) return;               // recently failed: skip
  const deadline = new Promise<void>((r) => setTimeout(() => { cooldownUntil = Date.now() + 60_000; r(); }, 8_000));
  await Promise.race([fill(records).catch(() => { cooldownUntil = Date.now() + 60_000; }), deadline]);
}
```
Anything skipped is picked up later by a "refresh missing pairs" route the UI calls when the records are on screen.

### Env vars
None for the public server. A self-hosted deployment would add its own base URL variable.

### Gotchas
- The public demo server **429s bursts and sometimes refuses connections outright** (including from some home/CI IPs). Space anchors about 1 s apart, honour `Retry-After`, and stop a run early on connection refused/DNS/timeouts instead of logging one error per record.
- Log one warning per anchor, not per record. A per-record error flood hides real problems.
- Cap how long a create/save path may wait (about 8 s) and add a cooldown (about 60 s) after any failure; otherwise every following request pays the full retry chain (observed: ~53 s stalls).
- A refresh route that is throttled should answer 200 with a `{ throttled, retryAfterMs }` flag, not 429, if your client globally toasts on 429.
- Coordinates are `lng,lat` in the URL; mixing the order silently routes across the wrong place.
- Mode change, anchor move and record move must all invalidate; that is why the key encodes all three.
- Public servers are for light use. Respect their usage policy or self-host for volume.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. Anchors are per-team; results live in one cached document per team and the UI marks transit estimates with a leading "~".
