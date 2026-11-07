---
name: audit-catalog-leaks
description: Check this public catalog (working tree, staged changes, unpushed commits and full git history) for anything that identifies a source repo or exposes a system's security posture, using the private ../pim index as the list of identifiers. Use when the user asks "did we expose anything", before pushing, and as the last step of map-integrations.
---

# audit-catalog-leaks

Read-only audit. Findings are reported **in chat only**: never write an identifier, a matched line,
or a finding into any file in this repo (it is public). If something must be recorded, it goes in
`../pim/security/findings.md`.

## 1. Build the identifier list from PIM (in memory, never in a file here)

If `../pim` doesn't exist, say so and fall back to step 3's posture grep plus a manual review.

Collect from `../pim`:
- repo names and remote owners/paths from `repos.md` (every row, including work repos)
- project IDs, project names, domains and `prj_…` IDs from `infra/*.md`
- distinctive names from `repos/*.md` (product/business names, function names, people, vendors)
- file and function names cited in `security/findings.md`

Put them in a shell variable for an `-E` regex (escape dots). Drop terms that are ordinary English
or generic tech words (`priority`, `orders`) — they only produce noise — and keep them in mind as
a manual check instead. Add the owner's name and email handle.

## 2. Grep for identifiers

```bash
git grep -nIiE "$T" -- . ':!.remember'                                     # working tree
git diff --cached | grep -iE "$T"                                           # staged
git log -p --all -i -G"$T" --pretty=format:'@@COMMIT %h %s' \
  | grep -iE "^@@COMMIT|^[+-].*($T)"                                        # all history
```

Triage each hit: a placeholder (`<project-id>`, `prj_xxxx`) or generic word is a false positive;
anything else is a leak.

## 3. Grep for security posture

Statements about an adopter's live state are leaks even with no name attached. In the tree and in
**removed** history lines (`git log --all -p | grep '^-'`), look for phrases like:

`not (yet )?deployed|not yet implemented|no adopter|one adopter|some adopters|a minority|wide open|if true|no auth|without auth|anyone can|reuse[sd]? .*secret|hardcod|in production|actually happened|incident`

A forward-looking Gotcha ("verify deployed rules match the repo copy") is fine; a statement about
what an adopter currently does or lacks is not.

## 4. Check what is public

- `gh repo view --json visibility` and `git log origin/main..main --oneline`: history that is
  already pushed is public; unpushed commits and the working tree can still be fixed cheaply.
- `git ls-files` outside `catalog/`: make sure nothing private (`.remember/`, `.env*`, notes) is
  tracked.

## 5. Report

- **Working tree / unpushed**: each leak with `file:line`, and the fix (reword generically, or move
  the playbook to `../pim/playbooks/` and drop its README row).
- **Pushed history**: each leak with commit, file, what it reveals, and severity. Recommend
  checking whether the underlying issue is still open on the real system before considering a
  history rewrite (force-push; forks and caches may keep the old commits anyway). Rewriting history
  is the user's call — never do it unasked.
- **Clean**: say exactly what was checked.

Offer to log new history leaks in `../pim/security/findings.md`.
