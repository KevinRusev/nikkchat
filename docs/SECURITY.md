# Security

What NikkChat protects today, how it does it, and what it doesn't do yet. The version
written for users is at [nikkchat.com/security](https://nikkchat.com/security).

## The short version

NikkChat is **not end-to-end encrypted**. Messages and media are encrypted on the way to the
servers (TLS) and while they are stored (by Google Cloud), and the security rules only let
the two people in a chat open it. Calls go directly between the two devices.

Without end-to-end encryption, whoever runs the servers can technically read what is stored
there. That is me, and Google, who host it. I don't read anyone's chats, but nobody should
have to take my word for it, which is why end-to-end encryption is on the roadmap and why
this page doesn't claim more than the code does.

## Threat model

**Who NikkChat protects users from**

- Someone on the network between a device and the servers. Everything travels over TLS.
- Another NikkChat account. The rules refuse any account that isn't one of the two people in
  a chat, for its messages, its media folder, its typing state and its call records.
- Someone holding an unlocked, signed-in device who tries to change the email or password or
  delete the account. Those ask for the password again.
- A password that has already leaked somewhere else. New passwords are checked against known
  breaches.

**Who NikkChat does not protect users from**

- Anyone with access to the Firebase project: me, Google, or someone who compromises either.
- Someone who takes over the user's device or account.
- The other person in the chat.
- Anyone a media link is passed to. Each file gets a long, random link that only appears
  inside the chat, and the link on its own opens the file.

## Access rules

There is no server of mine in front of the data, so Firebase's security rules are the
access control. They follow one idea: a chat's id is its two members' ids, sorted and
joined, and the id is the authority.

| Data | Who can reach it |
|---|---|
| A chat and its messages | The two members. Another account can't read it, write to it or add itself to it. |
| A message | Only its sender can edit it. The other member can mark it read or react. |
| A chat's media folder | The two members, for uploading, fetching through the SDK, listing and deleting. |
| Typing state | The two members. |
| A call record | The caller creates it; only the caller and the person called can read or change it. A device can only listen for calls addressed to its own account. |
| Profiles | Any signed-in account can read a name, @username, photo and bio. Phone numbers are not stored on profiles. |
| Collections | The owner. Other signed-in accounts only while the collection is public. |

Chats created by early builds have random ids. For those, membership comes from the chat's
member list, which the storage rules look up in Firestore before opening the chat's media
folder.

The rules are tested against the Firebase emulators: 120 checks as three accounts (two who
share a chat, one who doesn't), plus a two-browser run of the real screens that fails on any
"permission denied".

## Messages

- Stored in Cloud Firestore, encrypted at rest by Google. Not end-to-end encrypted.
- Deleting a conversation removes every message in it, for both people.

## Photos, videos and voice notes

- Stored in Google Cloud Storage, encrypted at rest by Google. Not end-to-end encrypted.
- Photos are redrawn and saved as a new image (WebP) in the browser before they upload, so
  the location, camera model and other embedded details are left behind. If the browser
  can't redraw a photo it is sent as it is; a setting adds a second metadata pass.
- Deleting a message or a conversation takes the photo out of the chat, but the stored file
  isn't erased yet.

## Calls

- Voice and video use WebRTC. Sound and picture travel directly between the two devices,
  encrypted by WebRTC, and no NikkChat server carries them. To find the direct route, each
  device asks a public STUN server for its own address.
- Setting up a call means the two devices exchange connection details, including network
  (IP) addresses, through the Realtime Database. Only the caller and the person called can
  read them.
- There is no relay (TURN) server yet. On networks that block direct connections, a call
  may not connect.
- Calls from a blocked account don't ring.

## Accounts

- Sign-in with Google, Microsoft, or an email and password. Password hashing and sessions
  are Firebase Auth's.
- New passwords need at least 10 characters and are checked against known breaches through
  Have I Been Pwned's range API, which never receives the password or its full hash.
- Sign-up verifies a phone number by SMS, with a cooldown between codes. The number shows
  there is a person behind the account and allows a password reset by text.
- Changing the email or password, or deleting the account, asks for the password again.
- All sessions can be signed out at once.
- Deleting the account removes the profile and photo and frees the username. Messages
  already sent stay in the other person's chat.

## Privacy choices

- **No analytics or ad SDKs.** The only outside scripts are Google's: the sign-in helper and
  reCAPTCHA, which checks for bots when a number is verified.
- **No feed and no follower counts.** Profiles can be opened, but nothing ranks or recommends
  them.
- **Search matches names and usernames, never phone numbers.**
- **Online status is a switch.** While it is off, NikkChat doesn't publish whether someone is
  online or when they were last here.

## Not built yet

- **End-to-end encryption** for messages and media.
- **Notifications** while NikkChat is closed.
- **A relay server** for calls on networks that block direct connections.
- **A private block list.** The accounts someone has blocked are stored with their profile.
  The app never shows that list to anyone, but it isn't yet locked away from other signed-in
  accounts.
- **Erasing stored files** when a message or a conversation is deleted.
- **Two-factor sign-in.** The phone number is verified at sign-up but isn't a second factor.

## A note on the earlier design

An earlier build encrypted photos, videos and voice notes in the browser with AES-256-GCM,
one key per file, and stored that key next to the message. That guarded against a leak of the
storage bucket on its own, not against anyone who could read the database, so it was never
end-to-end encryption. The code is still in the project, but uploads don't use it today, and
this document describes what runs.

## Reporting

Found something? Email **kevinrusev1@gmail.com** with a clear repro.
I will respond.
