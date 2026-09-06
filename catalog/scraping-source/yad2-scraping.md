# Yad2 (scraping source)

- **category**: scraping-source
- **provider**: Yad2 (Israeli classifieds site)
- **reusable**: partial — Yad2-specific parsing (selectors, Hebrew field labels), but the "spoofed-Googlebot-UA Playwright scraper" pattern is directly reusable for any other Radware/bot-wall-protected site. See [facebook-scraping](facebook-scraping.md) for the sibling session-cookie-based pattern.
- **docs**: https://www.yad2.co.il (no public API; site scraped directly)

## Overview
Yad2 is a large Israeli classifieds site (real estate, cars, general marketplace). Its listing pages sit behind Radware Bot Manager, which reliably fingerprints and blocks Playwright-driven Chromium — even with `navigator.webdriver` patched — while leaving its own image CDN completely open. The working approach is to spoof a Googlebot user-agent for page loads, and fall back to a browser bookmarklet + CDN-only fetch when even that fails.

## Playbook

### Prerequisites
- A Playwright-capable browser automation surface (MCP tool, or a local `playwright`/`playwright-core` install with a cached Chromium binary).
- No account, API key, or login needed — everything scraped is public.

### Setup steps
1. Install Playwright and ensure a Chromium binary is cached locally (`npx playwright install chromium`, or rely on an already-cached one under `~/Library/Caches/ms-playwright` on macOS).
2. Do not attempt plain `page.goto()` against the target site — validate first whether the site fronts with a bot-wall (Radware, PerimeterX, Cloudflare, etc.) by loading a listing page headed and headless and diffing the response; if headless is blocked but headed works, or both are blocked, use the spoofed-UA approach below rather than trying to defeat fingerprinting with more stealth flags.
3. If the bot-wall blocks even the spoofed-UA approach on certain sub-resources (e.g. an image CDN with per-session tokens), check whether that resource sits on a *separate*, unprotected hostname (image CDNs are often just a CDN edge with no bot-wall) — if so, split the flow: browser for page HTML/JSON, plain HTTP fetch for assets.
4. Build one self-contained "visit and extract" routine per page type (search-results page, single-listing page) — do not try to persist browser state across multiple automation calls if your automation surface resets its sandbox between calls (true for a Playwright *MCP tool*'s `run_code_unsafe`-style APIs; not an issue for a long-lived Node script using `playwright`/`playwright-core` directly).

### Core pattern
```js
// One fully self-contained "visit and extract" unit — safe to call once per
// page, whether from a long-lived script or a call-by-call automation tool
// that doesn't persist state between invocations.
async function scrapeListingPage(browser, url) {
  const ctx = await browser.newContext({
    userAgent: "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)",
    viewport: { width: 1280, height: 900 },
    locale: "en-US",
  });
  const page = await ctx.newPage();
  await page.goto(url, { waitUntil: "domcontentloaded", timeout: 30_000 });
  await page.waitForTimeout(2000);

  // Dismiss a cookie/consent dialog if present, by a stable data-testid/role
  // selector rather than exact button text (text is locale-dependent).
  try {
    await page.locator('[data-testid="cookie-consent-accept"]').click({ timeout: 3000 });
  } catch { /* not present — fine */ }

  // Prefer a structured embedded-JSON payload (e.g. a Next.js __NEXT_DATA__
  // script tag) over ad-hoc DOM scraping wherever the site ships one — it's
  // far more stable across redesigns than CSS selectors.
  const data = await page.evaluate(() => {
    const nd = document.getElementById("__NEXT_DATA__");
    if (nd) {
      try { return JSON.parse(nd.textContent || "{}"); } catch { /* fall through */ }
    }
    // DOM fallback — key every field off a stable data-testid, never off
    // generated CSS class names or nth-child position.
    const val = (testid) => document.querySelector(`[data-testid="${testid}"]`)?.innerText?.trim() ?? null;
    return {
      title: val("heading"),
      price: val("price"),
      description: val("description"),
    };
  });

  await ctx.close();
  return data;
}

// Fetch a gallery/asset from the site's (frequently unprotected) media CDN
// server-side, once you have real URLs — cheaper and more reliable than
// in-page fetch + base64 chunking, and sidesteps the bot-wall entirely if the
// CDN host is separate from the bot-walled app host.
async function fetchAsset(page, assetUrl) {
  const res = await page.request.get(assetUrl);
  if (!res.ok()) return null;
  return { base64: (await res.body()).toString("base64"), contentType: res.headers()["content-type"] };
}
```

For infinite-scroll result pages, drive scrolling with `page.mouse.wheel()` (a "trusted", OS-level input event) rather than `window.scrollTo()` inside `page.evaluate()` — headless Chromium's lazy-load listeners often only fire on trusted scroll events, and a script-driven `scrollTo` silently produces zero new results.

Rate-limit/anti-bot handling, generically: throttle concurrent contexts (one browser context per concurrent "worker", not one per page), add small `waitForTimeout` pauses between navigations, and treat an unexpected interstitial page title/DOM shape as a hard stop rather than retrying blindly — retrying against a bot-wall usually just escalates the block.

### Env vars
(none — no API key or credentials; scraped via plain browser automation)

### Gotchas
- A page can look fully blocked in the automated browser while a real human-driven session on the same machine/IP loads fine — that's a strong signal of automation/CDP fingerprinting, not an IP-based block, and no amount of extra stealth flags will fix it; changing the UA to a known, allow-listed crawler (Googlebot) is what actually works because the site *wants* search engines to index it.
- If your automation tool resets all in-memory state between separate calls (true of some MCP-style "run code" tools), every browser context you open must be fully created, used, and closed within one call — nothing (not even a variable reference) survives to the next call.
- Don't assume one field maps directly to a "total cost" figure shown in a summary/sticky element — sites sometimes bundle multiple sub-amounts (e.g. base price + fees) into a single displayed total; parse and store the components separately if downstream code needs to recompute the total from parts.
- A label's unit/period can vary per listing (e.g. "monthly" vs "bimonthly" on an otherwise identical field) — always re-read the label text per item rather than assuming a fixed convention.
- A regenerated/minified browser-bookmarklet fallback (for the rare case even the spoofed-UA approach gets blocked) needs to be re-minified after every edit to its source — keep the two files in sync via a documented one-line build command rather than hand-editing the minified copy.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. Consumed as the primary listing source for a scraping pipeline, combining the spoofed-UA Playwright approach with a browser-bookmarklet fallback for cases where even that gets blocked.
