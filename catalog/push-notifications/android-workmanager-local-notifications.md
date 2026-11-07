# Android local notifications via WorkManager (reconciled reminders, no push server)

- **category**: push-notifications
- **provider**: Android Jetpack WorkManager + `NotificationManagerCompat` (no FCM, no backend)
- **reusable**: yes — a scheduler that diffs "desired" vs "enqueued" work, two Hilt workers and a publisher; the domain function that decides *what* is due is the only app-specific part.
- **docs**: https://developer.android.com/develop/background-work/background-tasks/persistent and https://developer.android.com/develop/ui/views/notifications/build-notification

Time-based reminders (e.g. "job in 30 minutes", a morning summary, an evening "tomorrow" digest) and a periodic poll that turns data changes into notifications, all scheduled on the device. Works offline and in a no-account mode, and needs no server or push tokens. For server-sent push see [fcm-push](fcm-push.md).

## Playbook

### Prerequisites
- `androidx.work:work-runtime-ktx`, `androidx.hilt:hilt-work` (+ Hilt), `androidx.core` for `NotificationCompat`.
- `POST_NOTIFICATIONS` runtime permission (API 33+), notification channels created at startup.
- A pure function that returns the reminders due from the current data: `(data, leadMinutes, now, zone) -> List<Reminder(key, due, targetId)>`.

### Setup steps
1. Make the `Application` a `Configuration.Provider` with a `HiltWorkerFactory`, and remove WorkManager's default initializer from the manifest.
2. Create channels (reminders, summaries, events) once at startup.
3. Write a `Scheduler` singleton:
   - `periodic(...)` enqueues one unique 15-minute periodic worker (the minimum) as a safety net and event poller.
   - `reconcile(data, options, sent, uid)` computes the desired one-time reminders, enqueues missing ones with an initial delay, and cancels enqueued ones that are no longer desired.
4. Write a coordinator that collects the data + settings flows while signed in and calls `reconcile` on every change; on sign-out call `cancelAll()`.
5. Workers load a fresh snapshot (auth state + data, with a timeout), re-check that the item is still valid, then post through a publisher that respects permission and channel state.
6. Persist a small "sent" set of keys so a reminder or a daily summary fires only once even if work is re-enqueued.

### Core pattern
```kotlin
@Singleton
class NotificationScheduler @Inject constructor(@ApplicationContext private val context: Context) {
    private val work get() = WorkManager.getInstance(context)

    /** Safety net / poller. Offline-capable when there is nothing remote to poll. */
    fun periodic(offlineOnly: Boolean) {
        val net = if (offlineOnly) NetworkType.NOT_REQUIRED else NetworkType.CONNECTED
        val req = PeriodicWorkRequestBuilder<PollWorker>(15, TimeUnit.MINUTES)
            .setConstraints(Constraints.Builder().setRequiredNetworkType(net).build()).addTag(ALL).build()
        work.enqueueUniquePeriodicWork("app:poll", ExistingPeriodicWorkPolicy.UPDATE, req)  // UPDATE swaps constraints
    }

    suspend fun reconcile(reminders: List<Reminder>, sent: Set<String>, uid: String) {
        val now = Instant.now()
        val existing = work.getWorkInfosByTagFlow(ONCE).first().filter { !it.state.isFinished }
        val desired = linkedSetOf<String>()
        reminders.filter { it.key !in sent }.forEach { r ->
            val name = "$uid:${r.key}"
            desired += name
            if (existing.none { name in it.tags }) {
                val req = OneTimeWorkRequestBuilder<ReminderWorker>()
                    .setInputData(workDataOf("key" to r.key, "target" to r.targetId, "due" to r.due.toEpochMilli(), "uid" to uid))
                    .setInitialDelay(Duration.between(now, r.due).toMillis().coerceAtLeast(0), TimeUnit.MILLISECONDS)
                    .addTag(ALL).addTag(ONCE).addTag(name).build()
                // one slot per target, so a rescheduled item replaces its old reminder
                work.enqueueUniqueWork("app:$uid:${r.targetId}", ExistingWorkPolicy.REPLACE, req).await()
            }
        }
        existing.filter { info -> info.tags.none { it in desired } && info.state != WorkInfo.State.RUNNING }
            .forEach { work.cancelWorkById(it.id).await() }
    }

    suspend fun cancelAll() { work.cancelAllWorkByTag(ALL).await() }
    companion object { private const val ALL = "app_notifications"; private const val ONCE = "app_reminders" }
}

@HiltWorker
class ReminderWorker @AssistedInject constructor(
    @Assisted ctx: Context, @Assisted params: WorkerParameters,
    private val snapshot: Snapshot, private val coordinator: Coordinator,
) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result = try {
        val loaded = snapshot.load()                       // waits up to ~20s for auth + data
        when {
            loaded != null -> { coordinator.deliver(loaded.data, inputData, loaded.uid); Result.success() }
            snapshot.signedOut -> Result.success()          // nothing to do, don't retry forever
            else -> Result.retry()
        }
    } catch (e: Exception) {
        if (e is CancellationException) throw e
        Diagnostics.nonFatal("worker", e); Result.retry()
    }
}

class Snapshot @Inject constructor(private val repo: Repository, private val auth: AuthRepository) {
    val signedOut get() = auth.state.value == AuthState.SignedOut
    suspend fun load() = withTimeoutOrNull(20.seconds) {
        combine(auth.state, repo.data) { s, d -> s to d }
            .first { (s, d) -> s == AuthState.SignedOut || (s is AuthState.SignedIn && d != null) }
            .let { (s, d) -> if (s is AuthState.SignedIn && d != null) Loaded(s.user.uid, d) else null }
    }
}
```

```kotlin
// Publisher: silently skip when the user can't see it
fun enabled() = ContextCompat.checkSelfPermission(context, Manifest.permission.POST_NOTIFICATIONS) == PERMISSION_GRANTED &&
    NotificationManagerCompat.from(context).areNotificationsEnabled()
fun post(id: Int, channel: String, title: String, body: String, tap: PendingIntent): Boolean {
    if (!enabled() || manager.getNotificationChannel(channel)?.importance == NotificationManager.IMPORTANCE_NONE) return false
    manager.notify(id, NotificationCompat.Builder(context, channel).setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title).setContentText(body).setContentIntent(tap).setAutoCancel(true).build())
    return true
}
```

### Env vars
none.

### Gotchas
- **Reconcile, don't append.** Enqueue from a single "desired set" function on every data change and cancel the rest; otherwise deleted or rescheduled items keep firing. Tag each request with its stable name so the diff is cheap.
- **Deliver re-checks the data.** By the time a delayed worker runs the item may be closed, moved or deleted; load a fresh snapshot and drop stale reminders instead of trusting the input data.
- **Exactly-once needs a "sent" set.** WorkManager can re-run work (process death, `REPLACE`), and daily summaries are recomputed each reconcile. Persist sent keys (e.g. `"morning:2026-11-06"`) and filter by them.
- The periodic minimum is 15 minutes and is not exact; use one-time work with an initial delay for anything time-sensitive. Doze can still delay it by minutes — fine for reminders, not for alarms (use `AlarmManager.setExactAndAllowWhileIdle` for those).
- Use `ExistingPeriodicWorkPolicy.UPDATE` so changing constraints (e.g. switching between a no-network local mode and a cloud mode) takes effect without cancelling the schedule.
- Workers run with no UI process: the repository/auth must be able to start cold. Wrap the wait in a timeout, return `success()` when signed out (so work doesn't retry forever) and `retry()` only for transient failures.
- Re-throw `CancellationException` in workers; swallowing it breaks WorkManager's stop handling.
- Include the account uid in work names so a sign-out/sign-in as a different user can't deliver the previous user's reminders; cancel all on sign-out.
- Ask for `POST_NOTIFICATIONS` in context (when the user enables a reminder), and treat a disabled channel as "off" rather than an error.
- Notification tap → deep link: put a typed target in the intent extras, have the activity hand it to a small router, and let the main screen consume it once; with `singleTop` also handle `onNewIntent`.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace: a native Android app that schedules per-job reminders with a user-set lead time, morning and evening summaries, and a 15-minute poll that turns customer-side data changes into notifications; the poll drops its network constraint in an offline, account-less mode.
