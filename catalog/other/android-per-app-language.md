# Android per-app language (LocaleManager, localeFilters, LocaleConfig)

- **category**: other
- **provider**: Android platform (`LocaleManager`, AGP `androidResources`)
- **reusable**: yes — one tag list and one normalizer serve the whole app. Only the language list and the string labels change.
- **docs**: https://developer.android.com/guide/topics/resources/app-languages · https://developer.android.com/reference/android/app/LocaleManager

A native Android app that ships many locales, lets the user pick one in-app, and also shows up in the system's per-app language picker (Settings → Apps → app → Language). It complements [i18n-language-support](i18n-language-support.md), which covers web/React Native catalogs and RTL; this one is the native-Android (Compose or Views) variant with 10+ locales.

## Playbook

### Prerequisites
- Android 13+ (API 33) for `LocaleManager`; on older versions use AppCompat's `AppCompatDelegate.setApplicationLocales`.
- `res/values-<qualifier>/strings.xml` for each language.
- `android:supportsRtl="true"` in the manifest if any language is right-to-left.

### Setup steps
1. Add the string folders. Qualifier forms: `values-es`, `values-hi`, `values-ar`, `values-ja`, `values-ru`, `values-fr`, `values-de`, `values-iw` (Hebrew), and `values-b+pt+BR`, `values-b+zh+Hans`, `values-b+zh+Hant` for script- or region-specific ones.
2. Create `res/resources.properties` containing `unqualifiedResLocale=en`, so the default `values/` folder is declared as English.
3. In `app/build.gradle.kts` turn on `generateLocaleConfig = true` and list every locale in `localeFilters`. Use **aapt2 qualifier syntax**, matching the folder suffix. The build then generates `LocaleConfig` and injects the manifest attribute. Don't write that attribute yourself.
4. Define **one** list of supported tags, `SUPPORTED_LANGUAGES: List<Pair<String, Int>>` (BCP-47 tag to label string resource). Build both the picker dialog and the settings-row label from it.
5. Read the current choice with `Locale.toLanguageTag()`, and write it with `LocaleManager.applicationLocales`.
6. Put the Hebrew normalizer (below) in one function and use it at every comparison.
7. Add a JVM test that pins the list: no duplicates, expected size, every tag resolves to a label.
8. Whenever you add a language: new `values-*` folder, new `localeFilters` entry, new `SUPPORTED_LANGUAGES` entry. Missing the second one strips the resources silently.

### Core pattern
```kotlin
// app/build.gradle.kts
android {
    androidResources {
        generateLocaleConfig = true                 // builds LocaleConfig + manifest attribute for you
        localeFilters += listOf(                    // aapt2 syntax, NOT plain BCP-47 ("pt-BR" fails with "invalid config")
            "en", "iw", "es", "b+pt+BR", "hi", "ar", "b+zh+Hans", "b+zh+Hant", "ja", "ru", "fr", "de"
        )
    }
}
```
```kotlin
// The single source of truth for both the dialog and the settings-row label. Tags are exact BCP-47.
internal val SUPPORTED_LANGUAGES: List<Pair<String, Int>> = listOf(
    "en" to R.string.language_english,
    "iw" to R.string.language_hebrew,
    "pt-BR" to R.string.language_portuguese_brazil,
    "zh-Hans" to R.string.language_chinese_simplified,
    "zh-Hant" to R.string.language_chinese_traditional,
    // ...
)

// Android inconsistently round-trips the deprecated "iw" to "he". Compare through this.
internal fun normalizeLanguageTag(tag: String?): String? = when (tag) { "he" -> "iw"; else -> tag }

internal fun languageLabelRes(currentTag: String?): Int =
    SUPPORTED_LANGUAGES.firstOrNull { it.first == normalizeLanguageTag(currentTag) }?.second
        ?: R.string.language_system_default

// Read / write
val lm = context.getSystemService(LocaleManager::class.java)
val currentTag = lm.applicationLocales.get(0)?.toLanguageTag()          // NOT .language: that collapses zh-Hans / zh-Hant
lm.applicationLocales =
    if (tag == null) LocaleList.getEmptyLocaleList()                    // reset to system default
    else LocaleList.forLanguageTags(tag)
```
```kotlin
// Compose picker: "System default" + the list, a radio per row.
@Composable fun LanguagePickerDialog(currentTag: String?, onSelect: (String?) -> Unit, onDismiss: () -> Unit) {
    val options = listOf<Pair<String?, String>>(null to stringResource(R.string.language_system_default)) +
        SUPPORTED_LANGUAGES.map { (tag, res) -> tag to stringResource(res) }
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(stringResource(R.string.language_picker_title)) },
        text = {
            Column(Modifier.heightIn(max = 400.dp).verticalScroll(rememberScrollState())) {
                options.forEach { (tag, label) ->
                    val selected = normalizeLanguageTag(currentTag) == tag
                    Row(Modifier.fillMaxWidth().selectable(selected, onClick = { onSelect(tag) }).padding(vertical = 10.dp),
                        verticalAlignment = Alignment.CenterVertically) {
                        RadioButton(selected = selected, onClick = { onSelect(tag) })
                        Spacer(Modifier.width(14.dp)); Text(label, style = MaterialTheme.typography.bodyLarge)
                    }
                }
            }
        },
        confirmButton = {}
    )
}
```
```kotlin
// JVM test that keeps the list honest
@Test fun supportedLanguages_isWellFormed() {
    assertEquals(12, SUPPORTED_LANGUAGES.size)
    assertEquals(SUPPORTED_LANGUAGES.size, SUPPORTED_LANGUAGES.map { it.first }.toSet().size)
}
@Test fun he_normalizesTo_iw() = assertEquals("iw", normalizeLanguageTag("he"))
@Test fun scriptTagsPassThrough() = assertEquals("zh-Hans", normalizeLanguageTag("zh-Hans"))
```

### Env vars
None.

### Gotchas
- **`localeFilters` silently strips resources** for any locale not listed. The APK builds, and that language simply never appears. Add every new language there.
- **`localeFilters` takes aapt2 qualifier syntax**, the same as the resource folder suffix (`b+pt+BR`, `b+zh+Hans`), not a plain BCP-47 tag. `pt-BR` fails the build with "invalid config".
- **`generateLocaleConfig = true` replaces hand-written `locales_config.xml`** and the manifest `android:localeConfig` attribute. Writing your own as well causes a duplicate-attribute conflict.
- **`unqualifiedResLocale=en` in `res/resources.properties`** tells the build which language the default `values/` folder is. Without it the generated LocaleConfig may not list the default language.
- **Use `Locale.toLanguageTag()`, not `.language`.** `.language` drops the script and region, so `zh-Hans` and `zh-Hant`, or `pt-BR` and `pt-PT`, can't be told apart.
- **`iw`/`he`.** The app sets the deprecated `iw` code, and Android hands back `he` on some paths. Compare through `normalizeLanguageTag()` everywhere, and don't copy the comparison inline. Without it the picker shows "System default" while the app is actually in Hebrew.
- **Two hand-maintained lists drift.** The dialog options and the row label used to be separate lists; a tag in one and not the other showed "System default" for a language you could still select. Derive both from `SUPPORTED_LANGUAGES`.
- **Setting `applicationLocales` recreates the Activity.** State must live in a ViewModel or `rememberSaveable`, or the screen resets after the user picks a language.
- **RTL comes for free with `supportsRtl="true"`.** Compose directional APIs (`start`/`end`, `SwipeToDismissBox` directions) mirror automatically; hard-coded left/right does not.
- **Store listings are separate.** The Play Console listing text per language is not derived from the app's string folders; translate it separately (see [play-console-compliance](play-console-compliance.md)).

### Playbook confidence: high
Extracted from a real 12-locale implementation with a JVM test that pins the language list and the normalizer. The `values-*` qualifier forms and the AppCompat fallback for API < 33 are general knowledge, not from the source.

## Adoption
Used in **1** repo in this marketplace: a Compose app with 12 locales (one RTL, two script-qualified, one region-qualified), an in-app picker and the system per-app picker driven by the same generated `LocaleConfig`.
