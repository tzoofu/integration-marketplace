# Google Analytics 4

- **category**: analytics
- **provider**: Google
- **reusable**: yes — most repos use a thin `trackEvent()` wrapper (or raw `gtag.js`) around Google Analytics; one adopter additionally has a server-side Measurement Protocol sender the others lack. Candidate for `@marketplace/ga4` (client + optional server variant).
- **docs**: https://developers.google.com/analytics/devguides/collection/ga4

## Overview
Client-side page/event analytics via GA4. Three distinct integration shapes appear across repos: raw `gtag.js` loaded manually, `@next/third-parties/google`'s `<GoogleAnalytics>` helper (Next.js-specific, recommended when available), and Firebase Analytics' `firebase/analytics` SDK (when the repo already has a Firebase project, since a GA4 property is auto-created alongside it). All converge on the same app-level convention: a single `trackEvent(name, params)` wrapper that every component calls, never `gtag`/`logEvent`/`sendGAEvent` directly.

## Playbook

### Prerequisites
- A GA4 property + Measurement ID (`G-XXXXXXX...`), from Google Analytics admin — or, if the repo already has Firebase, the Measurement ID auto-provisioned under that Firebase project's linked GA4 property (Firebase Console → Project Settings → Your apps → web app → gtag config).
- (Server-side sends only) a Measurement Protocol API secret, created in GA4 Admin → Data Streams → your stream → Measurement Protocol API secrets.

### Setup steps
1. Decide which client integration fits: **Next.js app** → use `@next/third-parties/google`'s `<GoogleAnalytics gaId={...} />` (handles the script tags + `dataLayer` init for you); **already using Firebase** → use `firebase/analytics`'s `getAnalytics()` + `logEvent()`; **anything else / no framework helper** → hand-roll the two `<script>` tags for `gtag.js`.
2. Mount the analytics provider once, high in the tree (root layout / `_app`), gated on the env var being set — render nothing if it's missing, so local dev without a real Measurement ID doesn't error or spam warnings.
3. Write one `trackEvent(name, params)` wrapper function and export it as the *only* sanctioned way to fire events. Guard it so it's a safe no-op: on the server (`typeof window === "undefined"`), before the provider has mounted, and when the env var isn't configured.
4. Call `trackEvent()` fire-and-forget from UI event handlers and key lifecycle points (page view fires automatically via the provider; custom funnel/business events you fire yourself).
5. (Optional) document your event catalog — name, params, firing location — in one markdown file so events don't drift or get redefined ad hoc (one adopter maintains this as a dedicated `ANALYTICS.md`).
6. (Optional, server-side) for events that happen in server code with no browser context (e.g. an MCP tool call, a webhook), send directly to the GA4 Measurement Protocol collect endpoint with a `measurement_id` + `api_secret`, using a stable `client_id` (e.g. the user's email/uid — anything consistent per-actor).

### Core pattern

**Next.js app using `@next/third-parties/google` (recommended when on Next.js)**
```tsx
"use client";
import { GoogleAnalytics } from "@next/third-parties/google";

const GA_ID = process.env.NEXT_PUBLIC_GA_ID;

export function AnalyticsProvider() {
  if (!GA_ID) return null;
  return <GoogleAnalytics gaId={GA_ID} />;
}
```
```ts
import { sendGAEvent } from "@next/third-parties/google";

export function trackEvent(name: string, params?: Record<string, string | number | boolean | undefined>): void {
  // dataLayer only exists after <GoogleAnalytics> has mounted client-side —
  // guard on it rather than calling sendGAEvent unconditionally, which
  // console.warns on every call made before mount (or with no GA_ID at all).
  if (typeof window === "undefined" || !Array.isArray((window as any).dataLayer)) return;
  sendGAEvent("event", name, params ?? {});
}
```

**Firebase Analytics variant (when the repo already has a Firebase client SDK)**
```ts
"use client";
import { getAnalytics, logEvent, isSupported } from "firebase/analytics";

let analyticsPromise: Promise<import("firebase/analytics").Analytics | null> | null = null;

function getAnalyticsClient() {
  if (typeof window === "undefined") return Promise.resolve(null);
  if (!analyticsPromise) {
    analyticsPromise = isSupported()
      .then((supported) => (supported ? getAnalytics(getFirebaseApp()) : null))
      .catch(() => null);
  }
  return analyticsPromise;
}

export function trackEvent(name: string, params?: Record<string, unknown>): void {
  getAnalyticsClient().then((a) => { if (a) logEvent(a, name, params); });
}
```

**Raw `gtag.js` (no framework helper, e.g. non-Next.js or a helper-free setup)**
```tsx
import Script from "next/script"; // or a plain <script> tag outside Next.js

const GA_ID = process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID;

export function Analytics() {
  if (!GA_ID) return null;
  return (
    <>
      <Script src={`https://www.googletagmanager.com/gtag/js?id=${GA_ID}`} strategy="afterInteractive" />
      <Script id="ga-init" strategy="afterInteractive">
        {`window.dataLayer = window.dataLayer || [];
          function gtag(){dataLayer.push(arguments);}
          gtag('js', new Date());
          gtag('config', '${GA_ID}');`}
      </Script>
    </>
  );
}

export function trackEvent(eventName: string, params?: Record<string, string | number | boolean | undefined>) {
  if (typeof window === "undefined" || !(window as any).gtag) return;
  (window as any).gtag("event", eventName, params);
}
```

**Server-side send via GA4 Measurement Protocol (for events with no browser context)**
```ts
const MEASUREMENT_ID = process.env.NEXT_PUBLIC_GA_MEASUREMENT_ID;
const API_SECRET = process.env.GA_MP_API_SECRET;

export async function trackServerEvent(eventName: string, clientId: string, params?: Record<string, unknown>): Promise<void> {
  if (!MEASUREMENT_ID || !API_SECRET) return;
  try {
    await fetch(
      `https://www.google-analytics.com/mp/collect?measurement_id=${MEASUREMENT_ID}&api_secret=${API_SECRET}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ client_id: clientId, events: [{ name: eventName, params }] }),
      }
    );
  } catch {
    // analytics failures must never interrupt the actual server operation
  }
}
```

### Env vars
- `NEXT_PUBLIC_GA_MEASUREMENT_ID` or `NEXT_PUBLIC_GA_ID` (naming varies by repo — same concept, the GA4 Measurement ID)
- `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID` (when riding on a Firebase project's auto-linked GA4 property instead of a standalone one)
- `GA_MP_API_SECRET` (server-side Measurement Protocol sends only — never exposed as `NEXT_PUBLIC_*`)

### Gotchas
- Never call `logEvent`/`gtag`/`sendGAEvent` directly from feature code — always go through one `trackEvent()` wrapper, so the null-checks, async/support-detection, and "is analytics even configured" logic live in exactly one place instead of being re-implemented (and inevitably drifting) at every call site.
- `sendGAEvent` (from `@next/third-parties/google`) `console.warn`s on every call made before `<GoogleAnalytics>` has mounted — guard on `window.dataLayer` existing rather than calling it unconditionally, or local dev without a Measurement ID gets console spam.
- `trackEvent()` calls should be fire-and-forget, never awaited — a slow or blocked analytics request (ad blockers routinely block `google-analytics.com` and `googletagmanager.com`) must never delay or fail the actual user action it's attached to.
- Never pass free-text user-entered content as an event param (raw search queries, user-typed names/notes) — only enum-like strings, booleans, counts, and non-sensitive IDs. GA4 params can end up in exportable reports.
- If a repo widens tracking to include unauthenticated/anonymous visitors (not just signed-in users), that's a deliberate product decision with privacy/consent implications — don't silently change tracking scope on a "just make it consistent with the other repos" pass.
- The Firebase Analytics variant and the standalone GA4 variant use different Measurement IDs conceptually (one is auto-provisioned under a Firebase project, the other is a bare GA4 property) — don't assume they're interchangeable env var names across repos.

### Playbook confidence: high

## Adoption
Used in **4** repo(s) in this marketplace. All converge on a single `trackEvent()`-style wrapper, but the underlying transport varies: raw `gtag.js`, the Next.js-specific `@next/third-parties/google` helper, and Firebase Analytics' SDK (riding on an existing Firebase project's auto-linked GA4 property) all appear. One adopter also sends server-side events with no browser context through the GA4 Measurement Protocol endpoint, and one tracks all visitors (signed in or not) rather than only authenticated users.
