# Security

What NikkChat protects today, how it does it, and what it doesn't do yet. The version
written for users is at [nikkchat.com/security](https://nikkchat.com/security).

## The short version

Messages, photos, videos and voice notes are **end-to-end encrypted**, in chats between two
people and in groups. They are encrypted in the sender's browser with a key that only the
people in the chat hold. The servers, mine and Google's, store ciphertext.

Two limits. A chat is encrypted from the moment both people's apps have a key, which each
app makes the first time it opens; anything sent before that stays stored as it was sent.
And encryption covers what is said, not the fact that it was said: who talks to whom, and
when, is visible to the servers.

Calls and collections are not covered by this, and the copy never says they are.

## Threat model

**Who NikkChat protects users from**

- Someone on the network between a device and the servers. Everything travels over TLS.
- Anyone with access to the Firebase project (me, Google, or someone who compromises
  either), as far as the content of messages and media goes. They see ciphertext.
- Another NikkChat account. The rules refuse any account that isn't in a chat, for its
  messages, its media folder, its typing state and its call records.
- Someone a media link is passed to. A file from an encrypted chat is ciphertext; the link
  opens nothing readable.
- Someone holding an unlocked, signed-in device who tries to change the email or password or
  delete the account. Those ask for the password again.
- A password that has already leaked somewhere else. New passwords are checked against known
  breaches.

**Who NikkChat does not protect users from**

- The other people in the chat.
- Someone who takes over the user's device, or their account together with their recovery
  code.
- Someone who serves a modified app. NikkChat is a web app: its code comes from the server
  whenever it updates, so the encryption is only as trustworthy as that code. This is the
  standing limit of end-to-end encryption on the web.
- Someone who obtains a user's private key later and had also captured their ciphertext.
  Keys are long-lived; there is no forward secrecy.
- Metadata. The servers see who is in each chat, when messages are sent and read, what kind
  they are, reactions and call records.
- Anyone who is passed the link to a file sent before its chat was encrypted. That link
  opens the file.
- Someone with access to the servers holding a message back, deleting it, or delivering an
  old one a second time. They can't read a message or change what it says, but encryption
  doesn't prove a conversation is complete.
- Someone with access to the servers adding an account to a group. Membership is kept by the
  server, and members' apps hand the group's key to everyone on that list. The account would
  show in the member list like anyone else; the conversation doesn't yet announce it.

## How the encryption works

Everything runs on the browser's own WebCrypto. There is no crypto library in the bundle.

**Identity keys.** The first time the app opens for an account it makes an ECDH key pair on
the P-256 curve. The public half is published on the profile, and in a write-once list of
the account's current and earlier public keys. A key is named by the first eight bytes of
the SHA-256 of its public half, so a swapped key can't hide behind a known name.

**Two people.** Both apps derive the same chat key: ECDH between their own private half and
the other's public half, then HKDF-SHA-256 with the chat id as salt. Nothing is exchanged to
agree on it.

**Groups.** A group has a random AES-256 key per epoch. The member whose app makes it wraps
it once per member, with the pairwise key between the two of them. Copies are append-only in
the rules: a member can add a copy for someone who joined, but nobody can change or remove
one, so no member can lock another out. They go only when an admin deletes the group. The
first message after someone leaves or is removed starts a new epoch, wrapped only for the
people still there. A key is used for sending only if a current member handed it over; a
copy from anyone else is good for reading history and nothing more.

**Messages.** The private parts of a message (text, the media link and its per-file key,
file name, the quoted message) are sealed as one AES-256-GCM payload with a random 96-bit
nonce. The chat id and the sender's id are bound in as associated data, so a ciphertext
moved to another chat or re-posted under another name fails to open. The chat-list preview
is sealed the same way. An edit seals the message again.

**Media.** A file is encrypted in the browser with a key of its own before upload and stored
under a name that says nothing about it. That key travels inside the sealed message. The
first encrypted upload of a session reads a few bytes back; if the bucket won't allow that,
the file is withdrawn and nothing is sent, instead of quietly sending it unencrypted.

**The recovery code.** Private keys live in the browser's IndexedDB. A backup of them is
stored on the server, sealed with a key derived (PBKDF2-SHA-256) from a recovery code: 24
characters, 120 bits, generated on the device and shown once. The server never sees the
code. Entering it on a new browser brings the keys there. Without it, a new device can't
read earlier encrypted messages; the account can start over with a new key.

**Checking keys.** The server hands out public keys, so it could hand out a wrong one. Each
two-person chat has a safety code, thirty digits derived from both public keys, for the two
people to compare. The app also says when a contact's key is no longer the one this device
saw before.

**No silent downgrade.** A message goes unencrypted only when an account involved has no key
at all: its app hasn't been opened since encryption shipped, or its browser can't do the
cryptography. Between two people the chat shows no lock in that state. In a group where the
others are encrypting, a message from a member without a key arrives marked (next
paragraph). Once a device has seen a contact's key it refuses to send to them in the clear,
even if the key later disappears from their profile. And an account that has a key can't
send in the clear from a device that doesn't hold it; that device has to be unlocked with
the recovery code first.

**What isn't encrypted is treated as such.** What the app shows of an encrypted message comes
from the opened envelope and nowhere else: text or a file link stored in the clear beside it
is ignored, and the envelope can only supply the private parts, not who the message is from,
its type or its time. A message with no envelope, arriving after the conversation started
using encryption, is shown with a "Not end-to-end encrypted" mark, so one can't be planted
to look like the rest. A decrypted file is handed to the page only as a picture, video or
sound, never as a document that could carry script, because it lives on the app's own
origin and its type comes from the sender.

**In a group, a message is authenticated to the group.** Every member holds the same key, so
the cryptography shows a message came from a member, not which one. The rules are what stop
one member posting as another.

**What is not sealed.** Reactions, read and delivered ticks, timestamps, the kind of each
message, call records, profiles, group names and member lists.

**Collections** are outside it. Saving a photo to a collection stores the photo's key with
the collection, so the collection can show it, to its owner or (if public) to anyone signed
in. The app says so at the moment of saving.

## Access rules

There is no server of mine in front of the data, so Firebase's security rules are the
access control.

| Data | Who can reach it |
|---|---|
| A chat between two people, and its messages | The two members. The chat's id is their two ids, sorted and joined, and the rules treat the id as the authority. |
| A group, and its messages | Its members. Admins rename it and add or remove people; anyone can take themselves out; nobody else can change who is in it. |
| A message | Only its sender can write or edit it. Other members can mark it read or react. Between two people either can delete messages, which is how deleting the conversation clears it for both; in a group, people delete their own and admins can delete any. |
| A chat's media folder | Its members. |
| A group's wrapped keys | Its members, to read and to add copies. A copy can't be changed or removed; an admin clears the keys when the group is deleted. |
| Typing state | The two members of a chat. |
| A call record | The caller creates it; only the caller and the person called can read or change it. A device can only listen for calls addressed to its own account. |
| Profiles | Any signed-in account can read a name, @username, photo, bio and public key. Phone numbers are not stored on profiles. |
| An account's block list, push addresses and key backup | That account only. |
| An account's list of public keys | Any signed-in account can read it. Entries are written once. |
| Collections | The owner. Other signed-in accounts only while the collection is public. |

The rules are tested against the Firebase emulators: 208 allow/deny checks as four accounts,
plus a run of the real screens as three people in separate browsers. That run also reads what
the emulators stored and fails if a message's text, a file name or a preview is there in the
clear, and it writes to them directly, as someone with access to the servers could, to check
that planted text is ignored and a planted message is marked. The cryptography and the rules
for what a thread shows have their own checks, and the service worker's push handling is
driven on a production build.

## Messages

- Sealed as above, then stored in Cloud Firestore, which Google also encrypts at rest.
- In a chat between two people, deleting the conversation removes every message in it, for
  both. A group is deleted by an admin; leaving one only takes that person out.

## Photos, videos and voice notes

- Encrypted before upload in encrypted chats, as above.
- Photos are redrawn and saved as a new image (WebP) in the browser before that, so the
  location, camera model and other embedded details are left behind. If the browser can't
  redraw a photo it is sent as it is; a setting adds a second metadata pass.
- Deleting a message or a conversation takes the photo out of the chat, but the stored file
  isn't erased yet.

## Calls

- Voice and video are between two people and use WebRTC. Sound and picture are encrypted by
  WebRTC between the two devices. To find a direct route, each device asks a public STUN
  server for its own address.
- A relay for networks that block direct connections is built: a Cloud Function hands each
  side short-lived credentials for Cloudflare's TURN service, and the relay forwards the
  encrypted stream without a key to it. It is switched on separately from the site.
- Setting up a call means the two devices exchange connection details, including network
  (IP) addresses, through the Realtime Database. Only the caller and the person called can
  read them. The exchange isn't authenticated with the identity keys, so calls don't carry
  the guarantee messages do.
- Calls from a blocked account don't ring.

## Notifications

- Web push, off until someone turns it on in a browser. Built, and switched on separately
  from the site.
- A push carries the name of the sender or the group and the kind of thing that arrived
  ("Photo", "Voice message", "New message", a missed call). It never carries message text,
  which the server couldn't read anyway.
- Signing out or switching them off removes that browser's push address.

## Accounts

- Sign-in with Google, Microsoft, or an email and password. Password hashing and sessions
  are Firebase Auth's.
- New passwords need at least 10 characters and are checked against known breaches through
  Have I Been Pwned's range API, which never receives the password or its full hash.
- Sign-up verifies a phone number by SMS, with a cooldown between codes. The number shows
  there is a person behind the account and allows a password reset by text.
- Changing the email or password, or deleting the account, asks for the password again.
- All sessions can be signed out at once.
- The password signs the account in. It doesn't unlock encrypted messages: the recovery code
  does, and a password reset doesn't touch it.
- Deleting the account removes the profile and photo, the keys and their backup, the block
  list and the push addresses, and frees the username. Messages already sent stay in the
  other people's chats.

## Privacy choices

- **No analytics or ad SDKs.** The only outside scripts are Google's: the sign-in helper and
  reCAPTCHA, which checks for bots when a number is verified.
- **No feed and no follower counts.** Profiles can be opened, but nothing ranks or recommends
  them.
- **Search matches names and usernames, never phone numbers.**
- **Online status is a switch.** While it is off, NikkChat doesn't publish whether someone is
  online or when they were last here.
- **The block list is private.** Only the account that made it can read it, and blocked
  people aren't told.

## Not built yet

- **Forward secrecy.** Keys are long-lived until the account replaces them.
- **A safety code for groups**, and a notice in the conversation when someone is added.
- **Group calls**, and typing indicators in groups.
- **End-to-end encrypted collections.**
- **Erasing stored files** when a message or a conversation is deleted.
- **Two-factor sign-in.** The phone number is verified at sign-up but isn't a second factor.

## A note on the earlier design

An earlier build encrypted photos, videos and voice notes in the browser with AES-256-GCM,
one key per file, and stored that key next to the message. That guarded against a leak of the
storage bucket on its own, not against anyone who could read the database, so it was never
end-to-end encryption. The per-file keys are still how media is encrypted; what changed is
that the key now travels inside a message only the people in the chat can open.

## Reporting

Found something? Email **kevinrusev1@gmail.com** with a clear repro.
I will respond.
