# Local Filesystem Storage

- **category**: storage
- **provider**: none (custom `LocalStorageAdapter`, no cloud provider)
- **reusable**: no — this is a local-disk stand-in behind a `StorageAdapter` interface, explicitly structured so a cloud backend (S3/Firebase Storage/etc.) could be swapped in later, but no such backend exists in this repo yet.
- **docs**: https://nodejs.org/api/fs.html#fspromises-api

## Overview
A hand-rolled storage abstraction that persists everything — binary audio, job state, transcripts, summaries — as plain files/JSON under a local `data/` directory instead of a database or cloud bucket. Reach for this shape only for a single-instance/local-dev tool where you want a `StorageAdapter` seam ready for a future swap to real cloud storage, without paying for one yet.

## Playbook

### Prerequisites
- Node.js `fs/promises` and `path` (no third-party packages).
- A writable, persistent directory for the process (`process.cwd()/data` in this repo — not viable on typical serverless/read-only-filesystem deployments).

### Setup steps
1. Define a `StorageAdapter` interface describing every operation the app needs (put/get for each artifact type), independent of the backing implementation.
2. Implement one concrete adapter (`LocalStorageAdapter`) against the local filesystem; keep it the *only* class satisfying the interface for now.
3. Expose a single cached factory function (`getStorage()`) that lazily constructs and memoizes the adapter, so callers depend on the interface, never on `LocalStorageAdapter` directly — that's what makes swapping in a cloud adapter later a one-file change.
4. Lay out the data directory by artifact kind (`data/audio/`, `data/jobs/`, `data/transcripts/`), and always `mkdir(..., { recursive: true })` before writing — never assume the directory tree exists.
5. For "does this file exist" queries, prefer scanning a directory for a filename prefix (extension unknown ahead of time) over guessing the extension.
6. Read helpers should treat `ENOENT` as a normal "not found" (return `null`), and rethrow every other error — don't let a permissions or disk error silently look like "missing".

### Core pattern
```ts
// storage/index.ts — the seam
export interface StorageAdapter {
  putBlob(id: string, buffer: Buffer, ext: string): Promise<void>;
  getBlobPath(id: string): Promise<string>;
  hasBlob(id: string): Promise<boolean>;

  putRecord<T>(collection: string, id: string, data: T): Promise<void>;
  getRecord<T>(collection: string, id: string): Promise<T | null>;
  listRecords<T>(collection: string): Promise<T[]>;
}

let cached: StorageAdapter | null = null;
export function getStorage(): StorageAdapter {
  if (!cached) cached = new LocalStorageAdapter();
  return cached;
}
```
```ts
// storage/local.ts — the one implementation
import { promises as fs } from "node:fs";
import path from "node:path";

const DATA_DIR = path.join(process.cwd(), "data");

async function ensureDir(dir: string) {
  await fs.mkdir(dir, { recursive: true });
}

async function readJson<T>(filePath: string): Promise<T | null> {
  try {
    return JSON.parse(await fs.readFile(filePath, "utf-8")) as T;
  } catch (err) {
    if ((err as NodeJS.ErrnoException).code === "ENOENT") return null;
    throw err;
  }
}

export class LocalStorageAdapter implements StorageAdapter {
  async putBlob(id: string, buffer: Buffer, ext: string): Promise<void> {
    const dir = path.join(DATA_DIR, "blobs");
    await ensureDir(dir);
    const clean = ext.replace(/^\./, "").toLowerCase();
    await fs.writeFile(path.join(dir, `${id}.${clean}`), buffer);
  }

  async getBlobPath(id: string): Promise<string> {
    const dir = path.join(DATA_DIR, "blobs");
    const files = await fs.readdir(dir).catch(() => []);
    const match = files.find((f) => f.startsWith(`${id}.`));
    if (!match) throw new Error(`Blob not found for ${id}`);
    return path.join(dir, match);
  }

  async hasBlob(id: string): Promise<boolean> {
    return this.getBlobPath(id).then(() => true, () => false);
  }

  async putRecord<T>(collection: string, id: string, data: T): Promise<void> {
    const dir = path.join(DATA_DIR, collection);
    await ensureDir(dir);
    await fs.writeFile(path.join(dir, `${id}.json`), JSON.stringify(data, null, 2));
  }

  async getRecord<T>(collection: string, id: string): Promise<T | null> {
    return readJson<T>(path.join(DATA_DIR, collection, `${id}.json`));
  }

  async listRecords<T>(collection: string): Promise<T[]> {
    const dir = path.join(DATA_DIR, collection);
    await ensureDir(dir);
    const files = await fs.readdir(dir);
    const records: T[] = [];
    for (const f of files) {
      if (!f.endsWith(".json")) continue;
      const r = await readJson<T>(path.join(dir, f));
      if (r) records.push(r);
    }
    return records;
  }
}
```

### Env vars
None — path is derived from `process.cwd()`, not configured.

### Gotchas
- **No cloud backend exists behind this interface yet** — the `StorageAdapter` seam is aspirational, not proof that a swap is trivial in practice; verify no code elsewhere assumes local-file specifics (e.g. reading `getBlobPath()`'s result with `fs` directly) before actually swapping in S3/Firebase Storage.
- **Doesn't survive redeploys or scale past one instance** — files live on local disk, so this breaks on any platform with an ephemeral or non-shared filesystem (most serverless/container platforms), and two instances would each have their own inconsistent copy.
- **Extension-agnostic lookup by directory scan** — because the extension isn't always known ahead of time (upload could be any audio format), "does X exist" is implemented as "scan the directory for a filename prefix match," not a direct `fs.access` on a guessed path.
- **Treat `ENOENT` as domain-`null`, not an exception** — every read helper must special-case the not-found error code, otherwise normal "no data yet" callers get an unhandled throw instead of a clean `null`.
- **Always `mkdir(recursive: true)` before every write, not just once at boot** — cheap enough to call every time, and avoids ordering bugs if a fresh checkout or wiped `data/` dir is used.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. No cross-adopter variation to report — this is the sole real-world backing store for a local-dev/single-instance tool that persists uploaded binary media, job state, and derived text artifacts as files/JSON under a local directory, with no database or cloud storage involved.
