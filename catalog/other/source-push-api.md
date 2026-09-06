# Source Push API (planned)

- **category**: other
- **provider**: internal (spec for future generic third-party deal-source integrations, e.g. a retail chain's deals feed)
- **reusable**: no — not yet implemented; nothing to extract until the concrete design lands.
- **docs**: n/a (internal spec — see `docs/SOURCE_PUSH_API.md` in the adopting repo)

## Overview
A designed-but-unbuilt endpoint that would let an external deal-source provider push coupons via one authenticated call and have them fan out into the saved-items collection of every user subscribed to that source. Verified against the actual repo: the subscription side (`sources/{sourceId}` data model, self-subscribe/unsubscribe) is real and shipped; the push-receiving endpoint, its auth (`sourceKeys` collection), and the fan-out write logic are **spec-only — no route file, no `sourceKeys` collection, and no fan-out code exist in the codebase**. Everything below the Prerequisites/Setup/Core-pattern headers describes *intended* design pulled directly from `docs/SOURCE_PUSH_API.md`, not working code.

## Playbook

### Prerequisites
- An existing Firestore collection modeling "sources" with a `subscriberUids: string[]` array users can self-add/remove (already shipped — see `packages/core/src/firebase/sources.ts`).
- A Firebase Admin SDK server context capable of bypassing client Firestore rules (the repo already has this for its `/api/mcp` route, which this spec explicitly says to mirror).
- An existing hashed-token pairing pattern to copy (`packages/core/src/firebase/tokens.ts` / `domain/pat.ts`): generate a random token, store only its SHA-256 hash, show the plaintext once.

### Setup steps
1. Add a new collection (e.g. `sourceKeys/{sha256(key)}`) storing `{ sourceId, preview, createdAt }` — no client read/write; only an admin flow may create/delete, enforced in security rules. The API route reads it only via the Admin SDK.
2. Build an admin UI action that generates a random key, hashes it, writes the `sourceKeys` doc, and shows the plaintext key to the admin exactly once (mirrors `createPairingToken` in `tokens.ts`).
3. Add a `POST /api/sources/push` route handler (Node runtime, not edge, since it needs the Admin SDK) that: extracts the `Authorization: Bearer <key>` header, hashes it, looks up `sourceKeys/{hash}` to resolve `sourceId`, and 401s on a miss.
4. Validate the request body (a batch of coupon objects, e.g. up to ~50) with a schema validator (zod), same as the existing `/api/mcp` route does.
5. Read `sources/{sourceId}.subscriberUids`, then for each coupon × each subscriber, write a coupon document into that user's own coupon subcollection, tagged with the source's id/name and the provider's `externalId` (if given) for dedup.
6. Use batched/bulk writes (e.g. Firestore `BulkWriter`) since the write volume is subscribers × coupons.
7. Before writing, skip subscribers who already have a coupon with the same `sourceId` + dedup key (`externalId`, or `code` if no `externalId` was given).
8. Return a summary count (pushed / skipped duplicates / subscriber count).

### Core pattern
Design-intent sketch only — untested, not present in the repo. Sanitized/generic shape for a "push fans out to subscribers" endpoint secured by a hashed API key:

```ts
// app/api/sources/push/route.ts  (INTENDED DESIGN — not implemented)
import { z } from "zod";
import crypto from "node:crypto";
import { adminDb } from "@/lib/firebase-admin"; // Admin SDK, bypasses security rules

export const runtime = "nodejs";

const PushItemSchema = z.object({
  title: z.string(),
  code: z.string(),
  externalId: z.string().optional(), // used for de-dup
  // ...additional optional fields
});
const PushBodySchema = z.object({
  items: z.array(PushItemSchema).max(50),
});

async function resolveSourceId(bearerKey: string): Promise<string | null> {
  const hash = crypto.createHash("sha256").update(bearerKey).digest("hex");
  const doc = await adminDb.collection("sourceKeys").doc(hash).get();
  return doc.exists ? (doc.data()!.sourceId as string) : null;
}

export async function POST(req: Request) {
  const authHeader = req.headers.get("authorization") ?? "";
  const key = authHeader.replace(/^Bearer\s+/i, "");
  const sourceId = await resolveSourceId(key);
  if (!sourceId) return Response.json({ error: "Unauthorized" }, { status: 401 });

  const parsed = PushBodySchema.safeParse(await req.json());
  if (!parsed.success) return Response.json({ error: parsed.error.flatten() }, { status: 400 });

  const sourceSnap = await adminDb.collection("sources").doc(sourceId).get();
  const subscriberUids: string[] = sourceSnap.data()?.subscriberUids ?? [];

  const bulkWriter = adminDb.bulkWriter();
  let pushed = 0, skipped = 0;

  for (const uid of subscriberUids) {
    for (const item of parsed.data.items) {
      const dedupKey = item.externalId ?? item.code;
      const existing = await adminDb
        .collection(`users/${uid}/items`)
        .where("sourceId", "==", sourceId)
        .where("sourceDedupKey", "==", dedupKey)
        .limit(1)
        .get();
      if (!existing.empty) { skipped++; continue; }

      const ref = adminDb.collection(`users/${uid}/items`).doc();
      bulkWriter.set(ref, {
        ...item,
        sourceId,
        sourceDedupKey: dedupKey,
        createdAt: new Date(),
      });
      pushed++;
    }
  }
  await bulkWriter.close();

  return Response.json({ pushed, skipped, subscribers: subscriberUids.length });
}
```

### Env vars
None yet defined — the spec proposes a hashed-key `sourceKeys` Firestore collection instead of an env var per source.

### Gotchas
- The self-subscribe/unsubscribe side already works today and follows the same "caller may only add/remove their own uid" rule as an existing group self-join pattern — don't design a new client-writable path for `subscriberUids`, reuse this one.
- The spec explicitly rejects retroactive backfill (subscribing does not deliver past pushes) and unsubscribe cleanup (already-delivered items stay put) — v1 scope is deliberately narrow, don't over-build.
- Fan-out is `subscribers × items-per-call`, so batching/chunked writes are load-bearing, not optional, even at modest subscriber counts.
- Auth is meant to mirror an already-shipped hashed pairing-token pattern in the same repo (`tokens.ts`/`domain/pat.ts`) — copy that hashing/lookup approach rather than inventing a new one.

### Playbook confidence: medium (spec-only/unimplemented — no route file or `sourceKeys` collection exists in the repo; code above is a design-intent sketch, not verified working code)

## Adoption
Used in **1** repo in this marketplace. Only the subscription side (which sources a user follows)
is implemented there — the push-receiving endpoint itself remains spec-only.
