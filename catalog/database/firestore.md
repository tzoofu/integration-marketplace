# Firestore

- **category**: database
- **provider**: Firebase (Google Cloud)
- **reusable**: partial — collection schemas are app-specific, but the client-read/admin-write access pattern is reusable. See [firebase-admin-sdk](../other/firebase-admin-sdk.md) for the shared server bootstrap most adopters use to reach it. A couple of adopters diverge from this pattern by reading and writing entirely from the client SDK (see below).
- **docs**: https://firebase.google.com/docs/firestore and https://firebase.google.com/docs/firestore/security/get-started

## Overview
A serverless, realtime NoSQL document database from Firebase. Reach for it when you want live-updating UI (`onSnapshot`) without hand-rolling websockets, and you're fine with a security-rules-first (or Admin-SDK-gated) access model instead of a traditional API-per-table backend.

## Playbook

### Prerequisites
- A Firebase project (console.firebase.google.com) with Firestore enabled (native mode).
- `firebase` package (client SDK) for browser/mobile reads; `firebase-admin` for any server-side/Node access.
- A service account key (for Admin SDK) if any server-side writes are needed — either a downloaded JSON key or `FIREBASE_PROJECT_ID`/`FIREBASE_CLIENT_EMAIL`/`FIREBASE_PRIVATE_KEY` split into env vars.
- Firebase CLI (`firebase-tools`) to deploy `firestore.rules` / `firestore.indexes.json`.

### Setup steps
1. Create the Firebase project and add a Web app (and/or Android app) to get its config object.
2. Decide the access model up front — this determines almost everything else:
   - **Server-authoritative** (recommended default for a Next.js app with real business logic/validation): client reads live via the client SDK, ALL writes go through server API routes using the Admin SDK, and `firestore.rules` denies client writes outright (defense in depth even though the Admin SDK bypasses rules).
   - **Pure client SDK**: no server API layer at all; both reads and writes happen from the client (web or mobile), and `firestore.rules` is the *only* authorization boundary. Only choose this if you're comfortable writing and testing real security rules — see Variant B below.
3. Write `firestore.rules` starting from `allow read, write: if false;` on `{document=**}` and add narrow, explicit `match` blocks per collection.
4. `firebase deploy --only firestore:rules,firestore:indexes` to publish rules and any composite indexes.
5. If you need real-time reactive UI, subscribe with `onSnapshot` on the client; otherwise a one-shot `getDocs`/Admin SDK read is simpler and cheaper.

### Core pattern

**Variant A — Next.js server-authoritative (Admin SDK writes, client SDK live reads, rules deny client writes)**

```ts
// lib/firebase-admin.ts — server-only module
import "server-only";
import { initializeApp, getApps, cert } from "firebase-admin/app";
import { getFirestore } from "firebase-admin/firestore";

if (!getApps().length) {
  initializeApp({
    credential: cert({
      projectId: process.env.FIREBASE_PROJECT_ID,
      clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      // service-account private keys are stored with literal "\n" in env vars — must be unescaped
      privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, "\n"),
    }),
  });
}

export const adminDb = getFirestore();
```

```ts
// app/api/items/route.ts — every write goes through here, never through the client SDK
import { adminDb } from "@/lib/firebase-admin";

export async function POST(req: Request) {
  const body = await req.json();
  await adminDb.collection("items").doc(body.id).set(body, { merge: true });
  return Response.json({ ok: true });
}
```

```ts
// lib/firebase-client.ts — browser bundle, read-only against this data
import { initializeApp, getApps, getApp } from "firebase/app";
import { getFirestore } from "firebase/firestore";

const config = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};
const app = getApps().length ? getApp() : initializeApp(config);
export const db = getFirestore(app);
```

```
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // narrow, explicit read carve-outs only — e.g. a user reading their own profile
    match /users/{email} {
      allow read: if request.auth != null && request.auth.token.email == email;
      allow write: if false;
    }
    // everything else: server (Admin SDK) only, deny all direct client access
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

**Variant B — pure client SDK, read + write gated entirely by rules (no server API layer)**

```ts
// lib/firebase.ts (web)
import { initializeApp } from "firebase/app";
import { getFirestore } from "firebase/firestore";

const app = initializeApp({
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  appId: import.meta.env.VITE_FIREBASE_APP_ID,
});
export const db = getFirestore(app);
```

```ts
// services/itemService.ts — a thin CRUD wrapper is the only "backend" this needs
import { addDoc, collection, doc, getDoc, serverTimestamp, updateDoc } from "firebase/firestore";
import { db } from "../lib/firebase";

const COLLECTION = "items";

export async function createItem(data: Record<string, unknown>) {
  const ref = await addDoc(collection(db, COLLECTION), { ...data, createdAt: serverTimestamp() });
  return ref.id;
}

export async function getItem(id: string) {
  const snap = await getDoc(doc(db, COLLECTION, id));
  return snap.exists() ? { id: snap.id, ...snap.data() } : null;
}
```

```
// firestore.rules — the ONLY authorization boundary in this variant, test it like code
match /items/{itemId} {
  allow read, delete: if request.auth != null && resource.data.ownerId == request.auth.uid;
  allow create: if request.auth != null && request.resource.data.ownerId == request.auth.uid;
  allow update: if request.auth != null && resource.data.ownerId == request.auth.uid
    && request.resource.data.ownerId == resource.data.ownerId; // ownerId itself is immutable
}
```

A native-mobile client (e.g. Android/Kotlin) follows the same Variant B shape: the Firestore SDK is initialized from the platform config file (`google-services.json`, not env vars) instead of a JS config object, and a thin repository class wraps collection references the same way the web `services/*` layer does — the security boundary is still entirely `firestore.rules`. A serverless triggers/schedule layer (e.g. Cloud Functions) can sit alongside this variant purely for background jobs — see Gotchas.

### Env vars

Server-authoritative (Variant A) — Admin SDK, server-only:
- `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` (or a single `FIREBASE_SERVICE_ACCOUNT_JSON` / `FIREBASE_ADMIN_CLIENT_EMAIL` + `FIREBASE_ADMIN_PRIVATE_KEY` variant — naming differs per repo, pick one convention and stick to it)

Client SDK, either variant (browser-exposed, framework-dependent prefix):
- Next.js: `NEXT_PUBLIC_FIREBASE_API_KEY`, `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`, `NEXT_PUBLIC_FIREBASE_PROJECT_ID`, `NEXT_PUBLIC_FIREBASE_APP_ID`, `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`, `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
- Vite: `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_STORAGE_BUCKET`, `VITE_FIREBASE_MESSAGING_SENDER_ID`, `VITE_FIREBASE_APP_ID`

Native mobile (Android): none — config ships in a build-time `google-services.json` file, not env vars. Cloud Functions equally use CLI-managed project credentials, not env vars.

### Gotchas
- If you deny client writes in `firestore.rules`, do it explicitly even though the Admin SDK already bypasses rules — it's cheap defense-in-depth against a future accidental client-side write call.
- A rule like `match /users/{uid} { allow read: if isOwner(uid); }` is a common one-collection carve-out layered on top of an otherwise fully-locked-down ruleset (`{document=**} { allow read, write: if false; }`) — write the deny-all wildcard first, then punch narrow holes.
- Composite/array-contains query limitations bite in practice: Firestore cannot AND one `array-contains` with a separate `array-contains-any` in a single query, so a "my events OR my team's events" lookup has to run as two queries merged in application code.
- `recursiveDelete()` (Admin SDK) is the correct way to delete a document plus its subcollections atomically-ish — a plain `.delete()` orphans subcollections silently, and denormalized counters can make that orphaning invisible until it causes a bug.
- Rules for shared parent/child docs: a wildcard subcollection match (`match /{collectionId}/{documentId}`) in `rules_version = '2'` matches **zero or more** path segments, so it can accidentally also match the parent document itself and OR a permissive subcollection rule onto the parent's own read/write rule — exclude sensitive subcollection names explicitly and never assume a wildcard block only touches children.
- Firestore rules OR together every `match` block that structurally matches a given path — a broad rule and a narrow, stricter rule on an overlapping path both apply, so a secrets-style subcollection needs to be carved out of any broader wildcard rule, not just given its own stricter one.
- A repo missing `firestore.rules`/`firebase.json`/`.firebaserc` in-repo isn't necessarily undocumented — some teams manage the Firebase project config outside the app repo entirely; check for that before assuming rules are absent.
- `list` vs `get` matter for security: a `list` (query) can only be allowed if Firestore can prove every possible match satisfies the rule; a lookup performed by someone who by definition isn't a member yet (e.g. redeeming an invite code) can never be proven safe that way, so such collections must be looked up strictly by exact document ID (`get`) with `list: false`.
- iOS Safari can silently stall an already-open `onSnapshot` listener's network stream while backgrounded and not reliably reconnect on foreground — cycling `disableNetwork(db)` → `enableNetwork(db)` on the `visibilitychange` event forces every active listener to refresh.
- `preferRest: true` on the Admin SDK's Firestore instance avoids the gRPC transport, which both speeds up serverless cold starts and is required on runtimes without raw TCP/TLS support (e.g. Cloudflare Workers).
- Dynamically `import()` `firebase-admin/auth` (rather than a static top-level import) if your Admin SDK usage is otherwise Firestore-only — some serverless bundlers choke resolving `firebase-admin/auth`'s CommonJS/ESM-only dependency chain, breaking every route that merely imports the module.
- A thin Cloud Functions layer using the Admin SDK can coexist with a pure-client-SDK app (Variant B) purely for server-triggered (`onWrite`/`onCreate`) or scheduled logic (e.g. recomputing budget-cap alerts, sending push notifications) — this doesn't turn the app into Variant A, since the client still reads/writes directly and rules remain the real authorization boundary.
- Watch for security rules staged in the repo but explicitly **not yet deployed** (e.g. gated behind a comment and an expiry date) — read the rules file's own comments before assuming what's live in production matches what's checked in.

### Playbook confidence: high

## Adoption
Used in **7** repo(s) in this marketplace. Most adopters follow a server-authoritative pattern — client SDK for live reads, Admin SDK for all writes, with security rules denying direct client writes. A couple of adopters instead read and write entirely from the client SDK, with `firestore.rules` as the sole authorization boundary; one of those is a native-mobile client paired with a small Cloud Functions layer (Admin SDK) for background/scheduled writes.
