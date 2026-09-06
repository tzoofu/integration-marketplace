# Self-hosted MCP Server (OAuth 2.1 + PAT)

- **category**: other
- **provider**: Anthropic (Model Context Protocol) / self-hosted
- **reusable**: yes — a self-hosted, OAuth2.1+PAT-secured MCP server exposing the app's own data/actions as tools is a strong, high-value pattern, confirmed across four repos.
- **docs**: https://modelcontextprotocol.io/

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

### Env vars
`NEXTAUTH_SECRET` (or equivalent session secret, reused for OAuth JWTs) — or a dedicated `MCP_OAUTH_SECRET` when isolating it from session auth; a PAT prefix constant (e.g. `PAT_PREFIX`) for identifying/validating token format.

### Gotchas
- Reusing the app's session secret for MCP OAuth tokens is simpler but couples the two token types' blast radius — a dedicated secret is the more defensive choice if the MCP surface is externally reachable.
- A read/organize-only MCP server for a messaging-adjacent app is a deliberate safety boundary, not an oversight — don't "complete" it by adding a send-message tool without checking whether that was intentional.
- Publishing `.well-known/oauth-authorization-server` and `.well-known/oauth-protected-resource` isn't required for PAT-only usage, but is needed for MCP clients that expect to auto-discover OAuth endpoints (e.g. Claude Desktop's connector flow).
- Dynamic client registration (`/api/oauth/register`) is part of OAuth 2.1's expectations for MCP clients — without it, every MCP client needs a manually pre-registered client ID.

### Playbook confidence: high

## Adoption
Used in **4** repo(s) in this marketplace. All expose their own domain data/actions as MCP tools
at a single `/api/mcp` endpoint, secured by an OAuth 2.1 Authorization Code + PKCE flow and/or
Personal Access Tokens. Variation is mostly in JWT-signing-secret isolation (some reuse the app's
existing session secret, one mints a dedicated OAuth secret and its own JWT signing rather than
reusing it) and in tool scope — one adopter is deliberately read/organize-only by design, explicitly
deferring any "send" action to a separate, dedicated messaging integration rather than implementing
it itself.
