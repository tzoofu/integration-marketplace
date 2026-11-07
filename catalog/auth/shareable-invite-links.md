# Shareable invite links (hashed token, multi-use, auto-approve)

- **category**: auth
- **provider**: internal (custom)
- **reusable**: yes. Storage, redemption transaction and policy checks are framework-agnostic.
- **docs**: n/a

A signed-in member mints a link they can drop into a chat or group. Whoever opens it and signs in is admitted without waiting for manual approval: a "team" link joins them to a team, an "app" link only lets them into the app. Links expire, have a use cap, can be revoked, and die with their creator's standing.

## Playbook

### Prerequisites
- A sign-in flow (any provider) and an allowlist/approval gate you want to bypass for invitees.
- A datastore with transactions, and a team-membership model if you have "team" links.

### Setup steps
1. **Mint:** `token = randomBytes(24).toString("base64url")`. Store the doc under `sha256(token)`; the raw token exists only in the URL. Fields: `kind` (`team|app`), `teamId?`, `createdBy`, `expiresAt` (7 days), `uses`, `maxUses` (cap, e.g. 25), `redeemedBy: string[]`, `revokedAt?`.
2. **Public landing route** `/invite/<token>` (exempt from the auth redirect). It remembers the token (query/localStorage) and starts sign-in.
3. **Redeem from two entry points through ONE function:** sign-in route (signed-out invitee, runs before the access gate) and a signed-in redeem route.
4. Redeem order: look up by hash, reject if the user was explicitly rejected, verify the creator still has standing, consume in a transaction, then apply effects.
5. Notify admins only when something changed (new to app or newly joined a team).

### Core pattern
```ts
export const hashToken = (t: string) => createHash("sha256").update(t).digest("hex");

// Pure policy, easy to unit test
export function unusableReason(i, now: Date) {
  if (i.revokedAt) return "revoked";
  if (now.getTime() >= new Date(i.expiresAt).getTime()) return "expired";
  if (i.uses >= i.maxUses) return "exhausted";
  return null;
}

// Atomic, idempotent per user
export async function consumeInvite(token: string, email: string) {
  const ref = db.collection("invites").doc(hashToken(token));
  return db.runTransaction(async (t) => {
    const doc = await t.get(ref);
    if (!doc.exists) return { ok: false, reason: "not_found" } as const;
    const inv = doc.data()!;
    if (inv.redeemedBy?.includes(email))                       // retry or re-open by the same person
      return inv.revokedAt ? ({ ok: false, reason: "revoked" } as const) : ({ ok: true, inv, firstUse: false } as const);
    const reason = unusableReason(inv, new Date());
    if (reason) return { ok: false, reason } as const;
    t.update(ref, { uses: FieldValue.increment(1), redeemedBy: FieldValue.arrayUnion(email) });
    return { ok: true, inv, firstUse: true } as const;
  });
}

export async function redeemInvite(token: string, rawEmail: string) {
  const email = normalize(rawEmail);
  const invite = await getByToken(token);
  if (!invite) return { ok: false, reason: "not_found" };
  if (await isRejected(email)) return { ok: false, reason: "rejected" };      // admin rejection always wins
  const team = invite.kind === "team" ? await getTeam(invite.teamId) : null;
  const creatorOk = team ? team && !team.disabled && isMember(team, invite.createdBy) : await isAllowed(invite.createdBy);
  if (!creatorOk) return { ok: false, reason: "inviter_lost_access" };        // link dies with its creator's standing
  const consumed = await consumeInvite(token, email);
  if (!consumed.ok) return consumed;
  if (team) await addMember(team, email, { by: invite.createdBy, via: "link" }); else await allowUser(email, invite.createdBy, "link");
  return { ok: true, kind: invite.kind, redirect: team ? "/dashboard" : "/onboarding" };
}
```

### Env vars
None beyond your auth provider's.

### Gotchas
- Store only the hash; a database leak must not yield usable links.
- Make redemption **idempotent per email** inside the transaction, so a retried sign-in or a re-opened link does not burn another use, and a revoked link still blocks even a prior redeemer.
- A use cap bounds the damage of a link leaking into a public group; overflow users fall back to the normal manual-approval queue.
- Tie validity to the creator, not just the clock: if they leave the team or lose access, their links stop working.
- Check explicit rejection before anything else.
- Run the signed-out redemption path before the access gate, otherwise the invitee is bounced to "pending" first.
- Keep pure policy (`unusableReason`, TTL) in a module with no Node or datastore imports so client code and tests can use it.
- Notifying admins on every open is noise; do it only on first use that changed state.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. Two link kinds (join a team, or app access only) share one redemption function reachable from both the sign-in route and a signed-in route.
