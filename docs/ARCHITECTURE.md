# Architecture

High-level overview of how NikkChat is put together.

## Stack

- React 19 PWA in the browser
- Firebase backend (Auth, Firestore, Storage, Realtime DB)
- WebRTC for peer-to-peer calls

The actual source code is private; this is a design overview for review.

## Data model

### `users/{uid}`
Public profile fields: display name, @tag, avatar URL, bio.

### `chats/{chatId}`
Chat envelope: participants, last message preview, unread counters.

### `chats/{chatId}/messages/{msgId}`
Individual messages. Text is stored in Firestore; media is uploaded to Cloud
Storage as ciphertext only, with the decryption key in the message envelope.

### `collections/{collectionId}`
User-curated media sets. May be public (link/QR shareable) or private.
