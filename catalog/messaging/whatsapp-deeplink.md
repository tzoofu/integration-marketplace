# WhatsApp Deep Link

- **category**: messaging
- **provider**: WhatsApp / Meta
- **reusable**: yes — identical no-SDK, no-API-key `wa.me` deep-link pattern across all three repos; trivial to lift as a shared `@marketplace/whatsapp-link` helper.
- **docs**: https://faq.whatsapp.com/425247423114725 (Click to Chat / `wa.me` links)

## Overview
`wa.me` deep links open WhatsApp (web or native) with a phone number pre-selected and a message pre-filled in the compose box — no API key, no SDK, no WhatsApp Business API account. The user still has to hit send themselves; the app never sends a message programmatically. Used both for "contact this person" (per-item phone number) and "share to WhatsApp" (opens the picker with no fixed recipient).

## Playbook

### Prerequisites
- None. No account, API key, or business verification needed — this is a plain URL scheme.
- If you want a **branded button**, an inline SVG WhatsApp glyph (no external icon package required).

### Setup steps
1. Normalize the target phone number to E.164-ish digits-only with country code: strip all non-digits, then replace a leading local trunk `0` with the country code (e.g. Israel: `0` → `972`). If the "phone" field might already be a URL (e.g. an existing wa.me/messenger link stored in the same field), pass it through unchanged instead of mangling it.
2. Build the message text from a template with named placeholders (e.g. `{address}`, `{url}`) substituted from real data.
3. URL-encode the filled message and append it as `?text=` on `https://wa.me/<digits>`. Omit the phone segment entirely (`https://wa.me/?text=...`) for a generic "share" link with no fixed recipient.
4. Render as a plain `<a href=... target="_blank" rel="noopener noreferrer">` — no client library needed.
5. If per-user customizable templates are wanted, store the template string (with `{token}` placeholders) per-user/config, offer a small token-insertion UI, and cap the length (e.g. 500 chars) since WhatsApp truncates absurdly long pre-filled text anyway.

### Core pattern
```ts
// whatsapp-link.ts — generic wa.me deep-link builder
const PLACEHOLDER_RE = /\{(\w+)\}/g;

/** Strip non-digits and swap a leading local trunk "0" for the country code. */
export function normalizePhone(phone: string, countryCode = "972"): string {
  return phone.replace(/\D/g, "").replace(/^0/, countryCode);
}

/** Callback-form substitution — NOT sequential String#replaceAll calls.
 *  Two real pitfalls this avoids:
 *  1. Sequential replaceAll (one call per placeholder) lets a value substituted
 *     early (e.g. attacker-controlled user input) introduce literal "{token}"
 *     text that a LATER replaceAll call then matches and substitutes again.
 *  2. String#replaceAll(search, stringReplacement) treats the replacement string
 *     as a substitution pattern ($&, $`, $', $$ are live) — a user-supplied value
 *     of "$'" or "$&" can pull other parts of the template into the message.
 *     The callback form's return value is NOT run through GetSubstitution, so
 *     it's immune to this.
 */
export function fillTemplate(template: string, vars: Record<string, string>): string {
  return template.replace(PLACEHOLDER_RE, (match, key) => vars[key] ?? match);
}

/** Build a wa.me link to a specific contact, or a recipient-less share link
 *  when `phone` is omitted. */
export function buildWhatsAppHref(message: string, phone?: string): string {
  const base = phone ? `https://wa.me/${normalizePhone(phone)}` : "https://wa.me/";
  return `${base}?text=${encodeURIComponent(message)}`;
}

// Usage:
// const href = buildWhatsAppHref(
//   fillTemplate("Hi {name}! Interested in {item}: {link}", { name, item, link }),
//   contactPhone
// );
```

### Env vars
None — the recipient phone number is business/app config data, not a secret, and no API key is involved.

### Gotchas
- A stored "contact phone" field can sometimes already be a full URL (e.g. someone pasted a Messenger link into a phone field) — check for an `http` prefix and pass it through unchanged before attempting digit normalization, or you'll mangle a valid link.
- Sequential `String#replaceAll` calls (one per placeholder) are exploitable: an early-substituted value can itself contain literal `{otherToken}` text that a later `replaceAll` call re-substitutes. Use a single combined regex + callback-form replace instead (see Core pattern).
- `String#replaceAll(pattern, stringReplacement)` treats the replacement as a substitution pattern (`$&`, `` $` ``, `$'`, `$$`) — user-controlled values like a guest name of `"$'"` can leak other template content. Only the callback form of `replace`/`replaceAll` is safe against this.
- WhatsApp truncates very long pre-filled `text` values silently; cap user-editable templates client-side (e.g. 500 chars) rather than discovering the truncation point empirically.
- This is not the WhatsApp Business Platform / Cloud API — there's no message-sending, delivery receipts, or templates-for-business-verification involved. If a future use case needs the app to *send* messages programmatically (not just open a pre-filled compose box), that's a different, heavier integration (Meta's WhatsApp Business API), not this pattern.

### Playbook confidence: high

## Adoption
Used in **3** repos in this marketplace. Adopters vary mainly in two ways: whether the link targets a fixed per-recipient phone number or is recipient-less (a generic "share/invite" link with no fixed target), and whether the message template is a hardcoded string or a per-user-configurable template with insertable placeholder tokens and its own settings UI (autosaved on change, with a reset-to-default option). One adopter also treats this channel as a recognized inbound source name for content originating in a bridged chat group, alongside its outbound use.
