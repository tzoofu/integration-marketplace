# First-Run Spotlight Onboarding Tour

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — a dependency-free, data-driven spotlight tour (pure step/placement logic + one overlay component + one persisted per-account flag) that drops into any React app with a dashboard-style main screen.
- **docs**: n/a — internal pattern, no external provider.

A once-per-account guided tour that dims the screen, cuts a spotlight around a real UI control, and shows a card explaining it. Steps are plain data, target resolution and card placement are pure functions (unit-testable without a DOM), and "already seen" lives on the user's server-side profile so it never re-fires on a new device or after clearing storage.

## Playbook

### Prerequisites
- React 18+ (any framework; the example is Next.js App Router, client components).
- A per-user profile record you can PATCH (the "seen" flag lives there, not in `localStorage`).
- A media-query hook (`useMediaQuery`) and an analytics `trackEvent` helper (optional).
- A stable layout where the controls you want to point at are already rendered when the tour starts.

### Setup steps
1. **Tag spotlight targets in the real UI** with `data-onboarding="<id>"` on the actual control (tab bar, filter button, primary card, add button, notification bell…). Ids are the contract between the tour data and the markup.
2. **Write the steps as pure data** (`onboarding-steps.ts`): `id`, `title`, `body`, and `targets` listed **separately for mobile and desktop**, each a priority-ordered list of ids. Allow per-layout body copy (gestures differ: swipe vs. keyboard). Empty target list = centered card, used for intro/closing steps.
3. **Resolve the target by measuring, not by existence.** `resolveTarget` walks the layout's candidate ids and returns the first whose bounding box is larger than 0×0. A `hidden md:flex` element exists in the DOM but measures 0×0 — checking only for presence spotlights the top-left corner on the other layout.
4. **Place the card with a pure function** (`placeCard(rect, cardSize, viewport)`): below the target if it fits, else above, else pinned near the bottom with no arrow (target taller than free space, e.g. a full-height card). Reserve a `bottomInset` on mobile for a fixed tab bar. Clamp horizontally to the viewport; compute the arrow x relative to the card.
5. **Build the overlay component**: full-screen fixed container (`role="dialog"`, `aria-modal`, `aria-labelledby`), one absolutely-positioned div over the target whose giant `box-shadow: 0 0 0 9999px rgba(..)` dims everything else (simpler and more robust than an SVG mask for a single rectangle), and a card. Measure the card *after layout* (`useLayoutEffect`) because its height varies with copy and demos, and hide it (`invisible`) until placed.
6. **Keep the spotlight glued**: on step change `scrollIntoView({ block: "nearest" })` the target once, then re-measure on `resize` and on `scroll` (capture phase, so nested scroll containers count).
7. **State hook** (`useOnboarding(enabled, hasSeen)`): `step` is `0` when inactive, `1..N` while active. Start it in an effect when `enabled && !hasSeen`. Expose `next / prev / skip / finish`. Gate `enabled` off in read-only/preview modes.
8. **Persist on finish *and* skip**: `PATCH` the profile with `hasSeen: true`, fire-and-forget. Read the flag in the server component and pass it down as a prop so there is no client fetch and no flash.
9. **Add a replay path**: a settings button (and optionally a help page) that PATCHes `hasSeen: false` then does a hard `window.location.href = "/dashboard"` so the server component re-reads the profile.
10. **Optional closing question**: a final target-less step with Yes/No buttons that persists a preference in the *same* PATCH as the seen flag.
11. **Render conditionally** in the main screen — only when `step > 0` and no higher-priority modal (post-action prompts, debriefs) is open, so two overlays never stack.
12. **Tests**: unit-test `resolveTarget` (candidate order, 0×0 skip, target-less step) and `placeCard` (below / above / pinned / horizontal clamp / centered). Add a check that every id used in the steps appears as a `data-onboarding` attribute in source — otherwise a rename silently degrades a step to a centered card.

### Core pattern
```ts
// onboarding-steps.ts — pure, no DOM
export interface OnboardingStep {
  id: string;
  targets: { mobile: string[]; desktop: string[] };   // data-onboarding ids, priority order
  title: string;
  body: { mobile: string; desktop: string };
  closing?: boolean;                                   // final yes/no question step
}
export interface TargetRect { top: number; left: number; width: number; height: number }

export function resolveTarget(
  step: OnboardingStep,
  isDesktop: boolean,
  measure: (id: string) => TargetRect | null,
) {
  for (const id of isDesktop ? step.targets.desktop : step.targets.mobile) {
    const rect = measure(id);
    if (rect && rect.width > 0 && rect.height > 0) return { id, rect };
  }
  return null;
}

export function placeCard(
  rect: TargetRect | null,
  card: { width: number; height: number },
  vp: { width: number; height: number; bottomInset: number },
) {
  const gap = 14, margin = 12;
  const clampLeft = (x: number) => Math.min(Math.max(margin, x), vp.width - card.width - margin);
  if (!rect) {
    return { top: Math.max(margin, (vp.height - card.height) / 2),
             left: clampLeft((vp.width - card.width) / 2), arrow: null };
  }
  const cx = rect.left + rect.width / 2;
  const left = clampLeft(cx - card.width / 2);
  const arrowX = Math.min(Math.max(20, cx - left), card.width - 20);
  const below = rect.top + rect.height + gap;
  if (below + card.height <= vp.height - vp.bottomInset - margin)
    return { top: below, left, arrow: { side: "top" as const, x: arrowX } };
  const above = rect.top - gap - card.height;
  if (above >= margin) return { top: above, left, arrow: { side: "bottom" as const, x: arrowX } };
  return { top: Math.max(margin, vp.height - vp.bottomInset - card.height - margin),
           left: clampLeft((vp.width - card.width) / 2), arrow: null };
}
```

```ts
// useOnboarding.ts
let seenThisSession = false; // survives client-side nav; a stale cached server payload can't re-show the tour

export function useOnboarding(enabled: boolean, hasSeen: boolean) {
  const [step, setStep] = useState(0);
  useEffect(() => { if (enabled && !hasSeen && !seenThisSession) setStep(1); }, [enabled, hasSeen]);

  const markSeen = (extra: object = {}) => {
    seenThisSession = true;
    fetch("/api/me/preferences", {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ hasSeenOnboarding: true, ...extra }),
    }).catch(() => {});
  };

  // Track analytics from the closure, never inside a setState updater (StrictMode double-invokes updaters).
  const next = () => {
    track("onboarding_step", { step, step_id: STEPS[step - 1]?.id, action: step >= TOTAL ? "finish" : "next" });
    if (step >= TOTAL) { markSeen(); setStep(0); } else setStep(step + 1);
  };
  const prev = () => setStep(Math.max(1, step - 1));
  const skip = () => { track("onboarding_step", { step, action: "skip" }); markSeen(); setStep(0); };
  const answer = (enabled: boolean) => { markSeen({ someFeatureEnabled: enabled }); setStep(0); };
  return { step, next, prev, skip, answer };
}
```

```tsx
// OnboardingWalkthrough.tsx (essentials)
const measure = (id: string) => {
  const el = document.querySelector(`[data-onboarding="${id}"]`);
  if (!el) return null;
  const r = el.getBoundingClientRect();
  return { top: r.top, left: r.left, width: r.width, height: r.height };
};

// 1) re-measure on step/resize/scroll(capture)   2) useLayoutEffect: placeCard(rect, card.offset*, viewport)
// 3) capture-phase keydown so the app's own arrow-key shortcuts don't fire underneath the tour
<div className="fixed inset-0 z-[9998]" role="dialog" aria-modal="true" aria-labelledby="tour-title">
  <div className="absolute rounded-2xl ring-2 ring-primary transition-all duration-300 motion-reduce:transition-none"
       style={spot ? { ...spot, boxShadow: "0 0 0 9999px rgba(15,17,19,.62)" } : { inset: 0, boxShadow: "0 0 0 9999px rgba(15,17,19,.62)" }} />
  <div ref={cardRef} tabIndex={-1} aria-live="polite"
       className={cn("absolute w-[min(340px,calc(100vw-24px))] rounded-2xl border bg-card p-4", !placement && "invisible")}
       style={{ top: placement?.top ?? 0, left: placement?.left ?? 0 }}>
    {/* "n of N", skip, title, body, optional demo, progress dots, Back / Next (or Yes / No on the closing step) */}
  </div>
</div>
```

```ts
// Server side — accept the flag in the existing preferences PATCH route
if ("hasSeenOnboarding" in body) update.hasSeenOnboarding = body.hasSeenOnboarding === true;

// Server component: pass the flag as a plain prop (no client fetch, no flash)
<Dashboard hasSeenOnboarding={profile?.hasSeenOnboarding === true} />
```

**Optional demos inside cards**: drive tiny looping illustrations with the Web Animations API (`el.animate(keyframes, { iterations: Infinity })`, cancel on unmount) and skip them entirely under `prefers-reduced-motion` so the static first frame remains. For "list/grouping" demos, render from the same constants the real UI uses so the illustration can't drift.

### Env vars
none — no external services; analytics is optional.

### Gotchas
- **Check size, not existence.** Responsive-hidden elements still exist in the DOM with a 0×0 box; presence-only checks spotlight the viewport corner on the other layout. Keep separate mobile/desktop target lists.
- **Renaming/removing a target id silently degrades its step** to a centered card — no build error. Add a test that every id in the step data appears in source.
- **Persist server-side, not in `localStorage`.** A browser-scoped flag re-shows the tour on every new device/profile. When migrating from a localStorage flag, run a one-time backfill setting the flag `true` on all existing users so nobody who already dismissed it sees it again. Also keep a script to reset the flag for QA.
- **Stale router cache can re-fire the tour.** A client-side nav away and back can reuse a server payload rendered before the PATCH landed; the module-level `seenThisSession` guard prevents the flash. A hard reload always gets the fresh value.
- **Never fire analytics inside a `setState` updater** — StrictMode invokes updaters twice and double-counts events.
- **Measure the card after layout.** Its height depends on copy and demos; place it in `useLayoutEffect` and keep it `invisible` until the first placement to avoid a visible jump.
- **Re-measure on scroll with the capture flag** (`addEventListener("scroll", fn, true)`) or targets inside nested scroll containers drift away from the spotlight.
- **Swallow the app's own keyboard shortcuts** with a capture-phase `keydown` handler that calls `stopPropagation`, otherwise arrow-key vote/navigation shortcuts fire underneath the overlay. In RTL, ← means "next".
- **Reserve space for fixed bottom chrome** (`bottomInset`) so the card never lands under a mobile tab bar; when the target fills the screen, pin the card with no arrow rather than covering the target.
- **Don't stack overlays.** Suppress the tour while any post-action modal is open, and disable it in read-only/preview modes where the user can't act on the steps anyway.
- **Hard-navigate on replay** (`window.location.href`), not a client router push — only a full navigation re-runs the server component that reads the flag.
- **Silent PATCH failures** (`.catch(() => {})`) mean the tour can return on the next hard load; acceptable, but know it is a deliberate trade-off.
- **Docs drift.** Every UI change that moves or renames a tagged control must also update the tour data, the help/guide page copy, and first-run/empty screens; keep that as a written project rule.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter pairs the tour with a public, data-driven help page (inline `[[label]]` chips for on-screen controls) that expands on the same copy, reuses the tour's demo components on its public landing page, and ends the tour with a Yes/No preference question persisted in the same request as the seen flag.
