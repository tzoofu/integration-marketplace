# Response Cache Layer

- **category**: other
- **provider**: internal (custom) — built on Next.js's cache primitives, no external caching product
- **reusable**: yes — the "typed cache-tag function + centralized revalidate() choke point" pattern, and the disk-backed dev cache handler, are both framework-level and not app-domain-specific.
- **docs**: https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers

## Overview
A two-tier caching system that keeps repeated database reads out of the hot path. Tier 1 wraps every cacheable read with Next.js `"use cache"` + `cacheTag`/`cacheLife`, with invalidation centralized behind a small set of typed `revalidate*()` functions so busting a tag can't be typo'd. Tier 2 is a custom disk-backed cache handler that persists that same cache to a local directory for dev only (the platform's own handler is used in production), so a dev-server restart or bundler recompile doesn't repay a full backend scan.

## Playbook

### Prerequisites
- Next.js with the `"use cache"` directive enabled (`experimental: { useCache: true }` in `next.config.ts`, or its stable equivalent depending on Next.js version).
- A read layer (DB/API client) that the cache wraps — this pattern caches at the "typed accessor function" boundary, not inside the DB client itself.
- Deploying to a platform with its own production cache handler (e.g. Vercel) — the disk handler below is explicitly dev-only and must not run in production.

### Setup steps
1. Enable `"use cache"` in `next.config.ts` and set `cacheHandlers` to a per-environment value: `undefined` in production (use the platform default), and a custom disk handler locally.
2. Define one tag constant per logical collection/entity (e.g. `CACHE_TAGS.things`), plus a per-id tag helper (`thingCacheTag(id)`) and a per-user/per-key tag helper where a resource is looked up by something other than a numeric id.
3. Wrap every cacheable read in a function with `"use cache"` as its first statement, call `cacheTag(...)` with the relevant tag(s), and set `cacheLife(...)` — a short `{stale, revalidate, expire}` window for frequently-mutated collections, `"max"` for small/admin-curated ones that only change via explicit invalidation.
4. Write a small number of typed `revalidate*()` functions (never call `revalidateTag()` directly at call sites) — one for "bust the list", one for "bust one entity by id", one for "bust one user's data" — and call the matching one from every mutation path (route handlers, use-cases, scripts).
5. Implement the disk-backed dev cache handler implementing `get`/`set`/`refreshTags`/`getExpiration`/`updateTags`, gated off in production via an environment check (e.g. `process.env.VERCEL`).
6. Add a hard max-age ceiling in the disk handler independent of each entry's own `expire`, so tags with `cacheLife("max")` (effectively ~1 year) can't be served stale forever if a `revalidateTag()` call is ever missed.

### Core pattern
```ts
// lib/constants.ts — one tag per collection, plus per-id/per-key helpers
export const CACHE_TAGS = {
  things: "things",
  smallCollection: "smallCollection",
} as const;
export type CacheTag = (typeof CACHE_TAGS)[keyof typeof CACHE_TAGS];

export const CACHE_TAG_LIFE: Record<CacheTag, "max" | "hours"> = {
  things: "max",
  smallCollection: "max",
};

export function keyedCacheTag(key: string): string {
  return `entity-${key}`;
}

// lib/cache.ts — typed reads + the one sanctioned invalidation surface
import { cacheTag, cacheLife, revalidateTag } from "next/cache";
import { CACHE_TAGS, CACHE_TAG_LIFE, keyedCacheTag, type CacheTag } from "./constants";
import { readThings, readOneThing, readSmallCollection } from "./db";

export type ListRevalidateTag = Exclude<CacheTag, typeof CACHE_TAGS.things>;
export function revalidate(tag: ListRevalidateTag): void {
  revalidateTag(tag, CACHE_TAG_LIFE[tag]);
}

export async function getCachedThings() {
  "use cache";
  cacheTag(CACHE_TAGS.things);
  cacheLife({ stale: 30, revalidate: 60, expire: 300 });
  return readThings();
}

// Tagged only with its own id tag, not the shared list tag, so one entity's
// edit can't invalidate every other entity's cached detail read.
export async function getCachedThing(id: string) {
  "use cache";
  cacheTag(keyedCacheTag(id));
  cacheLife({ stale: 60, revalidate: 300, expire: 3600 });
  return readOneThing(id);
}

export function revalidateThing(id: string): void {
  revalidateTag(keyedCacheTag(id), "max");
}

// Busts the shared list tag AND every touched entity's own tag — pass every
// id a mutation actually created/updated/deleted, even from a partial batch.
export function revalidateThingList(ids: readonly string[]): void {
  revalidateTag(CACHE_TAGS.things, CACHE_TAG_LIFE[CACHE_TAGS.things]);
  for (const id of new Set(ids)) revalidateTag(keyedCacheTag(id), "max");
}

export async function getCachedSmallCollection() {
  "use cache";
  cacheTag(CACHE_TAGS.smallCollection);
  cacheLife("max");
  return readSmallCollection();
}
```

```js
// cache-handlers/disk-handler.js — dev-only persistent cacheHandlers.default
// Interface: https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers
const fs = require("fs");
const fsp = require("fs/promises");
const path = require("path");
const crypto = require("crypto");

const CACHE_DIR = process.env.APP_CACHE_DIR || path.join(process.cwd(), ".dev-cache");
const ENTRY_DIR = path.join(CACHE_DIR, "entries");
const TAGS_FILE = path.join(CACHE_DIR, "tags.json");
// Hard ceiling independent of any entry's own `expire`, since cacheLife("max")
// resolves to ~365 days — without this, a missed revalidateTag() call could
// serve a stale entry for a year instead of failing safe.
const MAX_AGE_MS = Number(process.env.APP_CACHE_MAX_AGE_HOURS ?? 12) * 60 * 60 * 1000;
fs.mkdirSync(ENTRY_DIR, { recursive: true });

function entryPath(cacheKey) {
  return path.join(ENTRY_DIR, crypto.createHash("sha256").update(cacheKey).digest("hex") + ".json");
}
async function writeAtomic(file, contents) {
  const tmp = `${file}.${process.pid}.${Math.random().toString(36).slice(2)}.tmp`;
  await fsp.writeFile(tmp, contents);
  await fsp.rename(tmp, file); // write-then-rename: never serve a partial entry
}

let tagTimestamps = new Map();
try {
  tagTimestamps = new Map(Object.entries(JSON.parse(fs.readFileSync(TAGS_FILE, "utf8"))));
} catch {}

module.exports = {
  async get(cacheKey, softTags) {
    try {
      const stored = JSON.parse(await fsp.readFile(entryPath(cacheKey), "utf8"));
      if (Date.now() > stored.timestamp + stored.expire * 1000) return undefined;
      if (Date.now() - stored.timestamp > MAX_AGE_MS) return undefined;
      const tagInvalidated = (tags) => tags?.some((t) => (tagTimestamps.get(t) ?? 0) >= stored.timestamp);
      if (tagInvalidated(stored.tags) || tagInvalidated(softTags)) return undefined;
      const buf = Buffer.from(stored.value, "base64");
      return {
        value: new ReadableStream({ start(c) { c.enqueue(new Uint8Array(buf)); c.close(); } }),
        tags: stored.tags, stale: stored.stale, timestamp: stored.timestamp,
        expire: stored.expire, revalidate: stored.revalidate,
      };
    } catch {
      return undefined; // any failure degrades to a plain cache miss
    }
  },
  async set(cacheKey, pendingEntry) {
    const entry = await pendingEntry;
    const reader = entry.value.getReader();
    const chunks = [];
    for (;;) {
      const { done, value } = await reader.read();
      if (done) break;
      chunks.push(Buffer.from(value));
    }
    const buf = Buffer.concat(chunks);
    await writeAtomic(entryPath(cacheKey), JSON.stringify({
      value: buf.toString("base64"), tags: entry.tags ?? [], stale: entry.stale,
      timestamp: entry.timestamp, expire: entry.expire, revalidate: entry.revalidate,
    }));
  },
  async refreshTags() {
    try {
      tagTimestamps = new Map(Object.entries(JSON.parse(fs.readFileSync(TAGS_FILE, "utf8"))));
    } catch {}
  },
  async getExpiration(tags) {
    return tags.reduce((max, t) => Math.max(max, tagTimestamps.get(t) ?? 0), 0);
  },
  async updateTags(tags) {
    const now = Date.now();
    for (const t of tags) tagTimestamps.set(t, now);
    await writeAtomic(TAGS_FILE, JSON.stringify(Object.fromEntries(tagTimestamps)));
    // best-effort eviction of matching entries so the dir doesn't grow unbounded
  },
};
```

```ts
// next.config.ts — wire the dev-only handler, skip it on the deploy platform
function localCacheHandlers() {
  if (process.env.VERCEL) return undefined; // production keeps the platform's own handler
  return { default: require.resolve("./cache-handlers/disk-handler.js") };
}

const nextConfig = {
  experimental: { useCache: true },
  cacheHandlers: localCacheHandlers(),
};
export default nextConfig;
```

### Env vars
- `APT_CACHE_DIR` (or equivalent) — overrides the disk cache directory, e.g. so tests can point it at a temp dir
- `APT_CACHE_DEBUG` — enables verbose cache hit/miss/invalidate logging
- `APT_CACHE_MAX_AGE_HOURS` — overrides the hard age ceiling on disk-cached entries
- `VERCEL` (or platform-equivalent) — gates which cache handler loads; not app-specific, set automatically by the deploy platform

### Gotchas
- `cacheLife("max")` resolves to a ~365-day expire internally — tags using it have *no* meaningful time-based expiry, so tag invalidation is the only thing that ever refreshes them. A disk-backed dev handler must add its own independent max-age ceiling as a fail-safe, or a missed `revalidateTag()` call serves year-stale data.
- Tag scope matters: a shared "list" tag and per-entity tags must be separate, or editing one entity busts every other entity's cached detail page. Route every mutation through a small number of typed `revalidate*()` helpers (never raw `revalidateTag()` at call sites) so this invariant can't be typo'd away.
- Views that reuse another tag on purpose (e.g. a filtered/narrowed read over the same underlying collection) should say so explicitly in a comment — it's the one case where NOT minting a new tag is correct, but it's non-obvious to a future reader.
- The disk handler's `get()` is not wrapped in try/catch by the framework — an unhandled throw surfaces as a render error, so every failure path inside the handler must degrade to a plain cache miss instead of throwing.
- Use write-then-rename (atomic) for on-disk cache entries; otherwise a crash mid-write can leave a corrupt/partial entry that gets served as a hit.
- Tie-breaking on invalidation timestamps should favor evicting (use `>=` not `>`) when an invalidation and a cache write land in the same millisecond — an extra redundant read is a better trade than serving stale data.
- The disk cache is dev-only and must be excluded on the deploy platform (e.g. gated on `process.env.VERCEL`) — the platform's own production cache handler is better and this pattern must not compete with it in production.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. That adopter applies the pattern across a handful of
distinct entity collections (a mix of frequently-mutated and admin-curated, small ones), with
per-entity and per-list tags kept separate, plus the disk-backed dev handler for cross-restart
persistence.
