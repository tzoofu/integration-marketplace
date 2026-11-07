# SSRF-safe link preview (Open Graph fetch)

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — a self-contained Go fetcher (dial-time IP filter, size/time limits, redirect cap, regex meta parsing) with no app-specific code; any app that unfurls user-supplied URLs server-side can reuse it as is.
- **docs**: https://ogp.me/ · https://owasp.org/www-community/attacks/Server_Side_Request_Forgery

Builds the preview card (title, description, thumbnail) for a URL found in a user's outgoing message by fetching the page server-side. Because the URL is user-controlled, the fetcher refuses to connect to loopback, private, link-local, CGNAT and multicast addresses — checked on the resolved IP at dial time, for the page, its redirects and the image — and caps time, bytes and redirects.

## Playbook

### Prerequisites
- Go 1.18+ (`net/netip`) — or any HTTP client that exposes a pre-connect hook on the resolved IP.
- Somewhere to keep the result: on the message row itself, and a content-addressed media store for the thumbnail.

### Setup steps
1. Extract the first `https?://…` URL from the message text and trim trailing punctuation (`.,;:!?)]}'"`).
2. Build one shared `http.Client` whose `net.Dialer.Control` parses the dialed address with `netip.ParseAddr`, `Unmap()`s it, and rejects anything that isn't a public unicast address. Set `Proxy: nil`.
3. Limit everything: 3s overall timeout (dial, TLS, headers, client), 512 KiB for HTML, 256 KiB for the image, max 3 redirects and only to http/https.
4. GET the page with a browser-like `User-Agent` and `Accept: text/html,application/xhtml+xml,image/*;q=0.8`. Require status 200 and a `text/html` or `application/xhtml+xml` content type.
5. Parse `<meta>` tags with a tolerant regex: `og:title`, `og:description`, `description`, `og:image`/`og:image:url`; fall back to `<title>`. HTML-unescape, collapse whitespace, truncate (title 200, description 300 runes).
6. Resolve `og:image` relative to the page URL, fetch it through the same client, and keep it only if it's `image/jpeg` or `image/png` within the size cap.
7. If there's no title and no description, return "no preview" rather than an empty card.
8. Cache: store the preview (url, title, description, thumbnail hash) with the sent message so it's never re-fetched for display; store the thumbnail by its SHA-256 so repeats dedupe. For high send rates add a small in-memory TTL cache keyed by URL (including negative results).
9. Treat every error as "send without a preview" and log it at debug — never block or fail the send because a preview failed. Respect a per-message "no preview" flag.

### Core pattern
```go
package linkpreview

import (
	"context"
	"errors"
	"fmt"
	"html"
	"io"
	"mime"
	"net"
	"net/http"
	"net/netip"
	"net/url"
	"regexp"
	"strings"
	"syscall"
	"time"
)

const (
	pageLimit  = 512 << 10
	imageLimit = 256 << 10
	timeout    = 3 * time.Second
)

var ErrBlocked = errors.New("address not allowed")

type Preview struct {
	URL, Title, Description string
	Image                   []byte
	ImageMime               string
}

type Fetcher struct{ client *http.Client }

var cgnat = netip.MustParsePrefix("100.64.0.0/10")

func publicOnly(ip netip.Addr) bool {
	return ip.IsValid() && !ip.IsLoopback() && !ip.IsPrivate() && !ip.IsLinkLocalUnicast() &&
		!ip.IsLinkLocalMulticast() && !ip.IsUnspecified() && !ip.IsMulticast() &&
		!ip.IsInterfaceLocalMulticast() && !cgnat.Contains(ip)
}

func New() *Fetcher { return newFetcher(publicOnly) } // tests inject an allow-loopback func for httptest

func newFetcher(allow func(netip.Addr) bool) *Fetcher {
	dialer := &net.Dialer{Timeout: timeout, Control: func(_, address string, _ syscall.RawConn) error {
		host, _, err := net.SplitHostPort(address)
		if err != nil {
			return err
		}
		ip, err := netip.ParseAddr(host)
		if err != nil || !allow(ip.Unmap()) { // Unmap: ::ffff:127.0.0.1 must count as loopback
			return fmt.Errorf("%w: %s", ErrBlocked, host)
		}
		return nil
	}}
	tr := &http.Transport{DialContext: dialer.DialContext, Proxy: nil,
		TLSHandshakeTimeout: timeout, ResponseHeaderTimeout: timeout, MaxIdleConns: 4}
	return &Fetcher{client: &http.Client{Transport: tr, Timeout: timeout,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			if len(via) >= 3 {
				return errors.New("too many redirects")
			}
			if req.URL.Scheme != "http" && req.URL.Scheme != "https" {
				return errors.New("redirect to a non-http url")
			}
			return nil // the new host is still dial-checked by Control
		}}}
}

var urlRe = regexp.MustCompile(`https?://[^\s<>"]+`)

func FirstURL(text string) string {
	return strings.TrimRight(urlRe.FindString(text), ".,;:!?)]}'\"")
}

func (f *Fetcher) Fetch(ctx context.Context, raw string) (*Preview, error) {
	u, err := url.Parse(raw)
	if err != nil || (u.Scheme != "http" && u.Scheme != "https") || u.Host == "" {
		return nil, errors.New("not an http(s) url")
	}
	ctx, cancel := context.WithTimeout(ctx, timeout) // one budget for page + image
	defer cancel()
	body, ctype, err := f.get(ctx, u.String(), pageLimit)
	if err != nil {
		return nil, err
	}
	if mt, _, _ := mime.ParseMediaType(ctype); mt != "text/html" && mt != "application/xhtml+xml" {
		return nil, errors.New("not an html page")
	}
	p, imgURL := parse(string(body))
	p.URL = raw
	if p.Title == "" && p.Description == "" {
		return nil, errors.New("page has no title")
	}
	if imgURL != "" {
		if iu, err := u.Parse(imgURL); err == nil && (iu.Scheme == "http" || iu.Scheme == "https") {
			if data, it, err := f.get(ctx, iu.String(), imageLimit); err == nil &&
				(strings.HasPrefix(it, "image/jpeg") || strings.HasPrefix(it, "image/png")) {
				p.Image, p.ImageMime = data, strings.SplitN(it, ";", 2)[0]
			}
		}
	}
	return &p, nil
}

func (f *Fetcher) get(ctx context.Context, u string, limit int64) ([]byte, string, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, u, nil)
	if err != nil {
		return nil, "", err
	}
	req.Header.Set("User-Agent", "Mozilla/5.0 (compatible; AppLinkPreview/1.0)")
	req.Header.Set("Accept", "text/html,application/xhtml+xml,image/*;q=0.8")
	resp, err := f.client.Do(req)
	if err != nil {
		return nil, "", err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return nil, "", fmt.Errorf("status %d", resp.StatusCode)
	}
	if resp.ContentLength > limit {
		return nil, "", errors.New("too large")
	}
	data, err := io.ReadAll(io.LimitReader(resp.Body, limit+1))
	if err != nil {
		return nil, "", err
	}
	if int64(len(data)) > limit {
		if strings.HasPrefix(resp.Header.Get("Content-Type"), "text/html") {
			data = data[:limit] // the <head> of a big page still has its meta tags
		} else {
			return nil, "", errors.New("too large")
		}
	}
	return data, resp.Header.Get("Content-Type"), nil
}

var (
	metaRe  = regexp.MustCompile(`(?is)<meta\s+[^>]*>`)
	attrRe  = regexp.MustCompile(`(?is)([a-z:_-]+)\s*=\s*("([^"]*)"|'([^']*)')`)
	titleRe = regexp.MustCompile(`(?is)<title[^>]*>(.*?)</title>`)
)

func parse(page string) (p Preview, imageURL string) {
	var desc string
	for _, tag := range metaRe.FindAllString(page, -1) {
		attrs := map[string]string{}
		for _, a := range attrRe.FindAllStringSubmatch(tag, -1) {
			attrs[strings.ToLower(a[1])] = a[3] + a[4]
		}
		switch strings.ToLower(attrs["property"] + attrs["name"]) {
		case "og:title":
			p.Title = clean(attrs["content"], 200)
		case "og:description":
			p.Description = clean(attrs["content"], 300)
		case "description":
			desc = clean(attrs["content"], 300)
		case "og:image", "og:image:url":
			if imageURL == "" {
				imageURL = strings.TrimSpace(html.UnescapeString(attrs["content"]))
			}
		}
	}
	if p.Title == "" {
		if m := titleRe.FindStringSubmatch(page); m != nil {
			p.Title = clean(m[1], 200)
		}
	}
	if p.Description == "" {
		p.Description = desc
	}
	return p, imageURL
}

func clean(s string, max int) string {
	s = strings.Join(strings.Fields(html.UnescapeString(s)), " ")
	if r := []rune(s); len(r) > max {
		s = string(r[:max]) + "…"
	}
	return s
}
```

Caller (never fails the send):

```go
func (s *Sender) previewFor(ctx context.Context, text string, noPreview bool) *linkpreview.Preview {
	u := linkpreview.FirstURL(text)
	if noPreview || u == "" {
		return nil
	}
	if p, ok := s.cache.Get(u); ok { // optional TTL cache, caches misses as nil too
		return p
	}
	p, err := s.previews.Fetch(ctx, u)
	if err != nil {
		s.log.Debug("link preview", "err", err) // debug only: URLs can be private
		p = nil
	}
	s.cache.Set(u, p, 10*time.Minute)
	return p
}
```

### Env vars
None.

### Gotchas
- The IP check must run on the resolved address at dial time (`Dialer.Control`), not on the host name before the request: a public-looking host can resolve to `127.0.0.1`, and DNS rebinding can switch answers between a pre-check and the connect. Control also runs for every redirect hop and for the image fetch, so they're covered for free.
- Call `Unmap()` before classifying: IPv4-mapped IPv6 (`::ffff:10.0.0.1`) otherwise slips past `IsPrivate`/`IsLoopback`.
- Block CGNAT `100.64.0.0/10` too — it's not in `IsPrivate` but is internal in practice (and one cloud's metadata endpoint lives there). Link-local covers `169.254.169.254`.
- `Proxy: nil` is required; with an environment proxy the dial goes to the proxy and the filter never sees the real target.
- Cap redirects (3) and allow only http/https schemes on each hop.
- Use one context timeout for the whole operation (page + image) so a slow image can't extend the total past the budget. Keep it short (≈3s) because it runs on the send path.
- Read through `io.LimitReader(limit+1)` and check `Content-Length` first. For oversize HTML, truncate rather than fail — the meta tags are in `<head>`. For oversize images, drop the image.
- Check `Content-Type` before parsing (don't run regexes over a PDF or a video), and only accept JPEG/PNG thumbnails — some clients and messaging protocols reject WebP/SVG thumbnails, and SVG can carry script.
- A regex-based meta parser is intentionally tolerant of broken HTML and avoids a full DOM parse; handle both `property=` and `name=` attributes, both quote styles, and HTML entities.
- Resolve relative `og:image` URLs against the *page* URL (`u.Parse(img)`).
- Return nothing when the page has neither title nor description, and treat every failure as "no preview": previews are best-effort and must never block or fail sending.
- Some platform APIs generate previews themselves (you only pass a "preview URL" flag); use this fetcher only for transports that need you to supply title/description/thumbnail.
- For tests, inject an `allow` func that permits loopback so `httptest` servers work, and add a test that the default fetcher refuses `127.0.0.1`.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
