# Firebase Crashlytics

- **category**: analytics
- **provider**: Google Firebase (Crashlytics)
- **reusable**: yes — standard Gradle-plugin + BOM setup any Android repo in this marketplace could reuse verbatim (init in `Application.onCreate()`, a thin manager wrapper, and a user-facing opt-out toggle). Only one Android repo in this marketplace uses it so far; another Android repo exists but doesn't use Crashlytics.
- **docs**: https://firebase.google.com/docs/crashlytics/get-started?platform=android

## Overview
Centralized crash reporting for a native Android app: uncaught crashes (and select handled exceptions) are recorded to Firebase Crashlytics via the standard `firebase-crashlytics` Gradle plugin + BOM, on top of/alongside any local on-device crash log.

## Playbook

**Note**: This playbook is synthesized from general Firebase Crashlytics documentation and standard SDK usage rather than extracted from a locally-available implementation, so treat it as a correct-by-the-book starting point rather than a verified extraction.

### Prerequisites
- A Firebase project with an Android app registered in it, and `google-services.json` downloaded into the app module (do not commit real project config to a public repo).
- Crashlytics enabled for the project in Firebase Console (Console → Build → Crashlytics → "Enable Crashlytics").

### Setup steps
1. Add the Google Services and Crashlytics Gradle plugins at the project level, and apply them in the app module's `build.gradle.kts`.
2. Add the Firebase BOM and the `firebase-crashlytics` (and typically `firebase-analytics`, though it can be deliberately omitted — see Gotchas) dependencies to the app module.
3. Place `google-services.json` in the app module root; gitignore it if the repo is public or shared beyond the team.
4. In your `Application` subclass's `onCreate()`, optionally call `FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(...)` based on a stored user preference (see step 6) before any crash could occur.
5. For handled exceptions you want surfaced in Crashlytics (not just uncaught crashes), call `FirebaseCrashlytics.getInstance().recordException(e)` at the specific catch site — don't wrap this indiscriminately around every try/catch, reserve it for exceptions worth centralized visibility into.
6. If you want a user-facing opt-out (common when a settings screen has a "crash reporting" toggle), back it with a `SharedPreferences` boolean and call `setCrashlyticsCollectionEnabled(enabled)` whenever it changes, plus once at startup to apply the stored value.
7. Build and run once, then trigger a test crash (e.g. a deliberate `throw RuntimeException("test crash")` behind a debug-only button) to confirm the crash appears in Firebase Console within a few minutes.

### Core pattern
```kotlin
// build.gradle.kts (project level)
plugins {
    id("com.google.gms.google-services") version "<latest>" apply false
    id("com.google.firebase.crashlytics") version "<latest>" apply false
}
```
```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:<latest>"))
    implementation("com.google.firebase:firebase-crashlytics")
    // implementation("com.google.firebase:firebase-analytics") // optional — see Gotchas
}
```
```kotlin
// A thin wrapper around the singleton, so call sites don't touch the SDK directly.
object CrashReporting {
    fun install(context: Context, collectionEnabled: Boolean) {
        FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(collectionEnabled)
    }

    fun setCollectionEnabled(enabled: Boolean) {
        FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(enabled)
    }

    fun recordException(e: Throwable) {
        FirebaseCrashlytics.getInstance().recordException(e)
    }
}
```
```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        val prefs = getSharedPreferences("prefs", MODE_PRIVATE)
        val enabled = prefs.getBoolean(KEY_CRASH_REPORTING_ENABLED, true)
        CrashReporting.install(this, enabled)
    }
}
```
```kotlin
// At a specific, deliberately-chosen catch site — not blanket-wrapped everywhere.
try {
    establishVpnConnection()
} catch (e: VpnEstablishException) {
    CrashReporting.recordException(e)
    // ... local handling ...
}
```

### Env vars
None — configuration is via `google-services.json` (checked into or excluded from the repo per its own policy) plus the Gradle plugin, not environment variables. A user-facing opt-out toggle, if present, is typically a `SharedPreferences` boolean, not an env var.

### Gotchas
- Crashlytics can be deployed **without** Firebase Analytics — they're separate products under the same BOM, and omitting Analytics is a legitimate, deliberate privacy choice (avoids adding behavioral tracking) rather than a misconfiguration. Don't assume the `firebase-analytics` dependency is required.
- `recordException()` is for exceptions you deliberately want centralized visibility into (rare, meaningful failure paths) — reserve it, don't sprinkle it into every catch block, or Crashlytics becomes noise instead of a signal.
- If you add a user-facing opt-out toggle, `setCrashlyticsCollectionEnabled()` must be called both at startup (to apply the stored preference) and on every toggle change — forgetting the startup call means the app defaults to Firebase's own default (collection enabled) until the user visits settings again.
- `google-services.json` is not a secret by cryptographic standards (Firebase's own docs say it's safe to expose in a client binary), but many teams still gitignore it as environment-specific config — check the specific repo's stated policy (e.g. a `PRIVACY.md`) rather than assuming either way.
- Crash reports can take several minutes to appear in Firebase Console after a test crash — don't conclude the integration is broken from an immediate empty dashboard.

### Playbook confidence: low
(Per the special-case note above: this playbook is synthesized from general Firebase Crashlytics documentation/best practice rather than extracted from a locally-available implementation, so treat it as a correct-by-the-book starting point rather than a verified extraction.)

## Adoption
Used in **1** repo(s) in this marketplace. It reports uncaught crashes (plus one deliberately narrow handled exception at a specific failure path) to Firebase Crashlytics as a centralized supplement to a local on-device crash-log file, with a user-facing Settings toggle to opt out. Deployed without Firebase Analytics alongside it — a deliberate choice, stated in project docs, to avoid adding behavioral tracking; Crashlytics is the only Firebase product integrated.
