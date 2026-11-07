# Nearest transit stations (static GeoJSON + haversine, no API)

- **category**: maps-geo
- **provider**: OpenStreetMap data (exported once to a static GeoJSON file) and internal (custom) helpers
- **reusable**: yes. Pure functions with no runtime dependency; swap the dataset for any city.
- **docs**: https://wiki.openstreetmap.org/wiki/Overpass_API

"Nearest light-rail / train stations and walking minutes" for any coordinate, plus a map overlay, without calling a routing or places API at runtime. A one-off script exports stations and lines from OSM into a static file served from `public/`; everything else is straight-line math.

## Playbook

### Prerequisites
- An Overpass query (or an extract) for `railway=station` / `station=light_rail` points and the line geometries for your area.
- A script that writes a `FeatureCollection` to `public/data/transit.geojson`.

### Setup steps
1. Export with a script (not at runtime): Points get `properties.kind` (`light_rail_station` | `rail_station`), `name`, optional `line`; LineStrings carry the routes for the map layer.
2. Load the file client-side for the map layer (fetch once, cache) and with `fs`/`fetch` server-side if needed.
3. Parse stations with a defensive reader, then call `nearestStations(coords, stations, { limit, maxMeters })`.
4. Render the layer on the map behind a user preference (default on), separate from record markers.

### Core pattern
```ts
export const WALK_METERS_PER_MIN = 80;   // ~4.8 km/h
export const DETOUR_FACTOR = 1.3;        // street grid vs straight line

export function haversineMeters(a: LatLng, b: LatLng) {
  const r = (d: number) => (d * Math.PI) / 180;
  const h = Math.sin(r(b.lat - a.lat) / 2) ** 2 +
    Math.cos(r(a.lat)) * Math.cos(r(b.lat)) * Math.sin(r(b.lng - a.lng) / 2) ** 2;
  return 2 * 6_371_000 * Math.asin(Math.min(1, Math.sqrt(h)));
}
export const walkMinutes = (m: number) => Math.ceil((m * DETOUR_FACTOR) / WALK_METERS_PER_MIN);

export function stationsFromGeoJSON(fc): Station[] {
  const out: Station[] = [];
  for (const f of fc?.features ?? []) {
    if (f?.geometry?.type !== "Point") continue;               // ignore lines
    const [lng, lat] = f.geometry.coordinates ?? [];
    const name = typeof f.properties?.name === "string" ? f.properties.name.trim() : "";
    if (!Number.isFinite(lng) || !Number.isFinite(lat) || !KINDS.has(f.properties?.kind) || !name) continue;
    out.push({ name, kind: f.properties.kind, line: f.properties.line || undefined, lat, lng });
  }
  return out;
}

export function nearestStations(c: LatLng, stations: Station[], { kind, limit = 2, maxMeters = 2500 } = {}) {
  return stations
    .filter((s) => !kind || s.kind === kind)
    .map((s) => ({ ...s, meters: haversineMeters(c, s) }))
    .filter((s) => s.meters <= maxMeters)
    .sort((a, b) => a.meters - b.meters)
    .slice(0, limit)
    .map((s) => ({ ...s, walkMinutes: walkMinutes(s.meters) }));
}
```

### Env vars
None.

### Gotchas
- Walking minutes are an estimate (straight line times a detour factor), not a route. Label them as approximate.
- Keep the module free of framework and database imports so client components, server code and unit tests can all use it.
- Skip features without a name or with a non-finite coordinate; OSM data has both.
- Regenerate the file by script when lines open; a stale static file is the main drift risk.
- Pair with an OSRM-style router only when you need real walking routes; this is the zero-cost tier.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. The same dataset feeds a toggleable map layer and the "nearest station" facts on a record.
