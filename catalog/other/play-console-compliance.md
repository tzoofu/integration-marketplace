# Google Play Console compliance checklist (sensitive permissions, foreground services, privacy policy, store listing)

- **category**: other
- **provider**: Google Play Console / Play Policy Insights
- **reusable**: yes — the checklist applies to any Android app that declares sensitive permissions, a `VpnService`, or `specialUse` foreground services. The declaration wording and the data-safety answers are per-app.
- **docs**: https://support.google.com/googleplay/android-developer/answer/9888170 · https://support.google.com/googleplay/android-developer/answer/10787469 · https://developer.android.com/guide/components/foreground-services

What to prepare *before* the first Play submission so review doesn't stall on declarations. None of it is a code defect: each item is a Play Console form, a hosted URL, or a piece of store-listing text. Pair it with [android-play-release-pipeline](../hosting-deploy/android-play-release-pipeline.md), which gets the bundle into the console.

## Playbook

### Prerequisites
- A Play Console developer account (an organisation account may need a D-U-N-S number and verification first).
- A build that already declares everything it uses in `AndroidManifest.xml`.
- A device or emulator reachable over `adb` for the demo videos.
- A place to host a public privacy-policy URL.

### Setup steps
1. **Audit the manifest against policy.** Run Android Studio's Play Policy Insights panel, or run lint with the Play insights lint jar on `ANDROID_LINT_JARS` as a one-off check: `ANDROID_LINT_JARS=<insights-lint.jar> ./gradlew :app:lintDebug`. Don't add the jar to `lintChecks`; it is a verification harness only. Typical flags: `PackageVisibilityPolicy` (`QUERY_ALL_PACKAGES`), `ForegroundServicesPolicy`, `VpnServicePolicy`. These are advisories tied to declaring the API at all. `tools:ignore` doesn't silence them, and they don't fail the build.
2. **Permissions Declaration Form** (App content → Permissions declaration) for each restricted permission. For `QUERY_ALL_PACKAGES`, justify it as core functionality (the user picks apps from the full installed list, including apps with no launcher icon) and explain why a `<queries>` launcher filter would silently shrink that list.
3. **Foreground service declaration** (App content → Foreground service) for each `specialUse` service: a functional description and a **demo video** that shows the user starting it and seeing the notification. Keep the manifest `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` text and the console text in step; refresh both when the service gains a new way to start.
4. **VPN disclosure.** If the app uses `VpnService`, the store listing must say so. For a local, on-device tunnel that never forwards traffic, say that plainly: the policy's "encrypts traffic to a remote endpoint" language assumes a remote VPN.
5. **Location.** Foreground-only location needs no background-location declaration. State in the Data Safety form that location is processed on-device and not collected, if that is true.
6. **Exact alarms.** `SCHEDULE_EXACT_ALARM` (user-granted special access) needs no console declaration, but the app must work without it. `USE_EXACT_ALARM` is a different, restricted permission.
7. **Data Safety form.** Answer from what the code really does. For an app with no account and no analytics, and with a crash reporter, declare the crash data and mark it processed with a user-controllable opt-out. Play can export and import the form as CSV, which is useful for keeping the answers in the repo and reviewing diffs.
8. **Privacy policy at a public URL.** Fill every placeholder (date, contact email), keep a table of permission to purpose to "leaves device?", and name every SDK that sends data. Host it anywhere that gives a stable HTTPS URL. A static PDF on Firebase Hosting works (see [firebase-hosting](../hosting-deploy/firebase-hosting.md)): keep the HTML as the editable source, re-render with headless Chrome, redeploy, and the URL in the console never changes.
9. **Store listing per locale.** Short description (80 characters max) and full description in every language you ship, plus the 512x512 icon, a 1024x500 feature graphic, and screenshots. Keep them as files in the repo (`store-assets/<locale>/`) so they are versioned. Mention the supported languages in the full text.
10. **Record the demo videos** with `adb`: start the flow, `adb shell screenrecord`, drive it with `adb shell input tap/swipe`, then pull the file. No UI-test framework is needed.
11. **Keep a living checklist file** in the repo with a box per item above and the raw lint findings it was derived from, so the next release or a second app can reuse it.

### Core pattern
```bash
# One-off policy audit (jar is NOT a project dependency)
ANDROID_LINT_JARS=/path/to/insights-lint.jar ./gradlew :app:lintDebug

# Demo video of a flow, from a connected device
adb shell screenrecord --time-limit 60 /sdcard/demo.mp4 &
adb shell input tap 540 1600          # drive the flow
adb pull /sdcard/demo.mp4

# Re-render the policy PDF from its HTML source, then redeploy the static site
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless --disable-gpu --no-pdf-header-footer \
  --print-to-pdf="hosting/privacy-policy.pdf" "file://$(pwd)/privacy-policy-src/privacy-policy.html"
firebase deploy --only hosting
```
```markdown
<!-- PLAY_STORE_CHECKLIST.md skeleton -->
## Before submitting
- [ ] Permissions Declaration Form: <permission> — justification: <core feature, why a narrower API doesn't work>
- [ ] Foreground service declaration + demo video: <service> — user-initiated, notification visible
- [ ] Listing discloses VPN use (local tunnel, nothing forwarded)
- [ ] Location is foreground-only; Data Safety says on-device only
- [ ] Exact alarms optional; app works without them
- [ ] Data Safety form matches the code (crash reporting declared with its opt-out)
- [ ] Privacy policy placeholders filled, hosted at a public URL, URL entered in the console
- [ ] Listing text + assets for every shipped locale
```
```text
<!-- Privacy policy permission table (one row per permission) -->
| Permission | Purpose | Data leaves device? |
|---|---|---|
| <PERMISSION> | <what the user sees it do> | No / <what, to whom> |
```

### Env vars
None.

### Gotchas
- **Advisories are not errors.** The policy lint IDs stay in the report forever. The fix is the console declaration, not a suppression or removing the feature.
- **`QUERY_ALL_PACKAGES` is the most reviewed permission.** A thin justification ("to list apps") is rejected. Tie it to the core purpose and show that narrower options fail.
- **A foreground-service declaration needs a video**, and the video must show a *user-initiated*, perceptible action. Auto-starting services need the trigger shown (a schedule firing, a countdown the user can cancel).
- **Disclose "VPN" in the listing** even if the tunnel is local. A listing that doesn't mention VPN use while the manifest declares `BIND_VPN_SERVICE` is a classic rejection.
- **Privacy policy placeholders get shipped.** `[DATE]` and `[CONTACT EMAIL]` left in a hosted policy fail review. Add a pre-submit check, and keep the "last updated" date and the footer in step by hand, since nothing updates them.
- **Data Safety must match the SDKs.** Adding a crash reporter changes the answers, and the privacy policy, and the in-app opt-out wording, together. Change all three in one commit.
- **A separate policy file for each audience drifts.** Keep one source file (HTML) and render the hosted copy from it.
- **Fonts in a PDF render depend on the machine.** If the HTML points at local `file://` fonts, the PDF only renders correctly on a checkout that has them. Note this in the source folder's README.
- **The listing is translated separately from app strings.** Per-locale store text, screenshots and the feature graphic are console uploads, not build outputs. Track them as files so no language is forgotten (see [android-per-app-language](android-per-app-language.md)).
- **The insights lint tool can crash** on newer Android Gradle Plugin versions in one of its own detectors, unrelated to your code. Read the findings that were reported, and don't treat the crash as a finding.

### Playbook confidence: medium
The checklist, the lint harness, the policy-hosting flow and the listing layout come from a real submission preparation. Play review outcomes and form wording acceptance were not available at extraction time, so the justification texts are patterns, not proven-accepted wording. Listing limits other than the 80-character short description, and Play CSV import/export behaviour, come from general Play knowledge, not from the source.

## Adoption
Used in **2** repos in this marketplace. The first is a native Android app with a local VPN service, two `specialUse` foreground services, `QUERY_ALL_PACKAGES`, a crash reporter with an opt-out, and a 12-locale store listing. The second is an app on a personal developer account (so a closed test with 12 testers for 14 days precedes production access), with a Data safety form covering contacts plus crash/analytics data, a hosted account-deletion URL, `READ_CONTACTS` behind an in-app prominent disclosure, and `POST_NOTIFICATIONS`.
