# Waze Deep Link

- **category**: maps-geo
- **provider**: Waze / Google
- **reusable**: yes — single-repo so far, but an identical no-SDK, no-API-key deep-link pattern to `whatsapp-deeplink.md`; trivial to lift as a shared `@marketplace/waze-link` helper.
- **docs**: https://developers.google.com/waze/deeplinks

## Overview
One-click `waze.com/ul` deep links that open Waze directly into turn-by-turn navigation to an address (falling back to raw coordinates when no address is available) — no API key, no SDK, just a URL scheme.

## Playbook

### Prerequisites
- None — no account, key, or SDK. Works as a plain anchor link on any platform where Waze (or a browser fallback) can handle the URL.

### Setup steps
1. Prefer a free-text address query over raw coordinates when you have one — it matches what a user would actually recognize/expect, and Waze's own geocoding handles typical formatting variance.
2. Build the link as `https://waze.com/ul?q=<url-encoded address>&navigate=yes`. The `navigate=yes` parameter is what makes Waze start turn-by-turn immediately instead of just centering the map on the pin.
3. Fall back to `https://waze.com/ul?ll=<lat>,<lng>&navigate=yes` only when there's no usable address string (e.g. lat/lng-only data).
4. Return `null`/`undefined` when you have neither an address nor coordinates, and hide the navigation affordance entirely rather than rendering a dead link.
5. Render as a plain `<a href=...>` (optionally `target="_blank" rel="noopener noreferrer"` if opened from a web context) — no client library needed.

### Core pattern
```ts
// waze-link.ts — generic Waze deep-link builder
export function buildWazeHref(
  location: { addressLine?: string; coordinates?: { lat: number; lng: number } }
): string | null {
  if (location.addressLine) {
    return `https://waze.com/ul?q=${encodeURIComponent(location.addressLine)}&navigate=yes`;
  }
  if (location.coordinates) {
    return `https://waze.com/ul?ll=${location.coordinates.lat},${location.coordinates.lng}&navigate=yes`;
  }
  return null;
}
```

### Env vars
None.

### Gotchas
- Omitting `navigate=yes` silently degrades the experience to "open Waze and show this pin" instead of starting navigation — easy to miss in testing if you don't actually tap through to a turn-by-turn screen.
- An address string is generally more reliable/recognizable to the end user than raw coordinates and should be preferred whenever available — reserve the coordinate fallback for records that genuinely have no address text.
- No offline/app-install handling is implied by this pattern — on a device without Waze installed, behavior (app store redirect vs. web fallback) is entirely up to Waze's own web handler, not something this app controls.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace, which generates a navigation link for a listing's address (falling back to coordinates when no address is available) so users can start driving directions in one tap.
