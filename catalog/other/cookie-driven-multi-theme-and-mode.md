# Cookie-driven multi-theme + light/dark/system mode (no flash, no theme library)

- **category**: other
- **provider**: internal (custom, Next.js App Router + Tailwind CSS v4 + CSS `oklch()`)
- **reusable**: yes — one small constants/parsers module, one client hook, and one CSS block; no dependency beyond React and Tailwind v4. The theme names, hue numbers and token set are the only app-flavored parts.
- **docs**: https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/oklch and https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme

## Overview
Lets the user pick one of N named colour themes (each is just a hue) and independently pick a display mode (`system` / `light` / `dark`). Both choices live in cookies, the server writes them onto `<html>` as `data-th` / `data-md` attributes during render, and CSS does all the work, including resolving `system` through `prefers-color-scheme`. Result: no inline script, no FOUC, no `next-themes`, no JS-derived theme props, so no hydration mismatches. Reach for it when you want several accent themes on top of a semantic-token design system. For the token layer itself see [design-system-semantic-tokens](design-system-semantic-tokens.md); for the native-Android counterpart see [compose-multi-theme-palettes](compose-multi-theme-palettes.md).

## Playbook

### Prerequisites
- Next.js App Router (server-rendered root layout that can read `cookies()`), Tailwind CSS v4 (`@theme inline`).
- A browser baseline with `oklch()` and `color-mix()` (all evergreen browsers).

### Setup steps
1. Create the constants + parsers module (Core pattern, step A). Parsers fall back to defaults on any unknown value, so a tampered or stale cookie can never break rendering.
2. In the root layout, read both cookies with `await cookies()`, parse them, and set `data-th` / `data-md` on `<html>`. Add `suppressHydrationWarning` because the client hook mutates those attributes later.
3. Write the CSS in this exact source order (all selectors share one specificity, so **source order is the cascade**): base block, per-theme hue overrides, forced-dark block, `@media (prefers-color-scheme: dark)` block for `[data-md="system"]`.
4. Parametrize every colour by two numbers: `--h` (accent hue) and `--nh` (neutral hue, a faint tint of the same family). A theme is then just `{ --h, --nh }`; the whole palette recomputes.
5. Define raw tokens (`--bg`, `--card`, `--ink`, `--acc`, status pairs …), alias them to semantic tokens, and expose the aliases to Tailwind in `@theme inline` so utilities like `bg-page` / `text-ink-muted` exist. Components then contain **no `dark:` variants and no raw colours**.
6. Add the client hook (step B) and a picker UI (a menu or segmented control) that calls `setTheme` / `setMode`. Show each theme's swatch using the same hue: `oklch(.55 .1 <hue>)`.
7. Optional: expose the same setters in a command palette ("Theme: Plum", "Mode: Dark").

### Core pattern

**A. `lib/appearance.ts` — shared by server layout, client hook and picker**
```ts
export const THEMES = ["paper", "slate", "clay", "plum"] as const;
export type Theme = (typeof THEMES)[number];
export const MODES = ["system", "light", "dark"] as const;
export type DisplayMode = (typeof MODES)[number];

export const THEME_COOKIE = "app-theme";
export const MODE_COOKIE = "app-mode";
export const DEFAULT_THEME: Theme = "paper";
export const DEFAULT_MODE: DisplayMode = "system";

export const THEME_HUES: Record<Theme, number> = { paper: 165, slate: 262, clay: 38, plum: 335 };

export const parseTheme = (v: string | null | undefined): Theme =>
  THEMES.includes(v as Theme) ? (v as Theme) : DEFAULT_THEME;
export const parseMode = (v: string | null | undefined): DisplayMode =>
  MODES.includes(v as DisplayMode) ? (v as DisplayMode) : DEFAULT_MODE;
export const themeSwatch = (t: Theme) => `oklch(.55 .1 ${THEME_HUES[t]})`;
```

**B. Root layout (server)**
```tsx
const cookieStore = await cookies();
const theme = parseTheme(cookieStore.get(THEME_COOKIE)?.value);
const mode = parseMode(cookieStore.get(MODE_COOKIE)?.value);
return <html data-th={theme} data-md={mode} suppressHydrationWarning>…</html>;
```

**C. `useAppearance()` — client hook; the DOM attribute is the store**
```ts
"use client";
const listeners = new Set<() => void>();
const subscribe = (l: () => void) => { listeners.add(l); return () => { listeners.delete(l); }; };

function write(attr: "data-th" | "data-md", cookie: string, value: string) {
  document.documentElement.setAttribute(attr, value);
  document.cookie = `${cookie}=${value}; path=/; max-age=31536000; samesite=lax`;
  listeners.forEach((l) => l());
}
const readTheme = () => parseTheme(document.documentElement.getAttribute("data-th"));
const readMode = () => parseMode(document.documentElement.getAttribute("data-md"));

export function useAppearance() {
  const theme = useSyncExternalStore(subscribe, readTheme, () => DEFAULT_THEME);
  const mode = useSyncExternalStore(subscribe, readMode, () => DEFAULT_MODE);
  const setTheme = useCallback((t: Theme) => write("data-th", THEME_COOKIE, t), []);
  const setMode = useCallback((m: DisplayMode) => write("data-md", MODE_COOKIE, m), []);
  return { theme, mode, setTheme, setMode };
}
```
Changing theme/mode is instant (attribute flip, no `router.refresh()`), and the cookie makes the next server render correct.

**D. CSS (order matters)**
```css
@import "tailwindcss";

:root, [data-th], [data-md] {          /* base = light, default hue */
  --h: 165; --nh: 85; color-scheme: light;
  --bg: oklch(.98 .006 var(--nh));  --card: oklch(1 0 0);
  --ink: oklch(.23 .01 var(--nh));  --mut: oklch(.46 .012 var(--nh));
  --line: oklch(.89 .012 var(--nh));
  --acc: oklch(.42 .09 var(--h));   --acc-ink: oklch(1 0 0);
  --acc-soft: oklch(.94 .03 var(--h)); --acc-text: oklch(.38 .09 var(--h));
  --ok: oklch(.4 .09 160);  --ok-bg: oklch(.94 .035 160);
  --bad: oklch(.45 .13 28); --bad-bg: oklch(.94 .03 28);
}
[data-th="slate"] { --h: 262; --nh: 255; }       /* themes = two numbers */
[data-th="clay"]  { --h: 38;  --nh: 60;  }
[data-th="plum"]  { --h: 335; --nh: 320; }

[data-md="dark"] {                                /* forced dark */
  color-scheme: dark;
  --bg: oklch(.19 .008 var(--nh)); --card: oklch(.23 .008 var(--nh));
  --ink: oklch(.94 .006 var(--nh)); --mut: oklch(.74 .012 var(--nh));
  --line: oklch(.34 .01 var(--nh));
  --acc: oklch(.76 .11 var(--h));  --acc-ink: oklch(.18 .02 var(--h));
  --acc-soft: oklch(.3 .05 var(--h)); --acc-text: oklch(.84 .1 var(--h));
  /* …brighter status hues… */
}
@media (prefers-color-scheme: dark) {
  [data-md="system"] { /* identical declarations to [data-md="dark"] */ }
}

@theme inline {                                   /* expose to utilities */
  --color-page: var(--bg);
  --color-content: var(--card);
  --color-ink: var(--ink);
  --color-accent: var(--acc);
  --color-on-accent: var(--acc-ink);
}
```
`system` is the only mode that needs the media query; `light` simply gets the base block, `dark` gets the forced block.

### Env vars
none.

### Gotchas
- **Cascade is source order.** The base selector `:root, [data-th], [data-md]` and the later `[data-th="x"]` / `[data-md="dark"]` rules have equal specificity, so reordering the blocks silently breaks hues or dark mode. Keep: base, hues, dark, system-media.
- **Dark values are duplicated** between `[data-md="dark"]` and the `system` media block (CSS cannot share a declaration list between a plain selector and a media-gated one without a preprocessor). Add a lint/test that the two blocks stay identical, or generate one from the other at build time.
- Because dark is separate from hue, each theme needs both a light and a dark lightness ladder to hit contrast. Keep lightness fixed per role and vary only hue/chroma, then check AA for the lowest-contrast hue (usually the yellow/amber theme) in both modes. Dark mode needs **brighter, lower-chroma** status colours than light mode.
- `color-scheme` must flip together with the tokens, otherwise native form controls, scrollbars and autofill stay light in dark mode.
- Cookie reading in the root layout makes every route dynamic. Acceptable for authenticated apps; for static marketing pages prefer a `prefers-color-scheme`-only approach or accept the dynamic cost.
- Use the DOM attribute as the client store (`useSyncExternalStore` with a server snapshot of the defaults). Deriving theme from React state or `localStorage` reintroduces flashes and mismatches; `suppressHydrationWarning` on `<html>` is required.
- Parse cookies defensively on the server: an unknown value (renamed theme, hand-edited cookie) must fall back to the default rather than emit an invalid `data-th`.
- Charts, canvases and PDF output cannot read CSS variables per theme. Keep chart colours as tokens (read via `getComputedStyle`) and let PDFs use a fixed palette.
- Setting the cookie client-side without `Secure` is fine for a non-sensitive preference, but keep `samesite=lax` and a long `max-age`. It is not a place for anything user-identifying.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. Sits on top of a semantic-token design system; the two together give a themeable, dark-mode-capable UI with zero `dark:` utilities. A second adopter would make it a candidate for a shared `appearance` package (constants + hook + base CSS).
