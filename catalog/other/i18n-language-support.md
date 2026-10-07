# i18n / language support (Hebrew + English, RTL)

- **category**: other
- **provider**: next-intl, i18next / react-i18next (+ `i18next-browser-languagedetector`, `expo-localization`), or internal (hand-rolled typed dictionary)
- **reusable**: yes — every adopter solves the same problem: two locales (`he`/`en`), RTL for Hebrew, a persisted choice, and a catalog-parity guard. The locale config, `dir` helper, parity check, and formatting helpers could be shared as a `@marketplace/i18n` package. The library wiring has to stay per-framework.
- **docs**: https://next-intl.dev/docs · https://www.i18next.com · https://react.i18next.com · https://docs.expo.dev/versions/latest/sdk/localization/

How an app picks a language, loads its UI strings, switches between Hebrew (RTL) and English
(LTR), and keeps both catalogs in sync. Use this playbook when an app needs to work in more than
one language, or as a checklist when adding a language to an app that has only one.

## Language support matrix (as implemented across adopters)

Every adopter supports exactly **`he` and `en`**. Variants are labelled by technical shape.

| Variant | Library | Supported | **Default** | Fallback for missing key | How the locale is chosen (precedence) |
|---|---|---|---|---|---|
| A. Next.js App Router, URL-prefixed | next-intl v4 (no middleware) | `en`, `he` | **`he`** | none (invalid locale segment → 404) | `/[locale]` URL segment is the source of truth. A bare `/` is resolved by `NEXT_LOCALE` cookie → `Accept-Language` → default |
| B. Next.js App Router, cookie-only | next-intl v4 | `en`, `he` | **`he`** | none | `locale` cookie → default (no URL prefix, no Accept-Language) |
| C. Vite React SPA (+ Express API) | i18next + react-i18next + browser language detector | `he`, `en` | **`he`** | `fallbackLng: 'he'` | localStorage → (navigator, effectively shadowed, see Gotchas) → default. The API gets `X-Language` / `Accept-Language` headers |
| D. React Native (Expo) | i18next + react-i18next + expo-localization | `en`, `he` | **device locale** (`he` if the device language is Hebrew, otherwise `en`) | `fallbackLng: 'en'` | persisted setting `'system' \| 'en' \| 'he'` → device locale |
| E. Next.js, hand-rolled | none (typed dictionary) | `he`, `en` | **`he`** | `he` → the key itself | `locale` cookie → default |
| F. Vite React SPA, hand-rolled | none (React context + dictionaries) | `he`, `en` | **`he`** | n/a (typed, so a missing key is impossible) | localStorage → default |
| G. Vite React SPA (+ Go API, chat/messaging content), hand-rolled | none (React context + typed dictionary) | `en`, `he` | **browser language** (`he` if `navigator.language` starts with `he`, otherwise `en`) | n/a (typed) | localStorage → `navigator.language` |

Several more apps in the marketplace are **Hebrew-only**. They set a static `<html lang="he" dir="rtl">` with no
switcher, or set `dir` per element on Hebrew content. They don't count as adopters, but the RTL
sections below apply to them too.

**Recommended default for a new Israel-facing web app:** variant A if you need shareable or SEO-indexable
per-language URLs, variant B if you don't (it's simpler). In both, `he` is the default and the
parity check is enforced in lint.

## Playbook

### Prerequisites
- One catalog file per locale (nested JSON for next-intl/i18next, or a TS dictionary for the hand-rolled variants).
- A font that covers Hebrew (for example `next/font/google` with `subsets: ["latin", "hebrew"]`; see [google-fonts](google-fonts.md)).
- A decision on where the locale lives (URL segment, cookie, localStorage, or device plus a setting). Use the matrix above to choose.

### Setup steps
1. Declare the locale list and the default **once**, as a typed constant (`["en", "he"] as const`, default `"he"`), and check every incoming value against that list before using it (`hasLocale(...)` or the equivalent).
2. Create `messages/en.json` and `messages/he.json`. The top-level keys act as namespaces (`common`, `nav`, `settings`, …).
3. Wire the library (see Core pattern for your variant).
4. Set `lang` and `dir` on the root element: `dir = locale === "he" ? "rtl" : "ltr"`. On web, add `suppressHydrationWarning` to `<html>` if a theme or locale script mutates it.
5. Build the switcher. It should persist the choice (cookie, localStorage, or a setting) and then re-render in the new locale. Use `router.replace` for a URL-prefixed locale, `window.location.reload()` for a cookie-only locale, `changeLanguage()` in an SPA, and `forceRTL` plus a restart on React Native.
6. Add a **parity check** that fails CI when the two catalogs don't have the same keys (lint script, unit test, or a compile-time type).
7. Write RTL-safe styles: logical properties (`ms-*`/`me-*`, `ps-*`/`pe-*`, `start-*`/`end-*`, `text-align: start`) rather than physical left/right.
8. Format dates and numbers with an **explicit** locale derived from the app locale. Never call bare `toLocaleString()`.

### Core pattern

**Shared config**
```ts
// i18n/config.ts
export const LOCALES = ["en", "he"] as const;
export type AppLocale = (typeof LOCALES)[number];
export const DEFAULT_LOCALE: AppLocale = "he";
export const isLocale = (v: unknown): v is AppLocale => LOCALES.includes(v as AppLocale);
export const dirFor = (l: AppLocale) => (l === "he" ? "rtl" : "ltr");
export const intlTag = (l: AppLocale) => (l === "he" ? "he-IL" : "en-US");
```

**Variant A: Next.js, `/[locale]` segment, next-intl, no middleware**
```ts
// i18n/routing.ts
import { defineRouting } from "next-intl/routing";
export const routing = defineRouting({ locales: ["en", "he"], defaultLocale: "he" });

// i18n/resolve-locale.ts: replaces next-intl middleware's first-visit negotiation
import { cookies, headers } from "next/headers";
import { hasLocale } from "next-intl";
export async function resolveLocaleFromRequest() {
  const c = (await cookies()).get("NEXT_LOCALE")?.value;
  if (hasLocale(routing.locales, c)) return c;
  for (const tag of ((await headers()).get("accept-language") ?? "").split(",")) {
    const lang = tag.trim().split(";")[0].split("-")[0];
    if (hasLocale(routing.locales, lang)) return lang;
  }
  return routing.defaultLocale;
}

// app/page.tsx (root, outside [locale])
export default async function Root() { redirect(`/${await resolveLocaleFromRequest()}`); }

// i18n/request.ts (wired via createNextIntlPlugin("./i18n/request.ts") in next.config.ts)
export default getRequestConfig(async ({ requestLocale }) => {
  const requested = await requestLocale;
  const locale = hasLocale(routing.locales, requested) ? requested : routing.defaultLocale;
  const messages = (await import(`../messages/${locale}.json`)).default;
  return { locale, messages /* optionally: mergeMessageOverrides(messages, overrides) */ };
});

// app/[locale]/layout.tsx: the real <html> lives here; root layout just returns children
export default async function LocaleLayout({ children, params }) {
  const { locale } = await params;
  if (!hasLocale(routing.locales, locale)) notFound();
  setRequestLocale(locale); // required: no middleware sets the request header
  const messages = await getMessages();
  return (
    <html lang={locale} dir={locale === "he" ? "rtl" : "ltr"} suppressHydrationWarning>
      <body>
        <NextIntlClientProvider messages={messages}>
          <LocaleCookieSync locale={locale} />
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}

// LocaleCookieSync.tsx ("use client"): remembers the choice for the next bare "/" visit
useEffect(() => {
  document.cookie = `NEXT_LOCALE=${locale}; path=/; max-age=31536000; samesite=lax`;
}, [locale]);

// Switcher: swap the first path segment, replace (not push) to avoid a duplicate history entry
const next = locale === "en" ? "he" : "en";
const segs = pathname.split("/"); segs[1] = next;
router.replace((segs.join("/") || `/${next}`) + location.search + location.hash);
```

**Variant B: Next.js, cookie-only, next-intl**
```ts
// i18n/request.ts
export default getRequestConfig(async () => {
  const c = (await cookies()).get("locale")?.value;
  const locale = c === "en" ? "en" : "he";
  return { locale, messages: (await import(`../messages/${locale}.json`)).default };
});
// app/layout.tsx
const locale = await getLocale();
// <html lang={locale} dir={dirFor(locale)}> <NextIntlClientProvider> (v4 inherits messages from the server config)
// Switcher
document.cookie = `locale=${other}; path=/; max-age=31536000; samesite=lax`;
window.location.reload();
```
Server components use `getTranslations("ns")` and client components use `useTranslations("ns")`. ICU plurals work out of the box: `{count, plural, one {# item} other {# items}}`.

**Variant C: Vite/React SPA, i18next**
```ts
i18n.use(LanguageDetector).use(initReactI18next).init({
  resources,                       // { he: { common, ... }, en: { common, ... } }, statically imported JSON
  defaultNS: "common", fallbackLng: "he", supportedLngs: ["he", "en"],
  detection: { order: ["localStorage", "navigator"], caches: ["localStorage"], lookupLocalStorage: "i18n-language" },
  interpolation: { escapeValue: false },
});
export function applyDirection(lang: string) {
  document.documentElement.dir = lang === "he" ? "rtl" : "ltr";
  document.documentElement.lang = lang;
}
// setLang: i18n.changeLanguage(l); applyDirection(l). Usage: t("title", { ns: "settings" })
// API client: send `X-Language` (+ Accept-Language). Server middleware resolves X-Language → Accept-Language → default
```

**Variant D: React Native (Expo), i18next + expo-localization**
```ts
export function deviceLocale(): AppLocale {
  return Localization.getLocales()[0]?.languageCode === "he" ? "he" : "en";
}
void i18next.use(initReactI18next).init({
  resources: { en: { translation: en }, he: { translation: he } },
  lng: deviceLocale(), fallbackLng: "en", interpolation: { escapeValue: false },
}); // synchronous at module load, so the first frame is already translated

I18nManager.allowRTL(true);
export function syncLocale(setting: "system" | AppLocale) {
  const target = setting === "system" ? deviceLocale() : setting;
  if (i18next.language !== target) void i18next.changeLanguage(target);
  const rtl = target === "he";
  if (I18nManager.isRTL !== rtl) { I18nManager.forceRTL(rtl); return { restartNeeded: true }; }
  return { restartNeeded: false };
}
// Call after persisted settings hydrate AND from the settings screen; show a "restart to apply" notice
export const rtlFlip = I18nManager.isRTL ? { transform: [{ scaleX: -1 }] } : {}; // for arrows and chevrons
```

**Variants E/F: hand-rolled typed dictionary (no library)**
```ts
// he.ts is the source of truth
export const he = { "nav.home": "בית", "items.count": "{n} פריטים" } as const;
// en.ts: compile-time parity, so a missing key is a type error
export const en: Record<keyof typeof he, string> = { "nav.home": "Home", "items.count": "{n} items" };
export function t(locale: AppLocale, key: keyof typeof he, vars?: Record<string, string | number>) {
  const s = (locale === "en" ? en[key] : he[key]) ?? he[key] ?? key;
  return vars ? s.replace(/\{(\w+)\}/g, (_, k) => String(vars[k] ?? "")) : s;
}
// Server reads the `locale` cookie; the client toggle sets the cookie and reloads
// (SPA variant: a React context holding {lang, dir, t, setLang}, persisted to localStorage)
```

**Catalog parity check (lint-time script; pick this or a unit test)**
```js
// scripts/check-i18n-parity.mjs, run from "lint": "eslint && node scripts/check-i18n-parity.mjs"
const flat = (o, p = "") => Object.entries(o).flatMap(([k, v]) =>
  v && typeof v === "object" ? flat(v, `${p}${k}.`) : [`${p}${k}`]);
const a = new Set(flat(en)), b = new Set(flat(he));
const missingHe = [...a].filter(k => !b.has(k)), missingEn = [...b].filter(k => !a.has(k));
if (missingHe.length || missingEn.length) { console.error({ missingHe, missingEn }); process.exit(1); }
```

**Optional: admin-editable runtime translation overrides (layered on variant A or B)**
```ts
// Storage: one doc per flattened key: translations/{dot.path.key} = { values: { en?, he? }, updatedAt, updatedBy }
// Pure merge: an override for a key no longer in the static catalog is ignored, so a stale doc can't bring back a deleted key
export function mergeMessageOverrides(base: Messages, overrides: Record<string, string>) {
  const baseFlat = flattenMessages(base), merged = { ...baseFlat };
  for (const [k, v] of Object.entries(overrides)) if (k in baseFlat) merged[k] = v;
  return unflattenMessages(merged);
}
// Server read is cached across serverless instances (not an in-memory Map):
export const getCachedOverrides = (locale) => unstable_cache(
  () => readOverridesFromDb(locale).catch(() => ({})),   // on failure, serve the static defaults
  ["translation-overrides", locale], { tags: ["translations"], revalidate: 300 })();
// After an admin save, a Server Action calls updateTag("translations") (Next 16) for read-your-own-writes.
// That Server Action must verify an ID token plus the admin role itself and never trust a client-supplied uid.
```
The admin UI imports both static catalogs and flattens them into rows grouped by namespace, with search.
Each locale gets its own input with its own `dir`, plus save and reset (reset deletes `values.<locale>`).
This covers UI strings only. Localized **database content** uses inline `_he` fields instead (see Gotchas).

**Formatting helpers**
```ts
export const formatDate = (d: Date, l: AppLocale) =>
  new Intl.DateTimeFormat(intlTag(l), { dateStyle: "medium", numberingSystem: "latn" }).format(d);
// Keep digits/countdowns LTR inside an RTL sentence:
export const ltrIsolate = (s: string) => `⁦${s}⁩`;
t("timeLeft", { time: ltrIsolate("04:59") });
```

**Mixed-direction user content (chat apps, any UI that shows text users wrote)**

The UI locale doesn't tell you the direction of user text. In a Hebrew UI an English message must
stay LTR, and in an English UI a Hebrew message must stay RTL.
```tsx
// dir="auto" on every container of user text: message bodies, quotes, transcripts, AI output, inputs/textareas
<div className="whitespace-pre-wrap" dir="auto">{msg.body}</div>
<textarea dir="auto" … />

// Number-only strings (phones, "+15551234567") have no letters, so dir="auto" resolves to the
// parent's direction in current Chrome: under RTL "+1555…" renders as "1555…+". Force LTR for those.
export const textDir = (s?: string | null): "auto" | "ltr" => (s && /\p{L}/u.test(s) ? "auto" : "ltr");

// On full-width blocks (headers, list rows), put the direction on an inner <bdi>, not on the block.
// dir on the block also flips its text-align, so a Latin name jumps to the far left of an RTL header.
<h2 className="truncate"><bdi dir={textDir(title)}>{title}</bdi></h2>
<span className="font-mono" dir="ltr">{phone}</span>   // phones, IDs, pairing codes, JIDs
```

**Hebrew full-text search (SQLite FTS5 `unicode61`)**

Hebrew attaches one-letter prefixes to the next word: ו "and", ה "the", ב "in", כ "as", ל "to",
מ "from", ש "that". `unicode61` indexes `ובבית` as one token, so a search for `בית` misses it.
Niqqud (vowel points) is also a problem: `unicode61` treats them as separators and splits a
pointed word into pieces. Add a hidden indexed column with prefix-stripped forms, computed in app
code, and strip niqqud from queries:
```sql
CREATE VIRTUAL TABLE search_fts USING fts5(
  doc_id UNINDEXED, content, terms,           -- terms: extra Hebrew forms, never displayed
  tokenize = 'unicode61 remove_diacritics 2'
);
```
```go
const hebrewPrefixes = "והבכלמש"

func isHebrewLetter(r rune) bool { return r >= 'א' && r <= 'ת' }

// StripNiqqud drops U+0591–U+05C7 marks (keeping maqaf ־, which becomes a space).
func StripNiqqud(s string) string { /* … */ }

// ExpandHebrew: for each Hebrew word, the forms left after removing up to two leading prefix
// letters, keeping at least 3 letters: "ובבית" -> "בבית בית".
func ExpandHebrew(s string) string {
	var terms []string
	for _, w := range strings.FieldsFunc(StripNiqqud(s), func(r rune) bool { return !isHebrewLetter(r) }) {
		rs := []rune(w)
		for i := 0; i < 2 && len(rs)-(i+1) >= 3 && strings.ContainsRune(hebrewPrefixes, rs[i]); i++ {
			terms = append(terms, string(rs[i+1:]))
		}
	}
	return strings.Join(terms, " ")
}
// index:  INSERT INTO search_fts (doc_id, content, terms) VALUES (?, ?, ExpandHebrew(content))
// query:  ... WHERE search_fts MATCH ?   with StripNiqqud(userQuery)
// snippet(search_fts, <content column>, …) still highlights the original text.
```
Because `terms` is computed in Go, a SQL-only migration can't backfill it. Have the migration
recreate the table and insert a marker row, then rebuild the index from the source tables in
app code right after migrations run, and drop the marker in the same transaction (idempotent on restart).

**LLM prompts in a bilingual app**

Don't tie AI output to the UI locale. Tell the model to write in the conversation's language, and
for mixed chats, in the language of the latest messages. Ask it to "write natural modern Hebrew and
keep names and English terms as written", or it tends to transliterate English names into Hebrew
letters. Transcription works the same way: ask for the original language, and store the
detected BCP-47 code next to the text instead of translating.

### Env vars
None. Locale config is code, not environment.

### Gotchas
- **next-intl without middleware (variant A):** you must call `setRequestLocale(locale)` in the `[locale]` layout, and on any page you want rendered statically, because nothing else sets the request locale. The bare `/` redirect and the `NEXT_LOCALE` cookie writer replace what the middleware used to do. One adopter dropped middleware because Next 16 `proxy.ts` always runs on the Node runtime, which a tried edge adapter couldn't support. Without middleware, **unprefixed deep links such as `/settings` return 404**: Next.js doesn't allow a `[...rest]` catch-all next to `[locale]`, so the stray segment is treated as an invalid locale. Every internal link must include the locale prefix.
- **Put `<html>`/`<body>` in the `[locale]` layout**, not the root layout, so `lang`/`dir` come from the URL.
- **Detector conflicts (variant C):** if your own React language context also reads localStorage and defaults to `he`, then calls `changeLanguage`, it overrides `i18next-browser-languagedetector`, and the browser language is silently never used. Keep one source of truth.
- **Pass the bare key with the namespace (i18next):** `t("title", { ns: "settings" })`, not `t("settings.title", { ns: "settings" })`. Check that a key exists in both locales before using it; agent-written code tends to invent plausible keys like `common.cancel` that don't exist.
- **React Native RTL needs a reload:** `I18nManager.forceRTL` only flips the layout after the JS bundle restarts. Strings change immediately. Tell the user to restart, and run `syncLocale` again after the settings store hydrates so an interrupted change still converges. Absolute-positioned UI (switch knobs) and directional glyphs (arrows, chevrons) don't mirror automatically; invert or `scaleX: -1` them.
- **Hebrew plurals and strict parity:** Hebrew uses CLDR `_one`/`_two`/`_many`/`_other`. A strict "identical key sets" parity test forces English to carry padding `_two`/`_many` keys. Either accept the padding or make the parity check plural-aware.
- **Safari/WebKit RTL horizontal scroll:** an `overflow-x: auto` row under `dir="rtl"` starts scrolled to the opposite end from Chrome and Firefox, which can hide the active tab. Call `activeEl.scrollIntoView({ inline: "nearest" })` on mount and on change, and don't rely on the default `scrollLeft`.
- **Use Tailwind logical utilities only** (`ms/me/ps/pe/start/end`), and `rtl:` variants for transforms and gradients. One adopter enforces zero physical `ml/mr/pl/pr/left/right` utilities by convention, and it removes most RTL bugs.
- **Explicit locale in formatting:** bare `toLocaleString()` / `toLocaleDateString()` uses the browser locale, not the app's. Pass `he-IL` or `en-US`. Add `numberingSystem: "latn"` to keep Western digits. `he-IL` gives 24-hour time.
- **Keep strings in the catalog:** domain utilities with inline `"he-IL" ? "נותרו X ימים" : "X days left"` strings bypass translation overrides and the parity check.
- **Localized database content** is a separate concern from the UI catalog: store optional `<field>_he` siblings (`title_he`, `name_he`) and pick at render (`locale === "he" && doc.name_he ? doc.name_he : doc.name`). Search and matching should check both fields.
- **Stored-but-unused preferences are a recurring smell.** Across adopters, a profile `language` field, a server `req.language`, and a system-config `locale` were each persisted and never read. If you store it, make sure something reads it. Otherwise don't store it.
- **When renaming the localStorage/cookie key, keep the legacy name** or migrate it. Changing it silently resets every existing user's language.
- **Monospace labels and Hebrew:** most monospace UI fonts (IBM Plex Mono, SF Mono, JetBrains Mono) have no Hebrew letters, so the browser falls back to a wide monospace Hebrew face that looks letter-spaced. Put a proportional Hebrew face in the mono stack (`"IBM Plex Mono", "IBM Plex Sans Hebrew", ui-monospace, …`). Also turn off `tracking-*` for small-caps-style labels under RTL (`[dir="rtl"] :is(.tracking-wider, .tracking-widest) { letter-spacing: normal }`). Hebrew has no capitals, and the spacing breaks up the word.
- **Hand-rolled SPA locale switch without a reload:** formatters outside React (`Intl.DateTimeFormat` cached at module scope) don't re-render when the locale changes. Either key the app subtree by locale (`<div key={locale} className="contents">`, which remounts everything, an acceptable cost for a rare toggle) or pass the locale into every formatter. `Intl.RelativeTimeFormat(tag, { numeric: "auto" }).format(0 | -1, "day")` gives "today"/"yesterday" ("היום"/"אתמול") without catalog keys.
- **Directional glyphs in text** (`←` back, `↩` reply) don't mirror. Wrap them in `<span className="inline-block rtl:-scale-x-100">`.
- **Hebrew in generated documents:** PDFs (see [react-pdf-renderer](react-pdf-renderer.md)) and government CSV/TXT exports have their own constraints (bidi-run splitting; some exports must be ISO-8859-8-i, because UTF-8 corrupts the Hebrew). The UI locale doesn't decide document language; legal documents are often always Hebrew.

### Playbook confidence: high
Extracted from real source across all adopters (four library-based, three hand-rolled). The mixed-direction content, Hebrew FTS5 and monospace-font sections come from a single adopter, verified with unit tests and in the browser.

## Adoption
Used in **7** repos in this marketplace, all bilingual Hebrew/English with RTL:
- next-intl in two Next.js variants: URL-prefixed without middleware, and cookie-only.
- i18next in a Vite SPA and in an Expo/React Native app (the only one that follows the device locale and falls back to English).
- Three hand-rolled typed dictionaries. One is a chat dashboard (Vite SPA + Go API) that follows the browser language and is the only adopter with Hebrew-aware full-text search.

Only one adopter has admin-editable runtime overrides. Parity is enforced by a lint script, a unit
test, or compile-time types in five adopters; the other two rely on convention or agent rules.
Several further apps are Hebrew-only RTL with no switching.
