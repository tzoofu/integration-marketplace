# In-App "What's New" Changelog

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — one pure data file + helpers, one per-user "last seen" field, a page, a nav badge and a one-time banner; drops into any signed-in React app with a user profile record. With a web and a native-Android adopter, the shared core is the typed entry list + listing screen; per-user seen tracking is an optional layer.
- **docs**: n/a — internal pattern, no external provider.

A user-facing changelog page that lists every feature, improvement and fix in plain words, newest first, with a red dot on what *this* person hasn't seen yet, a count badge on the nav item and a dismissible "N updates since your last visit" banner. Entries are code (a typed array in the repo), so they ship in the same commit as the change; role-tagged entries are hidden from people who can't use that feature. A test plus a standing agent rule keep it from going stale.

## Playbook

### Prerequisites
- React 18+ app with signed-in users (example: Next.js App Router, client components).
- A per-user profile document the user may update for one field (Firestore doc, DB row, etc.).
- User roles available on the client (e.g. `admin` / `editor` / `member`) if some changes only apply to some roles.
- Optional: an i18n `t()` helper (entries carry text per locale); a unit test runner.

### Setup steps
1. **Design first, then build.** Mock the page in the app's own visual language (date-grouped list, New/Improved/Fixed pills, filter chips, unseen dots, nav badge, mobile list, home-screen banner) and get sign-off before coding.
2. **Create the data file** (`lib/changelog.ts`, no framework imports so server and client share it): `ChangelogEntry { id, date, type, audience?, href?, linkLabel?, title, body }`, `title`/`body` per locale. Newest first. `id` = `YYYY-MM-DD-slug`, never renamed — it is what users' "last seen" points at.
3. **Backfill from git history.** Walk `git log --reverse --format='%ad %h %s' --date=short` and `git show --stat` for the meaningful commits; rewrite each user-visible change as a one-line benefit ("Export your data as CSV"), not the commit subject ("csv util"). Skip CI, refactors, tests. Merge noisy fix-up commits into one entry.
4. **Tag audiences.** `audience: ['admin']` (or any role list) for changes only some roles can see/use — admin settings, internal tools. Untagged = everyone.
5. **Pure helpers**: `visibleChangelog(entries, roles)`, `unseenIds(entries, lastSeenId)`, `groupByDate`, `releaseAgo(date, today)` → i18n key + count, `latestChangelogId()`, `changelogLines()` for AI/agent tools.
6. **Per-user "last seen"**: add `whatsNewSeen?: string | null` to the profile; allow the user to write only that field on their own record (string, bounded length) in your security rules / API.
7. **Expose from the app store**: `whatsNew` (role-filtered), `whatsNewUnseen: Set<string>`, `whatsNewReady` (profile loaded), `markWhatsNewSeen()` (writes the newest id; no-op if already latest or in any impersonation/preview mode).
8. **Page**: header + filter chips with counts, "New since your last visit" divider, then "Earlier"; each release = date column + card of items (pill, unseen dot, title, body, audience chip, "Try it →" link). Use flex-wrap so the date column stacks above the card on phones.
9. **Mark seen without losing the dots**: snapshot the unseen set on first render once the profile is loaded, then mark seen in an effect — the dots stay for this visit and clear next time.
10. **Badges + banner**: nav item with the unseen count, a dot on the mobile "More" tab, and a banner on the home page (shows when unseen > 0; "See what's new" links to the page, "Dismiss" marks seen).
11. **Agent tools (optional)**: a read-only `list_whats_new` tool (in-page WebMCP and/or a remote MCP server) using the same `visibleChangelog` + `unseenIds`.
12. **Enforce it**: unit tests (below) + a line in the repo's agent instructions (`CLAUDE.md`/`AGENTS.md`): *"Before every commit, if the staged diff changes anything a user can see or feel, add a changelog entry in the same commit."*

### Core pattern
```ts
// lib/changelog.ts — pure, shared by client, server and agent tools
export type ChangeType = 'new' | 'improved' | 'fixed'
export type Role = 'admin' | 'editor' | 'member'   // your app's roles
type Locale = 'en' | 'es'                           // your app's locales
type L = Record<Locale, string>

export interface ChangelogEntry {
  id: string            // 'YYYY-MM-DD-slug', stable forever
  date: string          // ISO date shipped
  type: ChangeType
  audience?: Role[]     // omit = everyone
  href?: string
  linkLabel?: L
  title: L
  body: L
}

export const CHANGELOG: ChangelogEntry[] = [
  { id: '2026-01-15-search', date: '2026-01-15', type: 'new', href: '/dashboard',
    title: { en: 'Quick search', es: 'Búsqueda rápida' },
    body: { en: 'Press ⌘K to jump to any page fast.', es: 'Pulsa ⌘K para ir rápido a cualquier página.' } },
  // newest first …
]

export const visibleChangelog = (e: ChangelogEntry[], role: Role) =>
  e.filter((x) => !x.audience || x.audience.includes(role) || role === 'admin')

/** Newer than lastSeenId. No (or unknown) lastSeen → only the newest release day, so new users aren't buried. */
export function unseenIds(entries: ChangelogEntry[], lastSeenId?: string | null): Set<string> {
  if (!entries.length) return new Set()
  const idx = lastSeenId ? CHANGELOG.findIndex((e) => e.id === lastSeenId) : -1 // position in the FULL list
  if (idx === -1) return new Set(entries.filter((e) => e.date === entries[0].date).map((e) => e.id))
  const newer = new Set(CHANGELOG.slice(0, idx).map((e) => e.id))
  return new Set(entries.filter((e) => newer.has(e.id)).map((e) => e.id))
}

export function groupByDate(entries: ChangelogEntry[]) {
  const out: { date: string; entries: ChangelogEntry[] }[] = []
  for (const e of entries) {
    const last = out.at(-1)
    if (last?.date === e.date) last.entries.push(e)
    else out.push({ date: e.date, entries: [e] })
  }
  return out
}

export const latestChangelogId = () => CHANGELOG[0]?.id ?? ''
```

```tsx
// page — snapshot unseen before marking seen (set-state-during-render, not in an effect)
const { whatsNew, whatsNewUnseen, whatsNewReady, markWhatsNewSeen } = useApp()
const [unseen, setUnseen] = useState<Set<string> | null>(null)
if (whatsNewReady && unseen === null) setUnseen(whatsNewUnseen)
const snapshotTaken = unseen !== null
useEffect(() => { if (snapshotTaken) markWhatsNewSeen() }, [snapshotTaken]) // eslint-disable-line react-hooks/exhaustive-deps
const shownUnseen = unseen ?? whatsNewUnseen
```

```
// Firestore rules variant — self may only set a bounded string
function selfWhatsNewSeenAllowed() {
  return !(request.resource.data.diff(resource.data).affectedKeys().hasAny(['whatsNewSeen']))
    || (request.resource.data.whatsNewSeen is string && request.resource.data.whatsNewSeen.size() <= 64);
}
// …and add 'whatsNewSeen' to the self-writable keys allowlist.
```

```ts
// tests/unit/changelog.test.ts — the guard rails
// unique ids ≤ 64 chars · ISO dates sorted newest-first · every locale non-empty
// every href resolves to a real page file
// href on an admin-only page ⇒ audience 'admin'; href on a page members can't open ⇒ has an audience
// each role only gets untagged entries + entries tagged with its own role
// unseenIds: null → newest day only; latest seen → empty; unknown id → newest day
```

### Env vars
None.

### Gotchas
- **Unseen is positional, against the full list.** Compare the last-seen id's index in the *unfiltered* array; an entry hidden by audience still marks the position, so role changes don't resurrect old items.
- **First visit / unknown id → newest release day only.** Otherwise a new user (or one whose last-seen id was removed) sees dozens of dots.
- **Freeze the unseen set before writing "seen".** Marking seen updates the profile subscription, which would wipe the dots mid-visit. Take the snapshot during render (React's "adjust state while rendering" pattern) — `setState` inside the effect trips `react-hooks/set-state-in-effect`.
- **Don't mark seen while impersonating** ("view as" another user) — write against the real signed-in user only, and skip it entirely in preview modes.
- **The security rule ships separately from the app.** If rules/API permissions aren't deployed alongside the code, the "seen" write is rejected silently and the badge/banner never clear — deploy the rules (or migration) first.
- **Audience tags are the leak vector.** An untagged entry about an admin-only page tells every user about a feature they can't (and shouldn't) see. Enforce it with the href-based test, and say "when in doubt, tag it" in the agent rule.
- **Write entries in benefit language, not commit language**, and give each locale its own natural copy (natural phrasing per language, `←` instead of `→` for links in RTL locales).
- **IDs are a contract.** Renaming an id breaks everyone's last-seen pointer (they fall back to "newest day"); fix typos in title/body, never in `id`.
- **Hook vs. rule for enforcement**: a pre-commit hook only covers commits made by whichever tool has it configured; a standing agent instruction + CI test covers both agent and human commits once the test is in CI.
- **Parallel sessions**: when another session is editing the same repo, stage only your own hunks — shared files (store, i18n, rules) will contain both features.

### Playbook confidence: high
Extracted from a real, locally-available implementation (Next.js App Router + Firestore client SDK, two locales incl. RTL), including unit and security-rules tests.

## Adoption
Used in **2** repos in this marketplace. Implemented with a single client-side store exposing the role-filtered list and unseen set, and security rules restricting the self-write to the one field. A native-Android adopter keeps a lighter variant: static per-release entries (version + date string + bullet `string-array` resources) and a screen listing them with the current version marked, with no per-user seen tracking, badge or banner.
