# Google Play Core In-App Update

- **category**: other
- **provider**: Google Play / Play Core (`app-update-ktx`)
- **reusable**: yes — the `AppUpdateManager` wrapper + priority-driven FLEXIBLE-vs-IMMEDIATE decision logic is generic and could be lifted into any other Android repo distributed via Play Store.
- **docs**: https://developer.android.com/guide/playcore/in-app-updates

## Overview
Google Play's in-app update API lets an Android app prompt the user to update (FLEXIBLE — downloads in the background, user chooses when to install) or force an update before continuing (IMMEDIATE — a full-screen blocking UI) without sending the user out to the Play Store listing. Which mode to use is normally driven by a per-release "update priority" value (0–5) set in Play Console: below some threshold you offer a flexible nudge, at or above it you force an immediate update.

## Playbook

### Prerequisites
- App is distributed through the Google Play Store (in-app updates only work for Play-installed builds, not sideloaded/debug builds — testing requires using the internal app-sharing/internal testing track).
- Play Console access to configure release priority per rollout.
- Gradle dependency on the Play Core in-app update artifact (`com.google.android.play:app-update` / the `-ktx` Kotlin extensions variant).

### Setup steps
1. Add the Play Core in-app update dependency to `build.gradle.kts` (the `-ktx` artifact for Kotlin coroutine/suspend-friendly APIs).
2. Create an `AppUpdateManager` via `AppUpdateManagerFactory.create(context)`.
3. On app start (typically in the main/launcher Activity's `onStart`/`onResume`), call `appUpdateManager.appUpdateInfo` to get an `AppUpdateInfo` — this reports `updateAvailability()`, `updatePriority()`, and which update types are allowed (`isUpdateTypeAllowed(...)`).
4. Write a pure decision function that maps `(updateAvailability, updatePriority)` to `None | Flexible | Immediate`, using a priority threshold (e.g. priority >= 4 → force IMMEDIATE, otherwise if an update is available → offer FLEXIBLE).
5. For FLEXIBLE: call `startUpdateFlowForResult(appUpdateInfo, activityResultLauncher, AppUpdateOptions.newBuilder(AppUpdateType.FLEXIBLE).build())`, register an `InstallStateUpdatedListener` to detect `InstallStatus.DOWNLOADED`, and prompt the user to restart (`appUpdateManager.completeUpdate()`) once the download finishes. Register/unregister the listener in `onStart`/`onStop` to avoid leaks.
6. For IMMEDIATE: call the same `startUpdateFlowForResult` with `AppUpdateType.IMMEDIATE` — Play handles the full-screen blocking UI itself; check the flow's result code and re-trigger the update check if the user somehow cancels/backs out of a supposedly mandatory update.
7. Guard against re-prompting a FLEXIBLE update repeatedly in the same process lifetime with a simple "already offered this session" boolean flag.
8. Set update priority per release in Play Console at rollout time — it is not something the app reads from its own config; it's an attribute of the release itself that Play serves back via `AppUpdateInfo`.

### Core pattern
```kotlin
// Pure decision logic — no Android/Play Core types, easy to unit test.
enum class UpdateMode { None, Flexible, Immediate }

const val FORCED_UPDATE_PRIORITY_THRESHOLD = 4

fun decideUpdateMode(
    updateAvailable: Boolean,
    updatePriority: Int,
): UpdateMode = when {
    !updateAvailable -> UpdateMode.None
    updatePriority >= FORCED_UPDATE_PRIORITY_THRESHOLD -> UpdateMode.Immediate
    else -> UpdateMode.Flexible
}

// Manager wrapper
class PlayUpdateManager(context: Context) {
    private val manager = AppUpdateManagerFactory.create(context)
    private var flexibleUpdateOffered = false

    private val listener = InstallStateUpdatedListener { state ->
        if (state.installStatus() == InstallStatus.DOWNLOADED) {
            // prompt user to restart, then: manager.completeUpdate()
        }
    }

    fun registerListener() = manager.registerListener(listener)
    fun unregisterListener() = manager.unregisterListener(listener)

    fun checkForUpdate(launcher: ActivityResultLauncher<IntentSenderRequest>) {
        manager.appUpdateInfo.addOnSuccessListener { info ->
            val available = info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE
            when (decideUpdateMode(available, info.updatePriority())) {
                UpdateMode.Immediate -> manager.startUpdateFlowForResult(
                    info, launcher, AppUpdateOptions.newBuilder(AppUpdateType.IMMEDIATE).build()
                )
                UpdateMode.Flexible -> if (!flexibleUpdateOffered) {
                    flexibleUpdateOffered = true
                    manager.startUpdateFlowForResult(
                        info, launcher, AppUpdateOptions.newBuilder(AppUpdateType.FLEXIBLE).build()
                    )
                }
                UpdateMode.None -> Unit
            }
        }
    }
}
```

### Env vars
None — update priority is set per-release in Play Console, not via app config or environment variables.

### Gotchas
- In-app updates only function for Play Store-installed builds; local/debug/sideloaded builds always report no update available, which can look like a bug during development — test via Play's internal app-sharing or internal testing track.
- FLEXIBLE updates require the app to detect `InstallStatus.DOWNLOADED` and explicitly call `completeUpdate()` (usually after prompting the user) — the update silently sits downloaded-but-not-installed otherwise.
- The `InstallStateUpdatedListener` must be registered/unregistered in sync with the Activity lifecycle (`onStart`/`onStop`) to avoid leaking it or missing state transitions.
- IMMEDIATE updates can still be interrupted by the user backing out of the flow — check the `ActivityResult` code and re-trigger the check rather than assuming the update always completes.
- Update priority is an attribute Google Play reports back for the *release*, configured by the developer in Play Console at rollout time — it is not read from the app's own remote config or environment.
- This playbook is synthesized from general Play Core/In-App-Update API documentation rather than extracted from a locally-available implementation, so treat it as a correct-by-the-book starting point, not a verified extraction.

### Playbook confidence: low

## Adoption
Used in **1** repo(s) in this marketplace. Implements the in-app update flow — a FLEXIBLE nudge or
an IMMEDIATE forced update chosen from the per-release priority set in Play Console — with a pure
decision function separated from the Android/Play Core lifecycle wiring.
