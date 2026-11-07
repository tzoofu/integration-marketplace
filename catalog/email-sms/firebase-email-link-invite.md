# Firebase Identity Toolkit — email sign-in link invites

- **category**: email-sms
- **provider**: Firebase / Google (Identity Toolkit REST API)
- **reusable**: partial — same direct-REST-call pattern in both repos (a genuinely different integration shape from the OAuth popup flow in `firebase-auth.md` despite sharing a provider), but single-purpose enough that a shared `sendInviteSignInLink()` helper is a plausible small extraction.
- **docs**: https://firebase.google.com/docs/reference/rest/auth#section-send-email-sign-in-link

## Overview
Server-side calls to `https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode` send Firebase's passwordless "email sign-in link" as a team-invite/onboarding email, doubling as both the invite and the recipient's first sign-in. The call is made directly against the REST API rather than through the Firebase Admin or client SDK, because the "send this link" call (`sendSignInLinkToEmail`) only exists in the client-side Firebase Auth SDK and can't be invoked cleanly from a server route.

## Playbook

### Prerequisites
- A Firebase project with Email/Password sign-in's "Email link (passwordless sign-in)" method enabled in the Firebase console.
- The project's public Web API key (`NEXT_PUBLIC_FIREBASE_API_KEY` or equivalent) — this is not a secret; it's the same key used to initialize the client Firebase SDK.
- An authorized domain in the Firebase console covering wherever `continueUrl` below points.
- A client-side route that can complete the sign-in (see Core pattern part 2).

### Setup steps
1. Enable the "Email link (passwordless sign-in)" provider for your Firebase Auth project.
2. Add your app's domain to the Firebase project's Authorized Domains list.
3. Server-side, build a small helper that POSTs to the Identity Toolkit `accounts:sendOobCode` endpoint with `requestType: "EMAIL_SIGNIN"`, the recipient's email, and a `continueUrl` pointing at a client route in your app that will finish the sign-in.
4. Call that helper from whatever server action triggers the invite (e.g. an admin "invite user" endpoint, or a "add team member" endpoint) — guard it so it only fires for genuinely new users, not existing members being re-added.
5. Build the client completion route: check `isSignInWithEmailLink(auth, window.location.href)`; if the email isn't recoverable from `localStorage` (the common case for an invite link opened on a different device/browser than the one that requested it), prompt the user to re-enter their email, then call `signInWithEmailLink(auth, email, window.location.href)`.
6. After `signInWithEmailLink` succeeds, mint whatever server-side session your app uses (e.g. exchange the resulting ID token for a session cookie via your own `/api/auth/session` endpoint) and redirect into the app.
7. Note there is no Firebase Console template customization available for this specific email type (unlike password-reset/email-verification templates) — recipients get Firebase's generic, unbranded email; if branding matters, that's a reason to move to a different invite mechanism instead of fighting this one.

### Core pattern
Server-side: send the invite link (generic, sanitized from the real helper — strip any app-specific import paths):
```ts
export async function sendInviteSignInLink(email: string): Promise<void> {
  const apiKey = process.env.NEXT_PUBLIC_FIREBASE_API_KEY;
  if (!apiKey) return;
  try {
    const res = await fetch(
      `https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode?key=${apiKey}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          requestType: "EMAIL_SIGNIN",
          email,
          continueUrl: `${APP_BASE_URL}/auth/complete-signin`,
          canHandleCodeInApp: true,
        }),
      }
    );
    if (!res.ok) {
      console.error("sendInviteSignInLink failed", res.status, await res.text());
    }
  } catch (e) {
    console.error("sendInviteSignInLink failed", e);
  }
}
```

Call site — only send the invite email for users genuinely new to the app:
```ts
const { alreadyAllowed } = await addUserToApp(email, invitedBy);
if (!alreadyAllowed) {
  sendInviteSignInLink(email);
}
```

Client-side completion route (`/auth/complete-signin`) — handle both the same-device auto-complete case and the far more common cross-device "ask for email again" case:
```tsx
"use client";
import { useEffect, useState } from "react";
import { isSignInWithEmailLink, signInWithEmailLink } from "firebase/auth";
import { clientAuth } from "@/lib/firebase-client";

const EMAIL_STORAGE_KEY = "emailForSignIn";

export default function CompleteSignIn() {
  const [needsEmail, setNeedsEmail] = useState(false);

  useEffect(() => {
    if (!isSignInWithEmailLink(clientAuth, window.location.href)) return; // invalid/expired link
    const stored = window.localStorage.getItem(EMAIL_STORAGE_KEY);
    if (stored) complete(stored);
    else setNeedsEmail(true);
  }, []);

  async function complete(email: string) {
    const result = await signInWithEmailLink(clientAuth, email, window.location.href);
    window.localStorage.removeItem(EMAIL_STORAGE_KEY);
    const idToken = await result.user.getIdToken();
    await fetch("/api/auth/session", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ idToken }),
    });
    window.location.href = "/dashboard";
  }

  // render needsEmail ? <email confirmation form calling complete(email)> : <loading state>
}
```

### Env vars
- `NEXT_PUBLIC_FIREBASE_API_KEY` — the Firebase Web API key, reused from whatever Firebase Auth setup already exists in the app (see `firebase-auth.md`); no separate key is needed for this feature.

### Gotchas
- **Invite emails triggered server-side are always opened on a different device/browser than the one that requested them** — the recipient, not the inviter, opens the link, so Firebase's same-device `localStorage` auto-complete key (`emailForSignIn`) is realistically never present. Design the "confirm your email" form as the primary path, not a rare fallback.
- **There is no Firebase Console template for this email** — unlike password-reset or email-verification, "email sign-in link" has no customizable template entry as of current Firebase docs; you get Firebase's generic, unbranded email and can't rebrand it without switching mechanisms.
- **Only invoke `sendSignInLinkToEmail`'s effect via the REST endpoint, not the client SDK, from a server route** — the client Firebase Auth SDK's `sendSignInLinkToEmail` has no server/Admin SDK equivalent, so a direct REST call to `accounts:sendOobCode` is the correct approach server-side, not a workaround to avoid.
- **Guard against re-inviting existing members** — the API key is not secret and there's no built-in Firebase rate limit visible to your app, so gate the call behind your own "is this genuinely a new user" check to avoid spamming already-active members every time an invite endpoint is hit.
- **The link doubles as the first sign-in, not just a notification** — treat `continueUrl` as a real, load-bearing part of your app's auth flow (must resolve to a page that calls `signInWithEmailLink`), not a cosmetic redirect target.

### Playbook confidence: high

## Adoption
Used in **2** repo(s) in this marketplace. Both call the same Identity Toolkit REST endpoint through an equivalent `sendInviteSignInLink`-style helper to invite new members/collaborators into an app — one gating it behind an admin/team-membership flow, the other behind a co-organizer invite flow. Both reuse the existing Firebase Web API key already present for auth rather than introducing a new env var.
