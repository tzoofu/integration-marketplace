# Firebase Auth

- **category**: auth
- **provider**: Firebase / Google
- **reusable**: yes — candidate for a shared `@marketplace/firebase-auth` client wrapper; most repos hand-roll the same Google sign-in pattern independently. Several add a server-side session or per-request token-verification layer on top, while others stay pure client SDK; one of those pure-client-SDK adopters is also a native-Android implementation, using Credential Manager instead of a web popup/redirect.
- **docs**: https://firebase.google.com/docs/auth

## Overview
Google OAuth sign-in via the Firebase Auth client SDK. Reach for it when you need "Sign in with Google" with zero custom OAuth plumbing and you're already on Firebase/Firestore for data. Two structurally different follow-ups exist after the client obtains a Firebase ID token: exchange it server-side for your own long-lived session cookie (most repos here), or skip the exchange and verify the ID token per-request instead.

## Playbook

### Prerequisites
- A Firebase project with Authentication enabled and the Google sign-in provider turned on (Firebase Console → Authentication → Sign-in method).
- Web: `firebase` npm package (client SDK) for the browser; `firebase-admin` (or a JWKS-only JWT verifier) on the server if you verify tokens server-side.
- Native Android: `androidx.credentials:credentials`, `androidx.credentials:credentials-play-services-auth`, `com.google.android.libraries.identity.googleid:googleid`, plus `google-services.json` and the Google Sign-In OAuth web client ID.
- Authorized domains for your app's origin(s) added in the Firebase Auth console (popups/redirects fail silently otherwise — surfaces as `auth/unauthorized-domain`).

### Setup steps
1. Create/select a Firebase project, register a Web App (and/or Android app), enable Google as a sign-in provider.
2. Copy the client config (`apiKey`, `authDomain`, `projectId`, `appId`, …) into public env vars — safe to expose, they're not secrets.
3. Initialize the client SDK once per app (guard against re-initializing on hot reload with `getApps().length ? getApps()[0] : initializeApp(...)`).
4. Decide your server trust model up front (see the two variants below) — retrofitting from "pure client SDK" to "server session" later means rewriting every protected route.
5. If you need admin-side verification, generate a service account key (Project Settings → Service Accounts) and store its `project_id` / `client_email` / `private_key` as server-only env vars — never `NEXT_PUBLIC_*`.
6. For native Android, add `google-services.json` (build-time config, not env vars) and register the SHA-1/SHA-256 fingerprint in the Firebase console.

### Core pattern

**Variant A — client sign-in + server session cookie**

```ts
// lib/firebase-client.ts
import { initializeApp, getApps } from "firebase/app";
import { getAuth, GoogleAuthProvider } from "firebase/auth";

const app = getApps().length
  ? getApps()[0]
  : initializeApp({
      apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
      authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
      projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
      appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
    });

export const clientAuth = getAuth(app);
export const googleProvider = new GoogleAuthProvider();
```

```ts
// hooks/useGoogleSignIn.ts (client)
import { signInWithPopup, signInWithRedirect } from "firebase/auth";
import { clientAuth, googleProvider } from "@/lib/firebase-client";

async function signIn(useRedirectFallback: boolean) {
  if (useRedirectFallback) {
    // iOS standalone PWA: popup gets bounced to Safari and loses the session —
    // fall back to a full-page redirect and resume via getRedirectResult() on load.
    await signInWithRedirect(clientAuth, googleProvider);
    return;
  }
  const result = await signInWithPopup(clientAuth, googleProvider);
  const idToken = await result.user.getIdToken();
  const res = await fetch("/api/auth/session", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ idToken }),
  });
  if (!res.ok) { await clientAuth.signOut(); /* show error */ return; }
  window.location.href = "/dashboard";
}
```

```ts
// app/api/auth/session/route.ts (server)
import { adminAuth } from "@/lib/firebase-admin";
import { createSessionToken, SESSION_COOKIE, SESSION_MAX_AGE } from "@/lib/session";

export async function POST(req: Request) {
  const { idToken } = await req.json();
  let decoded;
  try {
    decoded = await adminAuth().verifyIdToken(idToken);
  } catch {
    return Response.json({ error: "Invalid token" }, { status: 401 });
  }
  if (!decoded.email || !(await isEmailAllowed(decoded.email))) {
    return Response.json({ error: "Access denied" }, { status: 403 });
  }
  const token = await createSessionToken(decoded.email); // jose-signed JWT, your own claims
  const res = Response.json({ ok: true });
  res.headers.append(
    "Set-Cookie",
    `${SESSION_COOKIE}=${token}; HttpOnly; Secure; SameSite=Lax; Max-Age=${SESSION_MAX_AGE}; Path=/`
  );
  return res;
}
```

**Variant B — pure client SDK, no server layer**

```ts
// services/authService.ts
import { signInWithPopup } from "firebase/auth";
import { doc, setDoc, getDoc } from "firebase/firestore";
import { auth, db, googleProvider } from "../lib/firebase";

export async function signInWithGoogle() {
  const { user } = await signInWithPopup(auth, googleProvider);
  const ref = doc(db, "users", user.uid);
  if (!(await getDoc(ref)).exists()) {
    await setDoc(ref, { uid: user.uid, email: user.email, createdAt: new Date().toISOString() });
  }
  return user;
}
// Authorization is enforced entirely by firestore.rules (request.auth.uid checks) —
// there is no backend to trust, so rules ARE the security boundary.
```

**Variant C — per-request ID token verification, no session cookie**

```ts
// Zero-dependency variant (no firebase-admin): verify the ID token's signature
// directly against Google's public JWKS. Only the public project ID is needed.
import { createRemoteJWKSet, jwtVerify } from "jose";

const JWKS = createRemoteJWKSet(
  new URL("https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com")
);

export async function verifyFirebaseIdToken(idToken: string) {
  const projectId = process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID!;
  const { payload } = await jwtVerify(idToken, JWKS, {
    issuer: `https://securetoken.google.com/${projectId}`,
    audience: projectId,
  });
  return { uid: payload.sub as string, email: payload.email as string | undefined };
}

// Each protected route/tool call requires the client to send:
//   Authorization: Bearer <firebase-id-token>
// and re-verifies it — no server session state to manage or expire.
```

**Variant D — native Android via Credential Manager**

```kotlin
class AuthRepository(private val auth: FirebaseAuth = FirebaseAuth.getInstance()) {
    suspend fun signInWithGoogle(context: Context, webClientId: String): FirebaseUser {
        val googleIdOption = GetGoogleIdOption.Builder()
            .setFilterByAuthorizedAccounts(false)
            .setServerClientId(webClientId)
            .build()
        val request = GetCredentialRequest.Builder().addCredentialOption(googleIdOption).build()
        val response = CredentialManager.create(context).getCredential(context, request)
        val credential = response.credential as CustomCredential
        val googleIdTokenCredential = GoogleIdTokenCredential.createFrom(credential.data)
        val firebaseCredential = GoogleAuthProvider.getCredential(googleIdTokenCredential.idToken, null)
        return auth.signInWithCredential(firebaseCredential).await().user!!
    }
}
// No server-side session — same "rules are the boundary" model as Variant B,
// enforced via firestore.rules / storage.rules instead of a Next.js API layer.
```

### Env vars

Client (public, safe to expose) — Next.js:
`NEXT_PUBLIC_FIREBASE_API_KEY`, `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`, `NEXT_PUBLIC_FIREBASE_PROJECT_ID`, `NEXT_PUBLIC_FIREBASE_APP_ID`, `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`, `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` (some repos)

Client (public) — Vite:
`VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_APP_ID`

Server-only (Variant A/C with firebase-admin):
`FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` — or the `FIREBASE_ADMIN_*` naming (`FIREBASE_ADMIN_CLIENT_EMAIL`, `FIREBASE_ADMIN_PRIVATE_KEY`) some repos use instead

App session signing (Variant A):
`NEXTAUTH_SECRET` (used as a generic JWT-signing secret via `jose`, not actual NextAuth)

Admin allowlist gate (several repos):
`ADMIN_EMAIL` / `NEXT_PUBLIC_ADMIN_EMAIL`

Native Android: none — config comes from `google-services.json` (a build-time file, not env vars).

### Gotchas
- `signInWithPopup` is unreliable on iOS standalone PWAs — WebKit can bounce the popup out to Safari and lose the session entirely. Detect that case and fall back to `signInWithRedirect` + resume via `getRedirectResult()` on next page load; guard so only one mounted component instance resolves the redirect result (multiple hook instances would otherwise race to POST the session endpoint twice).
- `getRedirectResult()` resolves `null` on every normal page load where no redirect is pending — that's the expected common case, not an error; only throw/surface errors when it actually rejects with a code.
- Firebase's own default IndexedDB-based auth persistence can throw "Database is closing/hidden" when a popup backgrounds the tab mid-write (a known firebase-js-sdk bug, worse on Edge/Windows) — switching to `browserLocalPersistence` avoids that code path.
- Decide once whether you're doing a session-cookie exchange (Variant A) or per-request ID token verification (Variant C) — mixing them (one adopter explicitly chose the per-request-verification variant specifically to avoid a second, separately-expiring session token) adds a second source of truth for "is this user logged in."
- If you skip `firebase-admin` and verify ID tokens directly against Google's JWKS (Variant C's zero-dependency path), you only need the *public* project ID as issuer/audience — no service account required, and it avoids `firebase-admin/auth`'s CommonJS/`jose`-ESM interop issues that can crash edge/serverless bundlers that eagerly resolve it as an external module.
- Lazy-initialize the admin app/credentials on first use rather than at module import time — importing `firebase-admin/app` eagerly can throw during route/page data collection in environments where admin env vars aren't set yet.
- Normalize emails (case, whitespace) consistently across every read/write path before using them as a Firestore document key or allowlist check — inconsistent normalization silently creates duplicate/orphaned user records.
- In a pure-client-SDK setup (Variant B/D), `firestore.rules`/`storage.rules` ARE the entire security boundary — there is no server to double-check anything, so treat rule-writing with the same rigor as writing an authorization middleware.
- Google-only accounts can require a fresh sign-in (`auth/requires-recent-login`) before sensitive operations like account deletion — catch that error code and re-run `reauthenticateWithPopup` before retrying.

### Playbook confidence: high

## Adoption
Used in **7** repo(s) in this marketplace. Most adopters exchange the client-obtained Firebase ID token for a server-issued session cookie (Variant A). A minority skip the server layer entirely and rely on Firestore/Storage security rules as the sole authorization boundary (Variant B), and one of those verifies the ID token per-request against Google's public JWKS instead of maintaining a session (Variant C). One adopter is a native-Android app using Credential Manager rather than a web popup/redirect flow (Variant D).
