# KML/KMZ polygon zones (parse, point-in-zone with buffer, export)

- **category**: maps-geo
- **provider**: internal (custom), KML/KMZ files from Google My Maps, QGIS or similar
- **reusable**: yes. Pure functions plus one lazy-loaded dependency for KMZ.
- **docs**: https://developers.google.com/kml/documentation/kmlreference

Lets users draw areas of interest in any mapping tool, upload the file, and then filter records by "inside one of these zones". Includes a bounding-box prefilter so per-keystroke filtering over thousands of points stays cheap, an optional buffer in metres, and a KML writer so zones round-trip.

## Playbook

### Prerequisites
- `nanoid` (zone ids) and `jszip` (KMZ only, import it dynamically so it stays out of the main bundle).
- A zone type: `{ id, name, priority, coordinates: {lat,lng}[] }`.
- Browser `DOMParser` (client-side parse) or `@xmldom/xmldom` if parsing on the server.

### Setup steps
1. File input accepts `.kml,.kmz`. Read client-side; KMZ is a zip, so find the `.kml` entry by extension (don't assume the first entry).
2. Parse every `<Placemark>`: `<name>` and `<coordinates>`. KML tuples are `lng,lat[,alt]`; drop altitude, reverse to `{lat,lng}`.
3. Skip placemarks with fewer than 3 valid points. Derive `priority` from a leading integer in the name ("1 - Area"), else file order.
4. Store zones on the owning team/user; keep any shared library of named zones as a separate collection so both feed one picker.
5. Filter with `findZoneForPoint(point, zones, bufferMeters)`.

### Core pattern
```ts
export function parseKml(text: string): Zone[] {
  const doc = new DOMParser().parseFromString(text, "text/xml");
  const zones: Zone[] = [];
  for (const pm of Array.from(doc.querySelectorAll("Placemark"))) {
    const name = pm.querySelector("name")?.textContent?.trim() ?? "";
    const raw = pm.querySelector("coordinates")?.textContent?.trim() ?? "";
    const coordinates = raw.split(/\s+/).filter(Boolean)
      .map((c) => { const [lng, lat] = c.split(",").map(Number); return { lat, lng }; })
      .filter((c) => !isNaN(c.lat) && !isNaN(c.lng));
    if (coordinates.length < 3) continue;
    const m = name.match(/(\d+)/);
    zones.push({ id: nanoid(), name, priority: m ? parseInt(m[1]) : zones.length + 1, coordinates });
  }
  return zones.sort((a, b) => a.priority - b.priority);
}

export async function parseKmlOrKmz(file: File) {
  if (/\.kmz$/i.test(file.name)) {
    const { default: JSZip } = await import("jszip");
    const zip = await JSZip.loadAsync(file);
    const entry = Object.values(zip.files).find((f) => /\.kml$/i.test(f.name) && !f.dir);
    if (!entry) throw new Error("No KML inside the KMZ");
    return parseKml(await entry.async("text"));
  }
  return parseKml(await file.text());
}

// Ray cast
export function pointInPolygon(p, poly) {
  let inside = false;
  for (let i = 0, j = poly.length - 1; i < poly.length; j = i++) {
    const hit = (poly[i].lat > p.lat) !== (poly[j].lat > p.lat) &&
      p.lng < ((poly[j].lng - poly[i].lng) * (p.lat - poly[i].lat)) / (poly[j].lat - poly[i].lat) + poly[i].lng;
    if (hit) inside = !inside;
  }
  return inside;
}

// Buffer: flat-earth distance to each edge (fine under ~10 km)
function distToSegmentMeters(p, a, b) {
  const lat = 111_319.9, lng = lat * Math.cos((p.lat * Math.PI) / 180);
  const px = (p.lng - a.lng) * lng, py = (p.lat - a.lat) * lat;
  const bx = (b.lng - a.lng) * lng, by = (b.lat - a.lat) * lat;
  const len = bx * bx + by * by;
  const t = len === 0 ? 0 : Math.max(0, Math.min(1, (px * bx + py * by) / len));
  return Math.hypot(px - t * bx, py - t * by);
}
```
Bounding-box prefilter: cache a bbox per zone in a `WeakMap` keyed on the **coordinates array** and test the box before the ray cast. When `bufferMeters > 0`, inflate the box by the buffer (latitude pad = `buffer / 111319.9`, longitude pad divided by `cos(lat)`), otherwise buffer matches near the edge are silently dropped.

### Env vars
None.

### Gotchas
- Key the bbox cache on the coordinates array, **not** on the zone id: ids survive a geometry re-import, so an id-keyed cache serves a stale box after the next upload. A `WeakMap` is also garbage-collected with the payload.
- The filter runs per record on every search keystroke and almost always rejects, so the 4-comparison box test is where the speed comes from.
- Export (`zonesToKml`) must not mutate names (no forced leading digits), or export then re-import changes priorities on every cycle.
- Large imported libraries need compaction before storage: drop consecutive duplicate points, round to ~5 decimals, and simplify rings by a metre tolerance with the same edge-distance helper. Keep documents well under the datastore's document size limit.
- Replace a city's whole zone list in one write; partial list merges belong in the client.
- Government GIS sources reproject from a national grid; see `govmap-gis` for that import path.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. Zones come from two sources feeding one picker: per-team uploads and an admin-curated library keyed by city.
