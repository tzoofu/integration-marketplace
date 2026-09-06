# Free Map Tile Providers

- **category**: maps-geo
- **provider**: OpenFreeMap, OpenStreetMap raster tiles, Esri World Imagery, CARTO, OpenTopoMap, MapLibre demo glyph server (mix varies by repo)
- **reusable**: yes — no auth/API key involved in either repo; candidate for a shared `@marketplace/free-map-tiles` style-config package.
- **docs**: https://maplibre.org/maplibre-gl-js/docs/ , https://openfreemap.org/ , https://wiki.openstreetmap.org/wiki/Raster_tile_providers

## Overview
Map rendering (via MapLibre GL, through `react-map-gl`) using free, keyless vector/raster tile styles instead of a paid maps API (no Google Maps/Mapbox API key). One repo layers in OpenFreeMap vector styles and 3D building extrusions on top of the raster basemaps; the simpler repo uses raster-only styles.

## Playbook

### Prerequisites
- `maplibre-gl` + `react-map-gl` (the `/maplibre` entry point) as dependencies. No account or API key for any of the tile sources used.
- Awareness that keyless raster tile providers can silently start watermarking/rate-limiting without an HTTP-level error — spot-check a rendered tile visually, don't just check response status.

### Setup steps
1. Define a `StyleSpecification` (MapLibre style JSON) per "map type" your UI offers (e.g. streets / satellite / minimal / topo), each pointing at one raster or vector tile source. Keep this as a single exported `Record<MapType, StyleSpecification | string>` so every map instance in the app shares one source of truth instead of hand-copying tile URLs.
2. For raster sources, set `tileSize: 256` and the source's own attribution string (required by most of these providers' free-tier terms).
3. For a vector basemap served by URL (e.g. OpenFreeMap's hosted styles), pass the style URL directly as `mapStyle` rather than an inline spec — `react-map-gl`'s `mapStyle` prop accepts either.
4. If you need Hebrew/Arabic (or any RTL-script) map labels, register MapLibre's RTL text plugin once at module load, eagerly (not lazily) — see Gotchas for why.
5. For an optional 3D-buildings layer, add a `vector` source pointing at a build-extrusion-compatible vector tileset (OpenFreeMap's `planet` endpoint works, using OMT-compatible `source-layer: "building"`) and a `fill-extrusion` layer keyed off a footprint building-height property, with a fallback constant when that property is absent.
6. Support light/dark by swapping which style is active on the "streets" map type only (a raster OSM style has no native dark variant, so switch to a vector dark-styled URL instead); other map types (satellite/topo) don't need a dark variant.
7. Add a small custom control (a single "layers" button that expands to a mapType picker + a north-reset + a 3D toggle) rather than MapLibre's default navigation control, if you want tighter visual control over the basemap switcher UI.

### Core pattern
```ts
// map-tile-styles.ts — generic keyless MapLibre style config
import type { StyleSpecification } from "maplibre-gl";

export type MapType = "streets" | "satellite" | "minimal" | "topo";

// Public MapLibre demo glyph server — no key required, needed for any symbol
// (text label) layer even on raster-only styles.
const GLYPHS_URL = "https://demotiles.maplibre.org/font/{fontstack}/{range}.pbf";

const STREETS_STYLE: StyleSpecification = {
  version: 8,
  glyphs: GLYPHS_URL,
  sources: {
    osm: {
      type: "raster",
      tiles: ["https://a.tile.openstreetmap.org/{z}/{x}/{y}.png"],
      tileSize: 256,
      attribution: "© OpenStreetMap contributors",
    },
  },
  layers: [{ id: "osm", type: "raster", source: "osm" }],
};

const SATELLITE_STYLE: StyleSpecification = {
  version: 8,
  glyphs: GLYPHS_URL,
  sources: {
    esri: {
      type: "raster",
      // Esri uses {z}/{y}/{x} path order (row before col) — note the swap
      // relative to the {z}/{x}/{y} convention most other providers use.
      tiles: ["https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}"],
      tileSize: 256,
      maxzoom: 19,
      attribution: "© Esri, Maxar, Earthstar Geographics",
    },
  },
  layers: [{ id: "esri", type: "raster", source: "esri" }],
};

const TOPO_STYLE: StyleSpecification = {
  version: 8,
  glyphs: GLYPHS_URL,
  sources: {
    topo: {
      type: "raster",
      tiles: ["https://tile.opentopomap.org/{z}/{x}/{y}.png"],
      tileSize: 256,
      maxzoom: 17,
      attribution: "© OpenTopoMap contributors",
    },
  },
  layers: [{ id: "topo", type: "raster", source: "topo" }],
};

// Vector basemaps — pass the URL straight to `mapStyle`, don't try to inline it.
const MINIMAL_STYLE_URL = "https://tiles.openfreemap.org/styles/positron";
const DARK_STYLE_URL = "https://tiles.openfreemap.org/styles/dark";

export const MAP_STYLES: Record<MapType, StyleSpecification | string> = {
  streets: STREETS_STYLE,
  satellite: SATELLITE_STYLE,
  minimal: MINIMAL_STYLE_URL,
  topo: TOPO_STYLE,
};

// "streets" is the only type with a dark variant (a vector style swap);
// other map types have no native dark rendering.
export function resolveStyle(type: MapType, dark: boolean): StyleSpecification | string {
  if (type === "streets" && dark) return DARK_STYLE_URL;
  return MAP_STYLES[type];
}

// One-time RTL text plugin registration — call this once at module load in a
// client-only file, eagerly (not lazily): shaping a GeoJSON symbol layer
// before the plugin resolves can bake in mirrored glyphs permanently for
// that session, even after the plugin finishes loading.
export function registerRtlTextPluginOnce(): void {
  // import { setRTLTextPlugin, getRTLTextPluginStatus } from "maplibre-gl";
  // if (typeof window !== "undefined" && getRTLTextPluginStatus() === "unavailable") {
  //   setRTLTextPlugin(
  //     "https://unpkg.com/@mapbox/mapbox-gl-rtl-text@0.3.0/dist/mapbox-gl-rtl-text.js",
  //     false // lazy: false — see comment above
  //   );
  // }
}
```

### Env vars
None — every tile source used is keyless/public.

### Gotchas
- A keyless raster provider can start defacing tiles (e.g. a diagonal "API KEY REQUIRED" watermark) while still returning HTTP 200 with a structurally valid image — an HTTP status check alone won't catch this. Spot-check a rendered tile visually whenever you add or change a keyless raster basemap.
- Esri's World Imagery tile URL uses `{z}/{y}/{x}` path order, not the usual `{z}/{x}/{y}` — easy to transpose by habit and get a garbled/misaligned satellite layer.
- Register the RTL text plugin eagerly, not lazily, if you show non-Latin-script (Hebrew/Arabic) labels on a GeoJSON-backed symbol layer: the map can shape that layer into its internal tile representation before a lazily-loaded plugin resolves, and the already-shaped (wrong-direction) glyphs get cached for the rest of that session — a page reload is needed to fix it, not just waiting.
- If you expose a basemap switcher, export the full style map (and any custom layer/control definitions like a 3D-buildings layer) as shared constants from one module — multiple map instances hand-copying the same tile URLs/control styling will drift out of sync over time.
- WebGL context loss (mobile memory pressure, backgrounding) is a real failure mode with MapLibre — listen for `webglcontextlost`/`webglcontextrestored` on the map instance and show a retry affordance (remount the map component) rather than leaving a dead canvas.

### Playbook confidence: high

## Adoption
Used in **2** repos in this marketplace. One adopter renders an interactive point-marker map with clustering (via a clustering library) over these free tile styles; the other renders an admin-only map for drawing and editing geographic zone polygons, using only the raster layers.
