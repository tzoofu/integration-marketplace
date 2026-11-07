# Android local crash log with manual share

- **category**: analytics
- **provider**: internal (custom) on Android platform APIs (`Thread.UncaughtExceptionHandler`, `FileProvider`)
- **reusable**: yes — roughly 100 lines with no dependencies. The file naming, retention and report formatting are pure functions that port to any Android app.
- **docs**: https://developer.android.com/reference/java/lang/Thread.UncaughtExceptionHandler · https://developer.android.com/reference/androidx/core/content/FileProvider

Writes each uncaught crash (and a few chosen handled exceptions) to a small file in app-private storage, keeps the newest N, and lets the user share the latest one through the system share sheet. It works offline, with no Play Services, no account and no data leaving the device unless the user sends it. It can sit next to [firebase-crashlytics](firebase-crashlytics.md): the file is the fallback, Crashlytics is the central view.

## Playbook

### Prerequisites
- An `Application` subclass.
- `androidx.core` (for `FileProvider`).
- A Settings screen row ("Diagnostics" or "Share crash log").

### Setup steps
1. Write the pure helpers first: `crashLogFileName(timestamp, handled)`, `formatCrashReport(...)`, `crashLogsToPrune(fileNames, cap)`. They only take strings and numbers, so unit tests need no Android.
2. Write a `CrashLogManager` object that takes only a `Context` (so any entry point can call it). It exposes `install()`, `logHandled()`, `latestLogFile()` and `shareUri()`.
3. Call `CrashLogManager.install(this)` as the **first** line of `Application.onCreate()`.
4. Add `res/xml/file_paths.xml` with a `<files-path>` entry for the log directory, and declare the `FileProvider` in the manifest (`exported="false"`, `grantUriPermissions="true"`, authority `${applicationId}.fileprovider`).
5. Add the share row. Build the intent with the helper below and launch it. If there is no file yet, show a snackbar instead of an empty chooser.
6. Unit-test the three pure helpers, including "handled and unhandled names sort by time together".

### Core pattern
```kotlin
object CrashLogManager {
    const val CRASH_LOG_DIR = "crash_logs"
    const val MAX_RETAINED_LOGS = 5
    @Volatile private var installed = false

    // Never swallow the crash: the previous handler (system dialog, Play vitals, Crashlytics) always runs afterwards.
    fun install(context: Context) {
        if (installed) return
        installed = true
        val app = context.applicationContext
        val previous = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, e ->
            try { writeReport(app, buildReport(e), handled = false) }
            finally { previous?.uncaughtException(thread, e) }
        }
    }

    // Use at a few deliberate catch sites only; this is not a logging facility.
    fun logHandled(context: Context, tag: String, e: Throwable) =
        writeReport(context, buildReport(e, tag), handled = true)

    fun latestLogFile(context: Context): File? = logDir(context).listFiles()?.maxByOrNull { it.name }
    fun shareUri(context: Context, file: File): Uri =
        FileProvider.getUriForFile(context, context.packageName + ".fileprovider", file)

    private fun writeReport(context: Context, text: String, handled: Boolean) {
        val dir = logDir(context).apply { mkdirs() }
        File(dir, crashLogFileName(TS.format(Instant.now()), handled)).writeText(text)
        crashLogsToPrune(dir.list()?.toList().orEmpty(), MAX_RETAINED_LOGS).forEach { File(dir, it).delete() }
    }
    private fun logDir(c: Context) = File(c.filesDir, CRASH_LOG_DIR)
}

// UTC, fixed width: sorting file names IS sorting by time. No parsing needed.
private val TS: DateTimeFormatter = DateTimeFormatter.ofPattern("yyyyMMdd-HHmmss").withZone(ZoneOffset.UTC)

// Keep the timestamp immediately after "crash-" in both variants so handled and unhandled files interleave correctly.
internal fun crashLogFileName(ts: String, handled: Boolean = false) =
    if (handled) "crash-$ts-handled.txt" else "crash-$ts.txt"

internal fun formatCrashReport(ts: String, versionName: String, versionCode: Long, androidRelease: String,
                               sdkInt: Int, deviceModel: String, stackTrace: String) = buildString {
    appendLine("Time (UTC): $ts")
    appendLine("App version: $versionName ($versionCode)")
    appendLine("Android: $androidRelease (SDK $sdkInt)")
    appendLine("Device: $deviceModel")
    appendLine()
    append(stackTrace)
}

// Pure over names, not File objects: testable on the JVM.
internal fun crashLogsToPrune(names: List<String>, cap: Int) =
    if (names.size <= cap) emptyList() else names.sorted().dropLast(cap)
```
```xml
<!-- res/xml/file_paths.xml -->
<paths><files-path name="crash_logs" path="crash_logs/" /></paths>
<!-- manifest -->
<provider android:name="androidx.core.content.FileProvider" android:authorities="${applicationId}.fileprovider"
          android:exported="false" android:grantUriPermissions="true">
  <meta-data android:name="android.support.FILE_PROVIDER_PATHS" android:resource="@xml/file_paths" />
</provider>
```
```kotlin
// Share intent: return it, let the UI launch it. null = nothing to share yet.
fun diagnosticShareIntent(context: Context): Intent? {
    val file = CrashLogManager.latestLogFile(context) ?: return null
    val send = Intent(Intent.ACTION_SEND).apply {
        type = "text/plain"
        putExtra(Intent.EXTRA_STREAM, CrashLogManager.shareUri(context, file))
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    // The grant flag is needed on the chooser too, or the receiving app can't open the attachment.
    return Intent.createChooser(send, null).apply { addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION) }
}
```

### Env vars
None.

### Gotchas
- **Always delegate to the previous handler**, in a `finally`. Swallowing the exception hides the crash from Play vitals and Crashlytics, and leaves the process in an undefined state. A failure while writing the file must not stop delegation either.
- **Handler order with Crashlytics needs no code.** Firebase's init provider runs before any `Application.onCreate()` and installs its handler. When you call `getDefaultUncaughtExceptionHandler()` you already get that one, so your handler becomes the outermost layer: local file, then Crashlytics, then the Android default.
- **`FLAG_GRANT_READ_URI_PERMISSION` on the chooser as well as the inner intent.** Leaving it off fails silently and only on a device: the share sheet opens, but the target app can't read the attachment.
- **Name by UTC timestamp right after the prefix.** Then "keep the newest N" is a string sort, and a `-handled` suffix before `.txt` never shifts the timestamp digits, so handled and unhandled files interleave in time order.
- **`install()` is idempotent** with a volatile flag, because several entry points can call it in one process.
- **Install in `Application.onCreate()`**, not in the Activity. A service, tile or widget can be the first component in a fresh process and would otherwise crash unrecorded.
- **Keep handled-exception logging narrow.** One or two call sites for failures that matter. Turning it into a general logger makes retention (N files) useless.
- **A crash before `filesDir` is writable, or an out-of-space crash, writes nothing.** That is accepted; Crashlytics covers it when enabled.
- **State the sharing in your privacy policy.** The report holds stack trace, app version, Android version and device model. It is user-initiated and never sent automatically.

### Playbook confidence: high
Extracted from a real implementation with unit tests for the three pure helpers. The handler and share paths touch real files and the system chooser, so they aren't unit-tested. The "crash before `filesDir` is writable writes nothing" gotcha is general knowledge, not from the source.

## Adoption
Used in **1** repo in this marketplace: a native Android app that pairs the local file with Firebase Crashlytics, adds a user opt-out for the remote reporting, and uses one handled-exception call site.
