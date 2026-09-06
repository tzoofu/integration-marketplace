# Integration Marketplace

A catalog of every external/third-party integration used across my personal repos — organized as
**standalone implementation playbooks**, not just a usage map. Two goals:

1. Spot common patterns (auth, push notifications, analytics, maps, admin-approval workflows)
   across repos so they can eventually be shared instead of re-built per repo.
2. Be droppable into a **fresh Claude/Cursor session working on a different project**, with no
   access to any of the source repos — each integration file has enough (setup steps, a generic
   sanitized code pattern, gotchas) to implement the same integration from scratch.

**No repo names, project names, or business-identifying details are recorded anywhere in this
catalog** — not even as anonymous codenames. Every file describes an integration pattern and how
many repos in this marketplace adopt it, never which ones. This is deliberate: the catalog must
stay safe to hand to an unrelated project or session without linking back to (or identifying) the
private repos it was sourced from.

## Structure

The catalog is organized **per integration, not per repo**, in **category subfolders**:
`catalog/<category>/<integration-slug>.md` (e.g. `catalog/payments/bit-payment.md`). Grab just one
category folder to hand a new project only the relevant slice.

Each file has two parts — see `catalog/SCHEMA.md` for the exact shape:
- **`## Playbook`** — prerequisites, setup steps, a generic sanitized code pattern, env var names,
  gotchas. Self-contained; assumes zero repo context.
- **`## Adoption`** — how many repos in this marketplace use it, and a repo-agnostic note on any
  meaningfully different implementation variants — never which repos, by name or codename.

- `catalog/SCHEMA.md` — file shape and canonical category list. Read this first.
- `catalog/<category>/<integration-slug>.md` — one file per integration.
- `catalog/README.md` — index grouped by category + "shared patterns across repos" reuse
  candidates.
- `.claude/skills/map-integrations/` — the skill that (re)scans a repo and folds its findings into
  the per-integration playbooks, without ever recording that repo's identity.

See [catalog/README.md](catalog/README.md) for the full integration-by-integration index and reuse
candidates (Firebase Auth, FCM push, GA4, reCAPTCHA Enterprise, and WhatsApp deep-links all show up
in multiple repos with near-identical implementations).

## Adding another repo

Use the `map-integrations` skill (`.claude/skills/map-integrations/SKILL.md`) — point it at a repo
path (add it as an additional working directory first, or just give an absolute path) and it will
scan for integrations, bump the adoption count on each matching `catalog/<category>/<slug>.md` (or
create a new one), and update the index — without ever writing that repo's name or path into the
catalog.
