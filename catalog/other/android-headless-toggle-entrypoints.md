# Android headless toggle: one service, many entry points (app, Quick Settings tile, widget)

- **category**: other
- **provider**: internal (custom) on Android platform APIs (`Service`, `TileService`, `AppWidgetProvider`, `SharedPreferences`)
- **reusable**: yes — the shape (pure decision function, one pref writer, a service that is the source of truth, a prefs listener in the ViewModel) is independent of what the toggle does. Only the service body and the permission check change.
- **docs**: https://developer.android.com/develop/ui/views/quicksettings-tiles · https://developer.android.com/develop/ui/views/appwidgets

An app has one switchable mechanism (a VPN, a blocker, a timer) that must be flipped from the in-app button, the Quick Settings shade and a home-screen widget, and from a notification action or a scheduled alarm. Each entry point can be the first thing running in a fresh process, with no Activity and no ViewModel. This playbook keeps them all in agreement.

## Playbook

### Prerequisites
- A foreground `Service` (or `VpnService`) that does the real work and can be started with no UI.
- A single `SharedPreferences` file as the persisted source of truth.
- An `Application` subclass (its `onCreate()` runs before any other component in the process).
- minSdk high enough for `TileService` (24+). `Tile.subtitle` needs API 29+.

### Setup steps
1. **Own the state in prefs, not the ViewModel.** Define every key in one object (`AppPrefs`). Nothing else declares key strings.
2. **Make the service the source of truth.** It reads what to run from prefs on every start (only a small id travels in the Intent), and after it really starts or dies it writes the `active` flag itself. That corrects any optimistic write that turned out wrong (the system revoked the VPN, `establish()` failed after a reboot).
3. **Write a pure decision function** `decideToggle(active, hasPermission, config, …)` returning a sealed result: `Stop`, `Start(id)`, `DeepLinkToApp`. It takes plain values and touches no Android class, so it is JVM-testable.
4. **Write one executor** `executeToggle(context, decision, startService: (Intent) -> Unit)` that starts or stops the service and writes the `active` flag through a single writer function. Callers pass *how* to start (`startService` vs `startForegroundService`) and handle `DeepLinkToApp` themselves, because launching the Activity differs per entry point.
5. **Make the single writer refresh widgets.** `writeActive(context, active, …)` does the prefs put and then calls `Widget.updateAllWidgets(context)`. No other code writes the key.
6. **Add the tile**, **the widget**, and (optionally) a notification action. Each one only: reads prefs, calls `decideToggle`, calls `executeToggle`, and handles `DeepLinkToApp` its own way.
7. **Mirror prefs into the ViewModel** with one `OnSharedPreferenceChangeListener` registered in `init` and unregistered in `onCleared()`. Every key the tile, widget or service writes and the UI shows must be listed there. This is what keeps the in-app button current when something else changed state.
8. **Install in `Application.onCreate()`** anything every entry point depends on (crash handler, data migrations). It is the only component guaranteed to run first.

### Core pattern
```kotlin
// 1. Pure routing: no Context, no prefs.
sealed interface ToggleDecision {
    data object Stop : ToggleDecision
    data class Start(val configId: String) : ToggleDecision
    data object DeepLinkToApp : ToggleDecision      // needs consent or has nothing to run: send the user into the app
}

fun decideToggle(active: Boolean, hasConsent: Boolean, config: Config?): ToggleDecision = when {
    active -> ToggleDecision.Stop                    // checked BEFORE consent: stopping needs no permission,
    !hasConsent -> ToggleDecision.DeepLinkToApp      // and a consent-revoked-while-active tap must still stop
    config == null || !config.isRunnable() -> ToggleDecision.DeepLinkToApp
    else -> ToggleDecision.Start(config.id)
}

// 2. Execution, shared by ViewModel / tile / widget.
fun executeToggle(context: Context, d: ToggleDecision, startService: (Intent) -> Unit) = when (d) {
    ToggleDecision.Stop -> { startService(MyService.stopIntent(context)); writeActive(context, false) }
    is ToggleDecision.Start -> { startService(MyService.startIntent(context, d.configId)); writeActive(context, true, d.configId) }
    ToggleDecision.DeepLinkToApp -> Unit
}

// 3. The ONLY writer of the flag. Refreshes widgets every time (updatePeriodMillis is 0).
fun writeActive(context: Context, active: Boolean, configId: String? = null) {
    AppPrefs.get(context).edit {
        putBoolean(AppPrefs.KEY_ACTIVE, active)
        if (!active) remove(AppPrefs.KEY_ACTIVE_ID) else if (configId != null) putString(AppPrefs.KEY_ACTIVE_ID, configId)
    }
    ToggleWidgetProvider.updateAllWidgets(context)
}
```
```kotlin
// Tile: no Activity, no ViewModel.
class ToggleTileService : TileService() {
    override fun onStartListening() { super.onStartListening(); render() }      // tiles don't live-update; re-read on listen
    override fun onClick() {
        super.onClick()
        val p = AppPrefs.get(applicationContext)
        val d = decideToggle(p.getBoolean(AppPrefs.KEY_ACTIVE, false), Permission.hasConsent(this), AppPrefs.readSelected(p))
        if (d == ToggleDecision.DeepLinkToApp) { launchMainActivity(); return }
        executeToggle(this, d, ::startService)
        render()
    }
    private fun render() { qsTile?.apply { state = if (active) Tile.STATE_ACTIVE else Tile.STATE_INACTIVE; updateTile() } }
}
fun TileService.launchMainActivity() =                                           // tile-only: startActivityAndCollapse
    startActivityAndCollapse(PendingIntent.getActivity(this, 0, MainActivity.launchIntent(this), PendingIntent.FLAG_IMMUTABLE))
```
```kotlin
// Widget provider: intercept only YOUR action; everything else must reach super.
class ToggleWidgetProvider : AppWidgetProvider() {
    companion object {
        const val ACTION_TOGGLE = "com.example.app.widget.ACTION_TOGGLE"
        fun updateAllWidgets(context: Context) {
            val m = AppWidgetManager.getInstance(context)
            m.getAppWidgetIds(ComponentName(context, ToggleWidgetProvider::class.java)).forEach { render(context, m, it) }
        }
    }
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == ACTION_TOGGLE) handleToggle(context) else super.onReceive(context, intent)
    }
    override fun onUpdate(c: Context, m: AppWidgetManager, ids: IntArray) = ids.forEach { render(c, m, it) }
    private fun handleToggle(context: Context) {
        val p = AppPrefs.get(context)
        val d = decideToggle(p.getBoolean(AppPrefs.KEY_ACTIVE, false), Permission.hasConsent(context), AppPrefs.readSelected(p))
        if (d == ToggleDecision.DeepLinkToApp) { context.startActivity(MainActivity.launchIntent(context)); return }
        executeToggle(context, d, context::startForegroundService)               // widget tap: process may be fully backgrounded
    }
}
```
```kotlin
// ViewModel: the one listener that keeps the UI honest when someone else flipped the state.
private val prefsListener = SharedPreferences.OnSharedPreferenceChangeListener { p, key ->
    when (key) {
        AppPrefs.KEY_ACTIVE -> _isActive.value = p.getBoolean(key, false)
        AppPrefs.KEY_ACTIVE_ID -> _activeId.value = p.getString(key, null)
        // add every key the tile / widget / service writes that the UI displays
    }
}
init { prefs.registerOnSharedPreferenceChangeListener(prefsListener) }
override fun onCleared() = prefs.unregisterOnSharedPreferenceChangeListener(prefsListener)
```
```xml
<!-- manifest -->
<service android:name=".tile.ToggleTileService" android:exported="true"
         android:icon="@drawable/ic_tile" android:label="@string/tile_label"
         android:permission="android.permission.BIND_QUICK_SETTINGS_TILE">
  <intent-filter><action android:name="android.service.quicksettings.action.QS_TILE" /></intent-filter>
</service>
<receiver android:name=".widget.ToggleWidgetProvider" android:exported="false">
  <intent-filter><action android:name="android.appwidget.action.APPWIDGET_UPDATE" /></intent-filter>
  <meta-data android:name="android.appwidget.provider" android:resource="@xml/toggle_widget_info" />
</receiver>
<!-- Optional: long-press on the tile opens a settings screen -->
<intent-filter><action android:name="android.service.quicksettings.action.QS_TILE_PREFERENCES" /></intent-filter>  <!-- inside the Activity -->
```
`toggle_widget_info.xml` sets `android:updatePeriodMillis="0"`: the OS floor for a non-zero value is about 30 minutes, far too slow for a toggle that settles at once, so render only from `updateAllWidgets()`.

### Env vars
None.

### Gotchas
- **Check `Stop` before consent.** Stopping needs no permission. If a tap on an active toggle with revoked consent deep-links instead of stopping, the UI stays "on" forever.
- **Optimistic writes lie.** The tile, widget and ViewModel flip the flag before the service really starts. Only the service knows if the system revoked the VPN or `establish()` failed. Let the service write `active` after it really starts and again when it stops. Same-value `SharedPreferences` puts are no-ops, so the overlap is harmless.
- **A missing key in the ViewModel listener is a silent staleness bug.** The in-app button stops following the tile. This happened once for the main flag after the tile shipped. Review the listener whenever a new key is written outside the ViewModel.
- **Widget `onReceive()` must fall through to `super.onReceive()`** for everything except your own action. `AppWidgetProvider.onReceive()` is what dispatches `APPWIDGET_UPDATE`/`ENABLED`/`DELETED`, so overriding it without the fallback means the widget never renders.
- **`startForegroundService` for the widget, `startService` for the tile.** A widget tap can arrive with the process fully backgrounded. The service must call `startForeground()` promptly, or Android kills it with `ForegroundServiceDidNotStartInTimeException`.
- **Tiles don't live-update** while the shade stays open. They refresh in `onStartListening()` and right after your own click. If another path changes state while the shade is open, the tile is stale until the next open. That is standard Quick Settings behaviour.
- **Tile deep-link needs `startActivityAndCollapse(PendingIntent)`** on API 34+; a widget can't use it and starts the Activity directly. Share only the `Intent` factory between them.
- **Re-derive, don't share.** Headless entry points must rebuild their own state from prefs on every start. Don't depend on a ViewModel, a singleton filled in by an Activity, or an in-memory list.
- **Put one-time setup in `Application.onCreate()`** (data migrations, crash handler install, applying a stored opt-out). It is the only component guaranteed to run ahead of the service, tile and widget in a fresh process, unless a component declares its own `android:process`.
- **Keys live in one object.** A raw key literal anywhere else is a review failure; sweep for it after any prefs change.
- **An empty "run list" can be dangerous** (for a VPN that routes only chosen apps, an empty allow-list routes everything). Put that rule into the pure decision function (`isRunnable()`), so every entry point inherits it.

### Playbook confidence: high
Extracted from a real implementation with a service, a tile, a widget, a notification action and a ViewModel. The decision function has JVM unit tests in the source; the service, tile and widget paths are not unit-testable and are checked on a device. The `ForegroundServiceDidNotStartInTimeException` and `startActivityAndCollapse` API-level remarks come from general Android knowledge, not the source.

## Adoption
Used in **1** repo in this marketplace: a native Android app whose single foreground service is driven from the in-app button, the Quick Settings shade and a home-screen widget.
