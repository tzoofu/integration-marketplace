# Google Calendar

- **category**: other
- **provider**: Google
- **reusable**: yes — a generic "connect your Google Calendar via a separate OAuth client" + create/update/delete-event sync pattern, portable to any app that needs to mirror app-side scheduled events onto a user's own calendar.
- **docs**: https://developers.google.com/calendar/api/guides/overview and https://developers.google.com/identity/protocols/oauth2/web-server

## Overview
Lets an individual user connect their own Google Calendar — via a dedicated OAuth 2.0 client, separate from any app sign-in OAuth client — so that app-side scheduled events (e.g. appointments/visits) are automatically mirrored as Calendar events with attendee invites. Implements the full authorization-code flow (connect → callback → stored tokens) plus best-effort create/update/delete sync against the Calendar REST API, with automatic access-token refresh.

## Playbook

### Prerequisites
- A Google Cloud project with the Calendar API enabled.
- An OAuth 2.0 **Web application** client (Client ID + Secret) distinct from any other OAuth client used for sign-in — this needs the `calendar.events` scope, sign-in typically doesn't.
- An OAuth consent screen configured for the `calendar.events` scope, with the app's callback URL registered as an authorized redirect URI.
- A server-side token store keyed by user identity (DB table/collection) to persist refresh/access tokens per user.

### Setup steps
1. In Google Cloud Console, create (or reuse) a project, enable the Calendar API, and create an OAuth 2.0 Web application client.
2. Add `https://<your-domain>/api/google-calendar/callback` (or equivalent) as an authorized redirect URI.
3. Configure the OAuth consent screen requesting scope `https://www.googleapis.com/auth/calendar.events`; note that this is a Google "sensitive scope" — until the app passes Google's verification review it stays in "Testing" status, which caps refresh-token lifetime (commonly 7 days), so connected users must periodically reconnect.
4. Store the client ID/secret as server-only env vars.
5. Build a signed, short-lived "state" token (e.g. JWT) binding the OAuth flow to the initiating user's identity, since the callback redirect carries no app session.
6. Implement three routes: `connect` (builds the Google auth URL with the signed state and redirects), `callback` (verifies state, exchanges the code for tokens, persists them), and `disconnect` (deletes stored tokens).
7. Persist `{ accessToken, refreshToken, expiresAt }` per user; on each API use, check expiry with a skew buffer and transparently refresh via the token endpoint if needed.
8. Wire app-side event lifecycle (create/update/delete) to call the Calendar API, treating Calendar sync as strictly best-effort on top of the source-of-truth write — never let a Calendar failure roll back or block the primary write.

### Core pattern
```ts
// google-calendar.ts — OAuth + Calendar REST client
const CALENDAR_SCOPE = "https://www.googleapis.com/auth/calendar.events";
const TOKEN_ENDPOINT = "https://oauth2.googleapis.com/token";
const CALENDAR_API_BASE = "https://www.googleapis.com/calendar/v3";
const EXPIRY_SKEW_MS = 2 * 60 * 1000;

export function buildAuthUrl(state: string, redirectUri: string): string {
  const params = new URLSearchParams({
    client_id: process.env.GOOGLE_CALENDAR_CLIENT_ID!,
    redirect_uri: redirectUri,
    response_type: "code",
    scope: CALENDAR_SCOPE,
    access_type: "offline", // required to get a refresh_token
    prompt: "consent",      // forces a refresh_token on every connect, even re-connects
    state,
  });
  return `https://accounts.google.com/o/oauth2/v2/auth?${params.toString()}`;
}

export async function exchangeCodeForTokens(code: string, redirectUri: string) {
  const res = await fetch(TOKEN_ENDPOINT, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      code,
      client_id: process.env.GOOGLE_CALENDAR_CLIENT_ID!,
      client_secret: process.env.GOOGLE_CALENDAR_CLIENT_SECRET!,
      redirect_uri: redirectUri,
      grant_type: "authorization_code",
    }),
  });
  if (!res.ok) return null;
  const data = await res.json();
  if (!data.refresh_token) return null; // missing => request was misconfigured, not recoverable
  return {
    accessToken: data.access_token,
    refreshToken: data.refresh_token,
    expiresAt: new Date(Date.now() + data.expires_in * 1000).toISOString(),
  };
}

// Resolves a usable access token, refreshing if near-expiry. Returns null when
// the user never connected, or the refresh token was revoked/expired — callers
// must treat null as a silent no-op, never a hard error.
export async function getValidAccessToken(userKey: string): Promise<string | null> {
  const record = await getStoredTokens(userKey); // your token store
  if (!record) return null;
  if (new Date(record.expiresAt).getTime() - EXPIRY_SKEW_MS > Date.now()) return record.accessToken;

  const res = await fetch(TOKEN_ENDPOINT, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      refresh_token: record.refreshToken,
      client_id: process.env.GOOGLE_CALENDAR_CLIENT_ID!,
      client_secret: process.env.GOOGLE_CALENDAR_CLIENT_SECRET!,
      grant_type: "refresh_token",
    }),
  });
  if (!res.ok) { await deleteStoredTokens(userKey); return null; }
  const data = await res.json();
  const refreshed = { accessToken: data.access_token, expiresAt: new Date(Date.now() + data.expires_in * 1000).toISOString() };
  await saveStoredTokens(userKey, { ...record, ...refreshed });
  return refreshed.accessToken;
}

// sendUpdates=all is mandatory on write calls — without it attendees are added
// to the event but never receive an email invite.
export async function createCalendarEvent(accessToken: string, event: object): Promise<{ id: string } | null> {
  const res = await fetch(`${CALENDAR_API_BASE}/calendars/primary/events?sendUpdates=all`, {
    method: "POST",
    headers: { Authorization: `Bearer ${accessToken}`, "Content-Type": "application/json" },
    body: JSON.stringify(event),
  });
  return res.ok ? { id: (await res.json()).id } : null;
}
```

```ts
// Route sketch: connect -> callback -> disconnect
// GET /api/google-calendar/connect
//   -> if not configured, 503
//   -> sign a short-lived state token binding the current user's identity
//   -> redirect(buildAuthUrl(state, callbackUrl))

// GET /api/google-calendar/callback   (NO auth middleware — Google hits this directly)
//   -> verify `state` (identifies the user); reject if invalid
//   -> exchangeCodeForTokens(code, callbackUrl)
//   -> persist tokens for that user
//   -> redirect to a success page

// POST /api/google-calendar/disconnect
//   -> delete stored tokens for the current user
```

### Env vars
`GOOGLE_CALENDAR_CLIENT_ID`, `GOOGLE_CALENDAR_CLIENT_SECRET`

### Gotchas
- Use a **separate** OAuth client from any sign-in OAuth client — mixing scopes on one client complicates the consent screen and verification review.
- `calendar.events` is a Google "sensitive scope"; until the OAuth app passes Google's verification, it's stuck in "Testing" publishing status, which caps refresh-token lifetime at ~7 days — design the UX to degrade gracefully (silent no-op, "reconnect" prompt) rather than erroring when a token has expired.
- Always pass `access_type=offline` **and** `prompt=consent` when building the auth URL — omitting `prompt=consent` means Google won't reissue a `refresh_token` on a repeat authorization, silently breaking reconnect flows.
- The OAuth callback route runs with no app session in scope (Google redirects the browser directly) — authenticate the callback via a signed `state` param bound to the user, not session middleware.
- Distinguish "event gone on Google's side" (`404`/`410` — safe to clear the local event reference so a future edit recreates it) from a transient failure (network blip, `5xx` — must NOT clear the reference, or the next edit will "create" a duplicate event instead of retrying the update).
- Treat Calendar sync as strictly best-effort, layered on top of an already-committed source-of-truth write — catch and log all Calendar errors; never let them roll back or block the primary operation.
- When a background/fire-and-forget job triggers a cache invalidation (e.g. `revalidateTag`) outside of a request scope, it may silently no-op in some frameworks — accept a bounded staleness window rather than architecting around it.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace, to let a user connect their own Google Calendar (via a dedicated OAuth client, separate from the app's sign-in OAuth client) so app-side scheduled events auto-create/update/delete matching Calendar events with attendee invites.
