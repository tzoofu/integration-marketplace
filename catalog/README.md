# Integration index

Cross-repo index, organized **per integration** and grouped into **category subfolders** — see
`catalog/SCHEMA.md` for why and the entry format. Each integration file is a self-contained
**implementation playbook**: prerequisites, setup steps, a generic sanitized code pattern, env
vars, and gotchas — written so a fresh Claude/Cursor session with no access to the source repos
can act on it directly.

**No repo names, project names, or business-identifying details appear anywhere below** — not
even as anonymous codenames. The "Adopters" column is a count only. Each integration file's own
`## Adoption` section may add a repo-agnostic note on implementation variants where relevant, but
never says which repo has which variant.

## By category

### [auth](auth/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firebase-auth](auth/firebase-auth.md) | Firebase / Google | 9 | yes |
| [personal-api-tokens](auth/personal-api-tokens.md) | internal (custom) | 1 | yes |
| [shareable-invite-links](auth/shareable-invite-links.md) | internal (custom) | 1 | yes |

### [database](database/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firestore](database/firestore.md) | Firebase (Google Cloud) | 8 | partial |
| [firestore-read-cost-discipline](database/firestore-read-cost-discipline.md) | Firestore (Admin SDK + data cache) | 1 | yes |
| [sqlite-embedded](database/sqlite-embedded.md) | SQLite (`modernc.org/sqlite`, pure Go) | 1 | yes |

### [messaging](messaging/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [whatsapp-deeplink](messaging/whatsapp-deeplink.md) | WhatsApp / Meta | 5 | yes |
| [beeper-messaging](messaging/beeper-messaging.md) | Beeper | 1 | partial |
| [telegram-bot](messaging/telegram-bot.md) | Telegram (python-telegram-bot v20) | 1 | yes |
| [whatsapp-cloud-api](messaging/whatsapp-cloud-api.md) | Meta (WhatsApp Cloud API) | 1 | yes |

### [maps-geo](maps-geo/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [map-tiles](maps-geo/map-tiles.md) | OpenFreeMap / OSM / Esri / CARTO / OpenTopoMap | 2 | yes |
| [govmap-gis](maps-geo/govmap-gis.md) | GovMap (Israel Survey) | 1 | no |
| [nominatim-geocoding](maps-geo/nominatim-geocoding.md) | OpenStreetMap (Nominatim) | 2 | yes |
| [waze-deeplink](maps-geo/waze-deeplink.md) | Waze / Google | 3 | yes |
| [osrm-commute-routing](maps-geo/osrm-commute-routing.md) | OpenStreetMap (OSRM) | 1 | yes |
| [nearest-transit-stations](maps-geo/nearest-transit-stations.md) | OpenStreetMap data (static GeoJSON) | 1 | yes |
| [kml-zone-import](maps-geo/kml-zone-import.md) | internal (KML/KMZ) | 1 | yes |

### [ai-llm](ai-llm/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [openai-whisper](ai-llm/openai-whisper.md) | OpenAI | 1 | yes |
| [gemini-summarization](ai-llm/gemini-summarization.md) | Google (Gemini via Vercel AI SDK) | 1 | yes |
| [gemini-gated-structured-features](ai-llm/gemini-gated-structured-features.md) | Google (Gemini via `@google/genai` / Go `genai`) | 2 | yes |
| [pyannote-diarization](ai-llm/pyannote-diarization.md) | Hugging Face (pyannote.audio, local) | 1 | partial |
| [voice-note-transcription](ai-llm/voice-note-transcription.md) | Google Gemini audio / whisper.cpp (local) | 1 | yes |

### [email-sms](email-sms/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firebase-email-link-invite](email-sms/firebase-email-link-invite.md) | Firebase / Google (Identity Toolkit) | 2 | partial |

### [analytics](analytics/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [ga4-analytics](analytics/ga4-analytics.md) | Google | 5 | yes |
| [firebase-crashlytics](analytics/firebase-crashlytics.md) | Google Firebase (Crashlytics) | 2 | yes |
| [android-local-crash-log](analytics/android-local-crash-log.md) | internal (Android `UncaughtExceptionHandler` + `FileProvider`) | 1 | yes |

### [hosting-deploy](hosting-deploy/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [vercel-hosting](hosting-deploy/vercel-hosting.md) | Vercel | 7 | partial |
| [firebase-hosting](hosting-deploy/firebase-hosting.md) | Google Firebase Hosting | 2 | partial |
| [firebase-hosting-deploy](hosting-deploy/firebase-hosting-deploy.md) | Firebase | 1 | no |
| [android-play-release-pipeline](hosting-deploy/android-play-release-pipeline.md) | GitHub Actions + Google Play Developer API | 1 | yes |

### [payments](payments/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [bit-payment](payments/bit-payment.md) | Bit | 1 | yes |
| [paybox-payment](payments/paybox-payment.md) | Paybox | 1 | yes |

### [push-notifications](push-notifications/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [fcm-push](push-notifications/fcm-push.md) | Firebase (FCM) | 4 | yes |
| [browser-notification-api](push-notifications/browser-notification-api.md) | Web Platform (browser-native) | 1 | no |
| [notification-dispatch](push-notifications/notification-dispatch.md) | internal (custom, on FCM) | 2 | partial |
| [android-workmanager-local-notifications](push-notifications/android-workmanager-local-notifications.md) | Android WorkManager (no FCM) | 1 | yes |

### [admin-approval-workflow](admin-approval-workflow/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [admin-approval-workflow](admin-approval-workflow/admin-approval-workflow.md) | internal | 8 | partial |

### [storage](storage/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firebase-storage](storage/firebase-storage.md) | Firebase / Google Cloud Storage | 5 | yes |
| [image-pipeline](storage/image-pipeline.md) | internal (on Firebase Storage) | 1 | partial |
| [local-filesystem-storage](storage/local-filesystem-storage.md) | none (custom) | 3 | partial |
| [rclone-sqlite-backup](storage/rclone-sqlite-backup.md) | rclone + any S3-compatible store | 1 | yes |

### [other](other/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firebase-admin-sdk](other/firebase-admin-sdk.md) | Firebase | 3 | yes |
| [firebase-usage-dashboard](other/firebase-usage-dashboard.md) | Google Cloud (Monitoring + Billing Budget REST) | 1 | yes |
| [recaptcha-enterprise](other/recaptcha-enterprise.md) | Google | 2 | yes |
| [recaptcha-v3](other/recaptcha-v3.md) | Google | 1 | yes |
| [mcp-server](other/mcp-server.md) | Anthropic MCP | 6 | yes |
| [boi-exchange-rates](other/boi-exchange-rates.md) | Bank of Israel | 1 | yes |
| [react-pdf-renderer](other/react-pdf-renderer.md) | @react-pdf/renderer | 1 | yes |
| [play-core-in-app-update](other/play-core-in-app-update.md) | Google Play (Play Core) | 1 | yes |
| [web-share-clipboard](other/web-share-clipboard.md) | Web Platform (browser-native) | 2 | yes |
| [google-adsense](other/google-adsense.md) | Google | 1 | yes |
| [google-calendar](other/google-calendar.md) | Google | 1 | yes |
| [calendar-deeplinks](other/calendar-deeplinks.md) | Google Calendar / Outlook + .ics | 1 | yes |
| [shadcn-ui](other/shadcn-ui.md) | shadcn | 2 | partial |
| [design-system-semantic-tokens](other/design-system-semantic-tokens.md) | internal (Tailwind v4, optional shadcn/ui) | 2 | yes |
| [cookie-driven-multi-theme-and-mode](other/cookie-driven-multi-theme-and-mode.md) | internal (Next.js + Tailwind v4, CSS oklch) | 1 | yes |
| [pwa-manifest](other/pwa-manifest.md) | Web Platform (browser-native) | 2 | yes |
| [chrome-origin-trial](other/chrome-origin-trial.md) | Google Chrome | 1 | no |
| [webmcp-page-tools](other/webmcp-page-tools.md) | W3C WebMCP / Google Chrome | 1 | yes |
| [google-search-console](other/google-search-console.md) | Google | 1 | yes |
| [google-fonts](other/google-fonts.md) | Google Fonts | 3 | yes |
| [i18n-language-support](other/i18n-language-support.md) | next-intl / i18next / internal | 7 | yes |
| [legal-compliance](other/legal-compliance.md) | internal (Consent Mode v2 + axe-core) | 6 | yes |
| [ffmpeg-audio-processing](other/ffmpeg-audio-processing.md) | FFmpeg project | 2 | yes |
| [source-push-api](other/source-push-api.md) | internal (planned, spec-only) | 1 | no |
| [response-cache-layer](other/response-cache-layer.md) | internal (Next.js cache primitives) | 1 | yes |
| [listing-relevance-checker](other/listing-relevance-checker.md) | internal (browser automation) | 1 | partial |
| [admin-nav-system](other/admin-nav-system.md) | internal (custom) | 1 | yes |
| [admin-settings-editor](other/admin-settings-editor.md) | internal (custom) | 1 | yes |
| [command-palette](other/command-palette.md) | internal (custom) | 3 | yes |
| [swipe-card-engine](other/swipe-card-engine.md) | internal (custom) | 1 | yes |
| [view-mode-card-fields](other/view-mode-card-fields.md) | internal (custom) | 1 | yes |
| [super-admin-view-mode](other/super-admin-view-mode.md) | internal (custom) | 1 | yes |
| [android-calendar-sync](other/android-calendar-sync.md) | Android (CalendarContract) | 1 | no |
| [zxing-qrcode](other/zxing-qrcode.md) | ZXing | 2 | yes |
| [suncalc-sunset-cutoffs](other/suncalc-sunset-cutoffs.md) | suncalc (npm) | 1 | yes |
| [onboarding-spotlight-tour](other/onboarding-spotlight-tour.md) | internal (custom) | 1 | yes |
| [whats-new-changelog](other/whats-new-changelog.md) | internal (custom) | 2 | yes |
| [outbound-webhooks-hmac](other/outbound-webhooks-hmac.md) | internal (custom) | 1 | yes |
| [secrets-at-rest-aes-gcm](other/secrets-at-rest-aes-gcm.md) | internal (Go `crypto/cipher`) | 1 | yes |
| [ssrf-safe-link-preview](other/ssrf-safe-link-preview.md) | internal (custom) | 1 | yes |
| [sse-realtime-hub](other/sse-realtime-hub.md) | internal (Go `net/http` + `EventSource`) | 1 | yes |
| [android-headless-toggle-entrypoints](other/android-headless-toggle-entrypoints.md) | internal (Android `Service` / `TileService` / `AppWidgetProvider`) | 1 | yes |
| [android-per-app-language](other/android-per-app-language.md) | Android (`LocaleManager`, AGP `localeFilters`) | 1 | yes |
| [play-console-compliance](other/play-console-compliance.md) | Google Play Console / Play Policy Insights | 2 | yes |
| [compose-multi-theme-palettes](other/compose-multi-theme-palettes.md) | internal (Jetpack Compose + Material 3) | 1 | yes |
| [material-symbols-subset-font](other/material-symbols-subset-font.md) | Google Fonts Material Symbols + fontTools | 1 | yes |
| [android-contacts-import](other/android-contacts-import.md) | Android (ContactsContract) | 1 | yes |

## Shared patterns across repos (reuse candidates)

Every integration above with 2+ adopters is, by construction, a candidate for extraction into a
shared package. Notable ones:

- **Firebase Auth** ([auth/firebase-auth.md](auth/firebase-auth.md)) — Google sign-in with several
  session strategies in play: a custom server-issued session cookie, per-request Bearer/ID-token
  verification with no session cookie at all, pure client-SDK usage with no server layer, and two
  native-Android implementations using Credential Manager instead of a web popup/redirect (one also
  offers an account-less device-only mode that uploads its data when later connected to Google). One
  Go backend verifies the ID token against Google's JWKS itself (no Admin SDK) before minting its session.
- **Firestore + Firebase Admin SDK bootstrap**
  ([database/firestore.md](database/firestore.md),
  [other/firebase-admin-sdk.md](other/firebase-admin-sdk.md)) — most adopters share the same
  server-only Admin SDK init pattern; a few outliers do pure client-SDK read+write with no
  server layer at all (one native app routes between an on-device file store and Firestore behind
  one repository interface). Checked-in security rules don't always match what's deployed — verify the
  live ruleset, not just the repo copy.
- **Firebase Cloud Messaging** ([push-notifications/fcm-push.md](push-notifications/fcm-push.md))
  — near-identical token registration, stale-token cleanup, and dynamically-served service worker
  across the web adopters (two incompatible service-worker payload styles are in use, both
  documented as variants); one native-mobile adopter has no VAPID/service-worker concept at all.
  A fourth adopter sends from a Deno edge function straight to the FCM HTTP v1 API with a
  service-account OAuth2 exchange (no Admin SDK). Highest-value extraction candidate.
- **Firebase Storage** ([storage/firebase-storage.md](storage/firebase-storage.md)) — same
  upload/public-read-object pattern across most adopters via the Admin SDK server-side; one
  adopter uploads directly from a native client SDK gated by storage security rules instead.
- **Google Analytics 4** ([analytics/ga4-analytics.md](analytics/ga4-analytics.md)) — three
  distinct client shapes in play (raw `gtag.js`, a framework's built-in analytics wrapper, Firebase
  Analytics SDK, on web and native Android), plus one server-side Measurement Protocol sender.
- **reCAPTCHA Enterprise** ([other/recaptcha-enterprise.md](other/recaptcha-enterprise.md)) — both
  adopters verify via the Assessments API using a Firebase Admin-derived access token, fail-open on
  error. Distinct from the classic score-based
  [reCAPTCHA v3](other/recaptcha-v3.md) elsewhere in the marketplace (no GCP project needed).
- **WhatsApp `wa.me` deep-link**
  ([messaging/whatsapp-deeplink.md](messaging/whatsapp-deeplink.md)) — identical no-SDK pattern
  across adopters; one implementation's callback-form templating is meaningfully safer (avoids a
  sequential-replace/`$`-pattern injection bug the others are exposed to) and is documented as the
  recommended variant.
- **Free/keyless map tiles** ([maps-geo/map-tiles.md](maps-geo/map-tiles.md)) — no paid API key
  required by either adopter, though they pick different tile providers for the same "minimal
  style" slot (vector vs. raster).
- **Hosting/deploy platform** ([hosting-deploy/vercel-hosting.md](hosting-deploy/vercel-hosting.md))
  — same platform across most adopters; not a code package, worth a shared deploy-checklist doc
  instead. A couple of adopters' setups are low-confidence (no real linked-project config found, or
  a previously-documented feature that couldn't be re-verified against current source).
- **Admin-approval-workflow shape**
  ([admin-approval-workflow/admin-approval-workflow.md](admin-approval-workflow/admin-approval-workflow.md))
  — structurally similar (Firestore-backed state + role check) across adopters despite different
  domain objects; one SQL-backed adopter adds invite-only sign-up and per-resource roles granted
  directly or through groups.
- **Self-hosted MCP server (OAuth 2.1 + PAT)** ([other/mcp-server.md](other/mcp-server.md)) —
  several adopters each expose their own data as MCP tools behind an OAuth2.1(or 2.0)/PKCE + PAT
  scheme. The authorization-server half (DCR, PKCE code, rotating refresh, hashed PATs) is
  near-identical everywhere; implementations differ in signing-secret strategy and in transport (a
  long-lived server vs a stateless per-request server whose tool list is filtered by role). One
  adopter skips OAuth entirely: a thin stdio binary calls the app's REST API with a scoped personal
  API token ([auth/personal-api-tokens.md](auth/personal-api-tokens.md)).
- **Firebase email-link invite (Identity Toolkit REST)**
  ([email-sms/firebase-email-link-invite.md](email-sms/firebase-email-link-invite.md)) — both
  adopters call `identitytoolkit.googleapis.com/v1/accounts:sendOobCode` directly, with a
  near-identical server-send + client-completion pattern.
- **Web Share API / Clipboard API**
  ([other/web-share-clipboard.md](other/web-share-clipboard.md)) — both adopters implement the
  same share-with-clipboard-fallback pattern.
- **Google Fonts** ([other/google-fonts.md](other/google-fonts.md)) — all adopters load
  Hebrew-supporting typefaces; the Next.js ones via `next/font/google`, a Vite one via a CSS
  `@import` plus `@fontsource/*` packages.
- **Keyless OSM geocoding + Waze links** ([maps-geo/nominatim-geocoding.md](maps-geo/nominatim-geocoding.md),
  [maps-geo/waze-deeplink.md](maps-geo/waze-deeplink.md)) — forward geocoding in one adopter,
  browser-side reverse geocoding (plus OSRM road-snapping) in the other; identical no-key Waze
  `ul?q=` links in both web adopters, and a third (native Android) fires the same link as an intent.
- **ZXing** ([other/zxing-qrcode.md](other/zxing-qrcode.md)) — QR generation in one adopter,
  camera + still-photo barcode decoding in the other; a shared package would expose both.
- **shadcn/ui + command palette** ([other/shadcn-ui.md](other/shadcn-ui.md),
  [other/command-palette.md](other/command-palette.md)) — same scaffolding workflow on Next.js and
  Vite; the palettes range from a role-gated command launcher and a cross-entity search to a
  mixed launcher of record jumps, page navigation, actions on the open record and text templates.
- **Notification dispatch** ([push-notifications/notification-dispatch.md](push-notifications/notification-dispatch.md))
  — one adopter fans out in code (broadcast / group / filter-matched), the other is table-driven
  (one inserted row → in-app realtime + email + push).
- **i18n / language support** ([other/i18n-language-support.md](other/i18n-language-support.md)) —
  every adopter supports exactly `he` + `en` with RTL for Hebrew; all but two default to `he`
  (the native-mobile one follows the device locale and a chat SPA follows the browser language,
  both falling back to `en`). Also covers mixed-direction user content and Hebrew FTS5 search. Locale storage
  differs by shape (URL segment + cookie mirror, cookie-only, localStorage, device + persisted
  setting), as does the library (next-intl, i18next, hand-rolled typed dictionary). Shared
  extraction candidates: the locale constant/`dir` helper, a catalog-parity check, and
  locale-explicit `Intl` formatting helpers.

- **Legal compliance** ([other/legal-compliance.md](other/legal-compliance.md)) — privacy policy,
  terms, Israeli accessibility statement (IS 5568 / reg. 35), cookie consent, form consent and an
  axe audit, all required **per supported locale**. Ships with a copy-paste verifier prompt that
  audits an app per locale before implementing. Extraction candidates: the versioned consent store,
  `LegalLinks`, the legal-docs parity script, and the axe audit script.

- **Gated multi-user Gemini features**
  ([ai-llm/gemini-gated-structured-features.md](ai-llm/gemini-gated-structured-features.md)) — two
  adopters (TypeScript `@google/genai` and Go `genai`) put every call behind the same gate: kill
  switch, then the user's encrypted key or the app key, then per-user and global daily quotas. The
  gate is the extraction candidate; prompts stay per-app.
- **Semantic-token design system**
  ([other/design-system-semantic-tokens.md](other/design-system-semantic-tokens.md), drop-in CSS in
  [other/assets/design-tokens.css](other/assets/design-tokens.css)) — Tailwind v4 CSS-first tokens,
  AA-tuned light/dark values and one status-color registry, on Next.js + shadcn in one adopter and
  a plain Vite SPA in the other, which also fails the build on raw palette classes and sub-AA pairs.
- **ffmpeg audio prep** ([other/ffmpeg-audio-processing.md](other/ffmpeg-audio-processing.md)) —
  both adopters shell out to system `ffmpeg` before transcription; one normalizes and chunks long
  recordings on silence, the other only converts short voice notes to the engine's input format.
- **PWA manifest** ([other/pwa-manifest.md](other/pwa-manifest.md)) — Next.js `app/manifest.ts` in
  one adopter, a static `manifest.webmanifest` plus a `beforeinstallprompt` install button in a Vite
  SPA in the other; neither needs a service worker to be installable.
- **Local filesystem storage**
  ([storage/local-filesystem-storage.md](storage/local-filesystem-storage.md)) — files under a data
  directory; one server adopter has no database at all, another keeps media next to SQLite with a
  size cap and age-based pruning, and a native-Android app keeps its whole account-less data set in
  one on-device JSON file with export/restore and corrupt-file recovery.
- **Firebase Crashlytics** ([analytics/firebase-crashlytics.md](analytics/firebase-crashlytics.md)) —
  two Android adopters: one Crashlytics-only with a user opt-out, one Crashlytics + Firebase Analytics
  behind a single no-PII facade that is safe to call from JVM unit tests. The facade is the
  extraction candidate.
- **In-app "What's new" changelog** ([other/whats-new-changelog.md](other/whats-new-changelog.md))
  — a web adopter with typed entries, per-user last-seen id, role-gated entries, nav badge and
  one-time banner (enforced by unit tests and an agent rule); a native-Android adopter with static
  per-release resource entries and a list screen only. Shared core: entry list + screen. Pairs with
  [other/onboarding-spotlight-tour.md](other/onboarding-spotlight-tour.md).
- **Firebase Hosting for store-required pages**
  ([hosting-deploy/firebase-hosting.md](hosting-deploy/firebase-hosting.md)) — both adopters are
  Android apps that host only static pages a Play listing needs (privacy policy; landing and
  account-deletion pages in one); a tiny template, not a package.
- **Play Console compliance** ([other/play-console-compliance.md](other/play-console-compliance.md))
  — the same pre-submit checklist (sensitive-permission declarations, Data safety, hosted policy
  pages) across two Android apps; one also goes through the personal-account closed-test gate
  (12 testers for 14 days) before production.

## Single-adopter integrations (notable, not yet shared)

Everything above with exactly 1 adopter is a candidate worth watching for a second adopter before
extracting anything — see each file's own `## Adoption` note for context. Grouped by theme rather
than by (unnamed) adopter:

- **One cohesive internal feature set**: several single-adopter integrations (dead-listing
  detection via browser automation, an ownership-validated image upload/moderation pipeline,
  role-based admin navigation, a ⌘K command palette, a gesture-based swipe-card deck, a data-driven
  first-run spotlight onboarding tour ([other/onboarding-spotlight-tour.md](other/onboarding-spotlight-tour.md)), a persisted
  view-mode/card-field customization system, a session-scoped super-admin read-only tenant preview
  mode, a two-tier response cache layer, a Cloud Monitoring usage/cost dashboard
  ([other/firebase-usage-dashboard.md](other/firebase-usage-dashboard.md)), Firestore read-cost
  rules ([database/firestore-read-cost-discipline.md](database/firestore-read-cost-discipline.md))) happen to belong to the same adopter and share patterns
  with each other more than with anything else in the catalog.
- **In-browser agent tools (WebMCP)**
  ([other/webmcp-page-tools.md](other/webmcp-page-tools.md)) — per-page `document.modelContext`
  tools gated like the UI, with toast capture for outcomes. Pairs with
  [other/chrome-origin-trial.md](other/chrome-origin-trial.md) (turning the API on for visitors)
  and [other/mcp-server.md](other/mcp-server.md) (the same actions for clients outside the browser).
- **Self-hosted Go + SQLite service building blocks**: the official WhatsApp Cloud API
  ([messaging/whatsapp-cloud-api.md](messaging/whatsapp-cloud-api.md)) next to the Telegram bot
  playbook ([messaging/telegram-bot.md](messaging/telegram-bot.md)). Supporting pieces from the same
  stack that transfer to any Go service:
  [database/sqlite-embedded.md](database/sqlite-embedded.md),
  [other/secrets-at-rest-aes-gcm.md](other/secrets-at-rest-aes-gcm.md),
  [auth/personal-api-tokens.md](auth/personal-api-tokens.md),
  [other/outbound-webhooks-hmac.md](other/outbound-webhooks-hmac.md),
  [other/ssrf-safe-link-preview.md](other/ssrf-safe-link-preview.md),
  [other/sse-realtime-hub.md](other/sse-realtime-hub.md),
  [ai-llm/voice-note-transcription.md](ai-llm/voice-note-transcription.md) (Gemini or local
  whisper.cpp), and nightly off-site backups
  ([storage/rclone-sqlite-backup.md](storage/rclone-sqlite-backup.md)).
- **Israel-market integrations**: GovMap GIS, Bit/Paybox payment
  deep-links (static pre-generated links, no API/keys/webhooks), Bank of Israel
  exchange rates, classic reCAPTCHA v3, Hebrew-RTL server-side PDF rendering (working around a real
  upstream bidi-reordering crash) — clustered in adopters with an Israeli consumer/business focus.
- **Audio/AI pipeline**: OpenAI Whisper transcription, Gemini summarization (via Vercel AI SDK),
  local pyannote.audio speaker diarization (Python subprocess) — all in one single-tenant, local-first audio-processing adopter with no
  auth/database/messaging/payments/push/maps/analytics integrations of its own.
- **Native-mobile-only**: Google Play Core in-app updates appears in one native-mobile adopter
  whose source wasn't available locally during this catalog's last refresh — that playbook is
  synthesized from general provider documentation rather than extracted from real code, and marked
  `Playbook confidence: low` (its Crashlytics and Firebase Hosting siblings now have a second adopter,
  see above). A separate native-mobile adopter contributes on-device calendar sync
  mirrored into a cloud database, and QR code generation for invite codes.
- **Native Android app, extracted from real source**: a CI/CD path from push to a signed bundle on
  Play's internal track
  ([hosting-deploy/android-play-release-pipeline.md](hosting-deploy/android-play-release-pipeline.md)),
  one foreground service driven from the app, the Quick Settings shade and a widget with one
  decision function and one pref writer
  ([other/android-headless-toggle-entrypoints.md](other/android-headless-toggle-entrypoints.md)),
  a local crash file with a share sheet
  ([analytics/android-local-crash-log.md](analytics/android-local-crash-log.md), pairs with
  [analytics/firebase-crashlytics.md](analytics/firebase-crashlytics.md)), many-locale per-app language
  ([other/android-per-app-language.md](other/android-per-app-language.md), the native-Android
  counterpart to [other/i18n-language-support.md](other/i18n-language-support.md)) and the pre-submit
  Play Console declarations and policy hosting
  ([other/play-console-compliance.md](other/play-console-compliance.md), now shared — see above).
- **Web theming on top of semantic tokens**: cookie-driven named themes (hue-parametrized `oklch`) plus light/dark/system mode with no flash and no theme library ([other/cookie-driven-multi-theme-and-mode.md](other/cookie-driven-multi-theme-and-mode.md), extends [other/design-system-semantic-tokens.md](other/design-system-semantic-tokens.md)).
- **Native Android (Compose) app building blocks**: named multi-theme palettes with follow-system and
  live switching ([other/compose-multi-theme-palettes.md](other/compose-multi-theme-palettes.md), the
  native-Android counterpart to
  [other/design-system-semantic-tokens.md](other/design-system-semantic-tokens.md)), a subset
  Material Symbols icon font with a generated codepoint map
  ([other/material-symbols-subset-font.md](other/material-symbols-subset-font.md)), local
  WorkManager reminders reconciled from app data with no push server
  ([push-notifications/android-workmanager-local-notifications.md](push-notifications/android-workmanager-local-notifications.md),
  complementary to [push-notifications/fcm-push.md](push-notifications/fcm-push.md)) and address-book
  import with phone-based dedup ([other/android-contacts-import.md](other/android-contacts-import.md)).
- **Calendar deep-links (no-auth)**: `.ics` generation plus Google/Outlook calendar deep-links,
  distinct from the OAuth-based Google Calendar sync integration elsewhere in the catalog.
- **Draft-state admin settings editor**
  ([other/admin-settings-editor.md](other/admin-settings-editor.md)) — a multi-tab business
  settings screen where the page-level orchestrator owns draft/dirty state and a sticky
  save-bar, tab components stay purely presentational, and saves flow through one shared
  patch-and-merge helper rather than a full refetch.
- **Telegram bot as the whole UI** ([messaging/telegram-bot.md](messaging/telegram-bot.md)) — an
  async long-polling bot (no web frontend, no inbound ports) with namespaced inline-keyboard
  callbacks, opt-in consent + GDPR-style export/delete commands keyed by HMAC-pseudonymized user ids,
  and per-user two-language localization; deployed as a single Docker Compose container.
- **Direct browser Notification API**: foreground-only alerts, complementary to (not a replacement
  for) the FCM background-push integration in the same adopter.
- **Unimplemented/spec-only**: a designed-but-not-yet-built API for future generic third-party
  data-source push integrations.
