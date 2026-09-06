# Notification Dispatch

- **category**: push-notifications
- **provider**: internal (custom) — built on top of Firebase Cloud Messaging (see [fcm-push](fcm-push.md)), not re-described here
- **reusable**: partial — the dead-token cleanup and multi-strategy dispatch shape (broadcast / team / filter-matched) are portable; the specific filter-matching logic is domain-specific to whatever entity the notifications are about.
- **docs**: https://firebase.google.com/docs/cloud-messaging/send-message#send-messages-to-multiple-devices

## Overview
Custom targeting/fan-out logic layered on top of FCM that decides *which* users get a push for a given event, with cross-team dedup and dead-token cleanup. Distinct from `fcm-push.md`, which only covers token registration and raw transport — this is the "who" layer above the "how" layer.

## Playbook

### Prerequisites
- A working FCM send path already in place (`fcm-push.md`) — this pattern wraps it, it doesn't replace it.
- A way to look up: all registered tokens by user, group/team membership, and (if you want filter-matched targeting) whatever per-group preference/filter data drives "should this group be notified about this event."

### Setup steps
1. Build one low-level `dispatch(tokens, tokenRecords, notification)` function that is the **only** place `sendEachForMulticast` gets called, and the **only** place dead-token cleanup happens. Every higher-level strategy funnels through it so cleanup logic can never drift between strategies.
2. On top of `dispatch`, add one function per targeting strategy your product needs, e.g.:
   - **Broadcast** — every registered token, unconditionally.
   - **Group/team notify** — every member of a group *except* the actor who triggered the event (so you don't notify someone about their own action).
   - **Filter/criteria-matched** — evaluate each group's saved preferences/filters against the triggering entity, and only notify members of groups whose filter matches.
3. When one event needs to reach multiple groups (e.g. a batch of new items, each possibly matching a different subset of groups), **dedupe tokens globally across groups** — build a single `Set<token>` of "already sent to" as you iterate groups, so a user in two matching groups gets exactly one push, not one per group.
4. Wrap the dispatch call behind a **port/interface** (e.g. `TeamNotifier`) so calling code (an approval workflow, an API route) depends on an abstraction, not directly on FCM — makes it trivial to mock in tests or swap transport later.
5. Never `await` a notification send inline in a critical-path request handler if the send failing shouldn't fail the request — fire-and-forget with `.catch(console.error)` so a flaky push provider never blocks the actual business operation (e.g. approving a listing).

### Core pattern
```ts
import { getMessaging } from "firebase-admin/messaging";

// The ONLY function that talks to FCM and the ONLY place dead tokens get pruned.
async function dispatch(
  tokens: string[],
  tokenRecords: { userId: string; token: string }[],
  notification: { title: string; body: string; url?: string }
): Promise<void> {
  if (tokens.length === 0) return;
  const response = await getMessaging().sendEachForMulticast({
    tokens,
    notification: { title: notification.title, body: notification.body },
    webpush: { fcmOptions: { link: notification.url || "/" } },
  });
  response.responses.forEach((r, i) => {
    const code = r.error?.code;
    if (!r.success && (code === "messaging/registration-token-not-registered" || code === "messaging/invalid-registration-token")) {
      removeToken(tokenRecords[i].userId, tokenRecords[i].token).catch(console.error);
    }
  });
}

// Strategy 1: broadcast to everyone.
export async function notifyEveryone(notification: { title: string; body: string; url?: string }) {
  const tokenRecords = await getAllTokens();
  await dispatch(tokenRecords.map((r) => r.token), tokenRecords, notification);
}

// Strategy 2: notify a group, excluding the actor.
export async function notifyGroup(
  groupMemberIds: string[],
  excludeUserId: string,
  notification: { title: string; body: string; url?: string }
) {
  const tokenRecords = await getAllTokens();
  const memberSet = new Set(groupMemberIds);
  const targets = tokenRecords.filter((r) => memberSet.has(r.userId) && r.userId !== excludeUserId);
  await dispatch(targets.map((r) => r.token), targets, notification);
}

// Strategy 3: filter/criteria-matched fan-out across many groups, with
// cross-group token dedup (a user in 2 matching groups gets ONE push).
export async function notifyMatchingGroups(
  entity: unknown,
  groups: { memberIds: string[]; filter?: unknown }[],
  matches: (entity: unknown, filter: unknown) => boolean,
  notification: { title: string; body: string; url?: string }
) {
  const tokenRecords = await getAllTokens();
  const tokensByUser = new Map<string, string[]>();
  for (const r of tokenRecords) tokensByUser.set(r.userId, [...(tokensByUser.get(r.userId) ?? []), r.token]);

  const sentTokens = new Set<string>();
  for (const group of groups) {
    if (group.filter && !matches(entity, group.filter)) continue;
    const groupTokens: string[] = [];
    for (const userId of group.memberIds) {
      for (const token of tokensByUser.get(userId) ?? []) {
        if (!sentTokens.has(token)) { groupTokens.push(token); sentTokens.add(token); }
      }
    }
    if (groupTokens.length === 0) continue;
    const groupRecords = tokenRecords.filter((r) => groupTokens.includes(r.token));
    await dispatch(groupTokens, groupRecords, notification);
  }
}
```

**Port/adapter, so callers depend on an interface, not FCM directly**
```ts
export interface TeamNotifier {
  notifyTeam(team: Team, excludeUserId: string, notification: Notification): Promise<void>;
  notifyEveryone(notification: Notification): Promise<void>;
}

export const pushTeamNotifier: TeamNotifier = {
  notifyTeam: notifyGroup,
  notifyEveryone,
};
```

**Fire-and-forget from a critical-path handler** (an approval action must succeed even if the push fails)
```ts
await updateEntityStatus(id, "approved");
notifyMatchingGroups(entity, groups, matchesFilter, {
  title: "New item approved",
  body: entity.name,
}).catch(console.error); // deliberately not awaited — push failure must never fail the approval
```

### Env vars
None beyond whatever the underlying FCM transport needs (see `fcm-push.md`).

### Gotchas
- Centralize dead-token pruning in the single low-level `dispatch()` function. Duplicating the "check error code, delete token" logic in every strategy function is how cleanup silently drifts/breaks in one path while working in another.
- Cross-group dedup must build the `Set` of sent tokens incrementally as you iterate groups (not compute all matches up front then dedupe after) if the "one push per user" guarantee needs to hold even when notification *content* differs per group (e.g. per-group counts) — the first group a matching user is found in is the one whose content they get.
- Excluding the acting user from a group notification only works if your dedup key (email/uid) is normalized consistently (case, whitespace) — an unnormalized comparison can either double-notify the actor or fail to exclude them.
- Don't await these sends inline in a request path whose success shouldn't depend on push delivery — swallow with `.catch(console.error)` (or route to your error tracker) so provider flakiness never surfaces as a user-facing failure of the actual operation.
- Filter-matched targeting logic (matching an entity against a group's saved criteria) is inherently domain-specific — it is the one non-portable part of this pattern; only the dispatch/dedup/port shape generalizes.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. The adopter layers three targeting strategies (global broadcast, group notify excluding the acting user, and filter-matched fan-out against saved per-group criteria) over a single low-level dispatch/dedup/stale-token-cleanup function, with cross-group token dedup so a user matching multiple groups gets exactly one push.
