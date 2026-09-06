# Firebase Hosting

- **category**: hosting-deploy
- **provider**: Google Firebase Hosting
- **reusable**: no — a single static PDF deployed to satisfy a Play Console store-listing field, not a general web-app hosting use case; nothing about this setup is worth extracting as a shared package.
- **docs**: https://firebase.google.com/docs/hosting

## Overview
Firebase Hosting serves static files (and optionally proxies to Cloud Functions/Cloud Run) from a CDN under a `<project-id>.web.app` / `<project-id>.firebaseapp.com` domain, configured via `firebase.json` and deployed with the Firebase CLI. In its known usage here it hosts nothing more than one static PDF.

## Playbook

### Prerequisites
- A Firebase project (create one in the Firebase console or via `firebase projects:create`).
- The Firebase CLI installed and authenticated (`npm i -g firebase-tools`, `firebase login`).
- The static asset(s) to serve, already built/exported into a local directory.

### Setup steps
1. From the repo root, run `firebase init hosting` and select (or create) the target Firebase project.
2. Point the `public` directory at the folder containing your static build output (e.g. `hosting/`, `dist/`, `out/`).
3. Answer "configure as a single-page app" only if you actually need SPA rewrite-to-`index.html` behavior; for a handful of static files (like a single PDF) decline it so unmatched paths 404 normally instead of serving `index.html`.
4. Place the static asset(s) in the `public` directory.
5. Deploy with `firebase deploy --only hosting`.
6. The file is now reachable at `https://<project-id>.web.app/<path>` and `https://<project-id>.firebaseapp.com/<path>`.
7. To update the asset later, replace the file in the `public` directory and redeploy — Firebase Hosting versions each deploy and serves the latest atomically (no partial-update window).

### Core pattern
`firebase.json` — minimal static hosting config:
```json
{
  "hosting": {
    "public": "hosting",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"]
  }
}
```

`.firebaserc` — pins the CLI to a specific project so `firebase deploy` never prompts or targets the wrong one:
```json
{
  "projects": {
    "default": "<your-project-id>"
  }
}
```

Deploy command:
```bash
firebase deploy --only hosting
```

If the static asset is generated from an editable source (e.g. HTML rendered to PDF) rather than authored directly, keep the source alongside the deploy target and document the regeneration step, e.g.:
```bash
# render HTML -> PDF via headless Chrome, then redeploy
node render-pdf.js src/document.html hosting/document.pdf
firebase deploy --only hosting
```

### Env vars
None — Firebase Hosting deploys authenticate via the Firebase CLI's own login/service-account credentials, not app env vars.

### Gotchas
- A single-purpose static-hosting deploy (e.g. one PDF for a store-listing URL) doesn't need SPA rewrites, a build step, or a custom domain — resist adding hosting config complexity beyond what's actually served.
- Firebase Hosting deploys are atomic and versioned (each `firebase deploy` creates a new immutable release you can roll back to in the console) — there's no risk of visitors seeing a half-updated file mid-deploy.
- If the asset only exists to satisfy an external requirement (e.g. a store listing's required privacy-policy URL), confirm it isn't also supposed to be linked from inside the app itself — don't assume; check for both a direct URL reference and any in-app browser-intent/deep-link call site before concluding it's truly external-only.
- **Playbook confidence: low** — this playbook is synthesized from general Firebase Hosting documentation/best practice rather than extracted from a locally-available implementation, so treat it as a correct-by-the-book starting point rather than a verified extraction.

## Adoption
Used in **1** repo(s) in this marketplace, hosting a single static privacy-policy PDF solely to satisfy an app-store data-safety/store-listing field. Confirmed not linked to or opened anywhere inside the app itself — it exists purely to satisfy that external requirement.
