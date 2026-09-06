# Image Pipeline

- **category**: storage
- **provider**: internal (custom) — built on top of Firebase Storage (see [firebase-storage](firebase-storage.md)) and Next.js Image, neither of which is re-described here
- **reusable**: partial — the ownership-validation, MD5-hash duplicate-detection, and orphan-cleanup patterns are portable to any Storage-backed image app; the record-merge reconciliation logic is domain-specific to whatever "two records get merged" means in your app.
- **docs**: https://nextjs.org/docs/app/api-reference/components/image (framework-level resize/serve) — the lifecycle logic itself has no external docs, it's bespoke

## Overview
The custom lifecycle logic that sits *above* a raw Storage bucket: upload, ownership-scoped mutation, moderation/verification, cross-record duplicate detection via content hash, orphan cleanup, and merge-time reconciliation when two records combine. Reach for this shape once "upload a file to a bucket" (see the firebase-storage entry) stops being enough — i.e. once multiple owners can reference images, images can go stale/orphaned, or two owning records can be merged.

## Playbook

### Prerequisites
- An existing Storage-backed upload/delete layer (raw bucket plumbing) — this pipeline sits on top of it, doesn't replace it.
- A way to read Storage object metadata (MD5 hash, size) — used for duplicate detection, not just upload.
- Records with a stable, prefix-friendly ID so Storage paths can be namespaced per-owner (`{collection}/{id}/{uuid}.ext`).
- `next/image` (or an equivalent framework image component) if you want resizing/optimization for free instead of building a custom thumbnail pipeline.

### Setup steps
1. Namespace every object path under the owning record's ID (`{collection}/{id}/...`) so "does this URL belong to this record" is a prefix check, not a database lookup.
2. Add a pure URL⇄path module with no cloud-SDK dependency (testable, reusable from scripts) — see the firebase-storage entry's `storage-paths.ts` pattern.
3. On every mutation route that accepts client-supplied URLs (add/remove/verify), filter them through the ownership-prefix check before trusting them — never let a client claim another record's image.
4. Add a `verifiedImages`-style secondary field (subset of the main image list) so moderation/verification is tracked independently of raw attachment — lets admins flag reviewed images without touching the underlying array semantics.
5. Build duplicate detection as an admin-only scan: list all Storage objects under the shared prefix, group by MD5 hash, and union-find cross-record hash overlaps into clusters (a hash shared by 2 records is a pairwise dupe; transitively-connected hashes across 3+ records collapse into one cluster).
6. Build orphan cleanup as a diff: every Storage object under the prefix minus every record's known image URLs (converted to paths) = orphans. Always re-verify a specific orphan is still unreferenced immediately before the actual (irreversible) delete — time may have passed since the scan.
7. For merge-time reconciliation: classify each about-to-be-removed image as skip (exact MD5 duplicate of one the surviving record already has), keep-as-is (URL doesn't resolve to a known path — e.g. legacy format — nothing to move), or copy (do a server-side bucket-to-bucket copy into the surviving record's prefix, remembering the URL remap so any secondary fields like "verified" can follow the moved image).
8. Let framework image components (`next/image`) handle resize/format/responsive-serving — don't build a custom thumbnail/blurhash pipeline unless the framework option is proven insufficient.

### Core pattern

Ownership-scoped mutation (reject client-supplied URLs that don't resolve to this record's own prefix):
```ts
export function ownedUrls(id: string, urls: string[] | undefined): string[] {
  if (!urls?.length) return [];
  return urls.filter((url) => imageUrlToStoragePath(url)?.startsWith(`entities/${id}/`));
}

// in the PATCH route:
const addOwned = ownedUrls(id, body.add);          // never trust raw `add` URLs
const owned = new Set(existing.images ?? []);
const verifyOwned = body.verify?.filter((u) => owned.has(u)) ?? []; // verify only what's already attached
```

Cross-record duplicate detection via content hash (admin scan, not a live check on every upload):
```ts
const [files] = await bucket.getFiles({ prefix: "entities/" });
const byHash = new Map<string, Ref[]>();
for (const f of files) {
  const ref = pathToOwningRecord.get(f.name);
  if (!ref) continue; // orphan — handled by the separate orphan-cleanup pass
  const hash = f.metadata.md5Hash;
  if (!hash) continue;
  (byHash.get(hash) ?? byHash.set(hash, []).get(hash)!).push(ref);
}
const groups = [...byHash.entries()]
  .filter(([, refs]) => refs.length > 1)
  .map(([hash, refs]) => ({ hash, refs, crossRecord: new Set(refs.map((r) => r.recordId)).size > 1 }));
```

Merge-time classification (pure, no I/O — keeps the actual copy/delete calls separately parallelizable):
```ts
type MergeAction =
  | { kind: "skip"; url: string; hash: string }      // exact duplicate already on the survivor
  | { kind: "keepAsIs"; url: string }                // unresolvable URL — no bytes to move
  | { kind: "copy"; url: string; path: string };      // move into survivor's prefix

function classifyMergeImages(
  removeImages: string[],
  hashes: Map<string, string>,
  seenHashes: Set<string>
): MergeAction[] {
  const actions: MergeAction[] = [];
  for (const url of removeImages) {
    const path = imageUrlToStoragePath(url);
    const hash = path ? hashes.get(path) : undefined;
    if (hash && seenHashes.has(hash)) { actions.push({ kind: "skip", url, hash }); continue; }
    if (!path) { actions.push({ kind: "keepAsIs", url }); continue; }
    actions.push({ kind: "copy", url, path });
    if (hash) seenHashes.add(hash);
  }
  return actions;
}
```

Orphan cleanup (scan is informational and can use a stale/cached record list; the delete step must re-verify fresh):
```ts
// GET: informational — cached list is fine, nothing is destroyed
const orphans = files.filter((f) => !knownPaths.has(f.name));

// DELETE: re-verify against a FRESH read right before the irreversible call
const { known } = await knownGoodPaths(await readFreshRecords());
const toDelete = requestedPaths.filter((p) => !known.has(p));
await deleteStorageObjectsByPath(toDelete);
```

### Env vars
Same bucket var as the underlying Storage layer (see firebase-storage entry) — this pipeline introduces no new env vars of its own.

### Gotchas
- **A server-side bucket copy does not carry over ACLs** — after copying an object into the surviving record's prefix during a merge, you must re-apply the public/access ACL explicitly, or the merged image silently 403s.
- **Never derive "belongs to this record" from anything except a prefix match on the actual Storage path.** Matching on substring of the URL, or trusting a client-supplied "this is mine" flag, reopens the same cross-record injection the ownership check exists to close.
- **A `verified`/moderation flag needs its own remap during merges.** When an image is classified `skip` (an exact duplicate of one the survivor already owns), the removed record's own "verified" flag has no image of its own to attach to anymore — resolve it by mapping the hash back to whichever surviving-record URL carries it, or the flag silently disappears even though the underlying bytes survive.
- **Orphan-scan staleness is asymmetric.** A GET/list of orphans can tolerate a cached/stale record snapshot (nothing is destroyed by looking). The DELETE step must not reuse that same stale snapshot — always re-fetch immediately before deleting, since an image could have been re-attached in between.
- **Union-find, not pairwise grouping, for cross-record duplicate clusters.** If record A and B share a hash, and B and C share a different hash, a naive pairwise grouping produces two disconnected cards instead of one 3-way cluster — collapse transitively-connected hash-groups before rendering to an admin.
- **Don't build a custom resize/thumbnail pipeline if the framework already does it.** `next/image` (or equivalent) can cover deviceSizes/imageSizes-based responsive serving entirely at read time; a bespoke resize step is extra infrastructure for no benefit unless you've hit a proven gap (e.g. non-image formats, server-side thumbnailing needs).
- **Delete the surviving record's write only after it succeeds, and clean up the losing record's Storage objects only after that.** Reversing the order risks orphaning a live image if the metadata write fails midway.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. No cross-adopter variation to report — the full lifecycle (ownership-scoped mutation, moderation/verification tracking, cross-record duplicate detection via content hash, orphan cleanup, and merge-time reconciliation) is implemented by the one adopter that layers this on top of its Storage-backed image uploads; image resize/serving itself is left entirely to the framework's image component rather than a custom pipeline.
