# Jetpack Compose multi-theme palettes (named themes + follow-system, live switch)

- **category**: other
- **provider**: internal (custom, on Jetpack Compose + Material 3)
- **reusable**: yes — three small files (token class + palette builder, mode enum + persistence, theme composable) plus an optional picker screen; nothing app-specific except the token names and hex values.
- **docs**: https://developer.android.com/develop/ui/compose/designsystems/custom and https://developer.android.com/develop/ui/views/layout/edge-to-edge

Lets the user pick one of several named color themes (e.g. a light default, a dark "night", two tinted light themes, a high-contrast dark one) or "follow the phone", and recolors the whole Compose app instantly. Every screen reads semantic tokens from one `CompositionLocal`, so a theme is just another instance of the token class. The native-Android counterpart of the web [design-system-semantic-tokens](design-system-semantic-tokens.md) playbook.

## Playbook

### Prerequisites
- Jetpack Compose with Material 3, `androidx.activity` 1.8+ (`enableEdgeToEdge`, `SystemBarStyle`).
- A DI singleton or plain object for settings (Hilt shown), `lifecycle-runtime-compose` for `collectAsStateWithLifecycle`.
- A designer's token table per theme: about 20 values (background, surface, ink, secondary inks, muted/faint, line color, control/pressed, accent, accent-ink, soft/soft2 tints, text-on-accent, strong, warn bg/fg, ok bg/fg, badge).

### Setup steps
1. Create a `data class AppColors(...)` with every semantic token the UI uses, each defaulted to the base light theme. Include a border ladder (one line color at several alphas) instead of ad-hoc alpha math in screens.
2. Add a private `themed(base, dark, …tokens)` builder that maps the designer's ~20 tokens onto the full class with `base.copy(...)`. Pass the light base for light themes and a hand-tuned dark base for dark themes, so tokens the designer didn't specify (secondary status hues) are still sensible.
3. Declare each palette as one `val XColors = themed(...)` call.
4. Create `enum class ThemeMode(val dark: Boolean?)` with `SYSTEM(null)` plus one entry per palette, `isDark(system)` and `colors(system)`.
5. Persist the enum **name** in SharedPreferences/DataStore through a singleton exposing `StateFlow<ThemeMode>`; parse with a migration map for old values.
6. Make the theme composable take both `dark` and `colors`, provide the colors through `staticCompositionLocalOf`, and copy the key tokens into Material's `colorScheme` so stock M3 components match.
7. In the activity, collect the mode, read `isSystemInDarkTheme()` once, and re-apply `enableEdgeToEdge` whenever `dark` flips.
8. Build the picker: one selectable card per `ThemeMode.entries`, each with a tiny preview drawn in that palette's own colors.
9. Rule for every screen: colors only from `AppTheme.colors.*`, never a hex literal, and use the "on-accent" token (not white) for anything drawn on the accent color.

### Core pattern
```kotlin
// Color.kt
data class AppColors(
    val bg: Color = Color(0xFFF4F5F7), val card: Color = Color.White, val inkText: Color = Color(0xFF0F1729),
    val text2: Color = Color(0xFF6B7280), val muted: Color = Color(0xFF9CA3AF),
    val brand: Color = Color(0xFF2747C0), val brandText: Color = Color(0xFF2747C0), val brandDark: Color = Color(0xFF1D3A9E),
    val onBrand: Color = Color.White,               // text/icons on a brand background
    val tint: Color = Color(0xFFEAEFFC), val tabPill: Color = Color(0xFFDCE4FA),
    val ink: Color = Color(0xFF0F1729),              // dark surfaces (dark cards, dark buttons) with white text
    val warnBg: Color = Color(0xFFFEF6E7), val warnText: Color = Color(0xFF92600E),
    val okBg: Color = Color(0xFFE6F4EE), val okText: Color = Color(0xFF166045), val badge: Color = Color(0xFFC2410C),
    val switchOff: Color = Color(0x330F1729), val pressed: Color = Color(0xFFEEF1F6),
    val border06: Color = Color(0xFF0F1729).copy(alpha = .06f), val border14: Color = Color(0xFF0F1729).copy(alpha = .14f),
    // …rest of the ladder
)
val LightColors = AppColors()
val DarkBase = AppColors(/* hand-tuned dark values for every field */)

private fun themed(base: AppColors, dark: Boolean, bg: Long, surface: Long, ink: Long, muted: Long, faint: Long,
                   line: Long, ctl: Long, press: Long, accent: Long, accentInk: Long, soft: Long, soft2: Long,
                   onAccent: Long, strong: Long, warnBg: Long, warnFg: Long, okBg: Long, okFg: Long, badge: Long): AppColors {
    fun b(pct: Int) = Color(line).copy(alpha = pct / 100f)
    return base.copy(
        bg = Color(bg), card = Color(surface), inkText = Color(ink), text2 = Color(muted), muted = Color(faint),
        switchOff = Color(ctl), pressed = Color(press),
        brand = Color(accent), brandText = Color(if (dark) accent else accentInk), brandDark = Color(accentInk),
        onBrand = Color(onAccent), tint = Color(soft), tabPill = Color(soft2),
        ink = if (dark) base.ink else Color(strong),   // dark themes keep a raised slate so white-on-ink stays readable
        warnBg = Color(warnBg), warnText = Color(warnFg), okBg = Color(okBg), okText = Color(okFg), badge = Color(badge),
        border06 = b(6), border14 = b(14), /* … */
    )
}
val ForestColors = themed(LightColors, dark = false, bg = 0xFFF2F4F0, surface = 0xFFFFFFFF, /* … */ onAccent = 0xFFFFFFFF, /* … */)
val YellowDarkColors = themed(DarkBase, dark = true, bg = 0xFF141518, /* … */ accent = 0xFFF5B800, onAccent = 0xFF141518, /* … */)
```

```kotlin
// ThemeSettings.kt
enum class ThemeMode(val dark: Boolean?) {
    SYSTEM(null), LIGHT_DEFAULT(false), NIGHT(true), FOREST(false), YELLOW_DARK(true);
    fun isDark(system: Boolean) = dark ?: system
    fun colors(system: Boolean): AppColors = when (this) {
        SYSTEM -> if (system) NightColors else LightColors
        LIGHT_DEFAULT -> LightColors; NIGHT -> NightColors; FOREST -> ForestColors; YELLOW_DARK -> YellowDarkColors
    }
    companion object {   // stored names survive renames through this map
        fun parse(name: String?) = when (name) { "LIGHT" -> LIGHT_DEFAULT; "DARK" -> NIGHT
            else -> entries.firstOrNull { it.name == name } ?: SYSTEM }
    }
}

@Singleton
class ThemeSettings @Inject constructor(@ApplicationContext context: Context) {
    private val prefs = context.getSharedPreferences("theme_settings", Context.MODE_PRIVATE)
    private val state = MutableStateFlow(ThemeMode.parse(prefs.getString("mode", null)))
    val mode: StateFlow<ThemeMode> = state.asStateFlow()
    fun set(mode: ThemeMode) { prefs.edit { putString("mode", mode.name) }; state.value = mode }
}
```

```kotlin
// Theme.kt
private val LocalAppColors = staticCompositionLocalOf { LightColors }
object AppTheme { val colors: AppColors @Composable get() = LocalAppColors.current }

@Composable
fun AppTheme(dark: Boolean = isSystemInDarkTheme(), colors: AppColors = if (dark) DarkBase else LightColors,
             content: @Composable () -> Unit) {
    val scheme = (if (dark) darkColorScheme() else lightColorScheme()).copy(
        primary = colors.brand, onPrimary = colors.onBrand, background = colors.bg, onBackground = colors.inkText,
        surface = colors.card, onSurface = colors.inkText, outline = colors.border14,
        secondaryContainer = colors.tint, onSecondaryContainer = colors.brandDark,
        surfaceContainerLowest = colors.bg, surfaceContainerLow = colors.card, surfaceContainer = colors.card,
        surfaceContainerHigh = colors.card, surfaceContainerHighest = colors.pressed,
    )
    CompositionLocalProvider(LocalAppColors provides colors, LocalContentColor provides colors.inkText) {
        MaterialTheme(colorScheme = scheme, content = content)
    }
}

// MainActivity.onCreate
setContent {
    val mode by themeSettings.mode.collectAsStateWithLifecycle()
    val systemDark = isSystemInDarkTheme()
    val dark = mode.isDark(systemDark)
    DisposableEffect(dark) {
        val style = SystemBarStyle.auto(android.graphics.Color.TRANSPARENT, android.graphics.Color.TRANSPARENT) { dark }
        enableEdgeToEdge(statusBarStyle = style, navigationBarStyle = style)
        onDispose {}
    }
    AppTheme(dark, mode.colors(systemDark)) { Root() }
}
```

```kotlin
// Picker card preview: a mini screen drawn in the palette it represents (not the current theme)
@Composable
fun ThemePreview(p: AppColors, modifier: Modifier = Modifier.size(76.dp, 64.dp)) {
    Column(modifier.clip(RoundedCornerShape(11.dp)).background(p.bg).border(1.dp, p.border06, RoundedCornerShape(11.dp))
        .padding(8.dp), verticalArrangement = Arrangement.spacedBy(4.dp)) {
        listOf(p.brand, p.badge).forEach { dot ->
            Row(Modifier.fillMaxWidth().height(12.dp).background(p.card, RoundedCornerShape(3.dp)).padding(horizontal = 4.dp),
                verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(4.dp)) {
                Box(Modifier.size(5.dp).background(dot, CircleShape))
                Box(Modifier.weight(1f).height(3.dp).background(p.switchOff, RoundedCornerShape(2.dp)))
            }
        }
        Spacer(Modifier.weight(1f))
        Box(Modifier.fillMaxWidth().height(14.dp).background(p.brand, RoundedCornerShape(4.dp)))
    }
}
// "Follow system" card: two previews side by side (light half, dark half) clipped into one thumbnail.
```

### Env vars
none.

### Gotchas
- **The stored value is the enum name.** Renaming or replacing entries (e.g. moving from light/dark to named themes) silently resets everyone to the default unless `parse` maps the old names. Keep "follow system" as the default so an update never changes anyone's colors unasked.
- **Text on the accent must come from an "on-accent" token.** A yellow or light-blue accent needs dark text. Audit every place that paints the accent as a background (primary buttons, FAB, checkboxes, switch knob, badges, small icon tiles) — they tend to hard-code white. A button helper can pick `onBrand` when its container is the brand color and white otherwise.
- **Dark themes need a separate "ink" surface.** In light themes "ink" (near-black) doubles as a dark card/button color with white text; in dark themes using the designer's "strong" (near-white) there would invert those components. Keep a raised slate instead.
- **Read `isSystemInDarkTheme()` in the composable and pass it in** (`isDark(system)`, `colors(system)`), so "follow system" reacts live when the phone switches mode.
- **Re-run `enableEdgeToEdge` when `dark` changes** (a `DisposableEffect(dark)`), or status/nav bar icons keep the previous theme's contrast.
- **Only the Material slots you copy are themed.** Stock M3 components (dialogs, text fields, progress indicators, date pickers) read `colorScheme`; anything not mapped keeps the default purple/grey. Map the surface-container family too, or dialogs come out with a mismatched tint.
- **Theme composable defaults:** `dark` and `colors` both have defaults derived from the system; callers that pass only one of them can get a light palette with dark-mode Material slots. Pass both from the activity.
- **Unspecified tokens fall back to the base.** Light palettes reuse the base light status colors (teal/red), dark ones the dark base's. If a tinted theme clashes with a status hue, add that hue to the builder rather than special-casing screens.
- The picker applies on tap (no "save"); a 300ms color crossfade is optional — Compose recomposes everything instantly anyway.
- Hard-coded colors in screens are the main leak. A grep in CI (`Color(0x` outside the theme package) keeps them out.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace: a native Android app with five named palettes (three light, two dark, one with a yellow accent for outdoor readability) plus "follow system", a picker screen with per-palette previews, and a migration from an earlier light/dark/system setting.
