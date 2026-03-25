# Security

This document describes NikkChat's security model honestly: what it protects
against, what it doesn't, and the trade-offs I made.

## Threat model

**Who NikkChat tries to protect users from:**
- A passive attacker on the network (TLS handles this).
- An attacker who breaches Cloud Storage and dumps the bucket (encrypted-at-rest
  there, but media is also encrypted by NikkChat before upload, so the attacker
  gets ciphertext only).
- A casual insider at the storage layer who can read files but not Firestore.
- A future me who shouldn't be able to look at users' images by clicking around
  the Firebase console.

**Who NikkChat does NOT fully protect users from:**
- An attacker with full Firebase project access (they can read both the encrypted
  blob in Storage **and** the media key in Firestore - same trust boundary).
- An attacker who compromises the user's device.
- A malicious recipient (E2E doesn't help once the legitimate recipient has the key).

This is a real trade-off - true E2E (think Signal) requires a key-exchange
protocol where the server never sees the keys. NikkChat is closer to "client-side
encryption against a storage breach" than "Signal-grade E2E." That's documented
honestly here rather than claimed away.

## Crypto primitives

| Primitive | Choice |
|---|---|
| Symmetric cipher | AES-256-GCM via `crypto.subtle` (Web Crypto API) |
| IV size | 96 bits, randomly generated per encryption |
| Key generation | `crypto.subtle.generateKey` with `extractable: true` |
| Key transport | Raw key, base64-encoded, in the message envelope |
| RNG | `crypto.getRandomValues` (browser CSPRNG) |

No third-party crypto libraries. The whole encryption layer is ~60 lines of
well-tested first-party code on top of Web Crypto.

## Account security

- Passwords are validated against a strength scorer in the UI; the actual hashing
  is delegated to Firebase Auth.
- Email verification is required before sensitive actions.
- Reauthentication is required for: deleting the account, changing email, changing
  password, removing 2FA.
- Phone-OTP sign-in has a per-number cooldown to limit SMS abuse.

## Privacy choices

- **No analytics SDKs.** No Google Analytics, no Mixpanel, no Sentry. The PWA
  ships with zero third-party trackers.
- **EXIF stripping toggle.** Off by default for performance, on for privacy
  hygiene. When on, location/camera/timestamp metadata is removed before upload.
- **No public feed.** Profiles are addressable but there is no algorithmic
  feed and no follower count.
- **Read receipts and typing indicators are mutual** - if you turn yours off,
  you don't see other people's either.

## Known gaps / planned work

- True E2E for text messages (Signal Protocol style key exchange) is on the
  roadmap but is non-trivial in a Firestore-heavy app and is currently scoped
  to media only.
- TURN relay for calls is not deployed - calls behind symmetric NATs will fail
  to connect. Self-hosted coturn is on the roadmap.
- 2FA via TOTP is not implemented yet. Phone OTP is available as a sign-in
  method but isn't a true second factor.

## Reporting

Found something? Email **kevinrusev1@gmail.com** with a clear repro.
I will respond.
