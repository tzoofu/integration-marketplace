# PWA Web App Manifest

- **category**: other
- **provider**: Web Platform (browser-native — Chrome/Edge/Android "Add to Home Screen", installable PWA)
- **reusable**: yes — the Next.js `app/manifest.ts` `MetadataRoute.Manifest` pattern (shape, icon/shortcut structure) is trivially portable to any Next.js App Router project; only the actual content (name, colors, shortcuts) is app-specific.
- **docs**: https://developer.mozilla.org/en-US/docs/Web/Manifest and https://nextjs.org/docs/app/api-reference/file-conventions/metadata/manifest

## Overview
Makes the app installable as a standalone app (own window, no browser chrome, home-screen icon) via a Web App Manifest, using Next.js's built-in `app/manifest.ts` route convention instead of a static `public/manifest.json`. This provides installability only — no offline caching or background sync, which would require a separate service worker.

## Playbook

### Prerequisites
- Served over HTTPS (or `localhost`).
- App icons at minimum 192x192 and 512x512 PNG (square, no transparency issues for maskable use if desired).

### Setup steps
1. Create `app/manifest.ts` (or `.js`) at the app root; Next.js automatically serves it at `/manifest.webmanifest` and links it from the page `<head>` — no manual `<link rel="manifest">` tag needed.
2. Export a default function returning a `MetadataRoute.Manifest` object with `name`, `short_name`, `description`, `start_url`, `scope`, `display: "standalone"`, `background_color`, `theme_color`.
3. Add `lang`/`dir` if the app is RTL or non-English.
4. Provide `icons` at 192x192 and 512x512 minimum, referencing static files served from `public/`.
5. Optionally add `shortcuts` (long-press/right-click quick actions on the installed icon) pointing at specific in-app routes.
6. No service worker is required for installability alone — add one separately only if offline caching/background sync is also needed.

### Core pattern
```ts
// app/manifest.ts — Next.js App Router manifest route convention
import type { MetadataRoute } from "next";

export default function manifest(): MetadataRoute.Manifest {
  return {
    id: "/",
    name: "My App",
    short_name: "MyApp",
    description: "Short description of what the app does",
    start_url: "/",
    scope: "/",
    display: "standalone",
    background_color: "#ffffff",
    theme_color: "#000000",
    icons: [
      { src: "/icon-192.png", sizes: "192x192", type: "image/png", purpose: "any" },
      { src: "/icon-512.png", sizes: "512x512", type: "image/png", purpose: "any" },
    ],
    shortcuts: [
      {
        name: "Quick action name",
        short_name: "Short name",
        url: "/some/route",
        description: "What this shortcut does",
      },
    ],
  };
}
```

### Env vars
none — static manifest content, no configuration or keys required.

### Gotchas
- Next.js's `app/manifest.ts` convention replaces `public/manifest.json` entirely — don't ship both, and don't manually add a `<link rel="manifest">` tag; Next wires it up automatically.
- Installability alone (via just the manifest) does **not** provide offline support — a separate service worker is required for that, and its absence is easy to mistake for "PWA support is broken" when actually only offline caching was never implemented.
- Icon `purpose: "any"` vs `"maskable"` matters on Android — a maskable icon needs extra safe-area padding baked into the image or Android's adaptive-icon mask will crop the actual artwork.
- `shortcuts` entries only appear after the app is installed and only on platforms that support the feature (mainly Android/Chrome OS) — don't rely on them as a discoverable UI affordance in the browser tab.
- `start_url` and `scope` should typically match (`/`) unless intentionally scoping the installed app to a sub-path — a mismatch can cause the installed app to unexpectedly "escape" into the browser chrome when navigating outside `scope`.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace, to make an app installable as a standalone app (own window, no browser chrome) with app icons and a couple of quick-action shortcuts to in-app routes.
