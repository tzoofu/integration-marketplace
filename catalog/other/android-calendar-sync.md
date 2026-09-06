# Android CalendarContract sync

- **category**: other
- **provider**: Android (on-device `CalendarContract` OS content provider — not a third-party/cloud API)
- **reusable**: no — Android-native, on-device OS integration; no other repo in this marketplace is a native mobile app.
- **docs**: https://developer.android.com/guide/topics/providers/calendar-provider

## Overview
Each device reads its own local calendar (via the Android `CalendarContract` content provider) and mirrors selected events into Firestore, one-way per device, so one member's upcoming events (and anything derived from them) are visible to the rest of a shared group without a centralized calendar service. The same sync path can also write reminder events back onto the device's calendar.

## Playbook

### Prerequisites
- `android.permission.READ_CALENDAR` and `android.permission.WRITE_CALENDAR` declared in the manifest and granted at runtime (Android's dangerous-permission flow — request via `ActivityResultContracts.RequestPermission()` or similar, and handle "not yet granted" as a normal, expected state).
- `androidx.work:work-runtime-ktx` (WorkManager) if you want periodic background sync rather than only foreground/on-open sync.
- A destination for the mirrored data (the source uses Firestore, but any backend works) plus its own security rules restricting write access to the owning user/group.

### Setup steps
1. Add the calendar permissions to `AndroidManifest.xml` and gate every `CalendarReader`/sync call behind a `ContextCompat.checkSelfPermission(...) == PERMISSION_GRANTED` check.
2. Write a thin read-only wrapper (`CalendarReader.readUpcoming`) that queries `CalendarContract.Events.CONTENT_URI` with a projection of just the fields you need (`_ID`, `TITLE`, `DTSTART`), a `DTSTART` range selection, and `DELETED = 0` — always exclude soft-deleted events, the provider doesn't do it for you.
3. Understand the core architecture constraint: `CalendarContract` can only ever see calendars actually synced to THIS device (via a Google/Exchange/etc. account signed into the OS) — there is no API for one device to read another device's calendar directly. Any "shared visibility" has to be built on top via your own backend, not via the calendar provider itself.
4. Write a background sync routine (a `CoroutineWorker` if using WorkManager) that: checks permission (no-op, not a failure, if ungranted — don't let WorkManager retry-loop something that isn't actually broken), reads upcoming device events, and upserts each into your own backend keyed by a stable `sourceEventId` so re-runs update rather than duplicate.
5. Schedule the worker both periodically (`WorkManager.enqueueUniquePeriodicWork`) and once eagerly on relevant app-open screens, so data feels fresh without waiting for the periodic interval.
6. If you need the write-back path (reminders written onto the device's own calendar), find a writable calendar via `CalendarContract.Calendars.CALENDAR_ACCESS_LEVEL >= CAL_ACCESS_CONTRIBUTOR`, then `ContentResolver.insert()` a new `Events` row with `CALENDAR_ID`, `TITLE`, `DTSTART`, `DTEND`, and `EVENT_TIMEZONE` — return `null` gracefully if no writable calendar exists (e.g. no account signed in yet) rather than throwing.

### Core pattern
```kotlin
// CalendarReader.kt — thin, permission-gated wrapper over CalendarContract
data class DeviceCalendarEvent(val eventId: Long, val title: String, val date: LocalDate)

object CalendarReader {
    // Every non-deleted event across every calendar on this device, from now
    // through daysAhead out. Callers must have already checked READ_CALENDAR.
    fun readUpcoming(context: Context, daysAhead: Long = 60): List<DeviceCalendarEvent> {
        val resolver = context.contentResolver
        val now = Instant.now()
        val end = now.plusSeconds(daysAhead * 86_400)
        val projection = arrayOf(CalendarContract.Events._ID, CalendarContract.Events.TITLE, CalendarContract.Events.DTSTART)
        val selection = "${CalendarContract.Events.DTSTART} >= ? AND ${CalendarContract.Events.DTSTART} <= ? AND ${CalendarContract.Events.DELETED} = 0"
        val args = arrayOf(now.toEpochMilli().toString(), end.toEpochMilli().toString())
        val events = mutableListOf<DeviceCalendarEvent>()
        resolver.query(CalendarContract.Events.CONTENT_URI, projection, selection, args, "${CalendarContract.Events.DTSTART} ASC")?.use { cursor ->
            val idIdx = cursor.getColumnIndexOrThrow(CalendarContract.Events._ID)
            val titleIdx = cursor.getColumnIndexOrThrow(CalendarContract.Events.TITLE)
            val startIdx = cursor.getColumnIndexOrThrow(CalendarContract.Events.DTSTART)
            while (cursor.moveToNext()) {
                val title = cursor.getString(titleIdx) ?: continue
                val date = Instant.ofEpochMilli(cursor.getLong(startIdx)).atZone(ZoneId.systemDefault()).toLocalDate()
                events.add(DeviceCalendarEvent(eventId = cursor.getLong(idIdx), title = title, date = date))
            }
        }
        return events
    }

    // Write-back path: mirror a reminder onto this device's own calendar as a
    // plain 30-minute event. Returns null if there's no writable calendar yet.
    fun insertReminder(context: Context, title: String, date: LocalDate): Long? {
        val resolver = context.contentResolver
        val calendarId = findWritableCalendarId(resolver) ?: return null
        val zone = ZoneId.systemDefault()
        val startMillis = date.atTime(LocalTime.of(9, 0)).atZone(zone).toInstant().toEpochMilli()
        val values = ContentValues().apply {
            put(CalendarContract.Events.CALENDAR_ID, calendarId)
            put(CalendarContract.Events.TITLE, title)
            put(CalendarContract.Events.DTSTART, startMillis)
            put(CalendarContract.Events.DTEND, startMillis + 30 * 60 * 1000)
            put(CalendarContract.Events.EVENT_TIMEZONE, zone.id)
        }
        return resolver.insert(CalendarContract.Events.CONTENT_URI, values)?.lastPathSegment?.toLongOrNull()
    }

    private fun findWritableCalendarId(resolver: ContentResolver): Long? {
        val projection = arrayOf(CalendarContract.Calendars._ID)
        val selection = "${CalendarContract.Calendars.CALENDAR_ACCESS_LEVEL} >= ?"
        val args = arrayOf(CalendarContract.Calendars.CAL_ACCESS_CONTRIBUTOR.toString())
        resolver.query(CalendarContract.Calendars.CONTENT_URI, projection, selection, args, null)?.use { cursor ->
            if (cursor.moveToFirst()) return cursor.getLong(0)
        }
        return null
    }
}
```

```kotlin
// CalendarSyncWorker.kt — periodic + on-open background mirror into your own backend
class CalendarSyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        // No-op (not a failure) if permission isn't granted yet or the user
        // hasn't finished onboarding — WorkManager shouldn't retry-loop this.
        if (ContextCompat.checkSelfPermission(applicationContext, Manifest.permission.READ_CALENDAR) != PackageManager.PERMISSION_GRANTED) {
            return Result.success()
        }
        val deviceEvents = CalendarReader.readUpcoming(applicationContext)
        for (event in deviceEvents) {
            // Upsert keyed by sourceEventId so re-runs update, not duplicate.
            backend.upsertMirroredEvent(sourceEventId = event.eventId, title = event.title, date = event.date)
        }
        return Result.success()
    }
}
```

### Env vars
none — on-device OS content provider, no external network config. (The Firestore destination used in the source app has its own project config, but that's covered by whatever Firebase/Firestore integration entry applies, not this one.)

### Gotchas
- `CalendarContract` only ever exposes calendars synced to the CURRENT device via a signed-in account (Google/Exchange/etc.) — there is no cross-device read API. Any "see another member's calendar" feature has to be built as your own mirror/sync layer on top, exactly as this pattern does; it is not something the provider gives you natively.
- Always filter `DELETED = 0` in the selection — the provider keeps soft-deleted rows around (for sync-protocol reasons) and they will otherwise leak into your results.
- Treat "permission not granted" and "no writable calendar found" as normal `Result.success()` no-ops in a background worker, not failures — otherwise WorkManager will keep retrying a condition that isn't actually transient, burning battery/wake-locks for nothing.
- `insertReminder`'s writable-calendar lookup can return null when no account is signed into the device yet — always handle that gracefully (skip the write) rather than crashing on a null calendar id.
- Query only the columns you actually need in the projection — `CalendarContract.Events` rows carry many fields, and requesting all of them is unnecessary overhead for a simple upcoming-events read.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter mirrors each device's local calendar events
into a cloud backend for shared visibility across users, and also writes reminder events back onto
the device's own calendar — included in the catalog as a data-source integration signal even
though it's an OS content provider, not an external company/SDK.
