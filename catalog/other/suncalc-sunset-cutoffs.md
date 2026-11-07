# Sunset-Based Order Cutoffs (suncalc)

- **category**: other
- **provider**: suncalc (open-source npm library, no service/account)
- **reusable**: yes — pure function of date + coordinates + timezone; no auth, network, or keys involved.
- **docs**: https://github.com/mourner/suncalc

Computes a weekly cutoff or reopen time that tracks the sun (e.g. "stop taking orders 18 minutes before sunset on Friday", "reopen 42 minutes after Saturday sunset") instead of a fixed clock time, so the time shifts correctly through the year. Reach for it when a business rule depends on candle-lighting/havdalah-style sunset offsets.

## Playbook

### Prerequisites
- `suncalc` (+ `@types/suncalc` for TypeScript). No API key, no network call at runtime.
- The business location's latitude/longitude and IANA timezone, fixed in code or settings.

### Setup steps
1. Pick the target weekday for the event (e.g. Friday for "entry", Saturday for "exit") and compute the *local calendar date* of its next occurrence in the business timezone — not the server's timezone. Use `Intl.DateTimeFormat` with `timeZone` and `formatToParts` to read weekday/year/month/day.
2. Build the target date at **12:00 UTC** of that local date and pass it to `SunCalc.getTimes(date, lat, lng)`. Noon avoids the date flipping to the neighbouring day for timezones near UTC.
3. Read `.sunset`, add the event's base offset (negative before sunset, positive after) plus an operator-configurable offset in minutes.
4. To compare against "now", convert the resulting instant back into the business timezone's `weekday * 1440 + hour * 60 + minute` ("week minutes") so it can live in the same ordering as fixed-time cutoff rules and wrap across the week boundary.
5. Expose the event as a configuration mode (`"entry" | "exit"`) plus an optional offset, stored with the rest of the cutoff rule, and keep the schema that validates saved settings in sync with the type (an omitted optional field is silently stripped by schema validation).

### Core pattern
```ts
import * as SunCalc from "suncalc";

const LAT = 0; // business latitude
const LNG = 0; // business longitude
const TZ = "Region/City";
const ENTRY_OFFSET_MS = -18 * 60_000; // before sunset
const EXIT_OFFSET_MS = 42 * 60_000; // after sunset
const WEEKDAY: Record<string, number> = { Sun: 0, Mon: 1, Tue: 2, Wed: 3, Thu: 4, Fri: 5, Sat: 6 };

export function sunsetEventUTC(mode: "entry" | "exit", offsetMinutes: number, now: Date): Date {
  const targetWeekday = mode === "entry" ? 5 : 6;
  let weekday = 0, year = 0, month = 0, day = 0;
  for (const p of new Intl.DateTimeFormat("en-US", {
    timeZone: TZ, weekday: "short", year: "numeric", month: "2-digit", day: "2-digit",
  }).formatToParts(now)) {
    if (p.type === "weekday") weekday = WEEKDAY[p.value] ?? 0;
    if (p.type === "year") year = parseInt(p.value);
    if (p.type === "month") month = parseInt(p.value) - 1;
    if (p.type === "day") day = parseInt(p.value);
  }
  const daysUntil = (targetWeekday - weekday + 7) % 7;
  const noonUTC = new Date(Date.UTC(year, month, day + daysUntil, 12));
  const { sunset } = SunCalc.getTimes(noonUTC, LAT, LNG);
  const base = sunset ? sunset.getTime() : noonUTC.getTime();
  return new Date(base + (mode === "entry" ? ENTRY_OFFSET_MS : EXIT_OFFSET_MS) + offsetMinutes * 60_000);
}
```

### Env vars
None.

### Gotchas
- Don't replace this with a hand-rolled solar formula; `suncalc` is the tested source and the offsets are the only business-specific part.
- `getTimes().sunset` can be `null`/invalid at extreme latitudes; the pattern above falls back to noon so the code never throws.
- Always derive the local date in the business timezone before calling `getTimes`; using the server's local date breaks around midnight and on hosts running in UTC.
- Week-minute comparison needs explicit wraparound handling (a Saturday-night reopen vs a Friday cutoff in the same week) — cover the boundary cases in unit tests before changing the math.
- Offsets are approximate halachic/customary values, not an authoritative zmanim calendar; if exact times matter, cross-check against a dedicated zmanim source.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. It feeds a per-weekday cutoff rule system that also supports plain fixed clock times.
