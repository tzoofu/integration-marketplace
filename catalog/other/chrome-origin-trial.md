# Chrome Origin Trial (WebMCP)

- **category**: other
- **provider**: Google Chrome (Origin Trials console)
- **reusable**: no — the registered token is bound to one specific production origin and cannot be reused by another repo's domain; the *pattern* (a conditional `<meta httpEquiv="origin-trial">` tag) is portable, but each adopting app needs its own token.
- **docs**: https://developer.chrome.com/origintrials/ and https://developer.chrome.com/docs/web-platform/origin-trials

## Overview
Registers a production origin for a Chrome origin-trial token, which flips on an experimental browser feature flag (here: **WebMCP**, letting the page register client-side tools an in-browser AI agent can call) for visitors on that specific origin during the trial period. Distinct from an actual MCP server implementation — this is purely a dependency on Google Chrome's origin-trial feature-flag program, gating whether the browser exposes the experimental API at all.

## Playbook

### Prerequisites
- A registered account at Chrome's [Origin Trials console](https://developer.chrome.com/origintrials).
- The exact production origin (scheme + host + port) the trial token will be issued for — tokens are origin-locked and do not transfer to a different domain, subdomain, or port.
- Knowledge of which experimental feature/trial you're opting into (e.g. WebMCP) and its trial ID.

### Setup steps
1. Go to Chrome's Origin Trials console and register for the specific trial (e.g. WebMCP).
2. Enter the exact production origin the site is served from; Chrome issues a token scoped to that origin only.
3. Store the token as a public (client-exposed) env var — it's meant to be visible in the page source, unlike a secret.
4. Render the token as a `<meta httpEquiv="origin-trial" content="TOKEN">` tag inside `<head>`, conditionally only when the token env var is set.
5. Confirm the experimental API works on the real production origin — `localhost` typically has these trials enabled by default regardless of a token, so local testing won't validate the token itself.
6. Track the trial's expiration date (origin trials are time-boxed) and plan to either re-register before expiry or drop the flag once the feature ships stable.

### Core pattern
```tsx
// In the root layout / document <head> — conditional origin-trial meta tag.
// Frameworks whose metadata API only emits `name=` (not `http-equiv=`) need
// this tag added directly rather than through the generic metadata object.
{process.env.NEXT_PUBLIC_ORIGIN_TRIAL_TOKEN && (
  <head>
    <meta httpEquiv="origin-trial" content={process.env.NEXT_PUBLIC_ORIGIN_TRIAL_TOKEN} />
  </head>
)}
```

```html
<!-- Plain HTML equivalent, for a non-framework page -->
<meta http-equiv="origin-trial" content="YOUR_TOKEN_HERE">
```

### Env vars
`NEXT_PUBLIC_WEBMCP_ORIGIN_TRIAL_TOKEN` (or equivalent public/client-exposed var holding the origin-trial token) — note it names a specific trial (WebMCP here); a different trial would use a differently-named var.

### Gotchas
- The token is bound to an **exact** origin (scheme + host + port) — it silently does nothing on any other origin, including `www.` vs bare-domain mismatches, a different port, or a preview/staging subdomain. There is no error surfaced when the token doesn't match; the experimental feature simply stays disabled.
- `localhost` is exempt from the origin-trial gate in Chrome for many trials — so local dev "just works" whether or not the token is set/valid, which can mask a broken or expired production token until it's actually deployed.
- Origin trials are time-boxed and expire — track the expiry date separately from the codebase (e.g. in ops docs), since an expired token silently stops working with no visible error, just like a wrong-origin token.
- Omitting the env var entirely is a safe, intentional fallback (feature simply off) — don't treat a missing token as a build error; gate the meta tag's rendering on its presence.
- This flag only controls whether the **browser** exposes the experimental API — it says nothing about whether the app's own client-side code that uses that API (e.g. a WebMCP tool-registration hook) is present or correctly implemented; those are separate, unrelated failure modes.

### Playbook confidence: medium

## Adoption
Used in **1** repo(s) in this marketplace, to opt a production origin into Chrome's WebMCP origin trial so browser-side WebMCP APIs are enabled for site visitors during the trial period.
