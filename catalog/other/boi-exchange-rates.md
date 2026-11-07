# Bank of Israel Exchange Rates

- **category**: other
- **provider**: Bank of Israel (BOI) Public API
- **reusable**: yes — a keyless public REST API; trivial to lift for any repo needing official ILS exchange rates.
- **docs**: https://www.boi.org.il/en/economic-roles/statistics/exchange-rates/

## Overview
Fetches official ILS exchange rates for foreign-currency invoices/documents from the Bank of Israel's public API. No API key is required. The rate is typically snapshotted/cached against the specific document date, not fetched live every time, since the official rate for a given date doesn't change once published.

## Playbook

### Prerequisites
- None — the API is public and keyless.
- A place to cache the fetched rate per (date, currency) pair (e.g. a `fxRates` collection/table) so you don't hit the API repeatedly for the same historical rate.

### Setup steps
1. Call `https://boi.org.il/PublicApi/GetExchangeRate?key=<CURRENCY_CODE>` (e.g. `USD`, `EUR`) to get the current published rate for that currency.
2. Treat `ILS` itself as a trivial rate of `1` — skip the network call entirely.
3. Build a cache key from `(date, currency)` — e.g. `${date}_${currency}` — and check it before calling the API, since a historical day's official rate never changes once published.
4. On a cache miss, fetch, parse the JSON body's rate field, and persist it to the cache alongside the currency and date for future lookups.
5. Return `null` (not throw) on a non-OK response or a missing/malformed rate field, and let the caller decide how to handle a missing rate (e.g. block invoice creation, or fall back to a manual rate entry) rather than crashing the whole request.

### Core pattern
```ts
const BOI_API = "https://boi.org.il/PublicApi/GetExchangeRate";

async function getCachedRate(cacheKey: string): Promise<number | null> {
  // e.g. a Firestore/SQL lookup — return the cached rate or null
  return null;
}

async function setCachedRate(cacheKey: string, rate: number, currency: string, date: string): Promise<void> {
  // persist { rate, currency, date, fetchedAt } for this cache key
}

export async function getIlsRate(currency: string, onDate: string): Promise<number | null> {
  if (currency === "ILS") return 1;

  const cacheKey = `${onDate}_${currency}`;
  const cached = await getCachedRate(cacheKey);
  if (cached !== null) return cached;

  const response = await fetch(`${BOI_API}?key=${encodeURIComponent(currency)}`);
  if (!response.ok) return null;

  const data = (await response.json()) as { currentExchangeRate?: number };
  const rate = data.currentExchangeRate;
  if (typeof rate !== "number") return null;

  await setCachedRate(cacheKey, rate, currency, onDate);
  return rate;
}
```

### Env vars
None — the API is public and keyless.

### Gotchas
- The endpoint returns the *current* published rate for a currency, not a historical rate for an arbitrary past date — "snapshot at issue date" means caching the rate the first time you look it up for that date, not that the API itself supports date-range historical lookups. Plan your caching strategy accordingly if you need true historical rates for old documents.
- Cache the result per `(date, currency)` so a document generated later for the same historical date reuses the originally recorded rate rather than silently drifting if you re-fetch.
- Fail soft (return `null`) rather than throwing on network/parsing errors — an exchange-rate lookup failure shouldn't take down document generation if the caller can degrade gracefully (e.g. prompt for a manual rate).

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace, fetching official ILS exchange rates for foreign-currency
invoices with the rate snapshotted/cached at the document's issue date.
