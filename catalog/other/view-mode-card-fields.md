# View-Mode & Card Field System

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — the "URL-persisted view mode + per-view field-visibility registry shared across renderers" pattern is portable to any multi-layout listing app (list/grid/map).
- **docs**: n/a — internal pattern.

## Overview
A persisted, URL-driven toggle between multiple listing renderings (swipe deck / list / map), backed by a shared per-user, per-view configurable "which fields show on the card" preference system, so the layout isn't hardcoded per view.

## Playbook

### Prerequisites
- A router/URL layer that supports reading and writing query params without a full navigation (e.g. `window.history.replaceState` — NOT a full router push/replace, which would add a history entry per keystroke/toggle and fight back-navigation).
- A per-user preferences store (a `PATCH` endpoint + a settings table/column) if you want the field-visibility choice to persist across sessions; otherwise it can just live in memory/URL for the session.

### Setup steps
1. Add a `view` field to your filter/state object (e.g. `"swipe" | "list" | "map"`), defaulted to whichever view is most common, and read/write it via the URL query string using `history.replaceState` (never a router-level push/replace) — so the toggle is instant, shareable/bookmarkable mid-session, but doesn't spam browser history and doesn't need a server round-trip to persist.
2. Define a `FIELD_META` registry: one array of `{ key, label, slot, views }` where `slot` says how the field renders (`"fact"` = inline label:value, `"badge"` = pill, `"block"` = a bigger free-form section) and `views` says which layout(s) can show it at all (e.g. a map popup has less room than a full list card).
3. Define per-view `DEFAULTS: Record<View, string[]>` — the field keys shown out of the box for each view, before any user customization.
4. Write one `resolveFieldKeys(view, userPrefs)` function that intersects the user's chosen keys against what's actually valid for that view (defends against a saved preference becoming invalid after a field is removed/renamed), caps the result at a max count, and falls back to `DEFAULTS[view]` when the user has no saved preference.
5. Write one `buildCardSections(view, userPrefs, item, ctx)` renderer that turns the resolved field keys into React nodes grouped by slot (`facts`, `badges`, `blocks`) — shared identically by every card/list/deck component so they can never render a field differently between views. A view with less room (e.g. a map popup) can use a parallel plain-text variant of the same renderer instead of full React nodes.
6. Expose a settings UI (checkboxes per view, respecting the max-count cap) that `PATCH`es the sanitized preference object to the backend; sanitize on the server too (never trust the client blindly) by re-running the same allowed-keys-per-view + max-count logic.

### Core pattern
```ts
// display-fields.ts — the per-view field-visibility registry
export type ViewMode = "swipe" | "list" | "map";
export type Slot = "fact" | "badge" | "block";

export interface FieldMeta {
  key: string;
  label: string;
  slot: Slot;
  views: ViewMode[];
}

const ALL: ViewMode[] = ["swipe", "list", "map"];
const CARDS: ViewMode[] = ["swipe", "list"];
export const MAX_OPTIONAL = 6;

export const FIELD_META: FieldMeta[] = [
  { key: "description", label: "Description", slot: "block", views: CARDS },
  { key: "price", label: "Price", slot: "fact", views: ALL },
  { key: "tags", label: "Tags", slot: "badge", views: ALL },
  { key: "contact", label: "Contact", slot: "block", views: CARDS },
];

export const DEFAULTS: Record<ViewMode, string[]> = {
  swipe: ["description", "price", "tags"],
  list: ["description", "price", "tags", "contact"],
  map: ["price"],
};

export function optionalFieldsForView(view: ViewMode): FieldMeta[] {
  return FIELD_META.filter((f) => f.views.includes(view));
}

// Intersects a user's saved choice against what's valid NOW for this view,
// deduplicates, and caps at MAX_OPTIONAL — never trust a stale saved pref.
export function resolveFieldKeys(view: ViewMode, prefs?: Partial<Record<ViewMode, string[]>>): string[] {
  const allowed = new Set(optionalFieldsForView(view).map((f) => f.key));
  const chosen = prefs?.[view] ?? DEFAULTS[view];
  const seen = new Set<string>();
  const out: string[] = [];
  for (const k of chosen) {
    if (allowed.has(k) && !seen.has(k)) {
      seen.add(k);
      out.push(k);
      if (out.length >= MAX_OPTIONAL) break;
    }
  }
  return out;
}

// Server-side sanitizer for the PATCH endpoint — same allow-list + cap logic,
// applied per-view, so a malformed/oversized client payload can't corrupt state.
export function sanitizeDisplayFields(input: unknown): Partial<Record<ViewMode, string[]>> {
  const out: Partial<Record<ViewMode, string[]>> = {};
  if (!input || typeof input !== "object") return out;
  const obj = input as Record<string, unknown>;
  for (const view of ALL) {
    const arr = obj[view];
    if (!Array.isArray(arr)) continue;
    const allowed = new Set(optionalFieldsForView(view).map((f) => f.key));
    const seen = new Set<string>();
    const clean: string[] = [];
    for (const k of arr) {
      if (typeof k === "string" && allowed.has(k) && !seen.has(k)) {
        seen.add(k);
        clean.push(k);
        if (clean.length >= MAX_OPTIONAL) break;
      }
    }
    out[view] = clean;
  }
  return out;
}
```

```ts
// use-view-mode.ts — URL-persisted view toggle (history.replaceState, never router.replace)
export function readViewFromUrl(params: URLSearchParams): ViewMode {
  return (params.get("view") as ViewMode) ?? "swipe";
}

export function setViewInUrl(view: ViewMode) {
  const url = new URL(window.location.href);
  if (view === "swipe") url.searchParams.delete("view"); // default omitted from URL
  else url.searchParams.set("view", view);
  window.history.replaceState(null, "", url.toString());
}
```

```tsx
// A single shared renderer used by every card/list/deck/popup component —
// prevents the same field ever rendering differently between views.
function buildCardSections(view: ViewMode, prefs: Partial<Record<ViewMode, string[]>> | undefined, item: Item, ctx: FieldCtx) {
  const keys = resolveFieldKeys(view, prefs);
  const facts: Node[] = [], badges: Node[] = [], blocks: Node[] = [];
  for (const key of keys) {
    const meta = FIELD_META.find((f) => f.key === key);
    const node = RENDERERS[key]?.(item, ctx);
    if (!meta || !node) continue;
    (meta.slot === "fact" ? facts : meta.slot === "badge" ? badges : blocks).push(node);
  }
  return { facts, badges, blocks };
}
```

### Env vars
none — purely internal state/preferences pattern, no external config.

### Gotchas
- Persist the view toggle via `history.replaceState`, never a router-level `push`/`replace` — the latter both spams browser history (one entry per toggle) and often triggers a full data re-fetch that a client-only rendering-mode switch shouldn't need.
- Keep the view toggle a session/URL-level thing, NOT server-persisted as a team/user default — a browser refresh resetting to the default view (while a mid-session toggle is still shareable via URL) is often the intended UX; conflating the two makes "share this URL" behave inconsistently for the recipient.
- Re-validate a user's saved field-visibility preference against the CURRENT allowed-keys-per-view every time it's read, not just at save time — a field that gets renamed or removed from the registry must not silently leave a dangling, unrenderable key in someone's old saved preference.
- Sanitize the preference payload identically on the server (same allow-list + max-count logic) as on the client — never trust a client-constructed array of field keys directly into storage.
- One shared renderer function (not one per view) is what keeps a given field's rendering (icon, label, formatting) from drifting between the list view and the deck view — resist the urge to let each card component format the same field key independently.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter lets each user choose among three listing
renderings (swipe deck, list, map) and which fields appear per view, with the toggle held only in
the URL/session (never persisted server-side as a default) and a shared field-visibility registry
sanitized identically on client and server. Map rendering itself is already covered by the
cataloged maps-geo integrations ([map-tiles](../maps-geo/map-tiles.md),
[govmap-gis](../maps-geo/govmap-gis.md)) — this entry is only the toggle/field-selection
mechanism.
