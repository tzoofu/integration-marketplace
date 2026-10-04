# Draft-State Admin Settings Editor

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — the "orchestrator owns draft/dirty state, dumb tab components, one atomic patch-save helper, sticky bottom bar" shape applies to any admin config/settings screen with multiple sections and a partial-update API, regardless of framework.
- **docs**: n/a — internal pattern, no external provider.

## Overview
An admin settings screen split into a thin page-level orchestrator (owns tab routing, a nullable "draft" copy of the settings, dirty-tracking, and the single save call) and pure presentational tab/section components (render `view`/`update` props, hold no save logic of their own). Saves are partial patches through one shared auth+fetch helper, merged optimistically into client state — never a full page refetch — and surfaced via a sticky bottom action bar instead of per-field save buttons.

## Playbook

### Prerequisites
- A client-side settings store (context/hook) that holds the current settings value and an optimistic patch-merge updater.
- An authenticated endpoint that accepts a **partial** patch of the settings document (not a full-document replace) and validates it server-side.
- A way to attach an auth credential (ID token, session cookie) to the save request.

### Setup steps
1. Split the settings data layer into: (a) pure types + defaults + a `parseSettings()` function with no framework/transport dependency, (b) a client hook/context exposing the current value plus an optimistic updater, and (c) if the source is cached/fetched server-side, a separate server-only loader. Keep parsing logic in exactly one place so client and server never diverge on what a valid value looks like.
2. Write **one** save helper (`saveSettings(patch)`) that every save call site uses: attach the auth credential, `POST`/`PATCH` the patch, parse a JSON error body on failure, throw with a user-facing message on `!res.ok`. Never hand-roll `fetch` + token-attach again in a new component.
3. Build a client-side optimistic updater that merges a successfully-saved patch straight into local state. **Never re-fetch or remount the settings provider after a save** — that re-renders from whatever the fetch/cache layer currently holds (which can be a stale cached copy) and makes a successful save visually "revert."
4. In the page-level orchestrator, hold draft state as `useState<Draft | null>(null)` — `null` means "no local edits, show the base value." Compute `const view = draft ?? base` and `const dirty = draft !== null && !deepEqual(draft, base)`.
5. Expose a single `update(fn: (d: Draft) => Draft)` setter that lazily seeds `draft` from `base` on first call (`setDraft(prev => fn(prev ?? base))`). This lets every section mutate the same draft without needing its own local state or fighting over initialization.
6. Give every tab/section its own presentational component that receives `view`, `update`, and any section-specific callbacks as props. **Zero save logic lives in a child** — a child that saves its own fields turns one atomic multi-field commit into N independent partial writes, which is almost never what "Save" is supposed to mean to the user.
7. Render one sticky, fixed-position bottom bar, animated in/out purely off the `dirty` boolean, with a discard action (`setDraft(null)`, plus resetting any section-local mirrors) and a save action that calls the save helper with only the fields owned by that save boundary, then calls the optimistic updater and clears the draft.
8. Route which tab/section is showing through one typed registry (`const TABS = {...} as const`) plus a single `tabHref()` helper — never compare against raw string literals scattered across nav links, redirects, and the page itself.
9. Checklist for adding a new field: (1) add it to the shared type, its default value, and the parser; (2) add it to the server-side validation schema **and** the write/update path it flows into — a validation library that isn't told about a field will silently strip it on save, which is the single most common bug in this pattern; (3) expose it in the correct tab component; (4) read it back through the client hook or server loader. Do all four in the same change — a partial update here is what causes "I added the field to the form but it doesn't save."

### Core pattern
```tsx
// settings-shared.ts — types, tab registry, draft derivation
export const TABS = { general: "General", billing: "Billing" } as const;
export type Tab = (typeof TABS)[keyof typeof TABS];
export function tabHref(tab: Tab) { return `/admin/settings?t=${tab}`; }

export type Draft = { name: string; billingEmail: string /* ...one field per editable setting */ };

export function fromSettings(s: Settings): Draft {
  return { name: s.name, billingEmail: s.billingEmail };
}

// settings-client.ts — one save helper, used by every save call site
export async function saveSettings(patch: Partial<Settings>) {
  const token = await getAuthToken();
  const res = await fetch("/api/admin/settings", {
    method: "POST",
    headers: { "content-type": "application/json", authorization: `Bearer ${token}` },
    body: JSON.stringify(patch),
  });
  const body = await res.json().catch(() => ({}));
  if (!res.ok) throw new Error(body.error ?? "Save failed");
  return body;
}

// SettingsPage.tsx — orchestrator: owns draft/dirty, renders dumb tabs + one sticky bar
function SettingsPage() {
  const { settings } = useSettings();
  const updateSettings = useUpdateSettings(); // optimistic merge into client state
  const [draft, setDraft] = useState<Draft | null>(null);
  const [submitting, setSubmitting] = useState(false);

  const base = fromSettings(settings);
  const view = draft ?? base;
  const dirty = draft !== null && JSON.stringify(draft) !== JSON.stringify(base);

  const update = (fn: (d: Draft) => Draft) => setDraft((prev) => fn(prev ?? base));

  const onSave = async () => {
    setSubmitting(true);
    try {
      const patch: Partial<Settings> = { name: view.name, billingEmail: view.billingEmail };
      await saveSettings(patch);
      updateSettings(patch); // merge, don't refetch
      setDraft(null);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <>
      <GeneralTab view={view} update={update} />
      {dirty && (
        <div className="fixed bottom-0 inset-x-0">
          <button onClick={() => setDraft(null)} disabled={submitting}>Discard</button>
          <button onClick={onSave} disabled={submitting}>{submitting ? "Saving…" : "Save"}</button>
        </div>
      )}
    </>
  );
}

// GeneralTab.tsx — purely presentational, no save logic
function GeneralTab({ view, update }: { view: Draft; update: (fn: (d: Draft) => Draft) => void }) {
  return (
    <input
      value={view.name}
      onChange={(e) => update((d) => ({ ...d, name: e.target.value }))}
    />
  );
}
```

### Env vars
none — purely internal UI/data-flow pattern, no external config beyond whatever the auth/transport layer already needs.

### Gotchas
- Never call a full-page refetch, router refresh, or provider remount after a successful save — it re-renders from whatever the fetch/cache layer holds at that moment (which may be a stale cached document that hasn't caught up to the write yet), and a perfectly successful save will visually look like it "didn't take" or "reverted." Always merge the just-saved patch into client state directly.
- A field added only to the client draft type and the UI, but not to the server-side validation schema's shape *and* its write/update block, gets silently dropped by the validator on every save — treat those two additions as one inseparable step, not two.
- If any single tab/section gets its own save button, "Save" no longer means one atomic commit — decide up front what the atomicity boundary is (usually: one tab = one save transaction) and never let a child component call the save helper on its own.
- Draft state must default to `null` (meaning "no pending edits") and be lazily seeded from the base value on first mutation, not eagerly initialized from the base on mount — eager initialization plus any parent re-render/prop change can silently reset in-progress user edits.
- Compute "dirty" via a structural/deep comparison against the base value, not a hand-maintained set of per-field "touched" booleans — the hand-maintained version drifts every time a field is added and eventually reports wrong dirty/clean state.
- Route tab/section identity through one typed constant map, not raw string literals — every additional place that links cross-tab (nav, a related settings page, deep links) is another spot a hardcoded string can drift out of sync with a renamed tab.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter uses this shape for a multi-tab business settings screen (5+ tabs) backed by a single Firestore document, with a couple of section-local mirrors (e.g. a phone number field, a small color/text sub-form) that intentionally save independently of the main tab because they're logically their own atomicity boundary rather than part of the tab's bulk patch.
