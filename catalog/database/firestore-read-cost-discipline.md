# Firestore read-cost discipline (indexed lookups, segmented list cache, read logging)

- **category**: database
- **provider**: Google Firestore (Admin SDK on a server runtime with a framework data cache)
- **reusable**: yes. These are rules and small helpers, independent of the data model.
- **docs**: https://firebase.google.com/docs/firestore/pricing, https://firebase.google.com/docs/firestore/query-data/indexing, https://firebase.google.com/docs/firestore/query-data/index-overview

Firestore bills **one read per document returned** (`select()` projections too), so a single "load the whole list and filter in memory" on a hot path can cost thousands of reads every time it runs. This file is the set of rules and helpers that keep a collection of several thousand documents cheap on the free tier. Pair it with [firebase-usage-dashboard](../other/firebase-usage-dashboard.md) to see whether it is working.

## Playbook

### Prerequisites
- Firestore (native mode) with the Admin SDK on a server that has a tagged data cache (Next.js `"use cache"` + `cacheTag`, or any equivalent).
- Knowledge of which environments hit the **billed** project. Local dev, scripts and tooling servers usually share production credentials, so their reads count too.

### Setup steps
1. **Inventory the hot paths.** Anything that runs once per item (per scraped record, per created doc, per API call) must never read the full collection.
2. **Replace full-list lookups with indexed queries.** One helper per lookup key, chunked at 30 for `in` queries (Firestore's `in` limit):
   ```ts
   export async function findExistingByKey(keys: string[]) {
     const found = new Set<string>();
     for (let i = 0; i < keys.length; i += 30) {
       const snap = await db.collection("items").where("key", "in", keys.slice(i, i + 30)).select("key").get();
       snap.forEach((d) => found.add(d.get("key")));
     }
     return found;
   }
   ```
   Do dedup inside the **create path** (HTTP 409 / "skipped" result), not with a pre-check GET.
3. **Store derived, indexable fields on the doc** instead of computing them per read: a normalized dedup key (`addressKey`-style) and a boolean `archived`. Stamp them in the single set of write helpers, and exclude them from any user-writable allowlist.
4. **Make the bulk list live-only.** The cached "whole list" reader queries `where("archived","==",false)`. Archived means "dead, or stale and never touched". Keep a separate, clearly named reader for the few jobs that need every doc (storage audit, backups, admin "show hidden").
5. **Split the cached list into segments** (e.g. hot = recently created, cold = the rest, each its own cache entry and tag). Invalidation takes the written doc and busts only its segment; a bare id busts both.
6. **Log bulk reads in dev** (below) and watch for a line that fires per request or per item.
7. **Declare composite indexes** (`firestore.indexes.json`) for any query combining an equality and a range on different fields, and deploy with `firebase deploy --only firestore:indexes`. Single-field queries are auto-indexed.
8. After any raw write that bypasses your helpers (a migration script, a seed), run an idempotent backfill (`--dry` first) that re-stamps the derived fields, or those docs vanish from every list.

### Core pattern

Dev-only bulk read log (no cost, no noise in production):

```ts
const MIN_DOCS = 50;
export function logBulkRead(label: string, docCount: number): void {
  if (process.env.NODE_ENV === "production" || docCount < MIN_DOCS) return;
  console.info(`[firestore-read] ${label}: ${docCount} docs`);
}
```

Segment-aware invalidation (types make "bust everything" impossible by accident):

```ts
type Revalidatable = string | { id: string; createdAt?: string };
export function revalidateList(target: Revalidatable) {
  const seg = typeof target === "string" ? undefined : segmentOf(target.createdAt); // "hot" | "cold" | undefined
  if (seg) revalidateTag(`items:${seg}`, "max");
  else { revalidateTag("items:hot", "max"); revalidateTag("items:cold", "max"); } // safe, pays both
}
```

Recovering a doc that was just overwritten (point-in-time recovery off, version retention 1 hour): read it at a past `readTime`, then write it back field-by-field.

```ts
const snap = await db.runTransaction((t) => t.get(ref), {
  readOnly: true,
  readTime: Timestamp.fromMillis(Date.now() - 20 * 60 * 1000), // must be within the retention window
});
```

### Env vars
None specific. Dev/production gating uses `NODE_ENV`.

### Gotchas
- **The busted-cache trap.** Every write busts the list tag, and the next read refetches it. A loop that writes, reads the cached list, then writes again pays a full scan **per iteration**. This is the most expensive mistake, so keep reads out of write loops.
- **A doc without the derived `archived` field is invisible**, because `== false` never matches a missing field. Every write path must stamp it, or you need the backfill above.
- **Changing the derivation rule is a migration.** Editing the dedup-key normalizer or the archive predicate requires re-running the backfill, or old docs keep stale values.
- **Writes are cheap, reads are not.** Never trade extra reads for fewer writes. A typical bill is thousands of reads per write.
- **Dedup lookups see everything.** Indexed queries run over the whole collection, archived docs included, so dedup keeps working even when the bulk list skips them.
- **`revalidateTag(tag, "max")` is stale-while-revalidate.** The next request still gets old data. When a caller immediately re-reads its own write, use an immediate-expiry variant for that call only.
- **Attribute spikes with audit logs.** Aggregate metrics can't say who read. See [firebase-usage-dashboard](../other/firebase-usage-dashboard.md) for the Data Access audit-log setup and a per-caller counter, and enable it before an incident rather than after.
- **Local tooling is production traffic.** For a heavy debugging session, run the Firestore emulator (`firebase emulators:start --only firestore`), load a backup into it, and point the process at it with `FIRESTORE_EMULATOR_HOST=127.0.0.1:8080` **per command**, never in the env file. Auth and Storage still go to production.
- **Test the decision logic, not the I/O.** Keep dedup/derivation rules in pure modules with unit tests, and verify the Firestore calls against the emulator.

### Playbook confidence: high
Extracted from a real, locally available implementation. The numbers are illustrative and the free-tier limits are Google's, so re-check them.

## Adoption
Used in **1** repo in this marketplace. It targets a collection in the low thousands of documents with a bulk list cached in two `createdAt` segments and indexed `find*` helpers for every dedup key.
