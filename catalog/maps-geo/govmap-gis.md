# GovMap (Israeli government GIS)

- **category**: maps-geo
- **provider**: Israel Survey / GovMap (govmap.gov.il)
- **reusable**: no — Israel-specific GIS, and the API token is origin-locked to one specific deployed domain, so the pattern can be copied but the credential itself cannot be shared across repos/domains.
- **docs**: https://www.govmap.gov.il/ (no public English API docs; the JS widget is `https://www.govmap.gov.il/govmap/api/govmap.api.js`)

## Overview
GovMap is the Israeli government's official GIS platform, exposing neighborhood/settlement boundary geometry (streets, municipal layers) through a browser-only JS widget rather than a plain REST API. It returns all geometry in the Israeli Transverse Mercator (ITM / EPSG:2039) projection in meters, which must be reprojected to WGS84 (lat/lng) before it's usable with any standard map library. The API token is origin-locked — it only authenticates requests coming from the specific domain it was registered against.

## Playbook

### Prerequisites
- A GovMap API token registered for your exact production origin (request via GovMap; it will 401 from any other origin, including `localhost`, so local testing has to go through a deployed preview domain or a token specifically registered for local dev).
- `proj4` (or an equivalent projection library) for ITM ↔ WGS84 conversion.
- The token must **never** be shipped to the client bundle unconditionally — gate it behind a server route that only privileged users (e.g. admins) can call, since GovMap itself does not scope what a token can query once loaded into a page.

### Setup steps
1. Store the token as a server-only env var (not `NEXT_PUBLIC_`-prefixed or equivalent), and add an authenticated API route that returns it only to the roles that need it (e.g. admins triggering a one-off import), rather than embedding it in every visitor's bundle.
2. On the client, lazy-load GovMap's widget script (`govmap.api.js`) once, guarding against double-injection (check for an existing `<script>` tag with that `src` before appending a new one).
3. Resolve a target city/settlement's location. Prefer GovMap's own `search` call with `isAccurate: true` filtered to `type === "settlement"` — it returns GovMap's own canonical spelling and an ITM centroid, which sidesteps fuzzy-matching your own city name against GovMap's naming (their official spelling can differ subtly from a commonly-used short form). Fall back to your own geocoder (see `nominatim-geocoding.md`) + fuzzy string matching only when the settlement search comes up empty (e.g. a compound municipality name that doesn't resolve from its short form).
4. Enumerate boundary entities near that center by calling the widget's location-based feature query in a small fixed grid of offsets around the center point (a single point query only covers a limited radius; tile out a few hundred/thousand meters in each direction to cover a whole city), deduping by name+city across grid points, with a short delay between calls to avoid rate-limiting.
5. Fetch each entity's actual polygon geometry via GovMap's `search` (matched by name) → `getSearchResultData` (returns WKT geometry) — this is a second round-trip per entity since the enumeration call only returns attributes, not geometry.
6. Parse the returned WKT (`POLYGON`/`MULTIPOLYGON`, with or without a Z ordinate) into rings of ITM coordinate pairs, take the largest ring if multiple are returned (no holes/multi-part support needed for typical use), then reproject every point from ITM to WGS84 before storing/rendering.
7. Wrap the whole import in a clear, translatable error when the underlying call fails — a token/origin mismatch is by far the most common failure and is otherwise indistinguishable from a generic network error.

### Core pattern
```ts
// govmap-import.ts — generic pattern: ITM reprojection + WKT parsing + a
// gridded neighborhood/boundary import against GovMap's JS widget.
import proj4 from "proj4";

const ITM =
  "+proj=tmerc +lat_0=31.7343936111111 +lon_0=35.2045169444445 +k=1.0000067 " +
  "+x_0=219529.584 +y_0=626907.39 +ellps=GRS80 " +
  "+towgs84=-24.0024,-17.1032,-17.8444,-0.33009,-1.85269,1.66969,5.4262 +units=m +no_defs";
const WGS84 = "+proj=longlat +datum=WGS84 +no_defs";

export function itmToWgs84(x: number, y: number): { lat: number; lng: number } {
  const [lng, lat] = proj4(ITM, WGS84, [x, y]);
  return { lat, lng };
}
export function wgs84ToItm(lat: number, lng: number): { x: number; y: number } {
  const [x, y] = proj4(WGS84, ITM, [lng, lat]);
  return { x, y };
}

// Parses POLYGON/MULTIPOLYGON WKT (with or without a Z/M ordinate) into rings
// of ITM points. Rings are always the innermost paren groups, so matching
// "(non-paren content)" globally finds every ring regardless of nesting depth.
export function parseWkt(wkt: string): Array<Array<{ x: number; y: number }>> {
  const body = wkt.replace(/^\s*(MULTIPOLYGON|POLYGON|MULTIPOINT|POINT)\s*(Z|M|ZM)?\s*/i, "");
  const ringMatches = body.match(/\(([^()]+)\)/g) ?? [];
  return ringMatches
    .map((ring) =>
      ring.slice(1, -1).split(",")
        .map((pair) => pair.trim().split(/\s+/).map(Number))
        .filter(([x, y]) => Number.isFinite(x) && Number.isFinite(y))
        .map(([x, y]) => ({ x, y }))
    )
    .filter((ring) => ring.length > 0);
}

const GOVMAP_SCRIPT_SRC = "https://www.govmap.gov.il/govmap/api/govmap.api.js";

export function loadGovmapScript(): Promise<void> {
  if (typeof window !== "undefined" && (window as any).govmap) return Promise.resolve();
  return new Promise((resolve, reject) => {
    const existing = document.querySelector<HTMLScriptElement>(`script[src="${GOVMAP_SCRIPT_SRC}"]`);
    if (existing) {
      existing.addEventListener("load", () => resolve());
      existing.addEventListener("error", () => reject(new Error("GovMap script failed to load")));
      return;
    }
    const script = document.createElement("script");
    script.src = GOVMAP_SCRIPT_SRC;
    script.onload = () => resolve();
    script.onerror = () => reject(new Error("GovMap script failed to load"));
    document.head.appendChild(script);
  });
}

// A small fixed grid of offsets (meters) to tile a location query across a
// city-sized area — a single point query only covers a limited radius.
const GRID_OFFSETS = [
  { dx: 0, dy: 0 }, { dx: 2500, dy: 0 }, { dx: -2500, dy: 0 },
  { dx: 0, dy: 2500 }, { dx: 0, dy: -2500 },
];

function sleep(ms: number) {
  return new Promise((r) => setTimeout(r, ms));
}

// Server-side token gate — never inline the token unconditionally:
// export const GET = withAdmin(async () =>
//   NextResponse.json({ token: process.env.GOVMAP_API_TOKEN ?? null })
// );
```

### Env vars
- `GOVMAP_API_TOKEN` — the origin-locked GovMap API token.

### Gotchas
- The token is validated by GovMap against the calling **origin**, not just checked as a static secret — it 401s from `localhost` even if it's a genuinely valid token, and will 401 from any domain other than the one it was registered for. There is no local-dev workaround short of a token registered for a local/staging origin.
- GovMap returns all geometry in ITM (meters), not WGS84 (degrees) — plotting it directly on any standard map library without reprojecting first produces coordinates off by orders of magnitude, not a subtly-wrong result.
- GovMap's own settlement naming can differ from a commonly-used short form for a compound municipality (e.g. a two-word official city name vs. the shorter name people normally use) — a location search using only the short form can come back empty even though the settlement exists under its full official name. Prefer GovMap's own settlement search result (which returns its canonical name) as the fallback anchor rather than guessing you've covered every naming variant.
- The widget only exposes geometry through a name-matched `search` → `getSearchResultData` two-step, not through the location/attribute query — budget for one extra round-trip per entity when planning rate limits/delays.
- Because grid-based enumeration is a fixed offset pattern, a very large or irregularly-shaped city may not be fully covered by one run — a documented limitation, not a bug; re-run centered on an uncovered area if needed.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace, which enumerates neighborhood boundary geometry to drive zone-based record filtering, with the token origin-locked to its deployed production domain.
