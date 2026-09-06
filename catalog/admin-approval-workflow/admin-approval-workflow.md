# Admin Approval / User-Management Workflow

- **category**: admin-approval-workflow
- **provider**: internal (not third-party in any repo) — built on Firebase/Firestore
- **reusable**: partial — structurally similar (Firestore-backed state + role-check enforcement) across repos even though the domain objects differ; worth a shared *pattern* doc rather than shared code. The pending/approved/rejected user-gating shape generalizes most cleanly; one native-mobile adopter shows the same shape can be enforced purely in security rules with no server API layer at all, the same way a couple of web adopters do.
- **docs**: https://firebase.google.com/docs/firestore/security/get-started (this is an internal pattern, not a vendor product — the closest thing to "official docs" is Firestore's own security-rules guide, since every variant below ultimately rests on it)

## Overview
An admin-gated state machine for moderating who/what enters a system — new-user sign-up approval, order/task lifecycle transitions, or content moderation — backed by a status/role field on a Firestore document. Reach for this whenever you need "someone must review and approve/reject before X becomes active," without pulling in a dedicated workflow-engine dependency.

## Playbook

### Prerequisites
- A Firebase project with Firestore already provisioned (see the `firestore` and `firebase-admin-sdk` marketplace entries for the underlying setup).
- A way to identify "is this caller an admin" — either a hand-maintained allowlist doc (`config/auth`) or a `role`/`isAdmin` field on the user's own profile doc.
- If using the server-API variant: a session/auth layer that produces a verified caller identity server-side (see the `firebase-auth` marketplace entry).

### Setup steps
1. Add a `status` (or `role`/`banned`) field to the resource's Firestore document, with an explicit enum of states (e.g. `pending` / `approved` / `rejected`, or a linear lifecycle like `new → confirmed → preparing → ready → completed`).
2. Decide where enforcement lives — pick one or combine for defense-in-depth:
   - **Server API + admin guard**: a Next.js API route wrapped in a `withAdmin`/`requireAdmin` helper that re-checks the caller's admin status server-side before mutating the status field. Use this when you also want side effects (notifications, audit logs, cascading writes) on transition.
   - **Firestore rules only**: no server route at all — the client calls `updateDoc`/`setDoc` directly, and `firestore.rules` is the sole gate. Use this for simpler apps that don't need side effects and want one less deploy target.
3. Seed the first admin manually (a Firestore console write or a one-off script) — every repo observed here treats "who is the first admin" as a bootstrap problem outside the app's own UI, not something the app can self-service without a chicken-and-egg problem.
4. If any state transition must be irreversible or auditable (e.g. can't demote the last super-admin, every approve/reject logged), enforce that invariant in the same place as the rest of the gating — a Firestore rule (`resource.data.role in [...]`) or an audit-log write parallel to the state change, not as a UI-only check.
5. Wire the admin UI (a table/list of pending items with approve/reject/revoke buttons) to call whichever of the two mutation paths you chose.

### Core pattern

**Variant A — server API route + admin-role guard**:
```ts
// app/api/admin/approvals/route.ts
import { NextRequest, NextResponse } from "next/server";
import { withAdmin, badRequest } from "@/lib/api-handlers";
import { approveResource, rejectResource, revokeResource } from "@/lib/approval";
import { getPendingResources } from "@/lib/db";

const ACTIONS = ["approve", "reject", "revoke"] as const;
type Action = (typeof ACTIONS)[number];

export const GET = withAdmin(async () => {
  const items = await getPendingResources();
  return NextResponse.json({ items });
});

export const POST = withAdmin(async (req: NextRequest) => {
  const { id, action } = await req.json();
  if (!id || !ACTIONS.includes(action as Action)) {
    return badRequest(`id and action (${ACTIONS.join("|")}) required`);
  }
  if (action === "approve") await approveResource(id);
  else if (action === "reject") await rejectResource(id);
  else await revokeResource(id);
  return NextResponse.json({ ok: true });
});
```
```ts
// lib/api-handlers.ts (the withAdmin guard, shape shared across repos)
export function withAdmin<T>(handler: (req: NextRequest) => Promise<NextResponse<T>>) {
  return async (req: NextRequest) => {
    const auth = await requireAuth(req);
    if (auth instanceof NextResponse) return auth;
    if (!(await isAdmin(auth.email))) {
      return NextResponse.json({ error: "Forbidden" }, { status: 403 });
    }
    return handler(req);
  };
}
```

**Variant B — pure Firestore-rules enforcement, no server API layer**:
```
// firestore.rules
function myRole() {
  return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
}
function isAdmin() {
  return request.auth != null && myRole() == 'admin';
}

match /resources/{resourceId} {
  allow read: if true;
  // Only an admin may flip the moderation/approval field; regular
  // create/update of the resource's own content is a separate rule.
  allow update: if isAdmin()
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['status']);
}

// A user can never grant themselves the admin role — role is settable
// only by an existing admin, and only via a path that already checks isAdmin().
match /users/{uid} {
  allow update: if (request.auth.uid == uid
      && !request.resource.data.diff(resource.data).affectedKeys().hasAny(['role']))
    || isAdmin();
}
```
```ts
// client — no server route, just a direct Firestore write gated by the rule above
await setDoc(doc(db, 'resources', resourceId), { status: 'approved' }, { merge: true });
```

### Env vars
- `ADMIN_EMAIL` / `NEXT_PUBLIC_ADMIN_EMAIL` — used by a couple of repos as a bootstrap single-admin identity; most repos instead store the admin list/role in Firestore itself and need no env var at all ("none additional" is the common case).

### Gotchas
- **Role escalation via full-document `.set()`**: if any client code does a full-document `.set()`/`updateDoc` rather than a field-scoped update, a user can silently overwrite their own `role`/`banned` field back to a privileged value unless the rule explicitly denies touching those keys (`!diff(...).affectedKeys().hasAny(['role','banned'])`). This is a proven real-world failure mode, not a hypothetical — a demoted member re-`.set()`-ing their own person doc back to a privileged role has actually happened.
- **Banned/role fields must be immutable by the subject**: rules can explicitly forbid the `role` key in both `create` and `update` unless the caller is already an admin; going further, stripping admin powers from a banned admin (checking ban status inside the admin-check function itself) ensures a ban can't be self-reverted by someone who was an admin a moment ago.
- **"Approve" as an idempotent no-op, not just a status flip**: an invite/approve handler should check whether the target is already allowed/new before acting, so re-inviting an already-approved user doesn't overwrite invite attribution or double-send a notification.
- **Defense-in-depth vs rules-only is a real architectural choice, not accidental duplication**: an adopter can keep `firestore.rules` as the actual enforcement even though a server API layer also exists, specifically because the Admin SDK bypasses rules entirely — the rules comments on sensitive collections can say as much explicitly ("this block is defense-in-depth/documentation only").
- **Audit trails must survive the subject's own deletion**: an audit-log write path can deliberately not scrub the acting user's ID on account deletion, citing GDPR/legitimate-interest exceptions — don't treat "always cascade-delete on user deletion" as a universal rule for audit collections.
- **List/enumeration must stay denied even when `get` is fine**: an invite-code or membership collection can allow `get`-by-known-ID for any signed-in user (needed for join flows) but must explicitly set `allow list: if false` — a Firestore rule can only permit `list` if it can prove every possible match satisfies the rule, which a "not yet a member" lookup can never satisfy.
- **A staged-but-undeployed ruleset can look live in documentation and not be**: a `firestore.rules` file can carry a header comment stating the generic create/join-flow rules are "Staged but NOT YET DEPLOYED" pending a wide-open temporary rule (with an expiry date), while accompanying project documentation describes the same rules in present tense as if already enforced. Always check a rules file's own header/deploy-log before trusting that what's on disk is what's actually live, especially when a repo's own docs have gotten out of sync with deployed reality before.
- **Secrets that must never appear on a document a member can `read`**: a join-password hash and each joiner's one-shot "attempt ticket" belong in a separate `secrets/{secretId}` subcollection specifically excluded from the generic member-readable wildcard match — putting a password hash directly on a freely-`get`-able parent document would permanently leak it even though it's "just a hash."
- **Bootstrap mode for the first admin**: an admin-role check can treat "no superAdmins configured yet" as "every admin is also a superAdmin" — a deliberate bootstrap escape hatch, not a bug, but one that silently stops applying the moment the first `superAdmins` entry is written.

### Playbook confidence: high
(extracted from real, locally-available source code across 7 repos)

## Adoption
Used in **7** repo(s) in this marketplace. Most adopters enforce approval/role transitions via a server API route wrapped in an admin-role guard, gating things like sign-up approval queues, order/task lifecycle transitions, and content-moderation flows. A minority enforce the same pending/approved/rejected or role-gating shape purely through Firestore (and, for one, Storage) security rules with no server API layer, including one native-mobile adopter whose confirmation is medium-confidence — its documentation describes rules in present tense that its own rules file marks as staged but not yet deployed.
