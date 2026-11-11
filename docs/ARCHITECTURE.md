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
[ Firestore: users, usernames, chats, messages, wrapped keys, collections ]
       |
       | (Realtime Database SDK)
       v
[ Realtime Database: presence, typing, call setup ]
       |
       | (Storage SDK over HTTPS)
       v
[ Cloud Storage: photos, videos, voice notes, avatars ]
```

Messages and media are end-to-end encrypted in the browser before they reach any of this.
What is stored is ciphertext, plus the fields the server needs to route and order it. See
[SECURITY.md](SECURITY.md) for the design and for what it doesn't cover.

## Data model

### `users/{uid}`
The profile other people see: display name, @username, avatar URL, bio and the account's
public encryption key, plus the owner's settings. Readable by signed-in accounts. Phone
numbers are not stored here; they stay in Firebase Auth.

Four collections hang off it:

| Path | Holds | Who can read it |
|---|---|---|
| `users/{uid}/blocked/{otherUid}` | the block list, one document per blocked account | the owner |
| `users/{uid}/devices/{deviceId}` | one push address per browser that has notifications on | the owner |
| `users/{uid}/keys/backup` | the private keys, sealed with the recovery code | the owner |
| `users/{uid}/pubkeys/{keyId}` | every public key the account has had, written once | any signed-in account |

The last one is what lets a message sealed under an older key still be opened after the
account has replaced its key.

### `tags/{username}`
The username registry: one document per @username, pointing at its owner. It makes
usernames unique and lets an invite link (`nikkchat.com/u/<name>`) resolve to a person.

### `chats/{chatId}`
Two shapes share this collection.

A chat between two people has an id made of their two user ids, sorted and joined with `_`,
so there is exactly one chat per pair and the id itself says who belongs to it.

A group has a random id and a document that carries its name, its members (up to 50) and
its admins. Admins rename it and add or remove people; anyone can take themselves out.

Both hold the last message's preview and time, which the chat list is built from without
reading any messages. For an encrypted text message that preview is sealed too.

### `chats/{chatId}/messages/{messageId}`
One message. In the clear: the sender, the type (text, photo, video, voice note, or a logged
call), the time, who it has been delivered to and read by, and reactions. Sealed in one
envelope: the text, the media link and that file's key, the file name, and the message it
replies to.

### `chats/{chatId}/keys/{epoch}`
Groups only. One document per key epoch, holding a copy of that epoch's key for each member,
each copy wrapped so only that member can open it. The rules let members add copies and
nobody change or remove one; an admin clears them when the group is deleted.

### `collections/{collectionId}`
A set of saved photos and videos with an owner, a name, a cover and a visibility: private,
or public (shown on the owner's profile).

### Realtime Database

| Path | Purpose |
|---|---|
| `presence/{uid}` | online and last seen, cleared through `onDisconnect`; nothing is written while the owner has online status switched off |
| `typing/{chatId}/{uid}` | typing indicator, in chats between two people |
| `calls/{callId}` | call setup: the offer, the answer, connection candidates and the call's status |

## Keys

Each account has an ECDH P-256 key pair, made in the browser the first time the app opens
for it. The private half stays in that browser's IndexedDB, and in a backup on the server
that only the recovery code opens. The public half goes on the profile.

- **Two people** need no key exchange. Each app combines its own private half with the
  other's public half and runs the result through HKDF with the chat id as salt. Both arrive
  at the same AES-256 key.
- **A group** has a random AES-256 key per epoch. The app that creates an epoch wraps the key
  once per member, using the pairwise key it shares with that member. The first message after
  someone leaves starts a new epoch, wrapped only for the people still there.
- **A new device** gets the keys from the backup document, which opens only with the recovery
  code. The code is generated on the device and shown once; the server never sees it.

Key setup is written to survive two tabs racing each other: local changes go through a
single IndexedDB transaction under a Web Lock, and the public half is published in a
Firestore transaction that refuses to overwrite a key another device has just published.

## How a message travels

1. The app gets the chat's key: derived for two people, unwrapped from its own copy for a
   group. If it can't (the device is locked, or a key that was there before has gone
   missing), it refuses to send instead of falling back to plain text.
2. The private parts of the message are sealed as one AES-256-GCM payload with a fresh
   nonce. The chat id and the sender's id are bound in, so the ciphertext can't be moved to
   another chat or re-posted under another name.
3. The message document is written: the envelope, plus the sender, type and time. The rules
   only accept it from a member, as themselves.
4. The chat document's preview is updated, sealed the same way.
5. Firestore's listeners deliver both to the other members, whose apps open them with the
   same key. A message that can't be opened says why, and offers the recovery code when that
   is what's missing. It is never an empty bubble.
6. What a member's app shows of the message comes from the opened envelope only. Text or a
   file link stored in the clear beside it is ignored, and the envelope can't overwrite who
   the message is from. A message with no envelope, arriving after the conversation started
   using encryption, is shown with a "Not end-to-end encrypted" mark.
7. Opening the chat marks the message read, which the sender sees as ticks.

A chat between two people stays unencrypted only while one of them has no key at all: their
app hasn't been opened since encryption shipped, or their browser can't keep one. It shows no
lock in that state.
A group is encrypted from its first message; a member who gets a key later is handed a copy
of the group's keys by another member's app.

## How a photo travels

1. The user picks a photo.
2. The browser redraws it onto a canvas and saves it as WebP. That makes it smaller and
   leaves the embedded location and camera details behind. A setting adds a second pass over
   the metadata. If the browser can't redraw the file, it is sent as it is.
3. In an encrypted chat, the file is encrypted with a key of its own.
4. It uploads to Cloud Storage under the chat's own folder, under a name that says nothing
   about it. The storage rules only accept it from a member of the chat.
5. The first encrypted upload of a session reads a few bytes back. If the bucket won't allow
   that, the file is deleted and nothing is sent: the app won't quietly send it unencrypted.
6. The message carries the file's link and its key inside the sealed envelope.
7. The other members' apps fetch the file, decrypt it in memory and show it. The decrypted
   copy is only ever handed to the page as a picture, video or sound, whatever type the
   sender claimed for it.

Videos and voice notes take the same route without the redraw.

## Calls (WebRTC)

Calls are between two people and device to device. Firebase is used only to set them up:

1. The caller creates an `RTCPeerConnection` and an offer, and writes a call record
   addressed to the other person under `calls/{callId}`.
2. Each signed-in device listens only for call records addressed to its own account. The
   callee's device sees the record and shows the incoming-call screen, unless the caller is
   on its block list.
3. On accept, the callee writes an answer, and both sides exchange connection candidates
   through the same record.
4. Once a route is found, sound and picture flow between the two devices, encrypted by
   WebRTC.
5. Hanging up ends the call on both sides, logs it in the chat and removes the record. A
   call nobody answers times out.

Devices find their public address through a STUN server. For networks that block direct
connections there is a relay: before a call, each side's app asks a Cloud Function for
short-lived credentials for Cloudflare's TURN service, and falls back to a direct-only
attempt if that takes more than four seconds. The relay forwards packets it has no key to.
It is built and switched on separately from the site, behind a build flag.

## Notifications

Web push through Firebase Cloud Messaging, also behind a build flag.

1. Turning notifications on in a browser stores that browser's push address under the
   account's `devices`.
2. A Cloud Function runs when a message is created, and the caller's app calls a second one
   when a call starts ringing. Each skips anyone who has blocked the sender and sends a
   data-only push to each of the recipient's addresses: a name, the kind of thing ("Photo",
   "Missed voice call"), and the chat to open. Never the text, which it couldn't read anyway.
3. The service worker shows the notification, unless a NikkChat window is already open and
   in front. (Safari withdraws push permission from a site that stays silent, so there every
   push is shown.) Tapping it opens that chat.
4. Addresses the push service reports as dead are removed. Signing out removes this
   browser's.

## Security rules

With no server in between, the rules are the access control:

- A chat, its messages, its media folder, its typing state and its call records are open
  only to its members. For a chat between two people the rules read the members straight
  from the id.
- Only a message's sender can write or edit it; other members can mark it read or react.
- Nobody can add themselves to a chat. In a group only an admin changes the member list,
  apart from a member leaving.
- A group's wrapped keys can be added to and never changed.
- An account's block list, push addresses and key backup are readable by that account only.
- A collection is readable by its owner, and by other signed-in accounts only while public.

They are tested two ways: 208 allow/deny checks against the Firebase emulators as four
accounts, and a scripted run of the real screens as three people in separate browsers that
fails on any "permission denied". That run also reads what the emulators stored and fails
if message text, a preview or a file name is there in the clear, and it writes to them
directly to check that planted text is ignored and a planted message is marked.

## PWA

Built with `vite-plugin-pwa`. The service worker:

- caches the app shell, so a repeat visit starts without the network
- installs new versions by itself and reloads an open app onto them (a first visit, where
  there is no earlier version, is left alone)
- keeps a cache of media already viewed
- shows push notifications and opens the right chat from them

An install prompt handles `beforeinstallprompt` on Chromium browsers and shows written steps
on iOS Safari, which has no such event.

Each screen except sign-in is its own chunk, so the public homepage doesn't load code that
only the sign-up form needs.

## Cloud Functions

Six small functions:

| Function | Does |
|---|---|
| Account emails | sends verification and password-reset emails through Resend |
| Sign out everywhere | ends every session of an account |
| Account clean-up | after an account is deleted, removes its profile, keys, block list, push addresses and avatar, and frees its username |
| Message push | sends the push for a new message |
| Call push | sends the push for an incoming call, once per call |
| Relay credentials | asks Cloudflare for short-lived TURN credentials and hands them to a signed-in account |

None of them holds a key. For an encrypted message they see what the database sees:
ciphertext, a sender, a type and a time.

## Why this stack

| Choice | Reason |
|---|---|
| Firebase over a custom backend | Solo developer. Auth, database, storage and realtime out of the box, priced predictably at this scale. |
| Firestore + Realtime Database together | Firestore's live queries suit the chat list and messages. Realtime Database's lower latency and `onDisconnect` suit presence, typing and call setup. |
| Rules as the backend | No server to run or patch, as long as the rules are strict and tested. |
| WebCrypto, no crypto library | The primitives are the browser's. The code I have to get right is the key handling, and that is what the tests aim at. |
| Long-lived identity keys with a recovery code, not a ratchet | A second device and a lost phone stay simple to explain and recover from. The price is no forward secrecy. |
| WebRTC with Realtime Database for setup | One less service to run. Call media never touches a server of mine. |
| PWA over native | One codebase, instant updates, every platform. App stores can come later. |
