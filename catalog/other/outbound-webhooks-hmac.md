# Outbound webhooks (HMAC-signed, SSRF-guarded)

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — an event filter, a durable outbox with retries, HMAC signing and a dial-time SSRF guard are app-agnostic; only the event types and payload `data` change per app.
- **docs**: n/a (receiver conventions follow the common `sha256=<hex>` scheme; see https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries)

Lets each user register HTTP endpoints that receive selected app events as signed JSON POSTs. Events are written to a durable outbox and delivered by background workers with exponential backoff, a per-endpoint delivery log, auto-disable after repeated failures, and an SSRF guard that checks the actual dialed IP so endpoints can't be pointed at cloud metadata or internal services.

## Playbook

### Prerequisites
- An internal event bus/hub the app already publishes to (or a place in the code where domain events happen).
- A SQL store for endpoints and the delivery outbox.
- An encryption helper (e.g. AES-256-GCM with a server key) for the per-user signing secret.

### Setup steps
1. Tables: `endpoints(id, user_id, url, description, events JSON, enabled, failure_count, last_status, last_error, last_delivery_at, last_success_at, disabled_reason, created_at, updated_at)` and `outbox(id, endpoint_id, user_id, resource_id, event_id, event_type, payload, status queued|delivering|delivered|failed, attempts, next_attempt_at, last_status, last_error, created_at, updated_at)`.
2. Define subscribable event types (e.g. `item.created`, `item.status`, `account.status`) plus a non-subscribable `webhook.test`.
3. Per-user signing secret: generate 24+ random bytes (hex, with a recognizable prefix such as `whs_`), store it **encrypted** with AAD bound to the owner row (e.g. `"webhook_secret:<user_id>"`). Offer "reveal" and "rotate" from the dashboard only.
4. Tap the event bus with a non-blocking buffered channel. An intake goroutine maps each internal event to a webhook type (drop ones that don't map, e.g. the user's own outgoing actions), builds the JSON envelope, and inserts one outbox row per enabled endpoint subscribed to that type — in one transaction.
5. Delivery workers (2 is plenty) claim the next due row atomically (`UPDATE … SET status='delivering', attempts=attempts+1 WHERE id=(SELECT … ORDER BY next_attempt_at LIMIT 1) RETURNING …`), decrypt the secret, re-check the URL, POST, and record the result.
6. Retry schedule: 10s, 1m, 5m, 30m, 2h, 6h (7 attempts ≈ 8.5h). After ~50 consecutive failed attempts, disable the endpoint with a `disabled_reason` and fail its queued rows.
7. SSRF guard: validate the URL at save time (`CheckURL`) and again on every attempt; enforce the IP rules in the `net.Dialer.Control` hook so DNS rebinding can't bypass them.
8. Routes (dashboard-session only): list/create/update/delete endpoints, `POST …/{id}/test` (send `webhook.test` synchronously and return the recorded delivery), `GET …/{id}/deliveries` (log), secret reveal/rotate.
9. Prune the outbox every few hours (keep ~7 days). On startup, reset `delivering` rows to `queued`.

### Core pattern
```go
package webhooks

import (
	"bytes"
	"context"
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/url"
	"strconv"
	"strings"
	"syscall"
	"time"
)

const requestTimeout = 10 * time.Second

var backoff = []time.Duration{10 * time.Second, time.Minute, 5 * time.Minute, 30 * time.Minute, 2 * time.Hour, 6 * time.Hour}

func NewSecret() string {
	b := make([]byte, 24)
	if _, err := rand.Read(b); err != nil {
		panic(err)
	}
	return "whs_" + hex.EncodeToString(b)
}

// Sign returns the X-Signature header value. When a timestamp is used, sign "<ts>.<body>"
// and send it as X-Webhook-Timestamp so receivers can reject replays.
func Sign(secret string, ts int64, body []byte) string {
	m := hmac.New(sha256.New, []byte(secret))
	fmt.Fprintf(m, "%d.", ts)
	m.Write(body)
	return "sha256=" + hex.EncodeToString(m.Sum(nil))
}

type Envelope struct {
	ID         string `json:"id"`          // evt_<random hex>, stable across retries
	Type       string `json:"type"`
	CreatedAt  string `json:"created_at"`  // RFC 3339
	ResourceID *int64 `json:"resource_id"` // null for user-level events
	Data       any    `json:"data"`
}

// ---- SSRF guard -------------------------------------------------------------

var ErrBlocked = errors.New("webhook target address is not allowed")

type Guard struct {
	AllowPrivate bool // opt-in for self-hosters delivering to LAN services
	OwnPort      int  // never deliver to this app's own HTTP port on a local address
}

var metadataIPs = []net.IP{net.ParseIP("169.254.169.254"), net.ParseIP("fd00:ec2::254"), net.ParseIP("100.100.100.200")}
var cgnat = &net.IPNet{IP: net.IPv4(100, 64, 0, 0), Mask: net.CIDRMask(10, 32)}

func (g *Guard) CheckIP(ip net.IP, port int) error {
	for _, m := range metadataIPs {
		if ip.Equal(m) {
			return fmt.Errorf("%w: cloud metadata address", ErrBlocked)
		}
	}
	if ip.IsUnspecified() || ip.IsLinkLocalUnicast() || ip.IsLinkLocalMulticast() || ip.IsMulticast() ||
		ip.IsInterfaceLocalMulticast() || ip.Equal(net.IPv4bcast) || (ip.To4() != nil && ip.To4()[0] == 0) {
		return fmt.Errorf("%w: %s", ErrBlocked, ip) // always refused, flag or not
	}
	local := ip.IsLoopback() || ip.IsPrivate() || cgnat.Contains(ip)
	if local && g.OwnPort != 0 && port == g.OwnPort {
		return fmt.Errorf("%w: the app's own port", ErrBlocked)
	}
	if local && !g.AllowPrivate {
		return fmt.Errorf("%w: %s is private or loopback", ErrBlocked, ip)
	}
	return nil
}

func (g *Guard) CheckURL(raw string) error {
	if len(raw) > 2048 {
		return errors.New("url is too long")
	}
	u, err := url.Parse(raw)
	if err != nil || (u.Scheme != "http" && u.Scheme != "https") || u.Host == "" {
		return errors.New("url must be an absolute http or https URL")
	}
	if u.User != nil {
		return errors.New("url must not contain credentials")
	}
	port := map[string]int{"http": 80, "https": 443}[u.Scheme]
	if p := u.Port(); p != "" {
		if port, err = strconv.Atoi(p); err != nil || port < 1 || port > 65535 {
			return errors.New("invalid port")
		}
	}
	host := strings.ToLower(strings.TrimSuffix(u.Hostname(), "."))
	if ip := net.ParseIP(host); ip != nil {
		return g.CheckIP(ip, port)
	}
	if host == "localhost" || strings.HasSuffix(host, ".localhost") {
		return g.CheckIP(net.IPv4(127, 0, 0, 1), port)
	}
	return nil // host names are checked again on the resolved IP at dial time
}

// control runs on the exact resolved address being dialed.
func (g *Guard) control(_, address string, _ syscall.RawConn) error {
	host, portStr, err := net.SplitHostPort(address)
	if err != nil {
		return err
	}
	ip := net.ParseIP(host)
	if ip == nil {
		return fmt.Errorf("%w: unresolved %q", ErrBlocked, host)
	}
	port, _ := strconv.Atoi(portStr)
	return g.CheckIP(ip, port)
}

func NewClient(g *Guard) *http.Client {
	d := &net.Dialer{Timeout: 5 * time.Second, Control: g.control}
	return &http.Client{
		Timeout: requestTimeout,
		Transport: &http.Transport{
			Proxy:                 nil, // a proxy would dial on our behalf and bypass the check
			DialContext:           d.DialContext,
			TLSHandshakeTimeout:   5 * time.Second,
			ResponseHeaderTimeout: requestTimeout,
			MaxIdleConns:          20,
			IdleConnTimeout:       60 * time.Second,
		},
		CheckRedirect: func(*http.Request, []*http.Request) error { return http.ErrUseLastResponse }, // 3xx = failure
	}
}

// ---- one delivery attempt ---------------------------------------------------

type Result struct {
	OK         bool
	HTTPStatus int
	Error      string
	RetryAt    int64
}

func Attempt(ctx context.Context, c *http.Client, g *Guard, secret, endpoint, eventID, eventType string, body []byte) (res Result, permanent bool) {
	if err := g.CheckURL(endpoint); err != nil {
		return Result{Error: err.Error()}, true
	}
	ctx, cancel := context.WithTimeout(ctx, requestTimeout)
	defer cancel()
	ts := time.Now().Unix()
	req, _ := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("User-Agent", "App-Webhooks/1")
	req.Header.Set("X-Webhook-Id", eventID)
	req.Header.Set("X-Webhook-Event", eventType)
	req.Header.Set("X-Webhook-Timestamp", strconv.FormatInt(ts, 10))
	req.Header.Set("X-Signature", Sign(secret, ts, body))
	resp, err := c.Do(req)
	if err != nil {
		var ue *url.Error
		if errors.As(err, &ue) {
			err = ue.Err // *url.Error repeats the URL, which may hold a secret path/query
		}
		msg := err.Error()
		if len(msg) > 300 {
			msg = msg[:300]
		}
		return Result{Error: msg}, false
	}
	io.Copy(io.Discard, io.LimitReader(resp.Body, 64<<10)) // drain a little for keep-alive, never store it
	resp.Body.Close()
	res = Result{HTTPStatus: resp.StatusCode, OK: resp.StatusCode >= 200 && resp.StatusCode < 300}
	if !res.OK {
		res.Error = fmt.Sprintf("HTTP %d", resp.StatusCode)
	}
	return res, false
}

// Worker side:
//   res, permanent := Attempt(...)
//   if !res.OK && !permanent && attempts < 7 {
//       res.RetryAt = now.Add(backoff[min(attempts-1, len(backoff)-1)]).Unix()
//   }
//   store.FinishAttempt(context.WithoutCancel(ctx), delivery, res, disableAfter /*50*/)
```

Receiver verification (Node/Express):

```ts
import crypto from "node:crypto";
import express from "express";

const app = express();
const SECRET = process.env.WEBHOOK_SECRET!;
const seen = new Set<string>(); // use a TTL store (Redis) in production

app.post("/hooks/app", express.raw({ type: "application/json" }), (req, res) => {
  const ts = Number(req.header("X-Webhook-Timestamp"));
  if (!Number.isFinite(ts) || Math.abs(Date.now() / 1000 - ts) > 300) return res.sendStatus(400);

  const expected =
    "sha256=" + crypto.createHmac("sha256", SECRET).update(`${ts}.`).update(req.body).digest("hex");
  const got = req.header("X-Signature") ?? "";
  if (got.length !== expected.length || !crypto.timingSafeEqual(Buffer.from(got), Buffer.from(expected)))
    return res.sendStatus(401);

  const id = req.header("X-Webhook-Id")!;
  if (seen.has(id)) return res.sendStatus(200); // retries reuse the same id: dedupe
  seen.add(id);

  const event = JSON.parse(req.body.toString("utf8"));
  queueForProcessing(event); // ack fast, work async
  res.sendStatus(204);
});
```

Python receiver:

```python
import hashlib, hmac, time

def verify(secret: str, body: bytes, ts_header: str, sig_header: str) -> bool:
    ts = int(ts_header)
    if abs(time.time() - ts) > 300:
        return False
    mac = hmac.new(secret.encode(), f"{ts}.".encode() + body, hashlib.sha256).hexdigest()
    return hmac.compare_digest("sha256=" + mac, sig_header)
```

### Env vars
- `APP_WEBHOOK_ALLOW_PRIVATE` — `true` lets endpoints target loopback/RFC 1918/CGNAT addresses (useful for self-hosters delivering to LAN services); metadata and link-local are blocked regardless.
- `APP_SECRET_KEY` (or equivalent) — the server key used to encrypt stored signing secrets.

### Gotchas
- Check IPs in `Dialer.Control`, not only when the URL is saved: a host name can resolve (or re-resolve, DNS rebinding) to an internal address later. Control sees the exact IP being dialed for every connection attempt.
- Set `Proxy: nil` on the transport. With `ProxyFromEnvironment`, the proxy dials the target and your IP check only sees the proxy.
- Disable redirects (`http.ErrUseLastResponse`) and count 3xx as failure; following a redirect would let a public endpoint bounce the request to an internal URL.
- Always block cloud metadata IPs (`169.254.169.254`, `fd00:ec2::254`, `100.100.100.200`), link-local, multicast, unspecified and `0.0.0.0/8` — even when the private-network flag is on. Treat `100.64.0.0/10` (CGNAT) as private. Never allow the app's own port on local addresses, or a webhook can call the app's own API.
- Reject URLs with userinfo (`https://user:pass@…`) and cap URL length; normalize trailing dots on host names before the `localhost` check.
- Strip the URL from transport errors (`*url.Error`) before storing them in the delivery log — endpoint URLs often embed a secret token in the path or query.
- Sign the exact bytes you send, and have receivers verify against the raw body (Express `express.raw`, not `express.json`) with a constant-time compare. Including a timestamp in the signed string plus a ±5 min window gives replay protection; a stable event id across retries lets receivers dedupe.
- Keep the event intake non-blocking (buffered channel + `select default`) so a slow DB never stalls the main event bus; log when an event is dropped.
- Write outbox rows in one transaction per event and deliver from the DB, so an endpoint outage or process restart loses nothing. Requeue `delivering` rows on startup.
- Classify permanent failures (missing/undecryptable secret, URL now blocked) and stop retrying those immediately; network errors and non-2xx retry with backoff.
- Auto-disable an endpoint after many consecutive failures and surface `disabled_reason` in the UI; fail its remaining queued rows instead of letting them pile up.
- Only drain a small bounded slice of the response body (for connection reuse) and never store response bodies.
- Encrypt the signing secret at rest with AAD tied to its owner row so a sealed value copied to another row won't decrypt. Manage endpoints, secret reveal/rotate and the delivery log from the signed-in dashboard only, not with API tokens.
- Offer a "Send test" action that delivers synchronously and returns the delivery record — it's the fastest way for users to debug signatures and firewalls.
- Prune the delivery log on a timer (e.g. older than 7 days) so the outbox table stays small.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
