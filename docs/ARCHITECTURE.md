# Architecture

How NikkChat is put together. The source code is private; this is a design overview for
review.

## High level

NikkChat is a single-page PWA in the browser talking to Firebase. There is no server of my
own. Auth, persistence, storage and realtime fan-out are Firebase services, and Firebase's
security rules decide who can read and write what.

```
[ Browser PWA ]  <->  [ Firebase Auth ]
       |
       | (Firestore SDK)
       v
[ Firestore: users, usernames, chats, messages, collections ]
       |
       | (Realtime Database SDK)
       v
[ Realtime Database: presence, typing, call setup ]
       |
       | (Storage SDK over HTTPS)
       v
[ Cloud Storage: photos, videos, voice notes, avatars ]
```

Nothing is end-to-end encrypted. Data is encrypted in transit and at rest by the platform;
see [SECURITY.md](SECURITY.md) for exactly what that does and doesn't cover.

## Data model

### `users/{uid}`
The profile other people see: display name, @username, avatar URL, bio, plus the owner's
settings. Readable by signed-in accounts. Phone numbers are not stored here; they stay in
Firebase Auth.

### `tags/{username}`
The username registry: one document per @username, pointing at its owner. It makes
usernames unique and lets an invite link (`nikkchat.com/u/<name>`) resolve to a person.

### `chats/{chatId}`
A chat between two people. Its id is their two user ids, sorted and joined with `_`, so
there is exactly one chat per pair and the id itself says who belongs to it. The document
holds the two members, the last message preview and its time, which the chat list is built
from without reading any messages.

### `chats/{chatId}/messages/{messageId}`
One message: sender, type (text, photo, video, voice note, or a logged call), the text or the
media link, the time, who it has been delivered to and read by, reactions, and the message it
replies to if any.

### `collections/{collectionId}`
A set of saved photos and videos with an owner, a name, a cover and a visibility: private,
or public (shown on the owner's profile).

### Realtime Database

| Path | Purpose |
|---|---|
| `presence/{uid}` | online and last seen, cleared through `onDisconnect`; nothing is written while the owner has online status switched off |
| `typing/{chatId}/{uid}` | typing indicator |
| `calls/{callId}` | call setup: the offer, the answer, connection candidates and the call's status |

## How a photo travels

1. The user picks a photo.
2. The browser redraws it onto a canvas and saves it as WebP. That makes it smaller and
   leaves the embedded location and camera details behind. A setting adds a second pass over
   the metadata. If the browser can't redraw the file, it is sent as it is.
3. The file uploads to Cloud Storage under the chat's own folder. The storage rules only
   accept it from one of the chat's two members.
4. The message document gets the file's download link.
5. The other person's app shows the photo from that link.

Videos and voice notes take the same route without the redraw.

## Calls (WebRTC)

Calls are device to device. Firebase is used only to set them up:

1. The caller creates an `RTCPeerConnection` and an offer, and writes a call record
   addressed to the other person under `calls/{callId}`.
2. Each signed-in device listens only for call records addressed to its own account. The
   callee's device sees the record and shows the incoming-call screen, unless the caller is
   blocked.
3. On accept, the callee writes an answer, and both sides exchange connection candidates
   through the same record.
4. Once a direct route is found, sound and picture flow between the two devices.
5. Hanging up ends the call on both sides, logs it in the chat and removes the record. A
   call nobody answers times out.

Devices find their public address through a STUN server. There is no TURN relay, so if a
network blocks direct connections the call doesn't connect. That is on the roadmap.

## Security rules

With no server in between, the rules are the access control:

- A chat, its messages, its media folder, its typing state and its call records are open
  only to its two members. For chats with the current id format the rules read the members
  straight from the id.
- Only a message's sender can edit it; the other member can mark it read or react.
- A collection is readable by its owner, and by other signed-in accounts only while public.

They are tested two ways: 120 allow/deny checks against the Firebase emulators, and a
scripted run of the real screens in two browsers at once that fails on any "permission
denied".

## PWA

Built with `vite-plugin-pwa`. The service worker:

- caches the app shell, so a repeat visit starts without the network
- installs new versions by itself and reloads open tabs onto them
- keeps a cache of media already viewed

An install prompt handles `beforeinstallprompt` on Chromium browsers and shows written steps
on iOS Safari, which has no such event.

Each screen except sign-in is its own chunk, so the public homepage doesn't load code that
only the sign-up form needs.

## Cloud Functions

Three small functions: one sends the account emails (verification, password reset) through
Resend, one signs an account out of every device, and one cleans up after an account is
deleted.

## Why this stack

| Choice | Reason |
|---|---|
| Firebase over a custom backend | Solo developer. Auth, database, storage and realtime out of the box, priced predictably at this scale. |
| Firestore + Realtime Database together | Firestore's live queries suit the chat list and messages. Realtime Database's lower latency and `onDisconnect` suit presence, typing and call setup. |
| Rules as the backend | No server to run or patch, as long as the rules are strict and tested. |
| WebRTC with Realtime Database for setup | One less service to run. Call media never touches a server. |
| PWA over native | One codebase, instant updates, every platform. App stores can come later. |
