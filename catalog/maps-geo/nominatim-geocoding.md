# OpenStreetMap / Nominatim (geocoding)

- **category**: maps-geo
- **provider**: OpenStreetMap Foundation (Nominatim public API)
- **reusable**: yes — generic, keyless geocoding; easy to lift wholesale into any repo that needs address → lat/lng.
- **docs**: https://nominatim.org/release-docs/latest/api/Search/

## Overview
Free-text and structured address geocoding (address string → `{ lat, lng }`) against OpenStreetMap's public Nominatim instance. No API key, but usage is governed by Nominatim's usage policy (a `User-Agent` identifying your app is effectively mandatory, and there's an informal rate limit — don't hammer it).

## Playbook

### Prerequisites
- None besides network access — no account, no key.
- A distinct `User-Agent` string identifying your app (Nominatim's usage policy expects this; requests without one are more likely to get throttled/blocked).
- Willingness to do a small amount of address cleanup yourself — Nominatim's match quality drops fast on addresses with extra descriptive text (parenthetical notes, cross-street phrases, proximity phrases like "near X").

### Setup steps
1. Call the public endpoint (`https://nominatim.openstreetmap.org/search`) with `format=json&limit=1`, always setting a real `User-Agent` header identifying your app, and `cache: "no-store"` (or equivalent) so you don't accidentally cache a stale/failed geocode result.
2. Try a **freeform** query first (`q=<cleaned address, city>`) — usually the best first attempt.
3. Fall back to a **structured** query (`street=`, `city=`, `country=`) when freeform comes up empty — structured queries are often more precise for a street + city combo, especially in non-Latin scripts.
4. Before querying, strip anything that hurts match quality: parenthetical notes, proximity phrases ("near X", "next to X" — take only the text before/after the phrase depending on which side has the actual address), cross-street indicators (take only the first of two streets), and comma-separated trailing segments beyond the first.
5. Restrict to the country you care about via `countrycodes=<cc>` to cut down on false positives from identically-named streets elsewhere in the world.
6. Layer in narrow, app-specific fallback heuristics only after the two generic attempts fail (e.g. a locale-specific compound-city-name variant, or stripping a language-specific prepositional prefix) — keep these as clearly-labeled, ordered fallback steps, not baked into the primary query.
7. Treat any non-OK response or an empty results array as "no match" (return `null`), not an exception — geocoding misses are routine, not exceptional.

### Core pattern
```ts
// geocode.ts — generic Nominatim geocoding wrapper with freeform + structured fallback
const NOMINATIM_URL = "https://nominatim.openstreetmap.org/search";
const USER_AGENT = "your-app-name/1.0"; // identify your app per Nominatim's usage policy

async function nominatimFreeform(query: string, countryCode: string): Promise<{ lat: number; lng: number } | null> {
  const url = `${NOMINATIM_URL}?q=${encodeURIComponent(query)}&format=json&limit=1&countrycodes=${countryCode}`;
  const res = await fetch(url, { headers: { "User-Agent": USER_AGENT }, cache: "no-store" });
  if (!res.ok) return null;
  const results = await res.json();
  if (!results.length) return null;
  return { lat: parseFloat(results[0].lat), lng: parseFloat(results[0].lon) };
}

async function nominatimStructured(
  street: string, city: string, countryCode: string
): Promise<{ lat: number; lng: number } | null> {
  const params = new URLSearchParams({ street, city, country: countryCode, format: "json", limit: "1" });
  const res = await fetch(`${NOMINATIM_URL}?${params}`, { headers: { "User-Agent": USER_AGENT }, cache: "no-store" });
  if (!res.ok) return null;
  const results = await res.json();
  if (!results.length) return null;
  return { lat: parseFloat(results[0].lat), lng: parseFloat(results[0].lon) };
}

/** Strips common noise (parentheticals, "near X" style proximity phrases,
 *  cross-street markers, trailing comma segments) that hurts match quality. */
function cleanAddress(address: string): string {
  let clean = address.replace(/\(.*?\)/g, "");
  // cross-streets / alternates: keep only the first
  clean = clean.split(/\s+(?:corner of|at)\s+/i)[0].split("/")[0];
  // first comma-segment only
  clean = clean.split(",")[0];
  return clean.replace(/\s{2,}/g, " ").trim();
}

export async function geocodeAddress(
  address: string | undefined,
  city: string | undefined,
  countryCode: string
): Promise<{ lat: number; lng: number } | null> {
  const cleaned = address ? cleanAddress(address) : undefined;
  const query = [cleaned, city].filter(Boolean).join(", ");
  if (!query) return null;

  try {
    let result = await nominatimFreeform(query, countryCode);
    if (result) return result;

    if (cleaned && city) {
      result = await nominatimStructured(cleaned, city, countryCode);
      if (result) return result;
    }
    return null;
  } catch {
    return null;
  }
}
```

### Env vars
None — the public Nominatim endpoint requires no API key, only a `User-Agent` request header.

### Gotchas
- Nominatim's public instance is a shared, rate-limited resource, not dedicated infrastructure — don't batch-geocode a large dataset without deliberate delays between requests, and expect occasional throttling under load. For high-volume production use, consider a self-hosted Nominatim instance or a paid geocoder instead.
- A missing/generic `User-Agent` header makes you more likely to be silently rate-limited or blocked — always set a real one identifying your app.
- Freeform vs. structured queries can each succeed where the other fails for the same address — always try both in sequence rather than picking one.
- Descriptive real-world address text (parentheticals, "near the church", cross-street references) actively hurts match rates — clean the address string before querying rather than passing raw user-entered text straight through.
- Always scope with `countrycodes` when you know the target country — otherwise an identically-named street in a different country can silently win the "closest match" and return a wildly wrong coordinate with no error to catch it.
- Treat empty results / non-OK responses as a normal "couldn't geocode this" outcome (return `null`) rather than throwing — real-world address data will always have some fraction that doesn't resolve.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace, which uses it both for record-level address geocoding and for resolving city-center coordinates as an input to a separate GIS lookup.
