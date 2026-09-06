# Catalog entry schema

The catalog is organized **per integration, not per repo**, and files live in **category
subfolders**: `catalog/<category>/<integration-slug>.md`. Each distinct integration (a provider +
what it's used for) gets one file.

**No repo names, project names, or business-identifying details are ever recorded anywhere in this
catalog.** The whole point is a portable reference that can be dropped into an unrelated project
or session without linking back to (or identifying) the private repos it was sourced from. Track
*adoption* (how many repos, and how implementations differ) without ever naming or labeling which
repo is which — not even with anonymous codenames like "Repo A". If a distinction matters for
correctness (e.g. "the native-mobile implementation skips VAPID"), describe it by its technical
shape, never by a repo identifier.

Two integrations that share a provider but do genuinely different things get separate files (e.g.
`catalog/payments/bit-payment.md` and `catalog/payments/paybox-payment.md`, not one
`payments.md`).

**Every file must be a self-contained implementation playbook** — written so that a Claude/Cursor
session with *zero access to the original source repos* could implement the same integration from
this file alone: real setup steps, a generic sanitized code pattern, gotchas.

## File shape

```markdown
# <Integration name>

- **category**: auth | database | messaging | scraping-source | maps-geo | ai-llm | email-sms | analytics | hosting-deploy | payments | push-notifications | admin-approval-workflow | storage | other
- **provider**: <company/product, e.g. Firebase, Google, Yad2, Bit — or "internal (custom)">
- **reusable**: yes | no | partial — could another repo in this marketplace reuse this as a shared package, or is it too app/region-specific? One line of justification.
- **docs**: <link to the official provider/API documentation, or "n/a" for purely internal patterns>

<1-3 sentence Overview: what this integration does, when you'd reach for it>

## Playbook

### Prerequisites
<accounts, SDKs, packages needed before starting>

### Setup steps
<numbered, concrete — enough that someone unfamiliar with any of the source repos could follow along>

### Core pattern
<a GENERIC, SANITIZED, copy-paste-ready code/config snippet in the real language/framework used.
Base it on the actual pattern observed in source, stripped of app-specific naming, business logic,
and branding — it should read like a reusable starter, not any one app's real code. If different
adopters implement it meaningfully differently (e.g. server-session-JWT vs pure-client-SDK), show
each variant briefly, labeled by its technical shape (e.g. "web/server variant", "native-mobile
variant") — never by which repo it came from.>

### Env vars
<names only, never values — group by variant if relevant>

### Gotchas
<non-obvious lessons pulled from the real source: edge cases handled, workarounds, things that
would surprise someone implementing this fresh — described generically, no repo-identifying
specifics (real component/file names, business-object names, etc.)>

### Playbook confidence: high | medium | low
<"high" when extracted from real, locally-available source; "medium" for a freshly-discovered
pattern not yet cross-checked against a second implementation; "low" when no source was available
locally and this was synthesized from general provider knowledge instead — say so explicitly in
Gotchas too, still without naming which repo/project was unavailable>

## Adoption
Used in **N** repo(s) in this marketplace. <0-3 sentences summarizing notable variation across
those repos, in purely technical terms — no repo names, codenames, file paths, or
business/domain-specific naming.>
```

## Categories (canonical list — extend only when nothing above fits)

| category | folder | meaning |
|---|---|---|
| `auth` | `catalog/auth/` | sign-in / identity (OAuth, Firebase Auth, NextAuth, SSO) |
| `database` | `catalog/database/` | persistence (Firestore, Postgres, Redis, etc.) |
| `messaging` | `catalog/messaging/` | chat/messaging platforms (Beeper, WhatsApp, Slack) |
| `scraping-source` | `catalog/scraping-source/` | external site scraped for data (Yad2, Facebook Groups) |
| `maps-geo` | `catalog/maps-geo/` | maps, geocoding, polygons |
| `ai-llm` | `catalog/ai-llm/` | LLM/AI provider usage |
| `email-sms` | `catalog/email-sms/` | transactional email or SMS providers |
| `analytics` | `catalog/analytics/` | product/usage analytics |
| `hosting-deploy` | `catalog/hosting-deploy/` | deploy target/platform config (Vercel, Firebase Hosting) |
| `payments` | `catalog/payments/` | payment/checkout processors |
| `push-notifications` | `catalog/push-notifications/` | web/mobile push (web-push, FCM) |
| `admin-approval-workflow` | `catalog/admin-approval-workflow/` | internal approval/role/status workflow tied to admin actions |
| `storage` | `catalog/storage/` | file/blob storage |
| `other` | `catalog/other/` | anything not covered above |

## Top-level index

`catalog/README.md` keeps one row per integration in a single sortable table grouped by category
(linking into the subfolders), with an adoption **count** (never repo names) per row, plus a
"shared patterns across repos" section highlighting any integration file with 2+ adopters — those
are the marketplace-reuse candidates.

## Adding a new repo

When mapping a new repo (see the `map-integrations` skill):
1. For each integration found, check whether a matching `catalog/<category>/<integration-slug>.md`
   already exists (same provider + same purpose-shape).
2. If it exists, **increment its "Adoption" count** and fold in a one-line generic note if this
   repo's implementation reveals a genuinely different variant or gotcha — never add a repo name,
   codename, or any identifying label. Update the top-level `reusable` line if a second adopter now
   shares the pattern. Only touch the `## Playbook` section itself if the new implementation
   reveals something meaningfully different worth folding into the Core pattern/Gotchas.
3. If it doesn't exist, create a new `catalog/<category>/<integration-slug>.md` following the shape
   above — this means **actually reading the repo's real implementation** (not just noting file
   paths) so the `## Playbook` section is genuinely actionable, not another shallow mapping entry.
   The new file's "Adoption" section says "Used in 1 repo" — nothing more identifying than that.
4. Update `catalog/README.md`'s index table and "shared patterns" section. Never write the repo's
   name, path, or any business-identifying detail into any catalog file.
