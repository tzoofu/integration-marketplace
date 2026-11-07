# Legal compliance (privacy, terms, accessibility statement, cookie consent): every supported language

- **category**: other
- **provider**: internal (custom). Uses Google Consent Mode v2 for gating GA/Ads, and `@axe-core/playwright` for auditing
- **reusable**: yes. Every web adopter ships the same three legal pages, and the consent store, legal-links footer and locale-parity check are framework-agnostic. They could become a shared `@marketplace/legal` package. The legal prose itself stays per app.
- **docs**: https://developers.google.com/tag-platform/security/guides/consent · https://www.gov.il/he/departments/legalInfo/accessibility-regulations · https://www.gov.il/he/departments/the_privacy_protection_authority · https://github.com/dequelabs/axe-core-npm

This file covers the legal surface an Israel-facing web app needs: a privacy policy, terms, an accessibility statement (הצהרת נגישות, IS 5568 / reg. 35), a cookie banner that really blocks trackers, consent on forms, and baseline accessibility. Every item has to exist **in every language the app supports**, not only the default one. The file has two halves. The **verifier prompt** audits an existing app. The **playbook** implements whatever the audit finds missing.

> Not legal advice. These patterns are engineering scaffolding. Have a lawyer review the final prose, especially the privacy policy (Amendment 13 to the Privacy Protection Law).

## Verifier prompt (paste into a session on the target app)

Paste this whenever the app's legal compliance needs checking: before launch, after adding a third-party script or a form, and **every time a language is added**.

```text
You are my legal compliance verifier. Don't rebuild anything yet.

Step 0: LANGUAGES. Find every locale the app supports: i18n config, messages/* catalogs,
locale constants, the <html lang> value, any language switcher. List them, with the default
and the text direction (rtl/ltr) of each. Every check below runs ONCE PER LOCALE. An item
that exists in one language but not another counts as ⚠️, not ✅.

Then SCAN the codebase and check what legal compliance features we ALREADY have. For each
item, mark it as ✅ EXISTS & OK / ⚠️ EXISTS BUT NEEDS FIX / ❌ MISSING. If an item is missing
or broken, show me how you would build it and wait for my confirmation.

A. Legal pages (check each one in every locale)
- Does /privacy-policy (or /privacy) exist? Does its content match what we ACTUALLY collect?
  Check package.json, analytics, forms, auth providers, storage, and any third-party scripts.
  Is it linked in the footer?
- Does /terms-and-conditions (or /terms) exist, and is it linked?
- Does /accessibility-statement (הצהרת נגישות) exist? It is required in Israel under IS 5568
  and the accessibility regulations (reg. 35). It must state the conformance level, the
  adjustments made, known limitations, and an accessibility coordinator's name, email and phone.
  A Hebrew version is mandatory even if the default language is something else.
- Is the "last updated" date the same across locales? Do the translations say the same thing,
  or is one of them stale?
- Does each page render with the right lang and dir for its locale?

B. Cookies and consent
- Is there a cookie banner? Does it actually BLOCK GA / Meta Pixel / TikTok / AdSense / Hotjar
  until the user accepts? Test the logic: trace when the script tag renders and what the
  Consent Mode default is. Nothing may load before a choice is made.
- Can the user reject as easily as accept? Is the consent saved (and versioned)? Can the user
  reopen settings and withdraw later?
- Is the banner (text, buttons, link to the cookies section) translated into every locale?
- Do we collect only the data we need? List every form and the fields it collects.

C. Forms and data
- Does every form that collects personal data have a privacy-policy link (pointing at the
  current locale's page) and an explicit, unticked consent checkbox?
- Is consent validated ON THE SERVER, and stored with a timestamp and policy version?
- Is the data validated, and kept out of console.log and server logs?
- Do data-subject rights work: export, deletion, a contact for requests? Do deletion and
  export endpoints check that the requester owns the data?

D. Accessibility (the main source of lawsuits in Israel). Run in every locale
- Run an axe-core check: color contrast, alt text, heading order, keyboard navigation,
  aria-labels, the lang attribute, and dir correctness in RTL.
- Is there a skip-to-content link, and is its label translated?
- Is zoom not locked? Are focus styles visible? Is prefers-reduced-motion respected?

E. Security, third parties and images
- List all third-party scripts (GA, FB, YouTube, chat widgets, fonts, maps, captcha). Is each
  one mentioned in the privacy policy, in every locale?
- Is the site HTTPS-only? Is there any mixed content?
- Are there images in /public or /assets with no alt text, or with a possible copyright risk?

Output format (one line per item, per locale where it applies):
[✅/⚠️/❌] Feature Name [locale] - File: src/... - Issue: ... -> Suggested fix: [one line] -> Do you want me to fix it? [y/n]

At the end, give me:
1. A summary table: rows are the checklist items, columns are the locales.
2. The list of missing items. Ask me which ones to implement.

Start now. Scan first, don't code yet.
```

## Playbook

### Prerequisites
- An i18n setup with a single typed locale list. See [i18n-language-support](i18n-language-support.md).
- A footer or nav component that appears on every public page.
- If you use analytics or ads: their IDs, and the full list of third parties (needed for the privacy policy and the consent version).
- `@axe-core/playwright` + `playwright` (dev) for the audit script.
- Real values for owner/legal entity name, contact email, accessibility coordinator, and the reply SLAs. Never ship a personal name or email as a code default.

### Setup steps
1. **Decide where legal content lives** (see Core pattern, "Content storage"):
   - Static per-locale files, versioned with code. This is the simplest option.
   - Admin-editable DB documents keyed by `(slug, locale)`.
   - The hybrid recommended below: static prose, with an admin-editable config for the facts that change (owner, email, dates, SLAs).
2. **Create the pages for every locale**: `privacy`, `terms`, `accessibility-statement`. Use one route that reads the locale (`/[locale]/privacy`, or a cookie). Don't create separate routes per language.
3. **Add a legal-docs parity check** that fails CI when any `(doc × locale)` pair is missing. Reuse the i18n catalog-parity mechanism.
4. **Add `LegalLinks`** (privacy, terms, accessibility, cookie settings) to the footer *and* to the mobile menu.
5. **Implement the consent store and banner.** Make the default fully denied, and put every tracker behind a conditional render plus `gtag('consent','default', denied)`.
6. **List cookies** (name, purpose, lifetime) in the privacy policy, in a `#cookies` section. The banner links to it.
7. **Forms**: an unticked consent checkbox linking to the locale's privacy page, a server schema that requires it, and `{consentAt, policyVersion}` stored with the submission.
8. **Data-subject rights**:
   - An authenticated export and delete for signed-in users.
   - For anonymous data: an email request flow, or a verified (OTP) self-service flow.
9. **Accessibility baseline**:
   - `lang` and `dir` on `<html>` per locale.
   - A translated skip link.
   - Visible focus styles.
   - No `maximum-scale` / `user-scalable=no`.
   - `prefers-reduced-motion` respected.
10. **Add the axe audit script** and run it per locale (Core pattern).
11. **When adding a language:**
    - Add all three legal docs, the banner strings, the skip-link label and the `LegalLinks` labels.
    - Rerun the verifier prompt.
    - Keep the Hebrew accessibility statement no matter what the new default is.

### Core pattern

**Content storage: hybrid (static prose per locale + admin-editable facts)**
```ts
// lib/legal/config.ts
export interface LegalConfig {
  ownerName: string;
  contactEmail: string;
  a11yCoordinatorName: string;
  a11yCoordinatorPhone: string;
  lastUpdated: string;                 // YYYY-MM-DD; doubles as policy/terms version
  dataRequestDays: number;
  accessibilityResponseDays: number;
}
export const DEFAULT_LEGAL: LegalConfig = {
  ownerName: "", contactEmail: "", a11yCoordinatorName: "", a11yCoordinatorPhone: "",
  lastUpdated: "", dataRequestDays: 30, accessibilityResponseDays: 14,
};
// per-field fallback so a partial/corrupt stored doc never blanks the page
export const mergeLegalConfig = (stored: Partial<LegalConfig> | null): LegalConfig =>
  Object.fromEntries(
    Object.entries(DEFAULT_LEGAL).map(([k, v]) => [k, (stored as any)?.[k] ?? v]),
  ) as LegalConfig;
// render "" as a visible placeholder so a missing fact is obvious in review, not silently empty
export const orPlaceholder = (v: string, locale: AppLocale) =>
  v || (locale === "he" ? "[לא הוגדר]" : "[not set]");
export const formatLegalUpdated = (iso: string, locale: AppLocale) =>
  new Intl.DateTimeFormat(intlTag(locale), { month: "long", year: "numeric", timeZone: "UTC" })
    .format(new Date(iso));
```

```tsx
// app/[locale]/privacy/page.tsx: one route, one content module per locale
import he from "@/legal/privacy.he";
import en from "@/legal/privacy.en";
const DOCS = { he, en } satisfies Record<AppLocale, LegalDoc>;   // compile-time parity

export default async function Privacy({ params }: { params: Promise<{ locale: AppLocale }> }) {
  const { locale } = await params;
  const cfg = mergeLegalConfig(await getStoredLegalConfig());
  const Doc = DOCS[locale];
  return (
    <main id="main-content" tabIndex={-1} lang={locale} dir={dirFor(locale)}>
      <Doc cfg={cfg} updated={formatLegalUpdated(cfg.lastUpdated, locale)} />
    </main>
  );
}
```

Admin-editable variant: store `legal_docs/{slug}_{locale}` = `{title, content, updatedAt}`. The admin action (session-checked) whitelists the slug and the locale, requires both fields, then revalidates the route. Render the content as plain text (`white-space: pre-wrap`) or as sanitized markdown, never as raw HTML. Swap `{{EMAIL}}` / `{{PHONE}}` tokens with the current contact settings when rendering.

**Legal-docs parity check (CI)**
```ts
// scripts/check-legal-parity.ts
import { LOCALES } from "@/i18n/config";
const DOCS = ["privacy", "terms", "accessibility-statement"] as const;
const missing = DOCS.flatMap(d => LOCALES.filter(l => !existsSync(`legal/${d}.${l}.tsx`)).map(l => `${d}.${l}`));
const bannerKeys = ["cookies.title", "cookies.body", "cookies.accept", "cookies.reject",
  "cookies.settings", "a11y.skipToContent", "legal.privacy", "legal.terms", "legal.a11y"];
for (const l of LOCALES) {
  const msgs = flatten(require(`../messages/${l}.json`));
  for (const k of bannerKeys) if (!msgs[k]) missing.push(`messages/${l}.json:${k}`);
}
if (missing.length) { console.error("Missing legal content:", missing); process.exit(1); }
```

**Consent store (pure, unit-testable)**
```ts
// lib/cookie-consent.ts
export const CONSENT_COOKIE = "cookie-consent";
export const CONSENT_VERSION = "v1";       // bump when the set of third parties changes -> re-ask everyone
export const CONSENT_MAX_AGE = 60 * 60 * 24 * 365;
export type ConsentChoice = "all" | "analytics" | "necessary";

export function parseConsent(raw?: string | null): ConsentChoice | null {
  if (!raw) return null;
  const [v, c] = decodeURIComponent(raw).split(":");
  return v === CONSENT_VERSION && ["all", "analytics", "necessary"].includes(c) ? (c as ConsentChoice) : null;
}
export const serializeConsent = (c: ConsentChoice) => `${CONSENT_VERSION}:${c}`;
// opt-IN: no choice yet (null) means denied
export const analyticsAllowed = (c: ConsentChoice | null) => c === "all" || c === "analytics";
export const adsAllowed = (c: ConsentChoice | null) => c === "all";
export const consentModeFor = (c: ConsentChoice | null) => {
  const a = analyticsAllowed(c) ? "granted" : "denied";
  const ads = adsAllowed(c) ? "granted" : "denied";
  return { analytics_storage: a, ad_storage: ads, ad_user_data: ads, ad_personalization: ads };
};
```

**Banner and withdrawal (client)**
```tsx
const setChoice = (next: ConsentChoice) => {
  const secure = location.protocol === "https:" ? "; Secure" : "";
  document.cookie = `${CONSENT_COOKIE}=${encodeURIComponent(serializeConsent(next))}; Max-Age=${CONSENT_MAX_AGE}; Path=/; SameSite=Lax${secure}`;
  window.gtag?.("consent", "update", consentModeFor(next));
  if (!analyticsAllowed(next)) clearCookies(/^_ga/);   // expire on host, .host and parent domain
  setState(next); setSettingsOpen(false);
};
// Show when choice === null || settingsOpen. "Reject" is a button with the same weight as "Accept"
// (no dark pattern). A "Cookie settings" button in LegalLinks calls openSettings().
// All strings come from t("cookies.*"); the policy link is the CURRENT locale's /privacy#cookies.
<section role="region" aria-label={t("cookies.title")} dir={dirFor(locale)}>…</section>
```

**Hard gate + Consent Mode v2 default**
```tsx
// layout.tsx (server) reads the cookie, so there is no banner flash and no script before hydration
const initialChoice = parseConsent((await cookies()).get(CONSENT_COOKIE)?.value);

export function GoogleAnalytics() {
  const { choice } = useCookieConsent();
  if (!GA_ID || !analyticsAllowed(choice)) return null;   // conditional render = nothing loads
  return (<>
    <Script id="ga-init" strategy="afterInteractive">{`
      window.dataLayer=window.dataLayer||[];function gtag(){dataLayer.push(arguments);}window.gtag=gtag;
      gtag('consent','default',${JSON.stringify({ ...consentModeFor(null), wait_for_update: 500 })});
      gtag('consent','update',${JSON.stringify(consentModeFor(choice))});
      gtag('js',new Date()); gtag('config','${GA_ID}');`}</Script>
    <Script src={`https://www.googletagmanager.com/gtag/js?id=${GA_ID}`} strategy="afterInteractive" />
  </>);
}
// Pixels / TikTok / AdSense / chat widgets: the same `if (!allowed(choice)) return null` wrapper.
```

**Form consent: client checkbox + server enforcement + record**
```tsx
<label className="flex items-start gap-2">
  <input type="checkbox" name="privacyAccepted" required checked={agreed} onChange={e => setAgreed(e.target.checked)} />
  <span>{t("forms.consent")} <a href={`/${locale}/privacy`} target="_blank" rel="noopener">{t("legal.privacy")}</a></span>
</label>
<button disabled={!agreed || busy}>{t("forms.submit")}</button>
```
```ts
const Schema = z.object({
  name: z.string().trim().min(1).max(100),
  phone: z.string().regex(/^0\d{8,9}$/),
  privacyAccepted: z.literal(true),           // the server rejects anything without consent
});
const data = Schema.parse(await req.json());   // never console.log(data)
await save({ ...data, consentAt: new Date().toISOString(), policyVersion: legal.lastUpdated, locale });
```

**Terms-acceptance gate (signed-in apps)**
```ts
// on accept: profile.acceptedTermsVersion = TERMS_VERSION; profile.acceptedTermsAt = now
if (profile?.acceptedTermsVersion !== TERMS_VERSION) redirect(`/${locale}/accept-terms`);
// bumping TERMS_VERSION (= lastUpdated) re-prompts everyone, in their own locale
```

**LegalLinks + skip link**
```tsx
const LINKS = ["privacy", "terms", "accessibility-statement"] as const;
export function LegalLinks({ locale }: { locale: AppLocale }) {
  const t = useT();
  return (
    <nav aria-label={t("legal.nav")}>
      {LINKS.map(s => <Link key={s} href={`/${locale}/${s}`}>{t(`legal.${s}`)}</Link>)}
      <CookieSettingsButton />
    </nav>
  );
} // render in the footer AND the mobile drawer

// layout.tsx: first focusable element
<a href="#main-content" className="sr-only focus:not-sr-only focus:fixed focus:top-2 focus:start-2 focus:z-[100]">
  {t("a11y.skipToContent")}
</a>
```

**Accessibility statement: required sections (reg. 35)**
1. A commitment paragraph.
2. Conformance: IS 5568 (WCAG 2.0 AA), the target level (2.1 AA), the test method (axe plus manual keyboard and screen-reader testing, naming browsers and devices), and the date of the last review.
3. The adjustments made, as a list (keyboard nav, skip link, contrast, alt text, RTL, zoom, reduced motion…).
4. Known limitations, stated honestly (third-party embeds, user-uploaded images, maps…).
5. The accessibility coordinator: name, email, phone, the reply SLA in working days, and what to include in a report.
6. Physical-premises accessibility, if the business serves customers in person.

Write it in Hebrew, plus every other supported locale.

**Privacy policy: required sections**
Controller and contact · data collected (a table: field, purpose, legal basis) · data about non-users, if any · purposes · processors and storage location (every third party, matching the scripts list) · sharing · cookies (`#cookies`: name, type, purpose, lifetime, plus a localStorage note) · retention · rights (access, correction, deletion, export, the reply SLA, the right to complain to the Privacy Protection Authority) · security · children · changes, with the last-updated date.

**Terms: sections**
Service description · eligibility · acceptable use · user content and IP · notice-and-takedown · disclaimer and liability limit · termination and account deletion · cancellation policy (consumer protection) where something is sold · governing law and jurisdiction (Israel) · changes · contact.

**axe audit, per locale**
```ts
// scripts/a11y-audit.ts: run with `tsx scripts/a11y-audit.ts <baseUrl>`
import { chromium } from "playwright";
import AxeBuilder from "@axe-core/playwright";
const PAGES = ["/", "/privacy", "/terms", "/accessibility-statement"];
const browser = await chromium.launch();
const ctx = await browser.newContext();
// pre-set the consent cookie so the banner doesn't cover the page under test
await ctx.addCookies([{ name: "cookie-consent", value: "v1%3Anecessary", url: base }]);
let bad = 0;
for (const locale of LOCALES) for (const p of PAGES) {
  const page = await ctx.newPage(); await page.goto(`${base}/${locale}${p}`);
  const { violations } = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa", "wcag21a", "wcag21aa"]).analyze();
  const serious = violations.filter(v => v.impact === "serious" || v.impact === "critical");
  serious.forEach(v => console.log(`[${locale}] ${p} ${v.id}: ${v.nodes.length} nodes`));
  bad += serious.length;
}
await browser.close(); process.exit(bad ? 1 : 0);
```

**Non-web (chat-bot) variant**
- An explicit opt-in with accept, decline (the bot still works, nothing is saved) and "view policy" buttons, in the user's language.
- A `/consent` command to change the choice later.
- Store `{userHash, consentGiven, consentAt, policyVersion}`. Read or write preferences only while consent is true.
- `/privacy` (show stored data), `/dataexport`, `/cleardata` (delete preferences *and* the consent doc).
- An inactivity retention TTL.

### Env vars
- `NEXT_PUBLIC_GA_ID` (or the equivalent analytics ID); any pixel or ads IDs
- `NEXT_PUBLIC_ADS_ENABLED` (kill switch, if ads are used)
- Bot variant: `USER_ID_SALT`, `DATA_RETENTION_DAYS`

### Gotchas
- **Make the default fully denied, not opt-out.** A common mistake is to load GA with `analytics_storage: granted` and gate only the ads. Under Amendment 13 and GDPR-style consent, analytics needs opt-in too. Conditional render *and* a denied Consent Mode default are both needed. Consent Mode alone still loads `gtag.js` and sends cookieless pings.
- **Look for SDKs that start analytics on import.** For example, initializing Firebase Analytics in the client bootstrap starts collection with no banner. Move `getAnalytics()` behind the consent gate. Grep for every SDK `init`, not just `<Script>` tags.
- **Version the consent.** A plain `"granted"` value in localStorage with no version means that adding a new tracker later never re-asks anyone. Store `vN:<choice>` and bump the version when the third-party list changes.
- **Make withdrawal reachable.** Without a "Cookie settings" link the banner never comes back, and withdrawal must be as easy as giving consent. When consent is revoked, also expire the `_ga*` cookies on the host, `.host` and the parent domain. A `gtag('consent','update')` call alone doesn't delete them.
- **Read the consent cookie on the server.** That's what avoids a banner flash and a tracker race on the first paint. localStorage-only consent can't be read on the server.
- **A checkbox checked only in the browser isn't consent.** The server must reject a payload without `privacyAccepted: true` and store the timestamp and policy version. Passive text ("by submitting you agree…") is weaker than a checkbox.
- **Self-service deletion and export must prove ownership.** Use the session for signed-in users, and an OTP to the phone or email for anonymous submitters. An endpoint that deletes "everything matching this phone number" without verification lets anyone erase someone else's data.
- **Locale drift.** The usual failure is a Hebrew policy that's current and an English one that's three tracker-changes behind. Keep one `lastUpdated` shared by all locales. Update every locale in the same commit, and have the parity script catch missing files. It can't catch stale content, so the verifier prompt compares them.
- **The Hebrew accessibility statement is mandatory** for an Israeli service even when the UI defaults to English. A bilingual single page (`<section lang="en" dir="ltr">` + `<section lang="he" dir="rtl">`) is acceptable when there's no full i18n.
- **Mixed-direction text in RTL legal pages.** Wrap emails, phone numbers, URLs and cookie names in `dir="ltr"` / `<bdi>`, or they render scrambled.
- **Don't ship real personal data as defaults.** Owner names and emails in `DEFAULT_*` constants end up in the public bundle and in forks. Use empty defaults with a visible `[not set]` placeholder.
- **Accessibility overlays and widgets don't make a site compliant**, and they're a known lawsuit trigger in themselves. Fix the markup instead.
- **Don't lock zoom.** `user-scalable=no` / `maximum-scale=1` in the viewport fails WCAG 1.4.4.
- **Small-business exemptions exist** under the accessibility regulations, but they are narrow. Still publish a statement explaining the exemption rather than none.
- **Pre-set the consent cookie in the axe audit,** or the banner overlays every page and skews the results. Run the audit in each locale *and* in dark mode, if the app has one.
- **Keep `console.log(formData)` out of the code.** Logs end up in hosting dashboards, which are processors not listed in the policy.

### Playbook confidence: high
Extracted from several real implementations: static TSX per locale, admin-editable DB docs, a hybrid config with static prose, a three-tier Consent Mode banner, a minimal accept/reject gate, a versioned terms-acceptance gate, an axe Playwright audit, and a chat-bot opt-in. The per-locale parity script and the server-side consent enforcement are recommended hardening that combines patterns seen separately. They have not yet been observed together in one adopter.

## Adoption
Used in **6** repos in this marketplace: 5 web apps with privacy and terms pages (4 of them also have an accessibility statement) and 1 chat bot with an opt-in consent flow.
- Content storage varies: static per-locale TSX, admin-editable DB documents per slug, and admin-editable legal facts with static prose.
- Cookie consent appears in two shapes, a versioned three-tier Consent Mode v2 banner and a minimal accept/reject gate. Apps with no analytics or ads skip the banner and say so in the policy.
- Legal content is localized in he + en in one adopter. The others are single-language, with one bilingual accessibility statement.
