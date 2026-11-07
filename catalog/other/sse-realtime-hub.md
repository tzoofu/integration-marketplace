# Server-Sent Events from an In-Process Pub/Sub Hub (Go + React Query)

- **category**: other
- **provider**: internal (custom), Go `net/http` + browser `EventSource` + TanStack React Query
- **reusable**: yes — the hub, the SSE handler and the client hook are app-agnostic; only the event names and the cache-update switch are per-app.
- **docs**: https://html.spec.whatwg.org/multipage/server-sent-events.html · https://developer.mozilla.org/en-US/docs/Web/API/EventSource · https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation

## Overview
A single Go process publishes domain events (`message.new`, `item.updated`, `job.done`, …) into an in-memory hub. Each event belongs to a tenant (account/workspace) and reaches only the users in that tenant's audience: the owner plus members. One SSE endpoint per signed-in user streams all of that user's tenants. Every payload carries `account_id`, heartbeats keep proxies from closing the stream and re-check the session, and the React client reconnects with backoff, then patches or invalidates react-query caches. Reach for it for live dashboards on a single-instance server, where one-way push is enough and WebSockets are overkill.

## Playbook

### Prerequisites
- Go 1.21+ server with a session or token auth middleware that puts the user in the request context.
- React 18 + `@tanstack/react-query` v5 on the client, served from the same origin as the API (cookies flow automatically to `EventSource`).
- If behind a reverse proxy: response buffering and compression disabled for the stream path.

### Setup steps
1. Create the hub (Core pattern) and keep **tenant → owner** and **tenant → members** maps in it. Fill them at startup from the database, and call `SetOwner`/`SetMembers` whenever ownership or membership changes.
2. Give every producer only the narrow `Publisher` interface (`Publish(accountID, type, data)`), and pass a `Nop` in tests.
3. Add `GET /api/events` behind auth. Subscribe, send initial state snapshots, then loop over events and a heartbeat ticker.
4. Inject `account_id` into every JSON payload at the edge, so producers can't forget it.
5. Configure the HTTP server without a `WriteTimeout` (or exempt the stream), and cancel request contexts on shutdown so streams end promptly.
6. Proxy: Caddy `flush_interval -1`, nginx `proxy_buffering off` (or send `X-Accel-Buffering: no`), and no gzip on the path.
7. Client: one `useEvents()` hook mounted once at the app root after sign-in, with a typed `apply(event)` switch that updates caches.

### Core pattern
**Hub**
```go
package hub

import (
	"slices"
	"sync"
)

type Event struct {
	AccountID int64
	UserID    int64
	Type      string
	Data      any
}

type Publisher interface{ Publish(accountID int64, eventType string, data any) }

// Hub is an in-process fan-out. Publish never blocks: a subscriber whose buffer is full misses
// the event (the client recovers by refetching on reconnect).
type Hub struct {
	mu      sync.RWMutex
	subs    map[chan Event]int64 // channel -> subscribed user
	owners  map[int64]int64      // account -> owner user
	members map[int64][]int64    // account -> other users with access
	taps    []func(Event)        // e.g. outbound webhook dispatcher
}

func New() *Hub {
	return &Hub{subs: map[chan Event]int64{}, owners: map[int64]int64{}, members: map[int64][]int64{}}
}

func (h *Hub) SetOwner(accountID, userID int64) {
	h.mu.Lock(); defer h.mu.Unlock()
	if userID == 0 { delete(h.owners, accountID); return }
	h.owners[accountID] = userID
}

func (h *Hub) SetMembers(accountID int64, users []int64) {
	h.mu.Lock(); defer h.mu.Unlock()
	if len(users) == 0 { delete(h.members, accountID); return }
	h.members[accountID] = append([]int64(nil), users...)
}

func (h *Hub) Subscribe(userID int64) (<-chan Event, func()) {
	ch := make(chan Event, 64)
	h.mu.Lock(); h.subs[ch] = userID; h.mu.Unlock()
	var once sync.Once
	return ch, func() {
		once.Do(func() { h.mu.Lock(); delete(h.subs, ch); h.mu.Unlock(); close(ch) })
	}
}

// Tap sees every event once per audience user; it runs inside Publish, so it must not block.
func (h *Hub) Tap(fn func(Event)) { h.mu.Lock(); h.taps = append(h.taps, fn); h.mu.Unlock() }

func (h *Hub) Publish(accountID int64, eventType string, data any) {
	h.mu.RLock(); defer h.mu.RUnlock()
	owner := h.owners[accountID]
	if owner == 0 {
		return // unknown tenant: deliver to nobody
	}
	for _, u := range append([]int64{owner}, h.members[accountID]...) {
		for _, fn := range h.taps {
			fn(Event{AccountID: accountID, UserID: u, Type: eventType, Data: data})
		}
	}
	ev := Event{AccountID: accountID, UserID: owner, Type: eventType, Data: data}
	for ch, user := range h.subs {
		if user != owner && !slices.Contains(h.members[accountID], user) {
			continue
		}
		select {
		case ch <- ev:
		default: // slow consumer: drop rather than stall every producer
		}
	}
}

type Nop struct{}

func (Nop) Publish(int64, string, any) {}
```

**SSE handler**
```go
var heartbeatInterval = 15 * time.Second

func (s *Server) events(w http.ResponseWriter, r *http.Request) {
	flusher, ok := w.(http.Flusher)
	if !ok {
		http.Error(w, "streaming unsupported", http.StatusInternalServerError)
		return
	}
	user := userFrom(r)
	ch, unsubscribe := s.hub.Subscribe(user.ID) // subscribe BEFORE the snapshot so nothing is missed
	defer unsubscribe()

	h := w.Header()
	h.Set("Content-Type", "text/event-stream")
	h.Set("Cache-Control", "no-cache")
	h.Set("Connection", "keep-alive")
	h.Set("X-Accel-Buffering", "no")
	w.WriteHeader(http.StatusOK)

	write := func(accountID int64, event string, data any) bool {
		b, err := withAccountID(accountID, data)
		if err != nil {
			return true // skip unmarshalable payloads, keep the stream
		}
		if _, err := fmt.Fprintf(w, "event: %s\ndata: %s\n\n", event, b); err != nil {
			return false
		}
		flusher.Flush()
		return true
	}

	// Initial snapshot so a (re)connecting client is immediately correct.
	for _, a := range s.store.AccessibleAccounts(r.Context(), user.ID) {
		write(a.ID, "connection", s.stateOf(a.ID))
	}

	t := time.NewTicker(heartbeatInterval)
	defer t.Stop()
	for {
		select {
		case <-r.Context().Done():
			return
		case ev, ok := <-ch:
			if !ok {
				return
			}
			if !s.mayRead(r, ev.AccountID) { // per-credential scope (e.g. a token limited to some tenants)
				continue
			}
			if !write(ev.AccountID, ev.Type, ev.Data) {
				return
			}
		case <-t.C:
			if !s.sessionStillValid(r) { // signed out, token revoked, user disabled
				return
			}
			if _, err := fmt.Fprint(w, ": heartbeat\n\n"); err != nil { // SSE comment line
				return
			}
			flusher.Flush()
		}
	}
}

// withAccountID adds "account_id" as the first field of a JSON object payload.
func withAccountID(accountID int64, data any) ([]byte, error) {
	b, err := json.Marshal(data)
	if err != nil {
		return nil, err
	}
	head := `{"account_id":` + strconv.FormatInt(accountID, 10)
	switch {
	case len(b) < 2 || b[0] != '{':
		return []byte(head + `,"data":` + string(b) + `}`), nil
	case string(b) == "{}":
		return []byte(head + "}"), nil
	}
	return []byte(head + "," + string(b[1:])), nil
}
```

**Server: let streams end on shutdown**
```go
reqCtx, cancel := context.WithCancel(context.WithoutCancel(ctx))
srv := &http.Server{
	Addr: addr, Handler: h,
	ReadHeaderTimeout: 10 * time.Second,
	IdleTimeout:       120 * time.Second,
	// no WriteTimeout: it would kill every SSE stream after N seconds
	BaseContext: func(net.Listener) context.Context { return reqCtx },
}
srv.RegisterOnShutdown(cancel) // otherwise Shutdown waits for open streams until its deadline
```

**React client hook**
```ts
import { useEffect, useRef, useState } from "react";
import { useQueryClient, type QueryClient } from "@tanstack/react-query";

type ServerEvent =
  | { type: "item.new"; data: { account_id: number; id: string; list_id: string } }
  | { type: "item.updated"; data: { account_id: number; id: string } }
  | { type: "connection"; data: { account_id: number; status: string } };

const EVENT_TYPES = ["item.new", "item.updated", "connection"] as const;
const keys = (accountId: number) => ({
  all: ["account", accountId] as const,
  items: ["account", accountId, "items"] as const,
  item: (id: string) => ["account", accountId, "item", id] as const,
});

function apply(qc: QueryClient, ev: ServerEvent) {
  const k = keys(ev.data.account_id); // every payload carries account_id → route to that tenant's cache
  switch (ev.type) {
    case "item.new":
      qc.invalidateQueries({ queryKey: k.items });
      break;
    case "item.updated":
      qc.invalidateQueries({ queryKey: k.item(ev.data.id) });
      qc.invalidateQueries({ queryKey: k.items });
      break;
    case "connection":
      qc.setQueryData([...k.all, "status"], ev.data.status); // patch directly when the payload is the state
      break;
  }
}

export function useEvents(enabled: boolean) {
  const qc = useQueryClient();
  const [state, setState] = useState<"open" | "connecting" | "closed">("connecting");
  const retry = useRef(0);

  useEffect(() => {
    if (!enabled) return;
    let es: EventSource | null = null;
    let timer: ReturnType<typeof setTimeout> | undefined;
    let stopped = false;

    const open = () => {
      if (stopped) return;
      setState("connecting");
      es = new EventSource("/api/events"); // same-origin: cookies are sent
      es.onopen = () => {
        retry.current = 0;
        setState("open");
        qc.invalidateQueries(); // catch up on anything missed while disconnected
      };
      for (const type of EVENT_TYPES) {
        es.addEventListener(type, (e) => {
          try {
            apply(qc, { type, data: JSON.parse((e as MessageEvent).data) } as ServerEvent);
          } catch { /* ignore malformed event */ }
        });
      }
      es.onerror = () => {
        es?.close(); // take over from the browser's fixed-interval auto-retry
        setState("closed");
        const delay = Math.min(30_000, 1000 * 2 ** retry.current++) + Math.random() * 500;
        timer = setTimeout(open, delay);
      };
    };
    open();
    return () => { stopped = true; clearTimeout(timer); es?.close(); };
  }, [enabled, qc]);

  return state; // show a small "reconnecting" badge when not "open"
}
```

### Env vars
- None specific. The stream relies on the app's existing session cookie or token auth.

### Gotchas
- **Named events need named listeners.** `event: item.new` frames are *not* delivered to `es.onmessage`. Register `addEventListener(type, …)` for every type, and keep the list in sync with the server's event names (a typed union helps).
- **Every payload carries the tenant id.** One stream multiplexes all of a user's tenants, so the client must route by `account_id`. Injecting it at the SSE edge (`withAccountID`) guarantees it even when a producer forgets.
- **Filter twice.** The hub filters by tenant audience (owner and members). The handler also filters by the *credential's* scope, because a token may be limited to a subset of tenants. Keep the hub's owner/member maps in sync on every share, unshare or ownership change. A stale map either leaks events or silently drops them.
- **Heartbeats do double duty.** A comment line (`: heartbeat`) every ~15 s keeps proxies and load balancers from timing out idle streams, and it's a natural point to re-validate the session. Without that re-check, a revoked user keeps receiving events until they reload.
- **Never block in Publish.** Use a buffered channel per subscriber and drop on full. One stuck browser tab must not stall the producers. Clients recover by invalidating everything on (re)connect, which makes dropped events self-healing.
- **Subscribe before snapshotting** initial state. Otherwise an event published between snapshot and subscribe is lost.
- **Proxies buffer by default.** Disable buffering (Caddy `flush_interval -1`, nginx `proxy_buffering off` or `X-Accel-Buffering: no`) and don't compress the stream. The symptom is events showing up only after a reload or in bursts.
- **No `WriteTimeout`** on the `http.Server` for the stream route, and cancel request contexts in `RegisterOnShutdown`, or graceful shutdown hangs on open streams until its deadline.
- **HTTP/1.1 connection limit**: browsers allow ~6 connections per host over HTTP/1.1, and each tab's EventSource holds one. Serve over HTTP/2 (any TLS proxy such as Caddy does by default), or keep one stream per tab rather than one per tenant.
- **`EventSource` can't set headers.** Same-origin cookies work. Bearer-token clients (CLI, agents) should use `fetch` with a streaming reader instead, or the server can accept the credential another way. Avoid long-lived tokens in query strings, since they end up in logs.
- **Taps run synchronously** inside `Publish` while the read lock is held. A tap (e.g. outbound webhooks) must enqueue and return, never do network I/O inline, and never call `Publish` (that deadlocks against a waiting writer).
- Keep sensitive content out of event payloads where an id is enough: send `{account_id, id}` and let the client refetch through the authorized REST route.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
