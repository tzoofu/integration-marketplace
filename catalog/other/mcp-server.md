# Self-hosted MCP Server (OAuth 2.1 + PAT)

- **category**: other
- **provider**: Anthropic (Model Context Protocol) / self-hosted
- **reusable**: yes — a self-hosted, OAuth2.1+PAT-secured MCP server exposing the app's own data/actions as tools is a strong, high-value pattern, confirmed across six repos; the OAuth authorization-server half (DCR + PKCE code + rotating refresh + hashed PATs) is identical enough to become a `@marketplace/mcp-oauth` package.
- **docs**: https://modelcontextprotocol.io/ (authorization: https://modelcontextprotocol.io/specification/latest/basic/authorization)

## Overview
Each repo exposes its own domain data (listings, events, tasks, coupons, etc.) as MCP tools at an `/api/mcp` endpoint, so an external AI client (Claude Desktop, Claude Code) can read/write the app's data directly. Access is secured either by a full OAuth 2.1 Authorization Code + PKCE flow (for interactive client sign-in) or by a long-lived Personal Access Token (PAT) minted from the app's own settings UI (for non-interactive/CLI use).

## Playbook

### Prerequisites
- An existing web app with its own session-based auth (NextAuth, Firebase Auth, etc.) — the MCP server rides on top of that as the "who is this token/session for" source of truth.
- An MCP server SDK (`@modelcontextprotocol/sdk`) for defining a streamable-HTTP or SSE MCP server and its tool handlers.
- A signing secret for OAuth/PAT tokens (can reuse the app's existing session secret, or mint a dedicated one).

### Setup steps
1. Define the MCP server and its tools in a dedicated module (e.g. `lib/mcp/server.ts` + `lib/mcp/tools/*.ts`), one tool per app capability you want to expose (e.g. `listTasks`, `createEvent`, `getCoupon`) with a zod/JSON schema for its input.
2. Add an OAuth 2.1 Authorization Code + PKCE flow scoped to this app: `/api/oauth/authorize`, `/api/oauth/callback`, `/api/oauth/register` (dynamic client registration), `/api/oauth/token`.
3. Add a Personal Access Token path as a lighter-weight alternative to full OAuth: a PAT table/collection keyed by a hashed token, mintable from a settings page, checked on every MCP request via an `Authorization: Bearer <pat>` header.
4. Add a single `/api/mcp` route that accepts either a valid OAuth access token or a valid PAT, resolves it to the underlying app user, and dispatches to the MCP server's tool handlers scoped to that user's data.
5. Optionally publish `.well-known/oauth-authorization-server` and `.well-known/oauth-protected-resource` discovery documents so MCP clients can auto-discover the OAuth endpoints instead of being hand-configured.
6. Decide whether MCP tools should be read/write or read-only-by-design — a messaging-adjacent app may deliberately keep its MCP tools read/organize-only and defer any "send" action to a separate, dedicated messaging MCP server.
7. Reuse or don't reuse the app's existing session-signing secret for the MCP OAuth JWTs — reusing it is simpler; minting a dedicated secret isolates a compromise of one token type from the other.

### Core pattern
```ts
// lib/mcp/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

export function createMcpServer(userId: string) {
  const server = new McpServer({ name: "app-mcp", version: "1.0.0" });

  server.tool(
    "listItems",
    "List the current user's items",
    { status: z.enum(["open", "done"]).optional() },
    async ({ status }) => {
      const items = await getItemsForUser(userId, status);
      return { content: [{ type: "text", text: JSON.stringify(items) }] };
    }
  );

  return server;
}

// app/api/mcp/route.ts
export async function POST(req: Request) {
  const auth = req.headers.get("authorization");
  const userId = await resolveUserFromOAuthOrPat(auth); // OAuth access token OR PAT
  if (!userId) return new Response("Unauthorized", { status: 401 });

  const server = createMcpServer(userId);
  return handleMcpRequest(server, req); // SDK transport glue
}
```

**Stateless, role-gated variant** (MCP SDK v2 split packages, web-standard Request/Response, e.g. a
Next.js route handler). Every request builds a fresh server holding only the tools the caller's
role may use, so `tools/list` itself is permission-filtered:

```ts
// app/api/mcp/route.ts
import { WebStandardStreamableHTTPServerTransport } from "@modelcontextprotocol/server";

export async function POST(req: Request) {
  const origin = publicOrigin(req);                       // env-configured public URL, not req.url behind a proxy
  const email = await resolveCaller(req.headers.get("authorization"), origin); // PAT hash lookup or JWT verify, then live roster/user check
  if (!email) {
    return new Response(null, {
      status: 401,
      headers: { "WWW-Authenticate": `Bearer resource_metadata="${origin}/.well-known/oauth-protected-resource"` },
    });
  }
  const data = await loadWhatThisUserMaySee(email);       // same scoping as the app's own reads
  const server = buildServer(data, TOOLS.filter((t) => !t.when || t.when(data)));
  const transport = new WebStandardStreamableHTTPServerTransport({ sessionIdGenerator: undefined, enableJsonResponse: true });
  await server.connect(transport);
  try { return await transport.handleRequest(req); } finally { void server.close(); }
}
export const GET = () => new Response(null, { status: 405 });   // no SSE stream / sessions when stateless
```

```ts
// Authorization server: consent page reuses the app's existing client-side sign-in
// GET  /.well-known/oauth-protected-resource  -> { resource: `${origin}/api/mcp`, authorization_servers: [origin] }
// GET  /.well-known/oauth-authorization-server -> endpoints, code_challenge_methods_supported: ["S256"], token_endpoint_auth_methods_supported: ["none"]
// POST /api/oauth/register   -> validate redirect_uris (https, or http on loopback only), store client, return client_id
// GET  /oauth/authorize      -> client page: validate client_id + exact redirect_uri server-side FIRST, sign in with the
//                               app's normal login, show "Allow <client> to act as <you>?"
// POST /api/oauth/authorize  -> body = query params + decision, Authorization: Bearer <app ID token>;
//                               store sha256(code) -> {clientId, redirectUri, codeChallenge, email, expiresAt: +5min}
// POST /api/oauth/token      -> authorization_code: in a transaction, delete the code doc, then check expiry,
//                               client/redirect match and PKCE S256; refresh_token: rotate (delete old, issue new)
//                               -> { access_token: short-lived signed JWT (aud = MCP resource URL), refresh_token }
```

### Env vars
`NEXTAUTH_SECRET` (or equivalent session secret, reused for OAuth JWTs) — or a dedicated `MCP_OAUTH_SECRET` when isolating it from session auth; a PAT prefix constant (e.g. `PAT_PREFIX`) for identifying/validating token format. Behind a proxy/load balancer also a public base URL (e.g. `MCP_PUBLIC_URL`) used as issuer, audience and in the discovery documents.

### Gotchas
- Reusing the app's session secret for MCP OAuth tokens is simpler but couples the two token types' blast radius — a dedicated secret is the more defensive choice if the MCP surface is externally reachable.
- A read/organize-only MCP server for a messaging-adjacent app is a deliberate safety boundary, not an oversight — don't "complete" it by adding a send-message tool without checking whether that was intentional.
- Publishing `.well-known/oauth-authorization-server` and `.well-known/oauth-protected-resource` isn't required for PAT-only usage, but is needed for MCP clients that expect to auto-discover OAuth endpoints (e.g. Claude Desktop's connector flow).
- Dynamic client registration (`/api/oauth/register`) is part of OAuth 2.1's expectations for MCP clients — without it, every MCP client needs a manually pre-registered client ID.
- A 401 from the MCP endpoint must carry `WWW-Authenticate: Bearer resource_metadata="…/.well-known/oauth-protected-resource"` — that header is what makes clients (Claude Code, MCP Inspector) start the OAuth flow. Serve the metadata route as a catch-all so the path-suffixed form (`/.well-known/oauth-protected-resource/api/mcp`) also answers.
- If the server reads/writes the database with an admin/service SDK, the database's security rules no longer apply. Every write tool must re-check what the rules and the UI allow for the caller (owner-only writes, manager-of-this-person, admin-only), and should write documents through the same pure doc-shape builders the web app uses so the two can't drift.
- Resolve the role from the user record on **every** request rather than from token claims — then removing a user or changing their role takes effect immediately, even for unexpired access tokens. Re-check on refresh too.
- Store codes, refresh tokens and PATs only as SHA-256 hashes (the hash can be the document id), in collections no client can read; make codes single-use inside a transaction and rotate refresh tokens. Compare secrets in constant time.
- Validate `client_id` + exact `redirect_uri` before rendering consent or redirecting anywhere; only `https://` or loopback `http://` redirect URIs at registration.
- A stateless server (no session id) can't make server→client requests, so in-tool elicitation/confirmation prompts don't work — have destructive tools require an explicit `confirm: true` argument and say "ask the user first" in the description.
- First-run symptom of a missing signing secret: the consent/authorize step succeeds but the token endpoint fails (503/500). Return a clear "not configured" error rather than a generic 500.
- Reachability differs by client: Claude Code, Cursor and Claude Desktop (via `mcp-remote`) connect from the user's machine, so a private-network/VPN-only host works; claude.ai / ChatGPT web connectors call from the vendor's cloud and need a publicly reachable endpoint.
- If production secrets are synced into the cluster by an external-secrets controller, patching the Kubernetes Secret directly is overwritten on the next refresh — add the signing secret to the upstream secret store.
- Integration-test the route handlers by calling the exported `POST`/`GET` functions directly against the database emulator (no HTTP server needed); assert per-role `tools/list` contents and the refusals (wrong role, not their report, missing `confirm`).

- **Hosted chat clients (e.g. ChatGPT connectors) need OAuth discovery, not just the endpoint.** Serve `/.well-known/oauth-protected-resource` (RFC 9728) pointing at your authorization server, plus `/.well-known/oauth-authorization-server` and OIDC discovery metadata, and return a `WWW-Authenticate: Bearer resource_metadata="…"` header on the 401. If the host has a WAF/geo-firewall or managed "AI bots" rule, exempt `/.well-known/`, the MCP endpoint and the OAuth routes, or the client gets a 403 before it reaches your code.

### Playbook confidence: high

## Adoption
Used in **6** repo(s) in this marketplace. Most expose their own domain data/actions as MCP tools
at a single `/api/mcp` endpoint, secured by an OAuth 2.1 Authorization Code + PKCE flow and/or
Personal Access Tokens. Variation is mostly in JWT-signing-secret strategy (shared session secret
vs a dedicated OAuth secret — see Gotchas), in transport (a long-lived server vs a stateless
per-request server whose tool list is filtered by the caller's role), and in tool scope — one adopter is deliberately read/organize-only by design, explicitly
deferring any "send" action to a separate, dedicated messaging integration rather than implementing
it itself. One adopter instead ships a thin stdio MCP binary that never touches the database: it only calls the app's own REST API with a scoped personal API token. Its write tools are gated by an env allow flag and a recipient allowlist, and the server re-checks every send.
