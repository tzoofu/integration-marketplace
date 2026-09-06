# Web Share API / Clipboard API

- **category**: other
- **provider**: Web Platform (browser-native — `navigator.share`, `navigator.clipboard`)
- **reusable**: yes — a share-with-clipboard-fallback hook/util is trivially shared across any repo; both source repos independently reinvented the same pattern.
- **docs**: https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share and https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/writeText

## Overview
Uses the OS-native share sheet (`navigator.share()`) to share a URL or text when available, falling back to `navigator.clipboard.writeText()` — either because `navigator.share` doesn't exist (desktop browsers largely lack it) or for plain "copy to clipboard" actions like copying a code snippet. No third-party service or SDK is involved; both APIs are built into the browser.

## Playbook

### Prerequisites
- Served over HTTPS (or `localhost`) — both APIs are blocked on insecure origins.
- A user-gesture context (click handler) — browsers reject `navigator.share()`/`clipboard.writeText()` calls not triggered by direct user interaction.

### Setup steps
1. No installation or account setup needed — both APIs are available in all evergreen browsers behind no flag.
2. Add a click handler that calls `navigator.share(...)` when present, catching (and ignoring) the user-cancellation `AbortError`.
3. Fall back to `navigator.clipboard.writeText(...)` when `navigator.share` is `undefined` (most desktop browsers) or when the action is a plain "copy" (not a "share").
4. Show transient UI feedback (e.g. a "Copied!" state that reverts after ~1.5–2s) so the user knows the fallback path succeeded, since clipboard writes have no native UI feedback.

### Core pattern
```ts
// share-or-copy.ts — generic share-with-clipboard-fallback util
export async function shareOrCopy(data: { title?: string; text?: string; url: string }): Promise<"shared" | "copied" | "failed"> {
  if (typeof navigator !== "undefined" && navigator.share) {
    try {
      await navigator.share(data);
      return "shared";
    } catch (err) {
      // AbortError = user dismissed the native share sheet; treat as a no-op, not a failure
      if (err instanceof Error && err.name === "AbortError") return "failed";
      // Some browsers throw for other reasons (e.g. no share targets) — fall through to clipboard
    }
  }
  try {
    await navigator.clipboard.writeText(data.url ?? data.text ?? "");
    return "copied";
  } catch {
    return "failed";
  }
}

// Plain copy button (no share-sheet branch), with transient UI feedback
export function useCopyToClipboard(resetMs = 1500) {
  // returns [copied, copy(text)] — component wires this to a "Copied!" label swap
}
```

```tsx
// Usage in a component
async function handleShare() {
  const result = await shareOrCopy({ title, url: `${window.location.origin}/items/${id}` });
  if (result === "copied") {
    setCopied(true);
    setTimeout(() => setCopied(false), 1500);
  }
}
```

### Env vars
none — browser-native APIs, no configuration or keys required.

### Gotchas
- `navigator.share` is largely absent on desktop Chrome/Firefox — always feature-detect (`if (navigator.share)`) rather than assuming availability, and always have the clipboard fallback ready.
- `navigator.share()` rejects with an `AbortError` when the user simply closes the native share sheet — treat that specific case as a silent no-op, not an error to surface to the user.
- Both APIs require a secure context (HTTPS/localhost) and must be invoked synchronously from within a real user-gesture event handler — calling them after an `await` or inside a `setTimeout` can silently fail in some browsers.
- `navigator.clipboard.writeText()` can reject in some embedded/webview contexts (e.g. missing focus, permissions policy) — wrap in try/catch and fail silently or show an inline error rather than throwing.

### Playbook confidence: high

## Adoption
Used in **2** repo(s) in this marketplace. One adopter uses the pattern to share a link to a piece of content via the OS share sheet (falling back to clipboard) alongside a separate plain "copy to clipboard" action for auxiliary text such as code snippets; the other uses it purely to share/copy a single generated item. Both arrived at the same share-or-copy shape independently.
