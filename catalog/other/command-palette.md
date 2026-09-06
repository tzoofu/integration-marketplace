# Command Palette (⌘K)

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — a global-hotkey, role-gated, search-fallback command palette is a generic pattern applicable to any dashboard app.
- **docs**: n/a — internal pattern, no external provider (loosely inspired by macOS Spotlight / VS Code's command palette, but no library dependency).

## Overview
A Cmd/Ctrl+K searchable command palette for navigation and actions, with role-gated commands (built from the same nav registry as the admin sidebar) and a "search the app" fallback row for unmatched queries.

## Playbook

### Prerequisites
- A modal/dialog primitive (source uses a Base UI `Dialog`, but any accessible modal works).
- Existing route list and/or a nav-config registry to source palette items from (optional — can also hardcode a flat item list).
- A router with imperative navigation (`router.push`).

### Setup steps
1. Define a `PaletteItem` shape: `{ id, label, description?, icon, group, keywords?, run() }`. Group items (`nav`, `actions`, `admin`, ...) purely for visual section headers.
2. Write a builder function `getCommandPaletteItems(ctx)` — not a static array — since actions close over live state (router, role flags, theme). Recompute it on every render so gating and available commands stay live with auth/role/theme changes.
3. Install ONE global `keydown` listener (in a top-level provider component, not per-page) that toggles the palette open on Cmd/Ctrl+K. Suppress the shortcut while focus is in an input/textarea/contenteditable, UNLESS the palette is already open (so ⌘K still closes it from anywhere).
4. Render a small visible `⌘K` button alongside the invisible hotkey — discoverability for users who don't know the shortcut exists.
5. In the palette component: do substring filtering across `label` + `description` + `keywords`, track an `activeIndex` for arrow-key navigation, wire `ArrowUp`/`ArrowDown`/`Enter`, and reset both the query and `activeIndex` every time the palette opens.
6. Add a synthetic "search fallback" row appended only when the query is non-empty AND no item matched — routing to the app's own search page/query param (e.g. `/dashboard?search=<query>`) instead of leaving the user with a dead end.
7. Focus the input on open via `requestAnimationFrame` (not synchronously) if the modal library mounts content in the same tick — a raw synchronous `.focus()` call can race the DOM node's mount.

### Core pattern
```ts
// command-palette-items.ts — builder, not a plain array (closures over live state)
export type PaletteGroup = "nav" | "actions" | "admin";

export type PaletteItem = {
  id: string;
  label: string;
  description?: string;
  icon: IconType;
  group: PaletteGroup;
  keywords?: string[];
  run(): void;
};

export function getCommandPaletteItems(ctx: {
  router: Router;
  isAdmin: boolean;
  resolvedTheme: string | undefined;
  setTheme(theme: string): void;
}): PaletteItem[] {
  const { router, isAdmin, resolvedTheme, setTheme } = ctx;
  const isDark = resolvedTheme === "dark";

  const items: PaletteItem[] = [
    { id: "nav-home", label: "Dashboard", icon: HomeIcon, group: "nav", run: () => router.push("/dashboard") },
    { id: "action-toggle-theme", label: isDark ? "Light mode" : "Dark mode", icon: isDark ? SunIcon : MoonIcon,
      group: "actions", keywords: ["theme", "dark", "light"], run: () => setTheme(isDark ? "light" : "dark") },
  ];

  if (isAdmin) {
    for (const item of adminNavItems()) {
      items.push({ id: `admin-${item.id}`, label: item.label, icon: item.icon, group: "admin", run: () => router.push(hrefFor(item)) });
    }
  }
  return items;
}
```

```tsx
// CommandPaletteProvider.tsx — the one global keydown listener
function isTypingTarget(el: Element | null): boolean {
  if (!el) return false;
  return el.tagName === "INPUT" || el.tagName === "TEXTAREA" || el.hasAttribute("contenteditable");
}

function CommandPaletteProvider({ isAdmin }: { isAdmin: boolean }) {
  const [open, setOpen] = useState(false);
  const router = useRouter();

  useEffect(() => {
    function onKeyDown(e: KeyboardEvent) {
      if (!(e.metaKey || e.ctrlKey) || e.key.toLowerCase() !== "k") return;
      if (!open && isTypingTarget(document.activeElement)) return;
      e.preventDefault();
      setOpen((prev) => !prev);
    }
    window.addEventListener("keydown", onKeyDown);
    return () => window.removeEventListener("keydown", onKeyDown);
  }, [open]);

  const items = getCommandPaletteItems({ router, isAdmin, resolvedTheme: "light", setTheme: () => {} });
  return (
    <>
      <button onClick={() => setOpen(true)}><kbd>⌘K</kbd></button>
      <CommandPalette open={open} onOpenChange={setOpen} items={items} />
    </>
  );
}

// CommandPalette.tsx — filtering + fallback row + keyboard nav
function CommandPalette({ open, onOpenChange, items }: { open: boolean; onOpenChange(v: boolean): void; items: PaletteItem[] }) {
  const router = useRouter();
  const [query, setQuery] = useState("");
  const [activeIndex, setActiveIndex] = useState(0);

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return items;
    return items.filter((i) => [i.label, i.description, ...(i.keywords ?? [])].filter(Boolean).join(" ").toLowerCase().includes(q));
  }, [items, query]);

  type Row = { kind: "item"; item: PaletteItem } | { kind: "search"; query: string };
  const rows: Row[] = filtered.length ? filtered.map((item) => ({ kind: "item", item }))
    : query.trim() ? [{ kind: "search", query: query.trim() }] : [];

  function runRow(row: Row) {
    if (row.kind === "search") router.push(`/search?q=${encodeURIComponent(row.query)}`);
    else row.item.run();
    onOpenChange(false);
  }

  function onKeyDown(e: React.KeyboardEvent) {
    if (!rows.length) return;
    if (e.key === "ArrowDown") { e.preventDefault(); setActiveIndex((i) => (i + 1) % rows.length); }
    else if (e.key === "ArrowUp") { e.preventDefault(); setActiveIndex((i) => (i - 1 + rows.length) % rows.length); }
    else if (e.key === "Enter") { e.preventDefault(); runRow(rows[activeIndex]); }
  }

  // ...render Dialog with an <input> (onKeyDown={onKeyDown}) and a <ul role="listbox"> of rows
}
```

### Env vars
none — purely client-side UI pattern, no external config.

### Gotchas
- Gate the global `keydown` listener on `isTypingTarget(document.activeElement)` only while the palette is closed — if you also gate it while open, users can't close the palette with ⌘K while their cursor is still in the search input.
- Build the item list every render (a function call, not a memoized static array with a coarse dependency list) — role/theme-gated items must recompute immediately on auth or theme change, not lag behind.
- The "search fallback" row must only appear when there are zero real matches AND the query is non-empty — showing it alongside real matches, or when the query is empty, clutters the list with a route to nowhere useful.
- If the modal library mounts its content synchronously on `open`, a plain synchronous `.focus()` in the same effect can fire before the input exists in the DOM — defer with `requestAnimationFrame` and cancel it on cleanup.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter wires the palette to a global ⌘K/Ctrl+K
listener with role-gated commands (regular vs. elevated-role items filtered out live), grouped
rows with keyboard navigation, and a synthetic search-fallback row when no command matches the
query.
