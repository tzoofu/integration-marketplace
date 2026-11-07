# Android release pipeline: GitHub Actions to Google Play internal track

- **category**: hosting-deploy
- **provider**: GitHub Actions + Google Play Developer API (via a third-party upload action)
- **reusable**: yes — the workflow is project-agnostic. Only the package name, the secret names and the Gradle property name change per app. Could be shipped as a copy-paste template or a reusable workflow.
- **docs**: https://docs.github.com/actions · https://developer.android.com/guide/app-bundle · https://github.com/r0adkll/upload-google-play

On every push to the main branch, build and test the app, produce a **signed release App Bundle (.aab)** with an auto-incrementing `versionCode`, and upload it to Play Console's **internal testing track as a draft**. A human still presses "release" in Play Console. Pull requests only run the cheap quality gate and never see signing material.

## Playbook

### Prerequisites
- A Play Console app that already exists, with at least one release created by hand (the API can't create the app, and the first upload usually has to be manual).
- An upload keystore (`.jks`) and its alias and passwords. Enrol in Play App Signing so the upload key is replaceable.
- A Google Cloud service account with the Play Developer API enabled, invited in Play Console (Users and permissions) with release permission for this app, and a JSON key for it.
- Firebase config (`google-services.json`) if the app uses Firebase plugins. It must exist at build time even if it is gitignored.
- Gradle wrapper, version catalog, and a lint/unit-test setup that already passes locally.

### Setup steps
1. Make the signing config **optional** in `app/build.gradle.kts`: read `keystore.properties` if the file exists, and create the `release` signing config only then. A local build without the file produces an unsigned artifact instead of failing. Gitignore `keystore.properties`, `*.jks` and `*.keystore`.
2. Make `versionCode` overridable from the command line: `-PversionCodeOverride=<n>`. CI passes the workflow run number, which always increases, so no commit is needed per upload. Local builds keep the checked-in value.
3. Base64-encode the secrets and store them as repository secrets: `KEYSTORE_BASE64` (`base64 -i upload.jks`), `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`, `GOOGLE_SERVICES_JSON` (base64 of the file), `PLAY_SERVICE_ACCOUNT_JSON` (the raw JSON text).
4. Add `.github/workflows/ci.yml` with three jobs (below): `build` (all events), `bundle` (push only, needs `build`), `play` (push only, needs `bundle`).
5. Push to main, watch the three jobs, then open Play Console → Testing → Internal testing and confirm the draft release exists. Release it by hand.
6. Pin every third-party action by **commit SHA** (keep the tag in a comment). Use `actions/*` by major tag if you accept that risk.

### Core pattern
```kotlin
// app/build.gradle.kts (relevant parts)
val keystorePropertiesFile = rootProject.file("keystore.properties")
val keystoreProperties = Properties().apply {
    if (keystorePropertiesFile.exists()) load(FileInputStream(keystorePropertiesFile))
}
android {
    defaultConfig {
        // Play rejects a reused versionCode. CI passes the run number; local builds keep the default.
        versionCode = (project.findProperty("versionCodeOverride") as String?)?.toInt() ?: 1
        versionName = "1.0.0"          // bumped by hand per release; CI never touches it
    }
    signingConfigs {
        if (keystorePropertiesFile.exists()) {
            create("release") {
                storeFile = file(keystoreProperties["storeFile"] as String)
                storePassword = keystoreProperties["storePassword"] as String
                keyAlias = keystoreProperties["keyAlias"] as String
                keyPassword = keystoreProperties["keyPassword"] as String
            }
        }
    }
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            if (keystorePropertiesFile.exists()) signingConfig = signingConfigs.getByName("release")
        }
    }
}
```
```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:                                   # quality gate: every push and PR
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17' }
      - uses: gradle/actions/setup-gradle@<full-commit-sha> # v4.x
      - name: Set up google-services.json
        run: echo "$GOOGLE_SERVICES_JSON" | base64 --decode > app/google-services.json
        env:
          GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
      - run: ./gradlew lint --console=plain
      - run: ./gradlew testDebugUnitTest --console=plain
      - run: ./gradlew assembleDebug --console=plain

  bundle:                                  # signed .aab: main-branch pushes only, never PRs
    needs: build
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17' }
      - uses: gradle/actions/setup-gradle@<full-commit-sha>
      - name: Set up google-services.json
        run: echo "$GOOGLE_SERVICES_JSON" | base64 --decode > app/google-services.json
        env:
          GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
      - name: Set up release signing
        run: |
          echo "$KEYSTORE_BASE64" | base64 --decode > upload-key.jks
          {
            echo "storeFile=../upload-key.jks"        # relative to app/, matches file(...) in the module
            echo "storePassword=$KEYSTORE_PASSWORD"
            echo "keyAlias=$KEY_ALIAS"
            echo "keyPassword=$KEY_PASSWORD"
          } > keystore.properties
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
      - run: ./gradlew bundleRelease -PversionCodeOverride=${{ github.run_number }} --console=plain
      - uses: actions/upload-artifact@v4
        with:
          name: app-release-aab
          path: app/build/outputs/bundle/release/app-release.aab
      - name: Remove signing material
        if: always()
        run: rm -f upload-key.jks keystore.properties

  play:                                    # separate job: the third-party action never sits next to the keystore
    needs: bundle
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    env:
      PLAY_SERVICE_ACCOUNT_JSON: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
    steps:
      - if: env.PLAY_SERVICE_ACCOUNT_JSON != ''
        uses: actions/download-artifact@v4
        with: { name: app-release-aab }
      - if: env.PLAY_SERVICE_ACCOUNT_JSON != ''
        uses: r0adkll/upload-google-play@<full-commit-sha> # v1.1.x
        with:
          serviceAccountJsonPlainText: ${{ env.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.app
          releaseFiles: app-release.aab
          tracks: internal
          status: draft
          changesNotSentForReview: true
```

### Env vars
Repository secrets (names only): `GOOGLE_SERVICES_JSON`, `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`, `PLAY_SERVICE_ACCOUNT_JSON`.
Gradle property: `versionCodeOverride`.

### Gotchas
- **`changesNotSentForReview: true` is required** with `status: draft` when the app's changes aren't ready for review (typically while the listing is still being set up). Without it the Play API rejects the upload.
- **Draft releases need a human step.** Nothing goes to testers until someone releases the draft in Play Console. That is the point: CI can't ship by accident.
- **Job separation is a security boundary.** The `bundle` job holds the keystore and no Play credential. The `play` job holds the Play credential and only receives the already-signed artifact. A compromised third-party action in `play` can't read the keystore, and vice versa.
- **Skip the Play job quietly until the secret exists.** An `env` var plus `if: env.X != ''` on each step lets you merge the workflow before the service account is ready. A secret can't be used directly in `if:`.
- **Always remove signing files** with an `if: always()` step. Self-hosted runners keep the workspace; hosted runners are discarded, but the habit is cheap.
- **Pin by SHA** for any third-party action that sees a credential. A moved tag is a supply-chain hole.
- **JDK mismatch.** CI used JDK 17 while local dev used 21. Don't use APIs or Gradle features that exist only in the newer JDK, or CI breaks and local doesn't. Pin one version in both if you can.
- **`google-services.json` must be reconstructed before every Gradle task** that applies the Google Services plugin, including `lint`. Without it every task fails immediately.
- **`versionCode` via run number is monotonic but not meaningful.** Re-running a failed workflow gets a new number only if you start a new run ("Re-run" reuses `run_number` with a different `run_attempt`). If a rerun uploads a bundle whose code the Play Console already has, the upload is rejected; push a new commit instead. If you ever move to another CI, keep the counter ahead of the last uploaded code.
- **Don't derive `versionCode` from `versionName`**, and don't let CI bump `versionName` through a commit. Keep both rules explicit so a hand-made release and a CI release can never collide.
- **R8 / minify has never run under test.** Release builds are minified, but CI only exercises debug. Install a release build on a device before the first public release and add keep rules only when something actually breaks.
- **Unsigned release is silent.** If `keystore.properties` is missing the build still succeeds and produces an unsigned artifact; Play will reject it later with a less obvious error. A guard step (`test -f keystore.properties`) in the `bundle` job makes the failure early and loud.
- **Play App Signing key reset** is the recovery path for a lost upload key. Keep the keystore backed up somewhere other than the repo.

### Playbook confidence: high
The workflow, the Gradle config and most gotchas are extracted from a real three-job workflow. Three points come from general GitHub Actions / Play knowledge rather than the source, so verify them before relying on them: the "Re-run reuses `run_number`" part of the `versionCode` gotcha, the `test -f keystore.properties` guard suggestion, and the Play App Signing recovery note.

## Adoption
Used in **1** repo in this marketplace: a native Android app with a single module, Firebase plugins, a minified release build, and an optional local signing config.
