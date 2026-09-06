# Firebase Storage

- **category**: storage
- **provider**: Firebase / Google Cloud Storage
- **reusable**: yes — the same "Admin SDK bucket, per-object public ACL, canonical `storage.googleapis.com` URL" pattern repeats across most adopters with only the path prefix changing; the one client-SDK-direct-upload adopter is a different but still-portable pattern (Storage security rules gating a native client upload).
- **docs**: https://firebase.google.com/docs/storage

## Overview
Firebase Storage is Google Cloud Storage with a Firebase-flavored SDK and (optionally) declarative security rules. It's used here as plain object storage for user/admin-uploaded media — reach for it when you need durable file storage behind a Node backend that already uses Firebase Admin, or behind a mobile client that already uses the Firebase client SDK.

## Playbook

### Prerequisites
- A Firebase project with Storage enabled, and a bucket name (`<project>.appspot.com` or `<project>.firebasestorage.app`).
- Server-side: `firebase-admin` package, already-initialized Admin SDK app (service account JSON, or project-id/client-email/private-key triplet, or `applicationDefault()`).
- Client-side (native/mobile only): `firebase` client SDK plus deployed `storage.rules`.
- The bucket's env var(s) resolved before any call — every repo here reads it from `process.env` at module scope with a non-null assertion, so a missing var fails loudly (or with a bucket-not-found error) rather than silently.

### Setup steps
1. Enable Storage in the Firebase console / `firebase init storage`, which scaffolds `storage.rules` and registers it in `firebase.json`.
2. Initialize the Admin SDK once (module-level singleton, guarded by `getApps().length` so hot-reload/serverless re-invocation doesn't re-initialize).
3. Wrap `getStorage().bucket(BUCKET)` behind a small helper module (`storage-admin.ts` / `adminStorage()`) — never call `getStorage()` ad hoc from route handlers, so the bucket name and upload options (`public: true`, content-type) stay consistent everywhere.
4. Pick one canonical public-URL shape (`https://storage.googleapis.com/${BUCKET}/<path>`) and centralize URL⇄path conversion in a pure, dependency-free module (`storage-paths.ts`) so it's usable from both server routes and offline scripts/tests without importing `firebase-admin`.
5. Namespace object paths by owning entity (`{collection}/{id}/{uuid}.ext`, e.g. `items/{id}/{uuid}.ext`, `records/{id}/{uuid}.ext`) so ownership can be checked by prefix match alone.
6. If a native client uploads directly (no server route in between), write `storage.rules` instead of an API route, and deploy rules only after testing — a broken rule either locks out legitimate uploads or leaks reads.

### Core pattern

**Variant A — server-side upload via Admin SDK (the dominant pattern):**
```ts
import { randomUUID } from "crypto";
import { getStorage } from "firebase-admin/storage";

const BUCKET = process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET!;

function extForContentType(contentType: string): "png" | "webp" | "jpg" {
  if (contentType.includes("png")) return "png";
  if (contentType.includes("webp")) return "webp";
  return "jpg";
}

// The one sanctioned upload path — never construct URLs by hand elsewhere.
export async function uploadEntityImage(
  entityId: string,
  buffer: Buffer,
  contentType: string,
  ext?: string
): Promise<string> {
  const path = `entities/${entityId}/${randomUUID()}.${ext ?? extForContentType(contentType)}`;
  const bucket = getStorage().bucket(BUCKET);
  await bucket.file(path).save(buffer, { contentType, public: true });
  return `https://storage.googleapis.com/${BUCKET}/${path}`;
}

export async function deleteEntityImageFiles(urls: string[]): Promise<void> {
  if (!urls.length) return;
  const bucket = getStorage().bucket(BUCKET);
  const prefix = `https://storage.googleapis.com/${BUCKET}/`;
  const paths = urls
    .filter((u) => u.startsWith(prefix))
    .map((u) => u.slice(prefix.length).split(/[?#]/)[0]);
  await Promise.all(paths.map((p) => bucket.file(p).delete({ ignoreNotFound: true })));
}
```
Route handler shape (multipart upload, ownership-scoped):
```ts
export const POST = withAuth(async (req, auth, { params }) => {
  const { id } = await params;
  const formData = await req.formData();
  const file = formData.get("file") as File | null;
  if (!file) return badRequest("No file");

  const buffer = Buffer.from(await file.arrayBuffer());
  const url = await uploadEntityImage(id, buffer, file.type || "image/jpeg");
  // persist `url` on the owning record, then:
  return NextResponse.json({ ok: true, url });
});
```

**Variant B — native client uploads directly, gated by Storage security rules:**
```
// storage.rules
match /groups/{groupId}/photo.jpg {
  allow read: if request.auth != null && (
    firestore.get(/databases/(default)/documents/groups/$(groupId)).data.ownerUid == request.auth.uid ||
    firestore.get(/databases/(default)/documents/groups/$(groupId)).data.memberUid == request.auth.uid
  );
  allow write: if request.auth != null &&
    firestore.get(/databases/(default)/documents/groups/$(groupId)).data.ownerUid == request.auth.uid &&
    request.resource.contentType.matches('image/.*') &&
    request.resource.size < 5 * 1024 * 1024;
}
```
The Android client then uploads via the Firebase Storage client SDK directly to that path — no server route at all.

### Env vars
- `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` (most repos; must be public/client-exposed if any client code reads it)
- `FIREBASE_STORAGE_BUCKET`, `FIREBASE_PROJECT_ID` (an alternate naming convention used by one adopter — falls back to `${FIREBASE_PROJECT_ID}.firebasestorage.app` if unset)
- Admin SDK credentials: either `FIREBASE_SERVICE_ACCOUNT_JSON` (single-line JSON) or `FIREBASE_PROJECT_ID` / `FIREBASE_CLIENT_EMAIL` / `FIREBASE_PRIVATE_KEY`, or nothing (falls back to `applicationDefault()`)

### Gotchas
- **Public objects aren't optional in this pattern** — every upload here sets `{ public: true }` per-object rather than relying on bucket-level public access, so a plain `bucket.file(path).save()` without that flag silently produces an inaccessible object.
- **GCS copy does not carry ACLs.** A server-side `bucket.file(from).copy(bucket.file(to))` (used for merge/reconciliation flows) drops the source object's public ACL — you must call `.makePublic()` on the destination explicitly, or the copy 403s in the browser.
- **Two live URL shapes can coexist.** Older objects may be in the legacy Firebase-SDK download-URL shape (`.../o/<path>?alt=media&token=...`) while newer ones use the canonical `storage.googleapis.com/<bucket>/<path>` shape. Any path-derivation helper must handle both, or orphan/duplicate-detection logic will misclassify legacy-shape images as orphans.
- **Strip query strings/fragments before treating a URL as a path.** A URL→path conversion function that doesn't strip `?...`/`#...` lets a crafted suffix desync the derived Storage path from the real object key — this matters because these URLs are often round-tripped through client-supplied JSON (removal lists, verify lists).
- **Never trust client-supplied URL lists for mutation/deletion.** Filter any client-supplied "these URLs are mine" list down to ones that actually resolve to a path under that entity's own prefix before acting on it (`ownedUrls`-style helper) — otherwise one user's client can plant another entity's real object URL into their own record, and a later admin "remove" (which also deletes the underlying Storage object) destroys someone else's live file.
- **A Storage download-URL token is not an access-control boundary.** If the app persists a Storage SDK download URL (which embeds an unauthenticated bypass token) rather than a bare storage path, security rules on that path never get evaluated for anyone holding the URL string — true confidentiality requires storing the path and fetching through the authenticated SDK, not handing out the tokened URL.
- **Delete only after the owning record's write has succeeded, never before** — deleting the Storage object first and having the metadata write fail after leaves a broken reference; deleting after leaves at worst a harmless orphan (cleanable later).
- **Initialize the Admin SDK exactly once per process** — guard with `getApps().length` / a cached module-level app, since serverless re-invocation and Next.js HMR both re-run module code.

### Playbook confidence: high

## Adoption
Used in **5** repo(s) in this marketplace. Most adopters use the same server-side Admin-SDK-bucket, per-object-public-ACL, canonical-URL pattern to store admin/organizer-uploaded media (listing photos, landing-page assets, event galleries, attachments), differing mainly in path-prefix naming and the bucket env var's naming convention. One adopter instead uploads directly from a native mobile client, with Storage security rules — rather than a server route — as the access gate and size/type validation.
