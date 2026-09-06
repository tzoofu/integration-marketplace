# Google AdSense

- **category**: other
- **provider**: Google
- **reusable**: yes — the admin-gated per-slot placement config plus the "only count filled slots" `<ins>` pattern is a clean, portable pattern; single-repo so far.
- **docs**: https://developers.google.com/adsense/platforms/direct-implementation

## Overview
Displays Google AdSense ad units in configurable placements across the app. A global script (`adsbygoogle.js`) is loaded once with the publisher's client ID, and individual ad slots are rendered as `<ins class="adsbygoogle">` elements that AdSense fills asynchronously. Placement configuration (which slots exist, their labels, and per-slot/global enable state) is admin-managed and stored server-side rather than hardcoded.

## Playbook

### Prerequisites
- A Google AdSense account, approved for the site's domain.
- A publisher client ID (`ca-pub-XXXXXXXXXXXXXXXX`) and one or more ad slot IDs created in the AdSense dashboard.
- `ads.txt` published at the site root declaring the same publisher ID (required by AdSense to prevent unauthorized inventory reselling).

### Setup steps
1. Create an AdSense account and get it approved for the domain; note the publisher client ID.
2. Create ad units in the AdSense dashboard and note each unit's slot ID.
3. Publish `public/ads.txt` (or equivalent static-file root) with the line: `google.com, pub-<PUBLISHER_ID>, DIRECT, f08c47fec0942fa0`.
4. Set the publisher client ID as a public (client-exposed) env var.
5. Load the AdSense loader script once near the root of the app (e.g. in the root layout), gated by a config flag so it can be globally killed without a redeploy.
6. Build a reusable ad-slot component that pushes to `window.adsbygoogle` on mount and only reserves visible space once AdSense actually reports the slot as filled — otherwise it collapses to zero height, avoiding empty ad-shaped gaps for blocked/unfilled slots.
7. Store placement config (slot IDs, per-placement enabled flags, and a global on/off switch) server-side, editable via an admin-only UI/API, so ad placement can be changed without a code deploy.

### Core pattern
```tsx
// AdsenseLoader.tsx — loads the AdSense script once, gated by a config flag
"use client";
import Script from "next/script";

const ADSENSE_CLIENT_ID = process.env.NEXT_PUBLIC_ADSENSE_CLIENT_ID;

export default function AdsenseLoader({ enabled }: { enabled: boolean }) {
  if (!enabled || !ADSENSE_CLIENT_ID) return null;
  return (
    <Script
      src={`https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=${ADSENSE_CLIENT_ID}`}
      strategy="afterInteractive"
      crossOrigin="anonymous"
      onError={() => {/* likely blocked by an ad blocker — safe to ignore */}}
    />
  );
}
```

```tsx
// AdSlot.tsx — a single ad unit that only takes up space once actually filled
"use client";
import { useEffect, useRef, useState } from "react";

declare global { interface Window { adsbygoogle?: object[] } }

function AdSlot({ slot, format = "auto" }: { slot: string; format?: string }) {
  const insRef = useRef<HTMLModElement>(null);
  const [filled, setFilled] = useState(false);

  useEffect(() => {
    const ins = insRef.current;
    if (!ins || ins.dataset.adsbygoogleStatus) return;
    try { (window.adsbygoogle = window.adsbygoogle || []).push({}); } catch { /* noop */ }
  }, []);

  useEffect(() => {
    const ins = insRef.current;
    if (!ins) return;
    // A responsive slot reserves height before AdSense decides whether to fill it,
    // so height alone isn't proof of a real ad — only "done" + real height counts.
    const evaluate = () => setFilled(ins.dataset.adsbygoogleStatus === "done" && ins.offsetHeight > 0);
    const resizeObs = new ResizeObserver(evaluate);
    resizeObs.observe(ins);
    const mutObs = new MutationObserver(evaluate);
    mutObs.observe(ins, { attributes: true, attributeFilter: ["data-adsbygoogle-status"] });
    return () => { resizeObs.disconnect(); mutObs.disconnect(); };
  }, []);

  return (
    <div className={filled ? "min-h-[100px]" : "h-0 overflow-hidden"}>
      <ins
        ref={insRef}
        className="adsbygoogle"
        style={{ display: "block" }}
        data-ad-client={process.env.NEXT_PUBLIC_ADSENSE_CLIENT_ID}
        data-ad-slot={slot}
        data-ad-format={format}
        data-full-width-responsive="true"
      />
    </div>
  );
}
```

```ts
// Admin-managed placement config shape (persisted server-side)
type AdPlacement = { id: string; label: string; slotId: string; enabled: boolean };
type AdsConfig = { placements: AdPlacement[]; enabled: boolean }; // `enabled` = global kill switch
```

### Env vars
`NEXT_PUBLIC_ADSENSE_CLIENT_ID` (or equivalent client-exposed var holding the AdSense publisher ID)

### Gotchas
- Give each rendered ad slot a stable-but-unique React `key` (e.g. `${pathname}:${slotId}`) so it remounts (and re-pushes to `adsbygoogle`) on route changes instead of silently going stale.
- A responsive `<ins>` element reserves layout height before AdSense decides whether to fill the slot — checking `offsetHeight > 0` alone is not proof of a real filled ad; combine it with `data-adsbygoogle-status === "done"` (watched via `MutationObserver`) to avoid ad-shaped empty boxes for blocked/unfilled inventory.
- `adsbygoogle.js` throws/fails silently when blocked by an ad blocker — the `Script` `onError` should be a no-op, not a hard failure, since the rest of the app must keep working with ads blocked.
- Keep a global `enabled` kill switch independent of per-placement `enabled` flags, so ads can be disabled site-wide instantly (e.g. for a policy issue) without touching every placement.
- `ads.txt` must exactly match the real publisher ID or AdSense will flag the inventory as unauthorized and may reduce/stop fill.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace, with ad units rendered in configurable placements gated by an admin-controlled per-slot enable/disable config.
