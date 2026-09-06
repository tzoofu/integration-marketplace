---
name: map-integrations
description: Scan a sibling repo for third-party/external integrations (auth, database, messaging, scraping sources, payments, push, analytics, etc.) and fold its findings into this marketplace's per-integration catalog/. Use when the user adds a new repo to map, or asks to (re)map an existing one.
---

# map-integrations

Builds and maintains the integration catalog in this repo (`integration-marketplace`). The catalog is organized **per integration, not per repo**, in category subfolders (`catalog/<category>/<integration-slug>.md`): one call = one repo scanned, and each integration found either bumps an existing file's "Adoption" count (if another repo already has that integration cataloged) or creates a brand-new file (if it's the first repo to have it).

**No repo names, project names, paths, or business-identifying details are ever written into any catalog file — not even as an anonymous codename.** The catalog must stay fully portable and never reveal which private repos it was sourced from. Track adoption as a count and describe variants in purely technical terms (e.g. "the native-mobile implementation skips VAPID"), never by which repo they came from.

Read `catalog/SCHEMA.md` first — it defines the file shape, the canonical category list, and exactly how to decide "bump existing file" vs "new file". Follow it exactly; only add a new category if truly nothing existing fits.

**Every catalog file is a self-contained implementation playbook**, not just a usage map — it must read as actionable to someone with zero access to the source repos: real setup steps and a generic sanitized code pattern, not just "see `lib/foo.ts`". This means creating a new file requires actually reading the relevant source code, not just noting that a file exists.

## Input

The user gives a path to a repo (usually one of the additional working directories, e.g. `../some-repo`). If they just say "map `<name>`" and it's already an additional working directory, use that path directly — don't ask for it again. The repo's name/path is used only to locate its source during this scan; it must never be written into any catalog file.

## Process

1. **Delegate the scan.** Launch a read-only investigation (Explore agent, or a fresh general-purpose agent if Explore isn't available) against the target repo — do not read the whole repo into your own context. Give it this checklist:
   - `package.json` — every third-party SDK dependency (firebase, googleapis, stripe, twilio, sendgrid, openai, anthropic, web-push, playwright, etc.) is a candidate integration.
   - `.env.example` / `.env.local.example` (or `.env.local` if that's all that exists) — list env var **names only**, never values. Env vars often reveal integrations no code path makes obvious yet.
   - Top-level config: `firebase.json`, `firestore.rules`, `vercel.json`, `.firebaserc`, any `adminsdk*.json` (note existence/purpose only, never contents — these are credential files).
   - `lib/`, `src/`, `app/api/**`, `hooks/`, `scripts/` — client init files (`*firebase*.ts`, `*-client.ts`) and API routes that call external hosts.
   - `.claude/skills/` and `.claude/commands/` — skill/command names often name an integration directly (a scraper skill implies a scraping-source integration, a Beeper-ingest skill implies a messaging integration).
   - Root-level docs (`README.md`, `CLAUDE.md`, any `*_SETUP.md`, `ANALYTICS.md`-style files) — skim for provider names, "webhook", "OAuth", "approval".
   - Grep for "approval", "admin", "role", "permission", "status" to catch internal admin/approval workflows that aren't a package dependency at all.

   Have it report back structured entries (category, provider, purpose, key files, env vars, confidence), plus a 2-3 sentence architecture overview — but tell it explicitly: never include secret values (only key names), and never carry the repo's own name, path, or any business-specific object/entity naming into its report any more than needed to write a generic playbook. For anything that looks like a **brand-new** integration (no existing catalog file will match), also have it read the actual key files' contents — imports, function shapes, config — not just note their paths, so a real `## Playbook` section can be written.

2. **Fold each finding into the per-integration catalog**, per `catalog/SCHEMA.md`:
   - For each integration the agent found, look for an existing `catalog/<category>/<integration-slug>.md` whose provider + purpose-shape matches (e.g. a new repo's "Firebase Auth" matches the existing `catalog/auth/firebase-auth.md`, not a new file).
   - **Match found**: increment that file's "Adoption" count and fold in a one-line *generic* note only if this repo's implementation reveals a genuinely different variant or gotcha worth capturing — described by technical shape, never by naming or labeling the repo. Update its top-level `reusable` line if a second adopter now shares the pattern — a second adopter usually flips `reusable` from "yes, single-repo" reasoning to a concrete "candidate for `@marketplace/<name>` package" statement. Only touch the existing `## Playbook` section if this implementation reveals something meaningfully different — don't rewrite it wholesale for a routine match.
   - **No match**: create a new `catalog/<category>/<integration-slug>.md` following the file shape in `catalog/SCHEMA.md`, with a real `## Playbook` section (setup steps + a generic sanitized code pattern derived from what the scan actually found — never invented) and an `## Adoption` section that just says "Used in 1 repo in this marketplace."
   - Two integrations from the same provider that do genuinely different things (e.g. two unrelated payment deep-links) still get separate files — don't force a merge just because the provider matches.

3. **Update `catalog/README.md`** (the cross-repo index):
   - Bump the adoption **count** for every integration row this repo now contributes to (never add a name or label); add new rows for brand-new integration files.
   - Recompute "Shared patterns across repos" — any integration file that now has 2+ adopters belongs there, with a one-line generic note on what's identical vs what differs between implementations (no repo names/labels).
   - Recompute "Single-adopter only" for anything still confined to one repo.

4. **Report back concisely**: which integrations were found (just names + categories), which existing catalog files got bumped vs which are brand new, and which look reusable now. Don't dump the full catalog into the chat — point at the files.

## Notes

- This is read-only against the target repo — never edit files in the repo being mapped.
- Re-mapping an already-cataloged repo: re-run the scan (things drift), diff mentally against what's already reflected in `catalog/**/*.md` for a matching integration, and update the Playbook/Adoption content in place — never introduce a repo-identifying label to track "which one changed."
- If the target repo isn't yet an additional working directory, that's fine as long as its absolute path is readable — just pass the absolute path to the investigation agent.
