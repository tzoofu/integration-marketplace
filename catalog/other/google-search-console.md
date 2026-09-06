# Google Search Console site verification

- **category**: other
- **provider**: Google
- **reusable**: yes — the dual verification pattern (static file + meta tag) is a trivial copy-paste for another repo's domain.
- **docs**: https://support.google.com/webmasters/answer/9008080

## Overview
Google Search Console requires proof of domain ownership before it will surface search performance and indexing data for a site. This is done via two independent, simultaneously-present verification methods: a static HTML file served at the domain root, and a `<meta name="google-site-verification">` tag rendered in the page `<head>` via framework metadata.

## Playbook

### Prerequisites
- A Google Search Console property created for the target domain (Google Account with access to https://search.google.com/search-console).
- Ability to deploy a static file to the site's public root and to set an environment variable read at build/render time.

### Setup steps
1. In Google Search Console, add the property for your domain and choose the "HTML tag" verification method (and/or "HTML file" method — both can be used together for redundancy).
2. For the HTML file method: download the file Google gives you (named like `google<verification-id>.html`, containing a single line `google-site-verification: google<verification-id>.html`) and place it in your app's public static assets directory so it's served at `https://yourdomain.com/google<verification-id>.html`.
3. For the meta tag method: copy the verification token Google gives you (the `content` value of the meta tag) into an environment variable (e.g. `GOOGLE_SITE_VERIFICATION`) rather than hardcoding it.
4. Wire that env var into your framework's page metadata/head config so it renders as `<meta name="google-site-verification" content="...">` on every page.
5. Deploy, then click "Verify" in Search Console for each method you configured.

### Core pattern
Static file (place at `public/google<verification-id>.html`):
```
google-site-verification: google<verification-id>.html
```

Next.js App Router metadata (any root `layout.tsx`):
```tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  // ...other metadata fields
  verification: {
    google: process.env.GOOGLE_SITE_VERIFICATION,
  },
};
```

For non-Next.js frameworks, the equivalent is rendering a raw tag in `<head>`:
```html
<meta name="google-site-verification" content="{{GOOGLE_SITE_VERIFICATION}}" />
```

### Env vars
- `GOOGLE_SITE_VERIFICATION`

### Gotchas
- If the env var is unset, Next's `metadata.verification.google` simply omits the tag rather than erroring — safe for local/dev environments without the secret configured.
- The static-file method's filename itself encodes the verification ID; the file content is a fixed one-line string referencing that same filename — don't rename the file without also updating its content.
- Both methods can coexist safely; keeping both means losing one (e.g. removing the static file during a redeploy) doesn't break verification.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
