# Architecture

This document describes how NikkChat is put together at a high level. The actual
source code is private; this is a design overview for review.

## High level

NikkChat is a single-page PWA in the browser talking to a Firebase backend. There
is no custom server. Everything that needs to be authoritative (auth, persistence,
storage, realtime fan-out) is delegated to Firebase services. Everything that
needs to be private (media, in particular) is encrypted client-side first.

```
[ Browser PWA ]  <->  [ Firebase Auth ]
       |
       | (Firestore SDK over WSS)
       v
[ Firestore: users, chats, messages, collections ]
       |
       | (RTDB SDK over WSS)
       v
[ Realtime DB: presence, typing, call signaling ]
       |
       | (Storage SDK over HTTPS)
       v
[ Cloud Storage: encrypted blobs only ]
```

## Data model

### `users/{uid}`
Public profile fields (display name, @tag, avatar URL, bio). Avatars are
plaintext public assets - they are deliberately not encrypted because they are
intended to be visible.

### `chats/{chatId}`
A chat envelope: participants, last message preview, unread counters per user.
Used as a denormalized index so the chat list can render without joins.

### `chats/{chatId}/messages/{msgId}`
Individual messages. For text, the content is stored as-is in Firestore (Firestore
data is encrypted at rest by Google but not E2E). For media, only the **storage
path** and the **media key** travel through Firestore; the actual bytes live in
Cloud Storage as ciphertext.

### `collections/{collectionId}`
A user-curated set of media references. May be public (link/QR shareable) or
private. Each item points back to the encrypted blob and includes the media key.

### Realtime DB

| Path | Purpose |
|---|---|
| `presence/{uid}` | online/offline + last-seen, written via `onDisconnect` |
| `typing/{chatId}/{uid}` | typing indicator with TTL |
| `calls/{callId}` | WebRTC SDP offers/answers + ICE candidates |

## Encryption flow (media)

1. User picks a file (image / video / voice).
2. Optional: EXIF metadata is stripped via `exifr` if the user has the toggle on.
3. A fresh AES-256-GCM key + 96-bit IV are generated via `crypto.subtle`.
4. The file is encrypted in the browser; the resulting ciphertext is uploaded
   to Cloud Storage as `application/octet-stream`.
5. The base64-encoded raw key is attached to the message envelope in Firestore.
6. Recipients fetch the ciphertext, import the key, and decrypt locally.
7. Decrypted blobs are kept in an in-memory + IndexedDB cache so scrolling
   doesn't re-decrypt.

The server never sees the key in unwrapped form because the key lives in
Firestore (which is owned by the same Firebase project, not a separate trust
boundary - see [SECURITY.md](SECURITY.md) for what this does and doesn't protect
against).

## Calls (WebRTC)

Calls are peer-to-peer. The Firebase stack is used **only for signaling**:

1. Caller creates an `RTCPeerConnection`, generates an SDP offer.
2. Offer is written to `calls/{callId}/offer` in Realtime DB.
3. Callee subscribes, gets the offer, generates an answer, writes it back.
4. Both sides exchange ICE candidates via `calls/{callId}/candidates/{caller|callee}`.
5. Once ICE succeeds, audio/video flows directly between the peers.
6. Hangup writes a `status: "ended"` field; both sides tear down.

If P2P fails (symmetric NAT), the call simply doesn't connect - I'm not running
a TURN relay yet. That's on the roadmap.

## PWA

The app is built with `vite-plugin-pwa`. The service worker:
- pre-caches the app shell so cold start is instant
- uses a network-first strategy for Firestore/Storage SDK fetches
- shows a cached "you're offline" view of the chat list when the network is gone

A custom install prompt component watches `beforeinstallprompt` on Chromium and
shows platform-specific instructions on iOS Safari (which doesn't fire that event).

## Why this stack

| Choice | Reason |
|---|---|
| Firebase over a custom backend | Solo dev. Auth + DB + storage + realtime out of the box, all priced predictably for this scale. |
| Firestore + RTDB together | Firestore's reactive queries are great for the chat list. RTDB's lower-latency presence + `onDisconnect` is unbeatable. |
| Web Crypto over a JS library | First-party, audited, fast (uses native subtle crypto under the hood), no supply-chain risk. |
| WebRTC over RTDB signaling | One less service to run. Signaling traffic is tiny. |
| PWA over native | Solo dev velocity. One codebase, instant updates, every platform. App stores can come later. |
