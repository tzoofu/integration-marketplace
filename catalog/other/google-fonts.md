# Google Fonts (next/font/google)

- **category**: other
- **provider**: Google Fonts
- **reusable**: yes — identical `next/font/google` usage across adopters for Hebrew-supporting typefaces; a shared `@marketplace/fonts` preset (font choice + `display`/subset config) would remove duplicated setup.
- **docs**: https://nextjs.org/docs/app/building-your-application/optimizing/fonts

## Overview
Loads Hebrew-supporting Google typefaces (Assistant, Heebo) at build time via Next.js's built-in font optimizer. Next self-hosts/inlines the font files at build time — there is no runtime request to Google's font CDN, and no API key or env var is required for the font loading itself.

## Playbook

### Prerequisites
- Next.js App Router project (font loading shown here is the `next/font/google` module, available since Next 13).
- Know which Google Font family name(s) you need (must match the exact name Google Fonts uses, e.g. "Heebo", "Assistant").
- If the UI is RTL/Hebrew (or another non-Latin script), confirm the font family actually ships a `hebrew` (or relevant) subset — not all Google Fonts do.

### Setup steps
1. Import the desired font(s) directly from `next/font/google` in the root layout (or wherever the font is scoped) — no package install needed, it's bundled with Next.js.
2. Instantiate each font with `subsets`, `display`, and a CSS custom-property `variable` name.
3. Apply the returned `.variable` class name(s) to the `<html>` or `<body>` element so the CSS variable is available globally.
4. Reference the font via `var(--font-<name>)` in CSS, or rely on a Tailwind/`font-sans` utility mapped to that variable.
5. Optionally pass `weight: [...]` if the family isn't variable-width, to only ship the weights you actually use.

### Core pattern
```tsx
// app/layout.tsx
import { SomeGoogleFont } from "next/font/google";

const someFont = SomeGoogleFont({
  subsets: ["latin", "hebrew"], // match script(s) actually needed
  display: "swap",              // avoid invisible-text flash while loading
  variable: "--font-some-font",
  // weight: ["400", "700"],    // only needed for non-variable font families
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="he" dir="rtl" className={`${someFont.variable} h-full`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

```css
/* tailwind.config or globals.css — map the CSS var to a utility */
:root {
  --font-sans: var(--font-some-font);
}
```

### Env vars
none — font loading itself requires no credentials or configuration.

### Gotchas
- Not every Google Font ships every subset — verify the family actually has a `hebrew` (or other non-Latin) subset before committing to it; picking the wrong family silently falls back to a system font for unsupported glyphs.
- `display: "swap"` is used across observed adopters to prevent invisible text during font load (FOIT); omitting it risks a flash of invisible text.
- Multiple fonts can be composed by exposing multiple CSS variables (e.g. `--font-heebo`, `--font-assistant`) and switching between them per-component via Tailwind classes, rather than only ever using the `<html>`-level default.
- Because Next inlines/self-hosts these fonts at build time, there's no client-side network call to Google's font CDN and thus no related CSP/privacy concern to configure.

### Playbook confidence: high

## Adoption
Used in **2** repos in this marketplace. Both adopters load Hebrew-supporting typefaces for a
Hebrew-first RTL UI; one loads a single family while the other loads two, otherwise following the
identical setup pattern.
