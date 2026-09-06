# Browser Notification API (direct, foreground)

- **category**: push-notifications
- **provider**: Web Platform (browser-native `Notification` API)
- **reusable**: no — single-repo so far; a distinct, simpler sub-use from `fcm-push.md` (no service worker, no background delivery, no Firebase token registration).
- **docs**: https://developer.mozilla.org/en-US/docs/Web/API/Notification

## Overview
Requests OS notification permission and fires local/foreground desktop notifications directly via the browser's `Notification` constructor, paired with a Web Audio API chime for extra attention-grabbing. This is a separate, simpler code path from FCM background push (`fcm-push.md`): no service worker, no token registration, no server round-trip — it only works while the tab is open and in the foreground.

## Playbook

### Prerequisites
- None beyond a browser — this is pure Web Platform API, no third-party account or SDK needed.
- Must run in a secure context (`https`, or `localhost` for local dev) — `Notification` may be undefined or permission requests may silently fail otherwise.

### Setup steps
1. Guard every call: check `typeof window !== "undefined" && typeof Notification !== "undefined"` before touching the API — SSR and unsupported browsers both need a no-op fallback.
2. Read current permission state with `Notification.permission` (`"default" | "granted" | "denied"`) to decide whether to show a "enable notifications" prompt at all.
3. Request permission with `await Notification.requestPermission()` — must be triggered from a user gesture (a button click), browsers block silent/auto-triggered permission prompts.
4. On `"granted"`, fire notifications with `new Notification(title, { body, tag, ... })`.
5. (Optional but recommended) prime a Web Audio `AudioContext` on the same user gesture that requests permission — browsers suspend audio contexts until a user interaction, so priming early avoids a silent first chime later.
6. Wire `notification.onclick` to focus the window and close the notification — otherwise clicking it does nothing.

### Core pattern
```ts
"use client";

export type PermissionState = "default" | "granted" | "denied" | "unsupported";

export function getNotificationPermission(): PermissionState {
  if (typeof window === "undefined" || typeof Notification === "undefined") return "unsupported";
  return Notification.permission;
}

export async function requestNotificationPermission(): Promise<PermissionState> {
  if (typeof window === "undefined" || typeof Notification === "undefined") return "unsupported";
  if (Notification.permission === "granted" || Notification.permission === "denied") {
    return Notification.permission; // browsers refuse to re-prompt once decided
  }
  try {
    return await Notification.requestPermission();
  } catch {
    return "denied";
  }
}

export function fireOsNotification(title: string, body: string, options?: { tag?: string }): Notification | null {
  if (typeof window === "undefined" || typeof Notification === "undefined") return null;
  if (Notification.permission !== "granted") return null;
  try {
    const n = new Notification(title, { body, tag: options?.tag });
    n.onclick = () => { window.focus(); n.close(); };
    return n;
  } catch {
    return null;
  }
}
```

**Companion audio chime (Web Audio API, primed on first user gesture)**
```ts
let audioCtx: AudioContext | null = null;
let primed = false;

function ensureCtx(): AudioContext | null {
  if (audioCtx) return audioCtx;
  const Ctor = window.AudioContext ?? (window as any).webkitAudioContext;
  if (!Ctor) return null;
  try { audioCtx = new Ctor(); return audioCtx; } catch { return null; }
}

// Call this from the SAME click/keydown handler that requests notification
// permission — browsers keep new AudioContexts suspended until a user gesture.
export function primeAudio(): void {
  const ctx = ensureCtx();
  if (ctx?.state === "suspended") void ctx.resume().catch(() => {});
  primed = true;
}

export function playChime(): void {
  const ctx = ensureCtx();
  if (!ctx) return;
  if (ctx.state === "suspended") void ctx.resume().catch(() => {});
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  osc.type = "sine";
  osc.frequency.setValueAtTime(880, ctx.currentTime);
  gain.gain.setValueAtTime(0, ctx.currentTime);
  gain.gain.linearRampToValueAtTime(0.18, ctx.currentTime + 0.015);
  gain.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + 0.2);
  osc.connect(gain).connect(ctx.destination);
  osc.start();
  osc.stop(ctx.currentTime + 0.22);
}
```

### Env vars
None.

### Gotchas
- Once a user has answered `"granted"` or `"denied"`, calling `requestPermission()` again is a silent no-op — the browser will not re-prompt. A "denied" user must be told to change it manually in browser settings; there is no programmatic way back to "default".
- `requestPermission()` must originate from a direct user gesture (click/keypress handler) — calling it from a `useEffect` on mount, a timer, or after an `await` inside the handler can get silently ignored by the browser's popup blocker heuristics.
- This API only fires while the tab/window is open (foreground or backgrounded-but-alive) — it cannot wake a closed browser. If you need "notify even when the site is closed," that's FCM/web-push territory (`fcm-push.md`), not this API.
- `AudioContext` starts in a `"suspended"` state until resumed inside a user-gesture handler; priming it early (on the same click that requests notification permission) avoids the first real chime being silently dropped.
- Treat this as complementary to, not a replacement for, background push — a repo can run both: this API for "tab is open" immediacy, FCM for "tab/browser is closed."

### Playbook confidence: high

## Adoption
Used in **1** repo(s) in this marketplace. Applied as a foreground-immediacy complement to FCM background push — a desktop notification plus audio chime while the tab is open, independent of the background-push subscription.
