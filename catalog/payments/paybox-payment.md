# Paybox — payment deep-link

- **category**: payments
- **provider**: Paybox (payboxapp.com) — Israeli P2P/business payment app
- **reusable**: yes — identical pattern to [bit-payment](bit-payment.md): a pre-generated static "pay me" link with no API, no keys, no webhooks.
- **docs**: https://payboxapp.com/ (no public developer API; the personal payment link is generated from inside the Paybox app itself)

## Overview
Like Bit, Paybox has no merchant/developer API for this use case. The business generates a personal payment link once from inside the Paybox app and the storefront simply displays it as a button (plus the business phone number as a manual fallback) wherever a "pay via Paybox" option is offered.

## Playbook

### Prerequisites
- A Paybox account for the receiving business/individual.
- Generate a personal payment ("Pay Me") link from inside the Paybox app's own settings — a manual, one-time step, not an API call.
- The business's phone number, shown alongside the link as a manual-transfer fallback.

### Setup steps
1. Generate the payment link inside the Paybox app and copy the full URL.
2. Store it as a single public config value (env var or admin setting) — not a secret.
3. At checkout, when "Paybox" is the selected/enabled payment method, render a button using that URL with `target="_blank" rel="noopener noreferrer"`, alongside the business phone number as a text fallback.
4. Repeat the same button on the post-checkout order-tracking page.
5. Reconcile payment manually — there is no callback, so an admin marks the order as paid after confirming receipt in the Paybox app.
6. Only render the button when the configured link is non-empty; treat blank as "method unavailable," not an error.

### Core pattern
```tsx
// Config: a single fully-formed URL, generated manually via the Paybox app UI.
const PAYBOX_PAY_LINK = process.env.NEXT_PUBLIC_PAYBOX_PAY_LINK ?? "";

function PayboxPaymentButton({ businessPhone }: { businessPhone: string }) {
  if (!PAYBOX_PAY_LINK) return null; // treat missing config as "method unavailable"

  return (
    <div>
      <p>
        Pay via Paybox to <span dir="ltr">{businessPhone}</span>, or use the button below.
      </p>
      <a href={PAYBOX_PAY_LINK} target="_blank" rel="noopener noreferrer">
        Open Paybox and pay
      </a>
    </div>
  );
}
```

### Env vars
- `NEXT_PUBLIC_PAYBOX_LINK` (or equivalent public config key — a publicly-shown link, not a secret)

### Gotchas
- **There is no API** — same as Bit: the whole integration is a static link a human generates once inside the Paybox app. Don't look for a request/response contract that doesn't exist.
- **No payment confirmation callback** — reconcile manually; the app can't know payment succeeded on its own.
- **Show the button in both the checkout step and the tracking page** — users often pay after checkout rather than during it.
- **Treat the link as public config, not a secret.**
- **Amount is shown as plain text, not embedded in the link**, in the observed implementation — the customer enters the amount manually inside the Paybox app.
- Since Bit and Paybox share an identical UI/data-flow pattern in the observed source (same components render both, gated only by which payment method is selected), it's natural — and low-risk — to implement both from one shared "deep-link payment method" component, parameterized by provider name, label and link.

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. Implemented identically to the Bit pattern — a configured static deep link shown as a payment option at checkout and again on the order-tracking page.
