# Scoped personal API tokens

- **category**: auth
- **provider**: internal (custom)
- **reusable**: yes — hash-only storage, a prefixed random secret, a route→scope table guarded by a test, and a show-once UI apply to any app whose users want agents or scripts to call its API.
- **docs**: n/a

Lets signed-in users mint long-lived API keys for agents, MCP servers and scripts, sent as `Authorization: Bearer app_…` next to the normal browser session cookie. Each key carries scopes (e.g. `read`, `send`, `tpl`) and optionally a list of resources it may touch; every route declares the scope it needs in one table, and credential-management routes refuse tokens entirely.

## Playbook

### Prerequisites
- An existing session-based login (cookie) and a `users` table.
- A router that can report the matched route pattern for a request (e.g. chi's `Mux.Find`, or route metadata in Express/Fastify).
- `crypto/rand` and SHA-256 (any language's standard library).

### Setup steps
1. Create a tokens table: `id, user_id, name, prefix (display only), token_hash UNIQUE, scopes (JSON array), all_resources bool, created_at, last_used_at, use_count, revoked_at`. Add a join table `token_resources(token_id, resource_id)` if keys can be limited to specific resources.
2. Mint: 32 random bytes → base64url → prepend a short app prefix (`app_`). Store `sha256(token)` hex and the first ~10 chars as `prefix` for display. Return the plaintext exactly once in the create response.
3. Authenticate: if the request has `Authorization: Bearer …`, reject anything without the prefix, hash it, look up a non-revoked row joined to its user, and update `last_used_at`/`use_count`. Otherwise fall back to the session cookie. Attach the user and (if any) the token to the request context.
4. Define scope constants: `any` (any live token), `read`, `send`, `tpl`, and `session` (dashboard only — tokens refused). Keep a single map `"METHOD /pattern" → scope` covering every route; unlisted routes default to the strictest write scope.
5. Add a `scopeGuard` middleware after routing: for token requests look up the matched pattern's scope; answer `403 session_required` for `session` routes and `403 missing_scope` when the token lacks the scope. Browser sessions are never scope-limited.
6. Write a test that walks the router and fails for any route missing from the map, and for any map entry that is no longer a route.
7. Mark routes that manage credentials, secrets, webhooks, roles or tokens' own settings as `session`, so a leaked key can't escalate or exfiltrate secrets.
8. Token CRUD routes: list (never returns the hash), create (validates scopes, caps resource list length), revoke (sets `revoked_at`, scoped to `user_id`). When a *token* calls create, it may only mint a key with a subset of its own scopes and resources; when a token calls revoke, it may only revoke itself.
9. UI: a settings card listing keys (name, scope pills, `prefix…`, resources, "used X ago / never used", revoke with confirm) and a create form (name, resource select, scope chips) that switches to a "copy it now — you won't see it again" panel with a copy button.

### Core pattern
```go
package tokens

import (
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"encoding/hex"
	"net/http"
	"slices"
	"strings"
)

const Prefix = "app_"

// New returns the plaintext (show once) and the hash to store.
func New() (token, hash string) {
	b := make([]byte, 32)
	if _, err := rand.Read(b); err != nil {
		panic(err)
	}
	token = Prefix + base64.RawURLEncoding.EncodeToString(b)
	return token, Hash(token)
}

func Hash(token string) string {
	sum := sha256.Sum256([]byte(token))
	return hex.EncodeToString(sum[:])
}

// Display prefix kept in the DB so users can tell keys apart.
func DisplayPrefix(token string) string { return token[:len(Prefix)+6] }

// ---- authentication ---------------------------------------------------------

func (s *Server) authenticate(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if h := r.Header.Get("Authorization"); strings.HasPrefix(h, "Bearer ") {
			raw := strings.TrimPrefix(h, "Bearer ")
			if !strings.HasPrefix(raw, Prefix) {
				http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
				return
			}
			// SELECT u.*, t.* FROM tokens t JOIN users u ON u.id=t.user_id
			//   WHERE t.token_hash=? AND t.revoked_at IS NULL
			// then UPDATE tokens SET last_used_at=now, use_count=use_count+1 WHERE id=?
			u, tok, err := s.store.AuthToken(r.Context(), Hash(raw))
			if err != nil {
				http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
				return
			}
			r = withUser(withToken(r, tok), u)
		} else if c, err := r.Cookie("session"); err == nil {
			// Cookie auth on unsafe methods: also require an allowed Origin header (CSRF).
			u, err := s.store.UserForSession(r.Context(), c.Value)
			if err != nil {
				http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
				return
			}
			r = withUser(r, u)
		} else {
			http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
			return
		}
		next.ServeHTTP(w, r)
	})
}

// ---- scopes -----------------------------------------------------------------

const (
	ScopeAny     = "any"
	ScopeRead    = "read"
	ScopeSend    = "send"
	ScopeTpl     = "tpl"
	ScopeSession = "session" // dashboard only
)

var routeScopes = map[string]string{
	"GET /me":                       ScopeAny,
	"GET /me/tokens":                ScopeRead,
	"POST /me/tokens":               ScopeAny, // handler limits a token to minting a subset of itself
	"DELETE /me/tokens/{id}":        ScopeAny, // handler lets a token revoke only itself
	"GET /me/webhooks":              ScopeSession,
	"PATCH /settings/credentials":   ScopeSession,
	"GET /items":                    ScopeRead,
	"POST /items":                   ScopeSend,
	"POST /templates":               ScopeTpl,
}

func routeKey(method, pattern string) string {
	p := strings.TrimPrefix(pattern, "/api/v1")
	if len(p) > 1 {
		p = strings.TrimSuffix(p, "/")
	}
	return method + " " + p
}

func scopeFor(method, pattern string) string {
	if sc, ok := routeScopes[routeKey(method, pattern)]; ok {
		return sc
	}
	return ScopeSend // fail closed for unclassified routes
}

func (s *Server) scopeGuard(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		tok, isToken := tokenOf(r)
		if !isToken {
			next.ServeHTTP(w, r) // browser sessions are not scope-limited
			return
		}
		pattern := s.mux.Find(chi.NewRouteContext(), r.Method, r.URL.Path) // chi v5
		switch sc := scopeFor(r.Method, pattern); sc {
		case ScopeAny:
		case ScopeSession:
			writeError(w, 403, "session_required", "only from the signed-in dashboard, not with an API token")
			return
		default:
			if !slices.Contains(tok.Scopes, sc) {
				writeError(w, 403, "missing_scope", "this API key lacks the "+sc+" permission")
				return
			}
		}
		next.ServeHTTP(w, r)
	})
}
```

Completeness test (chi):

```go
func TestEveryRouteHasAScope(t *testing.T) {
	seen := map[string]bool{}
	_ = chi.Walk(newRouter(), func(method, route string, _ http.Handler, _ ...func(http.Handler) http.Handler) error {
		if !strings.HasPrefix(route, "/api/v1/") {
			return nil
		}
		key := routeKey(method, strings.ReplaceAll(route, "/*/", "/"))
		seen[key] = true
		if _, ok := routeScopes[key]; !ok {
			t.Errorf("route %q has no scope", key)
		}
		return nil
	})
	for key := range routeScopes {
		if !seen[key] {
			t.Errorf("routeScopes names %q, which is not a route", key)
		}
	}
}
```

Show-once create UI (React + TanStack Query):

```tsx
function CreateKey({ onDone }: { onDone: () => void }) {
  const [name, setName] = useState("");
  const [scopes, setScopes] = useState<Scope[]>(["read"]); // least privilege by default
  const create = useMutation({ mutationFn: () => api.createToken({ name: name.trim(), scopes, resource_ids: null }) });
  const key = create.data?.token; // plaintext only exists in this response

  if (key)
    return (
      <Panel>
        <strong>Copy your key now</strong>
        <p>You won't be able to see it again.</p>
        <code dir="ltr" className="select-all font-mono">{key}</code>
        <CopyButton text={key} />
        <button onClick={onDone}>Done</button>
      </Panel>
    );
  return (
    <form onSubmit={(e) => { e.preventDefault(); if (name.trim() && scopes.length) create.mutate(); }}>
      {/* name field, resource select ("All" or one), scope chips */}
    </form>
  );
}
```

### Env vars
None required. (The prefix is a code constant; there is no signing secret because tokens are random and verified by hash lookup.)

### Gotchas
- Store only `sha256(token)`. A fast hash is fine here (unlike passwords) because the token has 256 bits of entropy; it lets you look the token up by an indexed equality match.
- Check the prefix before hashing: it rejects garbage cheaply and makes leaked keys greppable by secret scanners (register the prefix pattern with your SCM's secret scanning if possible).
- Keep a short display prefix (`app_AbC12x…`) so users can match a key in a list without the server keeping the secret.
- Default unlisted routes to the strictest non-session scope (fail closed), and back the table with a router-walking test so a new route can't ship unclassified — and stale entries are caught too.
- Mark anything that reveals or replaces credentials, manages webhooks/roles/admin settings, or starts human-only flows (phone pairing, OTP login) as session-only. A stolen agent key then can't widen its own access.
- When a token creates a token, intersect: requested scopes must all be held by the caller, and requested resources must be within the caller's allowed set (default to the caller's set when none is given). Otherwise `POST /tokens` is a privilege-escalation path.
- A token may revoke itself but not the user's other keys; session callers may revoke any of their own keys. Always scope revoke/list queries by `user_id`.
- Resource-limited tokens should get `404` (not `403`) for resources outside their list, so they can't probe what exists.
- Record usage in the same request (`last_used_at`, `use_count`); show "never used" for idle keys so users can prune them. For long-lived requests (SSE streams) re-resolve the credential periodically *without* bumping the use counter, so a revoked key or deactivated user drops the stream.
- Cookie-authenticated unsafe requests need an Origin check (CSRF); bearer-token requests don't, because browsers don't attach them automatically.
- The plaintext appears only in the create response — don't cache it in client state beyond that panel, and never log the `Authorization` header.
- Return token metadata with RFC 3339 timestamps and `resource_ids: null` meaning "all", so clients can distinguish "all" from "none".

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
