# Firebase (Cloud Functions + project deploy)

- **category**: hosting-deploy
- **provider**: Google Firebase
- **reusable**: no — single-repo so far, and specific to a native-Android + Cloud Functions project shape rather than the Next.js-on-Vercel shape every other repo in this marketplace uses. Worth revisiting if a second Cloud-Functions-backed repo shows up.
- **docs**: https://firebase.google.com/docs/functions

## Overview
Deploy target for the server-side half of a Firebase project: Firestore/Storage security rules, Firestore indexes, and Cloud Functions, all deployed via the Firebase CLI under one project id and one pinned region. The client (a native Android app here) is distributed separately — this covers only the backend/rules deploy surface, not app distribution.

## Playbook

### Prerequisites
- A Firebase project with Firestore (and, if used, Storage) already provisioned in your target region.
- Node.js 20 (or whatever `engines.node` the Functions codebase pins) for building the Functions bundle.
- Firebase CLI installed and authenticated.

### Setup steps
1. `firebase init` (or hand-write `firebase.json`/`.firebaserc`) selecting Firestore, Storage (if needed), and Functions.
2. Pin `.firebaserc`'s `default` project to your Firebase project id — this makes every `firebase deploy` target the right project without an interactive prompt.
3. In `firebase.json`, set `firestore.location` to the region your Firestore database actually lives in, and set the same region in the Functions code's `setGlobalOptions({ region })` — colocating functions with their data avoids cross-region latency and egress cost on every trigger invocation.
4. Write Firestore/Storage security rules (`firestore.rules`, `storage.rules`) and Firestore composite indexes (`firestore.indexes.json`).
5. In the Functions package (`functions/`), declare a `predeploy` hook in `firebase.json` (`npm --prefix "$RESOURCE_DIR" run build`) so `firebase deploy` always ships compiled/current code, never a stale `lib/` build.
6. Write function handlers using v2 triggers (`firebase-functions/v2/*`) — e.g. `onDocumentWritten` for Firestore-triggered logic, `onSchedule` for cron-style recurring jobs.
7. Deploy everything with `firebase deploy`, or scope it: `firebase deploy --only firestore:rules,firestore:indexes` or `firebase deploy --only functions`.
8. Before deploying a rules change that *tightens* an existing `read`/`list` rule, verify it against the Firestore emulator (`firebase emulators:exec --only firestore "..."` plus `@firebase/rules-unit-testing`'s `assertSucceeds`/`assertFails`) — a rules deploy takes effect instantly and live, and a mistake surfaces as production `permission-denied` errors with no build/lint/type-check ever catching it.

### Core pattern
`firebase.json` — Firestore/Storage rules + a Functions codebase with a build predeploy hook:
```json
{
  "firestore": {
    "database": "(default)",
    "location": "<region>",
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  },
  "storage": {
    "rules": "storage.rules"
  },
  "functions": [
    {
      "source": "functions",
      "codebase": "default",
      "ignore": ["node_modules", ".git", "*.test.js", "*.test.ts"],
      "predeploy": ["npm --prefix \"$RESOURCE_DIR\" run build"]
    }
  ]
}
```

`.firebaserc` — pin the default project:
```json
{
  "projects": {
    "default": "<your-project-id>"
  }
}
```

`functions/package.json` — build/deploy scripts:
```json
{
  "engines": { "node": "20" },
  "main": "lib/index.js",
  "scripts": {
    "build": "tsc",
    "deploy": "firebase deploy --only functions"
  },
  "dependencies": {
    "firebase-admin": "^12.7.0",
    "firebase-functions": "^6.3.0"
  }
}
```

`functions/src/index.ts` — region-pinned v2 triggers, a document-write handler and a daily scheduled job:
```ts
import * as admin from "firebase-admin";
import { onDocumentWritten } from "firebase-functions/v2/firestore";
import { onSchedule } from "firebase-functions/v2/scheduler";
import { setGlobalOptions } from "firebase-functions/v2";

admin.initializeApp();

// Match the region to firebase.json's firestore.location to avoid cross-region latency/cost.
setGlobalOptions({ region: "<region>" });

export const onEntityChanged = onDocumentWritten("collection/{id}", async (event) => {
  await handleWrite(event.params.id, event.data?.before.data(), event.data?.after.data());
});

export const dailyRoutines = onSchedule("every day 08:00", async () => {
  await runDailyRoutines();
});
```

Deploy commands:
```bash
firebase deploy --only firestore:rules,firestore:indexes
firebase deploy --only functions
firebase deploy
```

### Env vars
None — deploy authenticates via Firebase CLI login/service-account credentials; runtime config for Cloud Functions (if any) is managed through Firebase's own config/secrets mechanism, not `.env` files.

### Gotchas
- **A rules deploy is instant and live with no staging window** — `firebase deploy --only firestore:rules` pushes straight to production. Test any tightened `read`/`list` rule against the Firestore emulator first; a client query that silently relied on a broader rule than the new one will only fail at runtime as `permission-denied`, never at build/lint/type-check time.
- **`list` queries can never be authorized the same way `get` lookups can** — Firestore security rules can only permit a `list`/query if every possible result is provably safe under the rule; a lookup-by-a-non-member-by-definition (e.g. redeeming an invite code) can never satisfy that, so such collections should force `allow list: if false` and only ever be read by exact-ID `get`.
- **Always declare a `predeploy` build hook for the Functions codebase** — without it, `firebase deploy --only functions` can ship a stale compiled `lib/` directory that doesn't match the current TypeScript source.
- **Pin the Functions region to match Firestore's region** (`setGlobalOptions({ region })` alongside `firestore.location` in `firebase.json`) — mismatched regions add latency and cross-region data transfer cost to every single trigger invocation.
- **This deploy surface is backend-only** — it says nothing about how the client app itself is distributed (e.g. a native Android app ships via Play Store, not Firebase Hosting); don't conflate "Firebase project configured" with "app distributed via Firebase."

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace, deploying Firestore rules, Storage rules, Firestore indexes, and Cloud Functions (notification-style business logic) for a region-pinned Firebase project. The client app itself is a native mobile app distributed separately (not via Firebase Hosting) — this integration covers only the backend/rules deploy surface.
