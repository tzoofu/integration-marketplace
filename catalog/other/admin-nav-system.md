# Role-Based Admin Navigation

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — the "one filtered config rendered as N surfaces" pattern (sidebar/drawer/dropdown) plus legacy-URL-alias resolution is a portable admin-shell pattern for any role-gated dashboard.
- **docs**: n/a — internal pattern, no external provider.

## Overview
A single source-of-truth admin navigation registry (sections of tabs/links, each optionally role-gated and optionally carrying a badge-count key) rendered as three different surfaces — desktop sticky sidebar, mobile hamburger drawer, compact header dropdown — via one shared filtering function, so the surfaces can never drift out of sync. A legacy-URL-alias table resolves pre-redesign bookmark/query-param shapes onto the current flattened leaf-id scheme.

## Playbook

### Prerequisites
- A role/permission signal available server- and client-side (e.g. `isAdmin`, `isSuperAdmin`) to filter the nav config.
- An icon library (the source uses `lucide-react`, but any icon set works).
- A lightweight badge-count endpoint if you want live counts (optional — the nav renders fine without it).

### Setup steps
1. Define one typed registry of nav sections → items (`kind: "tab" | "link"`), each item carrying an id, label, icon, optional `roleGated: boolean`, and optional `countKey` for a badge.
2. Write ONE filtering function (e.g. `filteredSections(role)`) that every surface calls — never let a surface (sidebar vs. drawer) apply its own separate filter, or the two will eventually show different items to the same user.
3. Render the desktop surface (sticky sidebar) and the mobile surface (hamburger drawer) from the exact same `filteredSections(role)` output — a drawer that renders the config as a full-width list, not a second hand-maintained nav tree.
4. Add a lazy badge-count fetch: only call the counts endpoint once the user has actually navigated into the admin route family (check `pathname` against a small allowlist of admin routes), so browsing the rest of the app never pays for it.
5. If migrating from an older nav shape, add a `LEGACY_ALIASES` map from old `?tab=`/`?subtab=` values to the new flattened leaf ids, and a `resolveLeaf(tab, subtab, role)` function that: (a) accepts a current id directly if it resolves under the caller's role, (b) else falls back to the alias table, (c) else falls back to a default leaf. Call this identically from every place a leaf is derived from the URL (initial state AND any resync effect) — a mismatch here is what silently strands back-navigation on the wrong tab.
6. Build `hrefFor(item)` as a single helper (tab → `/admin?tab=<id>`, link → its own `href`) so every surface links consistently.

### Core pattern
```ts
// admin-nav.ts — one config, filtered once, rendered everywhere
export type NavLeafId = string;

export type NavTab = {
  kind: "tab";
  id: NavLeafId;
  label: string;
  icon: IconType;
  countKey?: string;
  roleGated?: boolean; // e.g. super-admin only
};

export type NavLink = {
  kind: "link";
  id: string;
  label: string;
  href: string;
  icon: IconType;
};

export type NavItem = NavTab | NavLink;
export type NavSection = { id: string; label: string; items: NavItem[] };

export const DEFAULT_LEAF: NavLeafId = "overview";

export const NAV: NavSection[] = [
  {
    id: "queue",
    label: "Queue",
    items: [
      { kind: "tab", id: "pendingItems", label: "Pending items", icon: ClipboardIcon, countKey: "pendingItems" },
    ],
  },
  {
    id: "config",
    label: "Configuration",
    items: [
      { kind: "link", id: "settings", label: "Settings", href: "/admin/settings", icon: SettingsIcon },
      { kind: "tab", id: "danger", label: "Danger zone", icon: AlertIcon, roleGated: true },
    ],
  },
];

// THE one filtering pass. Sidebar, drawer, and dropdown all call this —
// never filter independently, or the surfaces will drift apart.
export function filteredSections(hasElevatedRole: boolean): NavSection[] {
  return NAV.map((s) => ({
    ...s,
    items: s.items.filter((i) => i.kind !== "tab" || !i.roleGated || hasElevatedRole),
  })).filter((s) => s.items.length > 0);
}

export function hrefFor(item: NavItem): string {
  return item.kind === "link" ? item.href : `/admin?tab=${item.id}`;
}

// Legacy pre-redesign `?tab=`/`?subtab=` values mapped onto today's leaves.
const LEGACY_ALIASES: Record<string, NavLeafId> = {
  oldFlatKey: "pendingItems",
  "nested.oldKey": "danger",
};

export function resolveLeaf(
  tab: string | null,
  subtab: string | null,
  hasElevatedRole: boolean
): NavLeafId {
  const allTabs = NAV.flatMap((s) => s.items).filter(
    (i): i is NavTab => i.kind === "tab" && (!i.roleGated || hasElevatedRole)
  );
  if (tab && allTabs.some((t) => t.id === tab)) return tab;
  const key = tab === "nested" && subtab ? `nested.${subtab}` : tab ?? "";
  const resolved = LEGACY_ALIASES[key];
  return resolved && allTabs.some((t) => t.id === resolved) ? resolved : DEFAULT_LEAF;
}
```

```tsx
// AdminNav.tsx — desktop sidebar; the mobile drawer renders the same
// filteredSections(role) list, just with drawer-specific styling.
function AdminNav({ hasElevatedRole, activeId, counts }: { hasElevatedRole: boolean; activeId: string; counts?: Record<string, number> }) {
  const sections = filteredSections(hasElevatedRole);
  return (
    <nav>
      {sections.map((section) => (
        <div key={section.id}>
          <div className="section-label">{section.label}</div>
          {section.items.map((item) => (
            <Link key={item.id} href={hrefFor(item)} className={item.id === activeId ? "active" : ""}>
              <item.icon />
              {item.label}
              {item.kind === "tab" && item.countKey && counts?.[item.countKey] ? (
                <span className="badge">{counts[item.countKey]}</span>
              ) : null}
            </Link>
          ))}
        </div>
      ))}
    </nav>
  );
}
```

### Env vars
none — purely internal UI/routing pattern, no external config.

### Gotchas
- The single biggest risk is two surfaces (sidebar/drawer/dropdown) each implementing their own filter pass over the raw config — they *will* drift (e.g. one hides a super-admin-only item, the other doesn't). Always route every surface through one shared filter function.
- Fetch badge counts lazily, gated on the current route actually being in the admin family — otherwise every page load in the whole app pays for an admin-only query.
- When adding legacy-URL back-compat, resolve the leaf identically in the initial-state computation and in any effect that resyncs from `useSearchParams()` — computing it two different ways is what silently breaks browser-back and generates `?back=` links landing on the wrong tab.
- Keep a "metadata lookup by id" helper (label/icon/description for a header) separate from the "access gate" helper (which filters by role) — conflating them makes it easy to accidentally leak a gated item's metadata, or to under-gate a lookup that's actually reachable from an already-filtered leaf id.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter renders a multi-section, role-gated admin
navigation (dozens of leaf tabs plus a few external links) across desktop, mobile, and dropdown
surfaces from one shared filter pass, with live badge counts and a legacy-URL-alias table for
pre-redesign bookmark links.
