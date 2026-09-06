# Firebase Cloud Messaging (FCM)

- **category**: push-notifications
- **provider**: Firebase / Google (FCM), VAPID
- **reusable**: yes — highest-value extraction candidate. Multiple adopters independently implement the same token-registration, stale-token-cleanup, and dynamically-served `firebase-messaging-sw` service-worker route — strong candidate for a shared `@marketplace/fcm-push` web package. Another adopter contributes a native-Android FCM consumer (server-triggered sends from Cloud Functions, no VAPID/service worker involved).
- **docs**: https://firebase.google.com/docs/cloud-messaging

## Overview
Push notifications delivered via Firebase Cloud Messaging. Two shapes appear in this marketplace: web push (browser subscribes with a VAPID key, a service worker shows the notification, works even when the tab is closed) and native Android push (the app registers automatically via `google-services.json`, no VAPID needed, delivery is handled by the OS + Play Services). Both share the same server-side send pattern: `sendEachForMulticast()` against a list of stored device tokens, followed by pruning any token FCM reports as dead.

## Playbook

### Prerequisites
- A Firebase project with Cloud Messaging enabled.
- Web push: a **VAPID key pair** generated in Firebase Console → Project Settings → Cloud Messaging → Web Push certificates.
- Native Android: `google-services.json` downloaded into the app module (gitignore it — it's environment config, not a secret by itself, but still shouldn't be committed to a public repo).
- A Firebase Admin SDK service account (server side) for sending — see `firebase-admin-sdk.md` in this catalog.
- A place to persist device tokens per-user/group (Firestore in every repo observed, but any store works).

### Setup steps
1. **Server**: initialize `firebase-admin` once (see `firebase-admin-sdk.md`) and expose `admin.messaging()`.
2. **Web client**: install `firebase` (JS SDK), initialize the client app with the standard `NEXT_PUBLIC_FIREBASE_*` config, and use `firebase/messaging`'s `isSupported()` before ever calling `getMessaging()` — Safari/older browsers throw otherwise.
3. **Web service worker**: add a route (e.g. `/firebase-messaging-sw.js` or an API route at that path) that serves a JS file which either (a) `importScripts` the `firebase-messaging-compat` SDK and calls `onBackgroundMessage`, or (b) listens to the raw `push` event and calls `self.registration.showNotification()` directly. Both are valid — see "Core pattern" for when to pick which. The route must be served from the site root so its default scope (`/`) covers the whole origin.
4. **Web client — subscribe flow**: request `Notification.requestPermission()` → register the service worker → call `getToken(messaging, { vapidKey, serviceWorkerRegistration })` → POST the returned token to your own backend, tied to the authenticated user.
5. **Server — persist token**: store the token in an array/set keyed by user (or group). Use an "arrayUnion"-style upsert so re-subscribing the same device is idempotent.
6. **Server — send**: on the triggering event, load all tokens for the target audience and call `getMessaging().sendEachForMulticast({ tokens, notification: {...} })` (or `data: {...}` if the SW reads a raw `push` event instead of Firebase's `onBackgroundMessage`).
7. **Server — prune dead tokens**: inspect `response.responses[i].error?.code`; on `messaging/registration-token-not-registered` or `messaging/invalid-registration-token`, delete that token from storage. Do this after every send — uninstalls and permission revocations are the normal case, not an edge case.
8. **Native Android**: add the `google-services` Gradle plugin + `firebase-bom` + `firebase-messaging` dependency, subclass `FirebaseMessagingService`, override `onNewToken` (persist it server-side) and `onMessageReceived` (build a `NotificationCompat` notification and show it via `NotificationManager`).

### Core pattern

**A. Web push — Firebase-compat service worker (uses `notification:` payload)**
```js
// app/api/firebase-messaging-sw/route.ts — served as a JS file at the site root
export async function GET() {
  const config = JSON.stringify({
    apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
    authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
    projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
    storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
    messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
    appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
  });

  const sw = `
importScripts('https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/10.12.2/firebase-messaging-compat.js');
firebase.initializeApp(${config});
const messaging = firebase.messaging();

messaging.onBackgroundMessage((payload) => {
  const title = payload.notification?.title ?? 'New activity';
  self.registration.showNotification(title, {
    body: payload.notification?.body ?? '',
    icon: '/favicon.ico',
    data: payload.data ?? {},
  });
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  const url = event.notification.data?.url || '/dashboard';
  event.waitUntil(clients.openWindow(url));
});
`.trim();

  return new Response(sw, {
    headers: { "Content-Type": "application/javascript", "Service-Worker-Allowed": "/", "Cache-Control": "no-cache" },
  });
}
```

**B. Web push — raw `push` event service worker (uses `data:` payload, no firebase-compat import needed at all)**
```js
export async function GET() {
  const script = `
self.addEventListener('push', function(event) {
  if (!event.data) return;
  var payload = {};
  try { payload = event.data.json(); } catch (e) {}
  var options = {
    body: payload.body || '',
    icon: '/icon-192.png',
    tag: payload.tag || 'push',
  };
  event.waitUntil(self.registration.showNotification(payload.title || 'New notification', options));
});

self.addEventListener('notificationclick', function(event) {
  event.notification.close();
  event.waitUntil(clients.openWindow('/'));
});
`;
  return new Response(script, {
    headers: { "Content-Type": "application/javascript; charset=utf-8", "Service-Worker-Allowed": "/", "Cache-Control": "no-cache, no-store, must-revalidate" },
  });
}
```
Note: sending with `notification:` in the FCM payload (pattern A) requires the compat SDK's `onBackgroundMessage`; sending with only `data:` (pattern B) lets you use a plain `push` listener with zero Firebase JS in the service worker — simpler, but you must build the notification title/body yourself from `data`.

**Web client — subscribe + register token**
```ts
"use client";
import { getToken, isSupported } from "firebase/messaging";

const VAPID_KEY = process.env.NEXT_PUBLIC_FIREBASE_VAPID_KEY;

export async function subscribeToPush(messaging: Messaging): Promise<string> {
  const reg = await navigator.serviceWorker.register("/firebase-messaging-sw.js", { scope: "/" });
  const token = await getToken(messaging, { vapidKey: VAPID_KEY, serviceWorkerRegistration: reg });
  if (!token) throw new Error("FCM returned an empty token");
  await fetch("/api/push/register", { method: "POST", body: JSON.stringify({ token }) });
  return token;
}
```

**Server — send + prune dead tokens (shared shape across all repos)**
```ts
import { getMessaging } from "firebase-admin/messaging";

async function sendToTokens(tokens: string[], notification: { title: string; body: string }) {
  if (tokens.length === 0) return;
  const response = await getMessaging().sendEachForMulticast({ tokens, notification });
  response.responses.forEach((r, i) => {
    const code = r.error?.code;
    if (!r.success && (code === "messaging/registration-token-not-registered" || code === "messaging/invalid-registration-token")) {
      removeStoredToken(tokens[i]).catch(console.error); // your own storage-layer delete
    }
  });
}
```

**Native Android — `FirebaseMessagingService`**
```kotlin
class MyMessagingService : FirebaseMessagingService() {
    override fun onNewToken(token: String) {
        // persist token server-side, tied to the current user/group
    }

    override fun onMessageReceived(message: RemoteMessage) {
        val title = message.notification?.title ?: return
        val body = message.notification?.body ?: ""
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle(title).setContentText(body)
            .setSmallIcon(R.mipmap.ic_launcher).setAutoCancel(true)
            .build()
        getSystemService(NotificationManager::class.java).notify(System.currentTimeMillis().toInt(), notification)
    }
}
```
```kotlin
// build.gradle.kts (app module)
plugins { alias(libs.plugins.google.services) }
dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.messaging)
}
```

### Env vars
- `NEXT_PUBLIC_FIREBASE_VAPID_KEY` (web push only)
- `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
- `NEXT_PUBLIC_FIREBASE_API_KEY`, `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`, `NEXT_PUBLIC_FIREBASE_PROJECT_ID`, `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`, `NEXT_PUBLIC_FIREBASE_APP_ID` (standard Firebase web config, needed by the SW route too since it's rendered server-side)
- Native Android: none — config comes from `google-services.json`, not env vars.

### Gotchas
- The service worker route MUST be served from the origin root (or with `Service-Worker-Allowed: /`) — a scope-mismatch is the #1 reason "push works in the open tab but not when the browser is closed."
- Always gate `getMessaging()`/`getAnalytics()`-style client SDK calls behind `isSupported()` — Safari and some in-app browsers don't support the Push API at all and will throw synchronously otherwise.
- Prune dead tokens on *every* send, not as a periodic cleanup job — token death (uninstall, permission revoked, browser data cleared) is routine, and stale tokens silently accumulate and slow down multicast sends otherwise.
- Web push requires a secure context (`https`, or `localhost` for dev) — check `window.isSecureContext` before showing a "subscribe" UI.
- The two service-worker payload styles (A: `notification` + compat SDK's `onBackgroundMessage`, B: `data` + raw `push` listener) are not interchangeable — pick one and keep the server's `sendEachForMulticast` payload shape (`notification:` vs `data:`) in sync with it.
- `sendEachForMulticast` (not the deprecated `sendMulticast`) is the current Admin SDK API — some older tutorials still show `sendMulticast`.
- Native Android needs no VAPID key and no service worker at all — those are exclusively a web-push concept; don't try to port that setup step to a mobile app.

### Playbook confidence: high

## Adoption
Used in **3** repo(s) in this marketplace. Two adopters implement the web-push shape (token registration, a dynamically-served service-worker route, stale-token cleanup) but differ in service-worker payload style — one uses the Firebase-compat `onBackgroundMessage` pattern with a `notification:` payload, the other a raw `push` listener with a `data:` payload and no Firebase JS in the worker at all. A third adopter is native-Android only: server-triggered sends from a serverless backend with no VAPID key or service worker involved.
