# Bit — payment deep-link

- **category**: payments
- **provider**: Bit (bitpay.co.il) — Israeli P2P payment app, owned by Bank Hapoalim
- **reusable**: yes — trivial to lift as a generic "deep-link payment button" pattern; no API keys, no webhooks, no server-side integration at all. See [paybox-payment](paybox-payment.md) for the sibling integration, which shares the same pattern.
- **docs**: https://www.bitpay.co.il/ (no public developer API; the "Pay Me" link is generated from the Bit consumer app's own settings)

## Overview
Bit has no merchant API — businesses accept payment by sharing a personal "Pay Me" deep link (or phone number) that opens the Bit app directly to a payment screen. This integration is nothing more than displaying that pre-generated static link as a button at checkout and again on an order-tracking page, alongside a plain-text instruction to pay by phone number as a fallback.

## Playbook

### Prerequisites
- A Bit account for the receiving business/individual.
- Generate a personal "Pay Me" link from inside the Bit consumer app (Settings → Pay Me / קבלת תשלום) — this is a one-time manual step done by a human, not an API call.
- The business's phone number, for the manual fallback instruction ("pay via Bit to this number") shown alongside the link.

### Setup steps
1. Generate the "Pay Me" link inside the Bit app and copy the full URL it produces (looks like `https://www.bitpay.co.il/app/me/<opaque-id>`).
2. Store that URL as a single config value (env var or admin-configurable setting) — treat it as configuration, not a secret; it's meant to be shown publicly.
3. At checkout, when "Bit" is the selected/enabled payment method, render a button/link using that URL with `target="_blank" rel="noopener noreferrer"`, plus the business phone number as a text fallback for users without the deep link.
4. Repeat the same button on any post-checkout tracking/status page, since the customer often opens the payment app after placing the order rather than during checkout.
5. Since there's no callback/webhook, mark the order as paid manually (e.g. an admin flips a status field once payment is confirmed via the Bit app's own transaction history) — bake this into your order-status model, don't try to auto-detect payment.
6. Only render the button when the configured link is non-empty — treat a missing/blank value as "this payment method isn't available," not an error.

### Core pattern
```tsx
// Config: a single fully-formed URL, generated manually via the Bit app UI.
const BIT_PAY_LINK = process.env.NEXT_PUBLIC_BIT_PAY_LINK ?? "";

function BitPaymentButton({ businessPhone }: { businessPhone: string }) {
  if (!BIT_PAY_LINK) return null; // treat missing config as "method unavailable"

  return (
    <div>
      <p>
        Pay via Bit to <span dir="ltr">{businessPhone}</span>, or use the button below.
      </p>
      <a href={BIT_PAY_LINK} target="_blank" rel="noopener noreferrer">
        Open Bit and pay
      </a>
    </div>
  );
}
```

### Env vars
- `NEXT_PUBLIC_BIT_LINK` (or equivalent public config key — this is a publicly-shown link, not a secret, so a `NEXT_PUBLIC_`/client-exposed var is appropriate)

### Gotchas
- **There is no API.** Don't go looking for a REST endpoint, SDK, or webhook — the entire "integration" is a static link generated once by a human inside the Bit consumer app. Any request/response shape you might expect from a "payment provider" doesn't exist here.
- **No payment confirmation callback.** The app has no way to know the payment succeeded. Orders paid via Bit must be reconciled manually (an admin marks the order paid after checking the Bit app's transaction list, sometimes cross-referenced against a screenshot the customer sends via WhatsApp).
- **Show the button in two places, not one**: at the checkout step (where the user picks the payment method) and again on the persistent order-tracking page, since users frequently pay after checkout completes rather than during it.
- **Treat the link as public, not secret** — it's designed to be shared/displayed, so a client-exposed env var is correct; don't route it through server-only secret storage.
- **Amount is not embedded in the link** in the observed implementation — the app shows the amount as plain text next to the button and expects the customer to enter it manually in the Bit app. If Bit's link format supports amount/reference query params for your account type, confirm that independently — it wasn't used in the source implementation.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. The deep-link button is configured as a single static URL and rendered at both the checkout step and the order-tracking page, with no API or webhook involved.
