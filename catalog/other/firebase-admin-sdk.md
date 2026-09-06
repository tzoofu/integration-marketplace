# Firebase Admin SDK / Service Account

- **category**: other
- **provider**: Firebase / Google Cloud IAM
- **reusable**: yes — identical bootstrap shape (service-account JSON/env → Admin app singleton) across repos; candidate for a shared `@marketplace/firebase-admin` bootstrap package.
- **docs**: https://firebase.google.com/docs/admin/setup

## Overview
The Firebase Admin SDK gives server-only, privileged access to Firestore, Auth, Storage, and FCM using a service-account credential instead of a signed-in user's client token. Every repo in this marketplace that runs server-side Firebase logic (Firestore writes, Auth token verification, FCM sends, Storage writes, or minting an OAuth access token for another Google API) bootstraps it the same way: a singleton Admin `App`, guarded against re-initialization, built from a service-account credential supplied via environment variables.

## Playbook

### Prerequisites
- A Firebase project with a service account (Project settings → Service accounts → Generate new private key), or Application Default Credentials in an environment that already has them (e.g. Cloud Run/Functions).
- `firebase-admin` installed (`npm install firebase-admin`).
- A server-only execution context (Next.js Route Handler, Server Action, Cloud Function) — never bundle this into client code, the private key must never reach the browser.

### Setup steps
1. Install `firebase-admin`.
2. Decide how the service-account credential reaches the process: either three discrete env vars (`project_id` / `client_email` / `private_key`) or one JSON blob env var containing the whole key file. Discrete vars are easier to rotate individually; a JSON blob is easier to paste from the downloaded key file in one shot.
3. Write a single bootstrap module that checks `getApps().length` (or caches the app in a module-level variable) before calling `initializeApp()`, so hot-reload / multiple imports never throw "app already exists".
4. When the private key comes from an env var, replace literal `\n` escape sequences with real newlines before passing it to `cert()` — most env var UIs/`.env` files can't hold real newlines.
5. Export typed accessors (`adminDb()`, `adminAuth()`, `adminMessaging()`, `adminStorage()`) that lazily resolve off the singleton app, rather than exporting the SDK singletons directly — this lets the module be imported safely even in code paths that don't end up using Firebase (avoids throwing at import time before env vars are configured).
6. Mark the module `"server-only"` (Next.js) or otherwise ensure it can never be pulled into a client bundle.
7. If a route needs to call a *different* Google API (e.g. reCAPTCHA Enterprise Assessments, an OAuth2-only Google endpoint) with the same service account's credentials, mint an access token via `getApp().options.credential.getAccessToken()` instead of provisioning a second credential.

### Core pattern
```ts
import "server-only";
import { cert, getApps, initializeApp, type App } from "firebase-admin/app";
import { getAuth, type Auth } from "firebase-admin/auth";
import { getFirestore, type Firestore } from "firebase-admin/firestore";

let app: App | null = null;

function getAdminApp(): App {
  if (app) return app;
  const existing = getApps();
  if (existing.length) {
    app = existing[0];
    return app;
  }

  const projectId = process.env.FIREBASE_PROJECT_ID;
  const clientEmail = process.env.FIREBASE_CLIENT_EMAIL;
  const privateKey = process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, "\n");

  app = initializeApp({
    credential: cert({ projectId, clientEmail, privateKey }),
  });
  return app;
}

export function adminDb(): Firestore {
  return getFirestore(getAdminApp());
}

export function adminAuth(): Auth {
  return getAuth(getAdminApp());
}

// Reuse the same service-account credential to call another Google API.
export async function getAdminAccessToken(): Promise<string> {
  const { access_token } = await getAdminApp().options.credential!.getAccessToken();
  return access_token;
}
```

### Env vars
`FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` (discrete form) — or `FIREBASE_SERVICE_ACCOUNT_JSON` (single-blob form); `NEXT_PUBLIC_FIREBASE_PROJECT_ID` / `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` when a public-facing project ID or bucket name is also needed client-side.

### Gotchas
- Guard initialization with `getApps().length` (or an equivalent cache) — a second `initializeApp()` call throws.
- `\n` in the private key env var must be unescaped to a real newline or `cert()` fails to parse the PEM.
- Importing `firebase-admin/auth` (or other submodules with heavy transitive deps like `jwks-rsa`/`jose`) at the top level can break *every* route in a Next.js app if the bundler treats it as an external module in an environment that can't resolve it (seen in one implementation targeting Cloudflare Workers) — dynamically `import()` submodules that aren't needed on every request, and initialize the Firestore client with `{ preferRest: true }` if the runtime lacks a working gRPC/TCP stack.
- Prefer building `Firestore`/`Auth` accessors as functions computed on demand rather than top-level `export const`, so importing the module never throws before env vars are configured (Next.js can statically collect route page data at build time, before secrets exist).
- Never let this module (or anything importing it) end up in a client bundle — the private key is a server secret.

### Playbook confidence: high

## Adoption
Used in **3** repo(s) in this marketplace. The credential is supplied either as three discrete
env vars or as a single JSON-blob env var, with the accessor scope varying from Firestore-only up
to Firestore + Auth + Storage + FCM. One adopter is lazily initialized and scoped to Firestore/Auth
only, using a dynamic import and a REST-preferring Firestore client to survive a bundling target
without a working gRPC/TCP stack.
