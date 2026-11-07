# Super-Admin Team View Mode

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — a cookie-gated, session-scoped "read-only preview" pattern that generalizes to any multi-tenant app where an elevated role needs to inspect another tenant's view without joining it or risking accidental writes.
- **docs**: n/a — internal pattern, no external provider.

An elevated-role-only mode that lets a super admin browse any other team's dashboard/detail pages read-only, without becoming a member, via a short-lived preview cookie that every write path deliberately ignores.

## Playbook

### Prerequisites
- A role signal distinguishing an elevated "can preview any tenant" role from ordinary tenant membership (the source keeps admin and super-admin as two independent booleans — one gates moderation actions, the other gates cross-tenant preview).
- A single per-tenant read-path resolver function ("which tenant is this request for") that every read page/route already calls, so a cookie check can be inserted there once instead of duplicated per page.

### Setup steps
1. Add a dedicated preview cookie, distinct from the durable "my active tenant" cookie, storing only the previewed tenant's id — make it a **session cookie with no `maxAge`**. A curiosity click into another tenant should not persist past the browser session.
2. Write one `getDashboardTenant(user)` resolver that: checks the elevated-role flag, then the preview cookie, then falls back to the user's own active tenant. Every read page/route calls this — never a page-local cookie read, or some surfaces will silently ignore preview mode.
3. Gate **setting** the cookie behind the elevated-role check server-side, in a dedicated route, re-validating the target tenant still exists and isn't disabled.
4. Gate **clearing** the cookie behind no special role (any signed-in user can exit preview), and also clear it implicitly whenever the user switches to one of their own real tenants — an explicit exit action alone leaves a stuck-in-preview edge case after normal navigation.
5. **Never let any write path resolve the acting tenant from the preview cookie.** Every mutation route must resolve the tenant through the ordinary membership-checked resolver, so even a tampered client request can't write into a previewed tenant. This is the real security boundary — not step 6.
6. Thread a single `viewingOtherTenant: boolean` out of the resolver into every mutable UI affordance (status changes, voting/consensus actions, notes/ratings, checklist toggles, scheduling, bulk actions, persisted filter writes) and disable each one locally. This is a UX convenience on top of step 5, not a substitute for it.
7. Render a persistent "preview mode" banner with an explicit exit action wherever `viewingOtherTenant` is true. Force any participation-gated content reveal (e.g. "see teammates' input once you've contributed yourself") to always fully reveal in this mode — a read-only previewer can never satisfy a participation gate, so leaving it enabled would just show them permanently incomplete data.

### Core pattern
```ts
// tenant-view-mode.ts
export const VIEW_TENANT_COOKIE = "admin-view-tenant"; // session cookie, no maxAge

export async function getDashboardTenant(user: { email: string; isElevatedRole: boolean }) {
  const viewedId = user.isElevatedRole ? readCookie(VIEW_TENANT_COOKIE) : null;
  if (viewedId) {
    const tenant = await getTenant(viewedId);
    if (tenant && !tenant.disabled) {
      return { tenant, viewingOtherTenant: !isMember(tenant, user.email) };
    }
  }
  const tenant = await getActiveTenant(user.email);
  return { tenant, viewingOtherTenant: false };
}

// POST /api/tenants/view   — elevated-role gated; validates target tenant, sets the cookie
// DELETE /api/tenants/view — any authed user; clears the cookie
// POST /api/tenants/active — switching to a real tenant also clears the cookie implicitly
//
// Every write route resolves the acting tenant via resolveActiveTenant(email)
// (membership-checked) and NEVER reads VIEW_TENANT_COOKIE — preview state
// cannot influence what a request is allowed to mutate.
```

### Env vars
none — purely internal cookie/routing pattern, no external config.

### Gotchas
- Making the preview cookie durable (a real `maxAge`) is the single biggest risk — it would silently keep rendering someone else's tenant days or weeks after one click. Keep it session-scoped.
- Disabling buttons client-side is a UX nicety, not a security control. If any write route ever resolves its acting tenant from the same cookie the read path uses, a previewer could write into a tenant they don't belong to — write paths must go through a separate, membership-checked resolver, full stop.
- Any content-reveal gate that normally unlocks on participation must be forced fully-open in preview mode — otherwise a read-only previewer sees a permanently "locked" view they can never unlock.
- Clearing the cookie implicitly on "switch to one of my own tenants" (in addition to an explicit exit action) avoids a confusing stuck-in-preview state after ordinary navigation.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter lets a super-admin-only role preview any other team's dashboard read-only via a session-scoped cookie, disabling every team-scoped write affordance client-side while the real write routes stay membership-gated server-side regardless of the cookie's state.
