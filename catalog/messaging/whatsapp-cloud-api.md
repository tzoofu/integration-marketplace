# WhatsApp Business Cloud API (Meta Graph API, direct)

- **category**: messaging
- **provider**: Meta (WhatsApp Business Platform, Cloud API on `graph.facebook.com`)
- **reusable**: yes. The Graph client, error-code mapping, signed webhook receiver with per-number routing, 24-hour-window guard and template CRUD only depend on Meta's API. Storage and UI wiring are the app-specific parts.
- **docs**: https://developers.facebook.com/docs/whatsapp/cloud-api · https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks/components · https://developers.facebook.com/docs/whatsapp/business-management-api/message-templates · https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes

This is the official two-way WhatsApp channel for a business number, called directly on Meta's Graph API with no BSP or Twilio in between. Outbound covers text, media, location, reactions and approved templates. Inbound messages and delivery statuses arrive on a public, HMAC-signed webhook. Reach for it when you need a supported, ban-safe business number. The Cloud API cannot read a personal number's inbox or history.

## Playbook

### Prerequisites
- A Meta developer app with the **WhatsApp** product ("Connect on WhatsApp" use case), plus a WhatsApp Business Account (WABA). Meta's test number works for development, but it can only message up to 5 verified recipients.
- Four values per number:
  | value | where |
  |---|---|
  | Phone number ID | App → WhatsApp → API Setup, under the "From" number |
  | WABA ID | same page |
  | Access token | same page (temporary, ≤24h). Use a System User token for production |
  | App secret | App settings → Basic → App secret |
- A public **https** URL for the webhook. Behind a reverse proxy (or a tunnel such as ngrok in dev), expose only the webhook path.
- A secret store for the access token and app secret, encrypted at rest, for example AES-GCM with AAD bound to the account id.

### Setup steps
1. **Validate input shape locally.** Phone number ID and WABA ID are numeric (about 5–30 digits). The token and app secret must be non-empty and of bounded length.
2. **Check the credentials with Meta before saving.** `GET /{phone_number_id}?fields=id,display_phone_number,verified_name,code_verification_status,quality_rating,messaging_limit_tier,name_status`, then `GET /{waba_id}` and page `GET /{waba_id}/phone_numbers`. Refuse the account if the WABA doesn't own the number. Reject duplicates on `phone_number_id`.
3. **Subscribe the app to the WABA's webhooks** with `POST /{waba_id}/subscribed_apps`. Without this, Meta sends nothing even though the callback "verifies". Record whether it succeeded and retry on the periodic refresh.
4. **Generate a random verify token per account** (stored, unique) and show the user the callback URL `https://<public-host>/<webhook-path>` together with that token.
5. **In Meta → WhatsApp → Configuration (Webhooks):** enter the callback URL and verify token, then **Verify and save**. Meta sends the GET handshake. Subscribe the fields `messages` (required), `message_template_status_update`, `phone_number_quality_update`, `phone_number_name_update` and `account_update`. Don't enable `history`, which is only for numbers shared with the Business app (coexistence).
6. **Test.** From a verified phone, message the business number. The webhook POST arrives, the chat opens, and the 24-hour window starts, so replies now work.
7. **Production.** Business settings → Users → **System users** → add an admin system user → assign the app and the WABA → **Generate token** with `whatsapp_business_messaging` + `whatsapp_business_management` and expiry **Never**. Replace the temporary token with it. Then add a real number (one not registered in the WhatsApp apps), set a payment method on the WABA, and complete business verification to raise the messaging tier.
8. **Periodic health refresh** (for example every few hours, and right after `account_update` / `phone_number_*` webhooks). Re-read the phone fields, record `token_invalid` or `error` health, and re-attempt `subscribed_apps` if it failed earlier.

### Core pattern

**Graph client (Go). Never forward the token off-host.**
```go
package cloudapi

type HTTPGraph struct {
	BaseURL string // "https://graph.facebook.com/v23.0"
	Client  *http.Client
}

func NewHTTPGraph(version string) *HTTPGraph {
	return &HTTPGraph{
		BaseURL: "https://graph.facebook.com/" + version,
		Client: &http.Client{Timeout: 30 * time.Second, CheckRedirect: func(req *http.Request, via []*http.Request) error {
			if len(via) >= 3 {
				return errors.New("too many redirects")
			}
			if len(via) > 0 && req.URL.Host != via[0].URL.Host {
				req.Header.Del("Authorization") // media redirects to a CDN: drop the bearer token
			}
			return nil
		}},
	}
}

type OutMessage struct {
	To       string    `json:"to"`
	Type     string    `json:"type"` // text|image|video|audio|document|sticker|location|reaction|template
	Context  *struct{ MessageID string `json:"message_id"` } `json:"context,omitempty"` // reply
	Text     *struct {
		Body       string `json:"body"`
		PreviewURL bool   `json:"preview_url,omitempty"`
	} `json:"text,omitempty"`
	Image    *OutMedia `json:"image,omitempty"`
	Document *OutMedia `json:"document,omitempty"`
	Template *OutTmpl  `json:"template,omitempty"`
	// video, audio, sticker, location, reaction follow the same shape
}
type OutMedia struct {
	ID       string `json:"id"` // from UploadMedia
	Caption  string `json:"caption,omitempty"`
	Filename string `json:"filename,omitempty"`
}
type OutTmpl struct {
	Name       string `json:"name"`
	Language   struct{ Code string `json:"code"` } `json:"language"`
	Components []struct {
		Type       string `json:"type"` // "body"
		Parameters []struct{ Type, Text string } `json:"parameters"`
	} `json:"components,omitempty"`
}

func (g *HTTPGraph) Send(ctx context.Context, token, phoneNumberID string, msg OutMessage) (wamid, waID string, err error) {
	body := struct {
		MessagingProduct string `json:"messaging_product"`
		RecipientType    string `json:"recipient_type"`
		OutMessage
	}{"whatsapp", "individual", msg}
	var res struct {
		Contacts []struct{ WaID string `json:"wa_id"` } `json:"contacts"`
		Messages []struct{ ID string `json:"id"` }      `json:"messages"`
	}
	if err := g.jsonCall(ctx, token, "POST", url.PathEscape(phoneNumberID)+"/messages", body, &res); err != nil {
		return "", "", err
	}
	if len(res.Messages) == 0 {
		return "", "", errors.New("send answer carried no message id")
	}
	if len(res.Contacts) > 0 {
		waID = res.Contacts[0].WaID // may differ from what you sent (normalized number)
	}
	return res.Messages[0].ID, waID, nil
}

// Upload, then send by media id.
func (g *HTTPGraph) UploadMedia(ctx context.Context, token, phoneNumberID string, data []byte, mime, name string) (string, error) {
	var buf bytes.Buffer
	mw := multipart.NewWriter(&buf)
	mw.WriteField("messaging_product", "whatsapp")
	mw.WriteField("type", mime)
	h := textproto.MIMEHeader{}
	h.Set("Content-Disposition", fmt.Sprintf(`form-data; name="file"; filename=%q`, name))
	h.Set("Content-Type", mime)
	part, _ := mw.CreatePart(h)
	part.Write(data)
	mw.Close()
	var res struct{ ID string `json:"id"` }
	err := g.call(ctx, token, "POST", g.BaseURL+"/"+url.PathEscape(phoneNumberID)+"/media", &buf, mw.FormDataContentType(), &res)
	return res.ID, err
}

// Inbound media: GET /{media_id} -> {url, mime_type, file_size}; then GET url with the bearer token.
func (g *HTTPGraph) Download(ctx context.Context, token, mediaURL string, maxBytes int64) ([]byte, error) {
	if u, err := url.Parse(mediaURL); err != nil || u.Scheme != "https" {
		return nil, errors.New("refusing a non-https media URL")
	}
	req, _ := http.NewRequestWithContext(ctx, "GET", mediaURL, nil)
	req.Header.Set("Authorization", "Bearer "+token)
	resp, err := g.Client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	data, err := io.ReadAll(io.LimitReader(resp.Body, maxBytes+1))
	if int64(len(data)) > maxBytes {
		return nil, fmt.Errorf("media larger than %d bytes", maxBytes)
	}
	return data, err
}

// Read receipt, and the typing indicator (only offered on the read receipt of an inbound wamid).
func (g *HTTPGraph) Typing(ctx context.Context, token, phoneNumberID, inboundWamid string) error {
	return g.jsonCall(ctx, token, "POST", url.PathEscape(phoneNumberID)+"/messages", map[string]any{
		"messaging_product": "whatsapp", "status": "read", "message_id": inboundWamid,
		"typing_indicator": map[string]string{"type": "text"},
	}, nil)
}
```

**Template CRUD (WABA-scoped)**
```go
// list: GET /{waba_id}/message_templates?fields=id,name,language,status,category,components,rejected_reason&limit=100
//       follow paging.next only if it stays on the Graph host.
// create (body-only):
func (g *HTTPGraph) CreateTemplate(ctx context.Context, token, wabaID, name, lang, category, body string, examples []string) (id, status string, err error) {
	comp := map[string]any{"type": "BODY", "text": body} // {{1}}, {{2}}…
	if len(examples) > 0 {
		comp["example"] = map[string]any{"body_text": [][]string{examples}} // one sample per param
	}
	var res struct{ ID, Status, Category string }
	err = g.jsonCall(ctx, token, "POST", url.PathEscape(wabaID)+"/message_templates", map[string]any{
		"name": name, "language": lang, "category": category, "components": []any{comp},
	}, &res)
	return res.ID, res.Status, err // usually PENDING
}
// delete (all languages of a name): DELETE /{waba_id}/message_templates?name=<name>
```
Validate locally before calling Meta: the name matches `^[a-z0-9_]{1,512}$`, the category is `MARKETING | UTILITY | AUTHENTICATION`, the language is one Meta supports (`en_US`, `he`, …), the body is at most 1024 characters, and `len(examples)` equals the number of `{{n}}` placeholders. To send, require `status == APPROVED` and exactly `paramCount` parameters.

**Webhook: GET handshake + signed POST with per-number routing**
```go
// GET ?hub.mode=subscribe&hub.verify_token=…&hub.challenge=…  -> echo challenge as text/plain
func VerifySubscription(mode, token, challenge string, tokenKnown func(string) bool) (string, bool) {
	if mode != "subscribe" || token == "" || challenge == "" || len(challenge) > 256 || !tokenKnown(token) {
		return "", false // answer 403
	}
	return challenge, true
}

func validSignature(appSecret string, body []byte, header string) bool {
	hexSig, ok := strings.CutPrefix(strings.TrimSpace(header), "sha256=")
	if !ok {
		return false
	}
	raw, err := hex.DecodeString(hexSig)
	if err != nil {
		return false
	}
	mac := hmac.New(sha256.New, []byte(appSecret))
	mac.Write(body)
	return hmac.Equal(raw, mac.Sum(nil))
}

type hookPayload struct {
	Entry []struct {
		ID      string `json:"id"` // WABA id
		Changes []struct {
			Field string `json:"field"` // messages | message_template_status_update | phone_number_* | account_update
			Value struct {
				Metadata struct{ PhoneNumberID string `json:"phone_number_id"` } `json:"metadata"`
				Contacts []struct {
					Profile struct{ Name string `json:"name"` } `json:"profile"`
					WaID    string                            `json:"wa_id"`
				} `json:"contacts"`
				Messages []json.RawMessage `json:"messages"`
				Statuses []struct {
					ID, Status, Timestamp, RecipientID string
					Errors []struct{ Code int; Title string }
				} `json:"statuses"`
				Event, MessageTemplateName, MessageTemplateLanguage, Reason string // template status
			} `json:"value"`
		} `json:"changes"`
	} `json:"entry"`
}

// POST handler: read the RAW body (sign over the exact bytes), then:
func HandleWebhook(ctx context.Context, body []byte, sigHeader string) error {
	var p hookPayload
	if err := json.Unmarshal(body, &p); err != nil {
		return ErrBadRequest
	}
	// 1. Resolve every account the payload names: phone_number_id for "messages",
	//    WABA id (entry.id) for template/quality/account changes. Unknown numbers are ignored.
	targets := resolveTargets(ctx, p)
	if len(targets) == 0 {
		return nil // 200, nothing for us
	}
	// 2. The body must verify against the app secret of EVERY target, else 401 and apply nothing.
	for _, acc := range targets {
		if !validSignature(acc.AppSecret(), body, sigHeader) {
			return ErrBadSignature
		}
	}
	// 3. Apply (idempotently): messages keyed by wamid; statuses only move forward
	//    sent < delivered < read (failed is terminal); template status -> APPROVED/REJECTED/PAUSED…
	for _, e := range p.Entry {
		for _, ch := range e.Changes {
			applyChange(ctx, e.ID, ch, targets)
		}
	}
	return nil
}
```

**Send guard: 24-hour window + rate limit + error mapping**
```go
func (b *Bridge) prepare(ctx context.Context, chat string, isTemplate bool) error {
	if !isTemplate {
		last := lastInboundAt(b.accountID, chat) // set on every inbound webhook message
		if last == 0 || time.Now().Unix() >= last+24*3600 {
			return ErrWindowClosed // permanent: don't retry; send an approved template instead
		}
	}
	return b.waitForSlot(ctx) // golang.org/x/time/rate, refuse if the wait would exceed ~10s
}

// Map Graph error codes to actionable sentinels.
func (e *GraphError) Is(target error) bool {
	switch target {
	case ErrTokenInvalid:
		return e.Code == 190 || e.Code == 102 || (e.HTTPStatus == 401 && e.Code != 131047)
	case ErrWindowClosed:
		return e.Code == 131047
	case ErrRateLimited:
		return e.Code == 4 || e.Code == 80007 || e.Code == 130429 || e.Code == 131048 || e.Code == 131056 || e.HTTPStatus == 429
	case ErrTemplate:
		return e.Code >= 132000 && e.Code < 133000
	case ErrNotFound:
		return e.Code == 100 && (e.Subcode == 33 || e.HTTPStatus == 404)
	}
	return false
}
```

### Env vars
- `APP_PUBLIC_URL`: public https base used to build the callback URL.
- `META_GRAPH_VERSION` (e.g. `v23.0`).
- `APP_CLOUD_SEND_RATE`: per-number sends per second (the source defaults to about 20/s, below Meta's per-number throughput cap).
- `APP_SECRET_KEY`: key for encrypting stored access tokens and app secrets.
- Per-account values (phone number ID, WABA ID, access token, app secret, verify token) live encrypted in the DB, not in env.

### Gotchas
- **Temporary vs permanent token.** The API Setup token lasts at most 24h, and Meta kills it sooner if the developer logs out of Facebook or changes their password. Symptoms are error 190 / HTTP 401 "token invalid" a few hours after setup. Don't keep regenerating it. Create a System User token with expiry "Never" and the two `whatsapp_business_*` permissions. The IDs and app secret stay the same, so a "replace token" action is enough.
- **Verification ≠ delivery.** The GET handshake succeeding proves nothing about messages. You also need the `messages` field subscribed **and** `POST /{waba_id}/subscribed_apps` done for that WABA. If nothing arrives, check both before suspecting your code. A POST that arrives with a 401 means the stored app secret is wrong.
- **Sign over the raw body.** Compute the HMAC on the exact bytes received, before any JSON re-encoding, and compare with `hmac.Equal`. With several apps or accounts behind one endpoint, resolve the payload's targets first and require a valid signature for **every** account it names. One app's secret must never let it write into another app's numbers.
- **Route by `phone_number_id`, not display number.** `messages` changes carry `value.metadata.phone_number_id`. Template, quality and account changes only carry the WABA id in `entry.id`, which may map to several numbers.
- **Webhooks are redelivered.** Insert inbound messages idempotently on the `wamid`. Status updates can arrive out of order, so only move status forward. `failed` carries `errors[0].code/title`, which you should surface to the user.
- **24-hour customer service window.** Free-form messages are allowed only within 24h of the contact's last inbound message (error 131047). A contact who never wrote starts with the window closed. Templates bypass it. Store `last_inbound_at` per chat and check it locally so users get a clear error, and so a scheduler marks the job as permanently failed instead of retrying.
- **No history, no groups.** The Cloud API only shows messages that pass through your webhook after you connect. Groups, polls, edits, status updates, presence and QR pairing don't exist here, so hide those controls for this account type. Messages sent from Meta's dashboard bypass your webhook entirely.
- **Typing indicator = read receipt.** Meta only offers "typing" attached to marking an inbound wamid read, inside the 24h window. Meta clears it after about 25s or when you reply, and there is no "stopped typing" call.
- **Media flow is two hops.** `GET /{media_id}` returns a short-lived signed CDN URL. Fetch it with the bearer token, refuse non-https URLs, cap the size, and strip `Authorization` on cross-host redirects. Keep signed media URLs out of error messages and logs.
- **Use the canonical `wa_id`.** Meta may answer a send with a normalized `wa_id` that differs from the number you sent. Key the chat by the returned id.
- **Template lifecycle.** New templates are `PENDING`. Approval, rejection (`reason`, where `NONE` means no reason) and pausing arrive via `message_template_status_update`. Approved templates can't be edited, so create a new name instead. Deleting removes every language of that name.
- **Quality and tier.** `quality_rating` (GREEN/YELLOW/RED) and `messaging_limit_tier` gate how many unique users a number can open conversations with per 24h. Show them, and refresh on `phone_number_quality_update`.
- **Paging links.** Follow `paging.next` only when it stays on the Graph host, and cap the page count.
- **Rate limits.** Errors 4, 80007, 130429, 131048 and 131056 (and HTTP 429) all mean back off. Pair-rate limiting (131056) is per recipient, so don't burst many messages to the same user.
- **Test number limits.** Meta's test number can only message up to 5 recipients you verified in the dashboard.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
