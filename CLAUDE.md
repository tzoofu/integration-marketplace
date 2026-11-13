# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A documentation-only catalog of third-party/external integrations used across the owner's private repos, written as
**standalone implementation playbooks**. There is no code, build, lint, or test step — all work is editing Markdown
under `catalog/`.

Each playbook must be usable by a fresh session on an unrelated project with zero access to the source repos: real
setup steps, a generic sanitized code pattern, env var names, gotchas.

## Hard rules (the catalog is public)

- **Never write a repo name, project name, path, codename ("Repo A"), cloud project ID, or business-identifying
  naming** into any file in this repo. Adoption is a count only; variants are described by technical shape
  (e.g. "the native-mobile variant skips VAPID").
- **Never record an adopter's security posture**: no open/undeployed rules, missing auth checks, reused secrets,
  past incidents, even anonymously. Turn findings into generic forward-looking Gotchas and tell the user about
  the real issue in chat.
- Env vars: names only, never values. Credential files: note existence/purpose only.
- Repo-specific or sensitive material (which repo uses what, project IDs, security findings, integrations too
  identifying to publish) belongs in the private companion repo **`../pim`** (its own `CLAUDE.md`, `SCHEMA.md`,
  `playbooks/`, `security/`). Nothing from `../pim` is ever copied back here. Moving a playbook to PIM means
  deleting it here *and* removing its row from `catalog/README.md`.

## Structure

- `catalog/SCHEMA.md` — the authoritative file shape (header bullets: category/provider/reusable/docs; `## Playbook`
  with Prerequisites, Setup steps, Core pattern, Env vars, Gotchas, Playbook confidence; `## Adoption`), the
  canonical category list, and the bump-vs-new-file rule. Read it before creating or editing a playbook.
- `catalog/<category>/<integration-slug>.md` — one file per *provider + purpose* (same provider doing different
  things → separate files). Categories are fixed; only add one when nothing in SCHEMA.md fits.
- `catalog/<category>/assets/` — optional copy-verbatim files linked from a playbook (e.g.
  `catalog/other/assets/design-tokens.css`); same no-identifiers rule.
- `catalog/README.md` — the index. Must stay in sync with the files: one table row per integration per category
  (with adopter count + reusable), plus "Shared patterns across repos" (every file with 2+ adopters) and
  "Single-adopter integrations". Any add/remove/rename/count change to a playbook requires updating it.

## Workflow

- Mapping a new repo (or re-mapping one) is done via the `map-integrations` skill
  (`.claude/skills/map-integrations/SKILL.md`): delegate a read-only scan of the target repo, then for each
  integration either bump the matching file's Adoption count (and the README row) or create a new playbook from
  the repo's real source — never invented. Never edit the repo being mapped.
- Implementing a playbook in another repo is done via the `implement-integration` skill
  (`.claude/skills/implement-integration/SKILL.md`): it reads `catalog/` and writes only into the target repo,
  never back into the catalog.
- Only touch an existing `## Playbook` section when a new adopter reveals a meaningfully different variant or gotcha.
  A second adopter usually updates the `reusable` line to a concrete shared-package statement.
- `Playbook confidence`: `high` = extracted from real local source; `medium` = single implementation, not
  cross-checked; `low` = synthesized from provider docs (say so in Gotchas).
