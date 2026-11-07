# Google reCAPTCHA v3

- **category**: other
- **provider**: Google
- **reusable**: yes — a self-contained, no-SDK pattern (load script → execute → verify via classic `siteverify` endpoint) portable to any repo needing bot-scoring on a sensitive form/sign-in action.
- **docs**: https://developers.google.com/recaptcha/docs/v3

## Overview
Classic score-based reCAPTCHA v3, distinct from reCAPTCHA Enterprise (see `recaptcha-enterprise.md`). The client silently executes an action to get a token (no challenge UI), and the server verifies that token via Google's public `siteverify` REST endpoint using a plain secret key — no Cloud project/service-account/OAuth machinery required, unlike Enterprise.

## Playbook

### Prerequisites
- A reCAPTCHA v3 site key + secret key pair from the classic reCAPTCHA admin console (https://www.google.com/recaptcha/admin) — separate from any Enterprise key, and much simpler to provision (no GCP project required).
- A server-only place to hold the secret key.

### Setup steps
1. Register a v3 site at the classic reCAPTCHA admin console, get a site key (public) and secret key (server-only).
2. On the client, lazily load `https://www.google.com/recaptcha/api.js?render=<SITE_KEY>` once, then call `grecaptcha.ready()` followed by `grecaptcha.execute(siteKey, { action })` right before submitting the protected form/action, to obtain a short-lived token. Preload the script early (e.g. on a layout mount) so the token is ready by the time the user submits, rather than adding load latency to the submit path.
3. Send the token alongside the action's normal payload to your API route.
4. On the server, POST `secret` + the token to `https://www.google.com/recaptcha/api/siteverify` as `application/x-www-form-urlencoded`.
5. Check `success === true`, that the returned `action` matches what you expected (prevents a token minted for a low-value action being replayed against a high-value one), and that `score` is above your minimum threshold (0.5 is Google's suggested default midpoint).
6. Make verification a no-op (return true) when the secret key isn't configured, so local/dev environments without a key don't get blocked — but do fail closed (return false) when a token was expected but missing.
7. Gate verification by caller context if relevant — e.g. skip the check entirely for already-privileged/admin callers, and only enforce it for a specific auth path (session-based sign-in) that's actually exposed to abuse.

### Core pattern
```ts
// Server-side verification
const VERIFY_URL = "https://www.google.com/recaptcha/api/siteverify";

export async function verifyRecaptcha(
  token: string | undefined,
  action: string
): Promise<boolean> {
  const secret = process.env.RECAPTCHA_SECRET_KEY;
  if (!secret) return true; // not configured — don't block
  if (!token) return false;

  const response = await fetch(VERIFY_URL, {
    method: "POST",
    headers: { "content-type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({ secret, response: token }),
  });
  if (!response.ok) return false;

  const result = (await response.json()) as {
    success?: boolean;
    score?: number;
    action?: string;
  };
  if (result.success !== true) return false;
  if (result.action && result.action !== action) return false;

  const minScore = Number(process.env.RECAPTCHA_MIN_SCORE ?? 0.5);
  return typeof result.score !== "number" || result.score >= minScore;
}
```

```ts
// Client-side token acquisition
declare global {
  interface Window {
    grecaptcha?: {
      ready(callback: () => void): void;
      execute(siteKey: string, options: { action: string }): Promise<string>;
    };
  }
}

const SITE_KEY = process.env.NEXT_PUBLIC_RECAPTCHA_SITE_KEY;
let scriptPromise: Promise<void> | undefined;

function loadScript(siteKey: string): Promise<void> {
  if (scriptPromise) return scriptPromise;
  scriptPromise = new Promise((resolve, reject) => {
    if (window.grecaptcha) return resolve();
    const script = document.createElement("script");
    script.src = `https://www.google.com/recaptcha/api.js?render=${siteKey}`;
    script.onload = () => resolve();
    script.onerror = () => {
      scriptPromise = undefined;
      reject(new Error("Failed to load reCAPTCHA"));
    };
    document.head.appendChild(script);
  });
  return scriptPromise;
}

export function preloadRecaptcha(): void {
  if (!SITE_KEY) return;
  loadScript(SITE_KEY).catch(() => {});
}

export async function getRecaptchaToken(action: string): Promise<string | undefined> {
  if (!SITE_KEY) return undefined;
  await loadScript(SITE_KEY);
  await new Promise<void>((resolve) => window.grecaptcha!.ready(resolve));
  return window.grecaptcha!.execute(SITE_KEY, { action });
}
```

### Env vars
`NEXT_PUBLIC_RECAPTCHA_SITE_KEY` (public site key), `RECAPTCHA_SECRET_KEY` (server-only secret key), `RECAPTCHA_MIN_SCORE` (optional, defaults to 0.5 if unset).

### Gotchas
- Don't confuse this with reCAPTCHA Enterprise — v3 verifies against the plain public `siteverify` endpoint with a shared secret key; Enterprise calls a Cloud API authenticated with an OAuth access token from a service account. They use different key pairs and are not interchangeable.
- Always check the returned `action` against what you expected — otherwise a token minted for one action can be replayed to pass verification on a different, more sensitive action.
- Preload the script (e.g. on page/layout mount) rather than only on submit — `grecaptcha.execute()` needs the script loaded and `grecaptcha.ready()` to have fired, which adds latency if done cold on submit.
- Treat an unconfigured secret key as "verification disabled" (return true), not as a hard failure — this keeps local dev and preview environments usable without provisioning real keys, at the cost of allowing all traffic through when unconfigured (only acceptable because it's an opt-in defense layer, not the sole line of defense).
- Not all callers necessarily need the check — e.g. an already-authenticated admin session may bypass it entirely, with the check reserved for a specific higher-abuse-risk auth path.

### Playbook confidence: medium

## Adoption
Used in **1** repo(s) in this marketplace. Applied as a score-based bot-check on a session
sign-in path, skipped for already-privileged callers and enforced only on the regular
session-based sign-in flow, with the client script preloaded ahead of submission.
