# Semantic-token design system (Tailwind v4 + shadcn/ui, light/dark, RTL)

- **category**: other
- **provider**: internal (custom, built on Tailwind CSS v4 + shadcn/ui)
- **reusable**: yes — the whole system is one CSS file plus a handful of conventions; it drops into any Next.js + Tailwind v4 project. Only the brand hues and the status names are app-flavored, and both are swappable by editing token values.
- **docs**: https://tailwindcss.com/docs/theme and https://ui.shadcn.com/docs/theming (drop-in file: [`assets/design-tokens.css`](assets/design-tokens.css))

## Overview
A calm green-on-cream/concrete visual language with a full dark mode, built entirely from CSS variables so components never mention a raw palette color. Three token layers (raw ramps → semantic tokens → per-theme values), a registry-style single source for status colors, WCAG AA contrast baked into every value, and RTL-safe conventions. Reach for it when starting a Tailwind v4 + shadcn app that needs a coherent, themeable, accessible look without a `tailwind.config.js`. The scaffolding half of the stack (component CLI, `components.json`) is in [shadcn-ui](shadcn-ui.md).

## Playbook

### Prerequisites
- Next.js (App Router) with Tailwind CSS v4 (CSS-first config, no `tailwind.config.js`).
- `shadcn` initialised (see [shadcn-ui](shadcn-ui.md)), `tw-animate-css`, `next-themes`, `lucide-react`, `clsx` + `tailwind-merge`.
- Two web fonts via `next/font/google`: a UI sans and a heading/numeric face. If the product is RTL or Hebrew/Arabic, pick families with those subsets.

### Setup steps
1. Replace the contents of your global CSS (e.g. `app/globals.css`) with [`assets/design-tokens.css`](assets/design-tokens.css), or merge its `@theme`, `@theme inline`, `:root`, `.dark` and `@layer base` blocks into yours. Keep the three imports at the top.
2. Load fonts in the root layout and expose them as the CSS variables the file reads (`--font-ui`, `--font-display`):
   ```tsx
   // app/layout.tsx
   import { Assistant, Heebo } from "next/font/google";
   const ui = Assistant({ subsets: ["latin", "hebrew"], variable: "--font-ui" });
   const display = Heebo({ subsets: ["latin", "hebrew"], variable: "--font-display" });
   // <html lang dir className={`${ui.variable} ${display.variable}`} suppressHydrationWarning>
   ```
3. Wire dark mode: `next-themes` with `attribute="class"`, `defaultTheme="system"`, `enableSystem`. The `@custom-variant dark (&:is(.dark *))` line in the CSS makes `dark:` utilities follow the `.dark` class.
4. `components.json`: `"cssVariables": true`, `"tailwind.config": ""`, `"tailwind.css"` pointing at the file from step 1, and `"rtl": true` if applicable.
5. Add shadcn components (`npx shadcn@latest add button card dialog …`). They already read the semantic tokens, so they pick up the palette with no edits.
6. Create the status registry (see Core pattern) before building any status-colored UI.
7. Optional blue (or any) accent scope: add `className="themed-accent"` to `<body>` or a subtree to swap `--primary`/`--accent`/`--ring` (and optionally fonts) without touching the base theme.
8. Adopt the rule that **no component hand-writes a raw palette class** (`bg-gray-*`, `text-blue-600`, `dark:bg-gray-900`). A one-line CI/grep check enforces it: `grep -rnE "(bg|text|border)-(gray|slate|zinc|neutral|blue|red|green)-[0-9]" components app`.

### Core pattern
Three layers, each mapping to the next:

```css
/* 1. raw ramps → utilities like bg-sand-100, text-ink-700, bg-brand-700 */
@theme {
  --color-brand-700: #157f4f;
  --color-sand-100: #f1f0ec;
  --color-ink-950: #101214;
  --color-status-positive: #168556;
  --color-status-positive-ink: color-mix(in oklab, var(--color-status-positive) 65%, black);
}

/* 2. semantic tokens → utilities bg-background, text-muted-foreground, bg-primary … */
@theme inline {
  --color-background: var(--background);
  --color-primary: var(--primary);
  --color-ring: var(--ring);
}

/* 3. values per theme */
:root { --background: #f6f6f4; --primary: #157f4f; --ring: #157f4f; }
.dark { --background: #101214; --primary: #25c07c; --ring: #25c07c; }
```

Components use only layer 2 (`bg-card`, `text-foreground`, `border-border`, `bg-primary/10`, `ring-ring`) plus the custom ramps for depth (`bg-sand-100`, `bg-ink-900`). Opacity modifiers work on every token.

**Status colors live in one registry** — never a per-component color map:

```ts
// lib/status.ts
export type Status = "neutral" | "info" | "positive" | "accent" | "caution" | "negative" | "muted";

export const STATUS_STYLES: Record<Status, { label: string; spine: string; pill: string; dot: string; solid: string; button: string }> = {
  positive: {
    label: "Positive",
    spine: "border-s-status-positive",                                   // card's colored inline-start edge (use border-s-4)
    pill: "bg-status-positive/18 text-status-positive-ink ring-1 ring-inset ring-status-positive/30",
    dot: "bg-status-positive",
    solid: "bg-status-positive text-ink-950",
    button: "border-status-positive/40 text-status-positive-ink hover:bg-status-positive/12",
  },
  // …one entry per status, same six keys
};
```

Text on a status tint uses the `-ink` token (a `color-mix` toward black in light mode, toward white in dark mode), because the raw hue on its own /10–/18 tint cannot reach 4.5:1.

**Theme-override class:** a single class re-points `--primary`, `--accent`, `--ring` (and optionally `--font-*`) for a subtree, with a matching `.dark .themed-accent` block. This gives a second brand accent (e.g. admin area vs. user area) from the same components.

**Signature patterns kept generic:** a colored inline-start "spine" on status cards (`border-s-4` + the registry's `spine`), monochrome amenity/filter chips (`bg-secondary` + one lucide icon, no per-item colors), and a single tinted "assistant tray" surface for any AI-generated content (`bg-accent`, inner surfaces `bg-card` without borders) so AI UI never gets restyled by hand.

**Theme-dependent rendering needs a `mounted` gate** (anything derived from `resolvedTheme`):

```tsx
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
if (!mounted) return <Skeleton className="h-9 w-9" />;
```

### Env vars
none — pure CSS and build-time tooling.

### Gotchas
- **Contrast is the main trap, and the first-draft brand colors fail it.** Values that had to be darkened: a mid green (`#1faa6b`) was 2.99:1 against white button text; a focus ring in the lighter green was 2.07:1 (needs 3:1, WCAG 1.4.11); a warning amber was 2.40:1 as text; a muted-foreground gray was 4.44:1 on the secondary surface; a blue used as text on its own accent tint was 4.37:1. Any new text color must clear 4.5:1 on `--background`, `--card` **and** `--secondary` (the lowest of the three).
- Opacity modifiers on body text (`text-muted-foreground/60`) silently drop below AA — avoid them for anything a user must read. Re-run an automated audit (axe or a small script over the token pairs) after every palette change.
- Dark mode needs its own, **brighter** status hues: the darkened light-mode values fall below 4.5:1 on the dark card. That is why the `.dark` block redefines the `--color-status-*` hues, not only the semantic tokens.
- Tailwind v4 is CSS-first; there is **no `tailwind.config.js`**. Custom breakpoints are `--breakpoint-*` in `@theme`. Use the named one (`wide:`) instead of `min-[1200px]:` — the arbitrary variant sorts before `md:` in the generated CSS and loses.
- Any inline SVG / canvas / map marker cannot use CSS tokens, so keep a **parallel hex map** for it and update it whenever the status hues change (the one place the "one source of truth" rule leaks).
- **RTL:** use logical utilities everywhere (`ps-`/`pe-`/`ms-`/`me-`/`start-`/`end-`/`border-s-*`), never `pl-`/`left-`/`border-l-*`. Set `dir` on `<html>` and give the Base UI/Radix direction provider the same value. A shadcn `rtl: true` flag only helps if the surrounding CSS is logical end to end.
- Hydration: gate every theme-derived prop together behind one `mounted` flag (setting only one causes a mismatch), put `suppressHydrationWarning` on `<html>` for the `.dark` class swap, and consider a `darkreader-lock` meta tag — the Dark Reader extension otherwise rewrites inline styles before React hydrates.
- Mobile overflow: `overflow-x: hidden` on `html` is not enough once pinch-zoom is allowed (don't lock `maximum-scale`; that violates WCAG 1.4.4). Add `overflow-x: clip` on `body` — **not** `hidden`, which turns body into a scroll container and breaks a sticky nav.
- Honour `prefers-reduced-motion` with one global rule (included in the asset) rather than per-component `motion-reduce:` variants.
- A shared tooltip/popover provider is required by Base UI primitives; mount it once in the theme-provider stack.
- Adding a color later: define it in `@theme`, give light and dark values in `:root`/`.dark`, reference the semantic class name only — never an arbitrary hex in a class.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. It is the styling layer on top of the same shadcn/ui scaffolding described in [shadcn-ui](shadcn-ui.md); the approach is generic enough to be extracted as a shared CSS package plus a status-registry template.
