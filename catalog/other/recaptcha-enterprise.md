# Google reCAPTCHA Enterprise

- **category**: other
- **provider**: Google
- **reusable**: yes — both verify via the Assessments API using a Firebase Admin-derived access token (no separate secret key actually needed despite `.env.example` suggesting one). Candidate for `@marketplace/recaptcha-enterprise`.
- **docs**: https://cloud.google.com/recaptcha/docs/create-assessment

## Overview
Bot/abuse scoring on a sensitive client action (sign-in, order submission): the client executes an Enterprise reCAPTCHA action to get a token, and the server verifies that token via the Assessments API before trusting the request. Distinct from classic reCAPTCHA v3 (see `recaptcha-v3.md`) — Enterprise assessments are called through the Google Cloud `recaptchaenterprise.googleapis.com` API using an OAuth access token (typically derived from an existing Firebase Admin / GCP service account) rather than a plain secret-key POST to `google.com/recaptcha/api/siteverify`.

## Playbook

### Prerequisites
- A Google Cloud project with the reCAPTCHA Enterprise API enabled and a site key created for it (Enterprise site keys are managed in Cloud Console, not the classic reCAPTCHA admin console).
- A service account (can be the same one used for Firebase Admin SDK — see `firebase-admin-sdk.md`) with permission to call the reCAPTCHA Enterprise Assessments API.
- The client loads Google's reCAPTCHA script and executes the Enterprise action to obtain a one-time token per protected action.

### Setup steps
1. Create an Enterprise key in Google Cloud Console (Security → reCAPTCHA Enterprise), scoped to your domain(s).
2. On the client, load `https://www.google.com/recaptcha/enterprise.js?render=<SITE_KEY>` and call `grecaptcha.enterprise.execute(siteKey, { action })` right before submitting the sensitive action (sign-in, checkout, etc.) to get a short-lived token.
3. Send that token to your server alongside the request it's protecting.
4. On the server, obtain an OAuth access token for a service account that has reCAPTCHA Enterprise access — reuse the Firebase Admin SDK's credential via `getApp().options.credential.getAccessToken()` rather than provisioning a separate key.
5. POST to `https://recaptchaenterprise.googleapis.com/v1/projects/{projectId}/assessments` with the token, site key, and expected action.
6. Check `tokenProperties.valid`, `tokenProperties.action` (must match what you expected), and `riskAnalysis.score` (0.0–1.0, higher = more likely human) against a minimum threshold (0.5 is a common default).
7. Decide fail-open vs fail-closed: both observed implementations fail *open* (treat the action as allowed) if the Assessments API call itself errors, to avoid an outage in Google's API blocking all sign-ins/orders — only an explicit low-score or invalid-token response blocks the action.

### Core pattern
```ts
import "server-only";
import { getApp } from "firebase-admin/app";

async function getAdminAccessToken(): Promise<string> {
  const { access_token } = await getApp().options.credential!.getAccessToken();
  return access_token;
}

export async function verifyRecaptchaEnterprise(
  token: string,
  expectedAction: string
): Promise<boolean> {
  const siteKey = process.env.NEXT_PUBLIC_RECAPTCHA_SITE_KEY;
  if (!siteKey) return true; // recaptcha not configured — don't block
  if (!token) return false;

  try {
    const projectId = process.env.GOOGLE_CLOUD_PROJECT_ID;
    const accessToken = await getAdminAccessToken();
    const res = await fetch(
      `https://recaptchaenterprise.googleapis.com/v1/projects/${projectId}/assessments`,
      {
        method: "POST",
        headers: { Authorization: `Bearer ${accessToken}`, "Content-Type": "application/json" },
        body: JSON.stringify({ event: { token, siteKey, expectedAction } }),
      }
    );
    if (!res.ok) return true; // fail open on API errors

    const data = (await res.json()) as {
      riskAnalysis?: { score?: number };
      tokenProperties?: { valid?: boolean; action?: string };
    };
    const valid = data.tokenProperties?.valid ?? false;
    const actionMatch = data.tokenProperties?.action === expectedAction;
    const score = data.riskAnalysis?.score ?? 0;
    return valid && actionMatch && score >= 0.5;
  } catch {
    return true; // fail open on unexpected errors
  }
}
```

### Env vars
`NEXT_PUBLIC_RECAPTCHA_SITE_KEY` (Enterprise site key, safe to expose to the client), `GOOGLE_CLOUD_PROJECT_ID` / `FIREBASE_PROJECT_ID` (project the key + service account belong to). A `RECAPTCHA_SECRET_KEY` sometimes appears in `.env.example` files but is not actually used by the Assessments-API flow — the OAuth access token from the service account is what authorizes the call.

### Gotchas
- Enterprise uses a completely different verification endpoint/auth model than classic reCAPTCHA v3 — don't POST the token to `siteverify` (that's the v2/v3 endpoint) and expect it to work.
- The service-account access token (not a static secret key) is what authorizes the Assessments API call — reuse the Firebase Admin credential instead of managing a second secret.
- Both known implementations fail open when the API call fails or reCAPTCHA isn't configured (`NEXT_PUBLIC_RECAPTCHA_SITE_KEY` unset) — an outage in Google's assessment API should not block core sign-in/checkout flows, only positive detection of bad tokens/low scores should.
- The `action` returned by the Assessments API must be checked against what you expected — otherwise a valid token minted for one action (e.g. `"pageview"`) could be replayed against a different, higher-value action.
- Can double as the provider for Firebase App Check (`ReCaptchaEnterpriseProvider`) at the same time as being used for a standalone server-side Assessment call — these are two separate integration points using the same site key.

### Playbook confidence: high

## Adoption
Used in **2** repo(s) in this marketplace. One adopter applies it as a single bot-check on a
sign-in path; the other uses it in two places — as the attestation provider for Firebase App
Check, and separately as a server-side risk assessment gating a form-submission endpoint against
spam/bot traffic.
