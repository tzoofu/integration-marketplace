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
| [firebase-auth](auth/firebase-auth.md) | Firebase / Google | 7 | yes |

### [database](database/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firestore](database/firestore.md) | Firebase (Google Cloud) | 7 | partial |

### [messaging](messaging/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [whatsapp-deeplink](messaging/whatsapp-deeplink.md) | WhatsApp / Meta | 3 | yes |
| [beeper-messaging](messaging/beeper-messaging.md) | Beeper | 1 | partial |

### [scraping-source](scraping-source/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [yad2-scraping](scraping-source/yad2-scraping.md) | Yad2 | 1 | partial |
| [facebook-scraping](scraping-source/facebook-scraping.md) | Facebook / Meta | 1 | partial |
| [behatsdaa-scraping](scraping-source/behatsdaa-scraping.md) | Behatsdaa | 1 | partial |

### [maps-geo](maps-geo/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [map-tiles](maps-geo/map-tiles.md) | OpenFreeMap / OSM / Esri / CARTO / OpenTopoMap | 2 | yes |
| [govmap-gis](maps-geo/govmap-gis.md) | GovMap (Israel Survey) | 1 | no |
| [nominatim-geocoding](maps-geo/nominatim-geocoding.md) | OpenStreetMap (Nominatim) | 1 | yes |
| [waze-deeplink](maps-geo/waze-deeplink.md) | Waze / Google | 1 | yes |

### [ai-llm](ai-llm/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [openai-whisper](ai-llm/openai-whisper.md) | OpenAI | 1 | yes |
| [gemini-summarization](ai-llm/gemini-summarization.md) | Google (Gemini via Vercel AI SDK) | 1 | yes |
| [pyannote-diarization](ai-llm/pyannote-diarization.md) | Hugging Face (pyannote.audio, local) | 1 | partial |

### [email-sms](email-sms/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firebase-email-link-invite](email-sms/firebase-email-link-invite.md) | Firebase / Google (Identity Toolkit) | 2 | partial |

### [analytics](analytics/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [ga4-analytics](analytics/ga4-analytics.md) | Google | 4 | yes |
| [firebase-crashlytics](analytics/firebase-crashlytics.md) | Google Firebase (Crashlytics) | 1 | yes |

### [hosting-deploy](hosting-deploy/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [vercel-hosting](hosting-deploy/vercel-hosting.md) | Vercel | 7 | partial |
| [firebase-hosting](hosting-deploy/firebase-hosting.md) | Google Firebase Hosting | 1 | no |
| [firebase-hosting-deploy](hosting-deploy/firebase-hosting-deploy.md) | Firebase | 1 | no |

### [payments](payments/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [bit-payment](payments/bit-payment.md) | Bit | 1 | yes |
| [paybox-payment](payments/paybox-payment.md) | Paybox | 1 | yes |

### [push-notifications](push-notifications/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [fcm-push](push-notifications/fcm-push.md) | Firebase (FCM) | 3 | yes |
| [browser-notification-api](push-notifications/browser-notification-api.md) | Web Platform (browser-native) | 1 | no |
| [notification-dispatch](push-notifications/notification-dispatch.md) | internal (custom, on FCM) | 1 | partial |

### [admin-approval-workflow](admin-approval-workflow/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [admin-approval-workflow](admin-approval-workflow/admin-approval-workflow.md) | internal | 7 | partial |

### [storage](storage/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firebase-storage](storage/firebase-storage.md) | Firebase / Google Cloud Storage | 5 | yes |
| [image-pipeline](storage/image-pipeline.md) | internal (on Firebase Storage) | 1 | partial |
| [local-filesystem-storage](storage/local-filesystem-storage.md) | none (custom) | 1 | no |

### [other](other/)
| Integration | Provider | Adopters | Reusable |
|---|---|---|---|
| [firebase-admin-sdk](other/firebase-admin-sdk.md) | Firebase | 3 | yes |
| [recaptcha-enterprise](other/recaptcha-enterprise.md) | Google | 2 | yes |
| [recaptcha-v3](other/recaptcha-v3.md) | Google | 1 | yes |
| [mcp-server](other/mcp-server.md) | Anthropic MCP | 4 | yes |
| [boi-exchange-rates](other/boi-exchange-rates.md) | Bank of Israel | 1 | yes |
| [react-pdf-renderer](other/react-pdf-renderer.md) | @react-pdf/renderer | 1 | yes |
| [play-core-in-app-update](other/play-core-in-app-update.md) | Google Play (Play Core) | 1 | yes |
| [web-share-clipboard](other/web-share-clipboard.md) | Web Platform (browser-native) | 2 | yes |
| [google-adsense](other/google-adsense.md) | Google | 1 | yes |
| [google-calendar](other/google-calendar.md) | Google | 1 | yes |
| [calendar-deeplinks](other/calendar-deeplinks.md) | Google Calendar / Outlook + .ics | 1 | yes |
| [shadcn-ui](other/shadcn-ui.md) | shadcn | 1 | partial |
| [pwa-manifest](other/pwa-manifest.md) | Web Platform (browser-native) | 1 | yes |
| [chrome-origin-trial](other/chrome-origin-trial.md) | Google Chrome | 1 | no |
| [google-search-console](other/google-search-console.md) | Google | 1 | yes |
| [google-fonts](other/google-fonts.md) | Google Fonts | 2 | yes |
| [ffmpeg-audio-processing](other/ffmpeg-audio-processing.md) | FFmpeg project | 1 | yes |
| [source-push-api](other/source-push-api.md) | internal (planned, spec-only) | 1 | no |
| [response-cache-layer](other/response-cache-layer.md) | internal (Next.js cache primitives) | 1 | yes |
| [listing-relevance-checker](other/listing-relevance-checker.md) | internal (browser automation) | 1 | partial |
| [admin-nav-system](other/admin-nav-system.md) | internal (custom) | 1 | yes |
| [command-palette](other/command-palette.md) | internal (custom) | 1 | yes |
| [swipe-card-engine](other/swipe-card-engine.md) | internal (custom) | 1 | yes |
| [view-mode-card-fields](other/view-mode-card-fields.md) | internal (custom) | 1 | yes |
| [android-calendar-sync](other/android-calendar-sync.md) | Android (CalendarContract) | 1 | no |
| [zxing-qrcode](other/zxing-qrcode.md) | ZXing | 1 | yes |

## Shared patterns across repos (reuse candidates)

Every integration above with 2+ adopters is, by construction, a candidate for extraction into a
shared package. Notable ones:

- **Firebase Auth** ([auth/firebase-auth.md](auth/firebase-auth.md)) — Google sign-in with several
  session strategies in play: a custom server-issued session cookie, per-request Bearer/ID-token
  verification with no session cookie at all, pure client-SDK usage with no server layer, and one
  native-mobile implementation using platform credential storage instead of a web popup/redirect.
- **Firestore + Firebase Admin SDK bootstrap**
  ([database/firestore.md](database/firestore.md),
  [other/firebase-admin-sdk.md](other/firebase-admin-sdk.md)) — most adopters share the same
  server-only Admin SDK init pattern; a couple of outliers do pure client-SDK read+write with no
  server layer at all. One adopter's checked-in security-rules file carries a comment marking part
  of its documented ruleset as staged but **not yet deployed** — a reminder that checked-in rules
  don't always match what's live in production.
- **Firebase Cloud Messaging** ([push-notifications/fcm-push.md](push-notifications/fcm-push.md))
  — near-identical token registration, stale-token cleanup, and dynamically-served service worker
  across the web adopters (two incompatible service-worker payload styles are in use, both
  documented as variants); one native-mobile adopter has no VAPID/service-worker concept at all.
  Highest-value extraction candidate.
- **Firebase Storage** ([storage/firebase-storage.md](storage/firebase-storage.md)) — same
  upload/public-read-object pattern across most adopters via the Admin SDK server-side; one
  adopter uploads directly from a native client SDK gated by storage security rules instead.
- **Google Analytics 4** ([analytics/ga4-analytics.md](analytics/ga4-analytics.md)) — three
  distinct client shapes in play (raw `gtag.js`, a framework's built-in analytics wrapper, Firebase
  Analytics SDK), plus one server-side Measurement Protocol sender.
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
  domain objects.
- **Self-hosted MCP server (OAuth 2.1 + PAT)** ([other/mcp-server.md](other/mcp-server.md)) —
  several adopters each expose their own data as MCP tools behind an OAuth2.1(or 2.0)/PKCE + PAT
  scheme.
- **Firebase email-link invite (Identity Toolkit REST)**
  ([email-sms/firebase-email-link-invite.md](email-sms/firebase-email-link-invite.md)) — both
  adopters call `identitytoolkit.googleapis.com/v1/accounts:sendOobCode` directly, with a
  near-identical server-send + client-completion pattern.
- **Web Share API / Clipboard API**
  ([other/web-share-clipboard.md](other/web-share-clipboard.md)) — both adopters implement the
  same share-with-clipboard-fallback pattern.
- **Google Fonts (`next/font/google`)** ([other/google-fonts.md](other/google-fonts.md)) — both
  adopters load Hebrew-supporting typefaces the same way.

## Single-adopter integrations (notable, not yet shared)

Everything above with exactly 1 adopter is a candidate worth watching for a second adopter before
extracting anything — see each file's own `## Adoption` note for context. Grouped by theme rather
than by (unnamed) adopter:

- **One cohesive internal feature set**: several single-adopter integrations (dead-listing
  detection via browser automation, an ownership-validated image upload/moderation pipeline,
  role-based admin navigation, a ⌘K command palette, a gesture-based swipe-card deck, a persisted
  view-mode/card-field customization system, a two-tier response cache layer) happen to belong to
  the same adopter and share patterns with each other more than with anything else in the catalog.
- **Israel-market integrations**: Yad2/Facebook scraping sources, GovMap GIS, Bit/Paybox payment
  deep-links (static pre-generated links, no API/keys/webhooks), Bank of Israel
  exchange rates, classic reCAPTCHA v3, Hebrew-RTL server-side PDF rendering (working around a real
  upstream bidi-reordering crash), a rewards-portal scraping source (Playwright-driven, manual
  login) — clustered in adopters with an Israeli consumer/business focus.
- **Audio/AI pipeline**: OpenAI Whisper transcription, Gemini summarization (via Vercel AI SDK),
  local pyannote.audio speaker diarization (Python subprocess), local filesystem storage, ffmpeg/
  ffprobe processing — all in one single-tenant, local-first audio-processing adopter with no
  auth/database/messaging/payments/push/maps/analytics integrations of its own.
- **Native-mobile-only**: Firebase Crashlytics, Google Play Core in-app updates, and a
  static-PDF-only Firebase Hosting deployment appear together in one native-mobile adopter whose
  source wasn't available locally during this catalog's last refresh — those three playbooks are
  synthesized from general provider documentation rather than extracted from real code, and marked
  `Playbook confidence: low`. A separate native-mobile adopter contributes on-device calendar sync
  mirrored into a cloud database, and QR code generation for invite codes.
- **Calendar deep-links (no-auth)**: `.ics` generation plus Google/Outlook calendar deep-links,
  distinct from the OAuth-based Google Calendar sync integration elsewhere in the catalog.
- **Direct browser Notification API**: foreground-only alerts, complementary to (not a replacement
  for) the FCM background-push integration in the same adopter.
- **Unimplemented/spec-only**: a designed-but-not-yet-built API for future generic third-party
  data-source push integrations.
