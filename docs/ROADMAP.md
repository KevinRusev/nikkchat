# Roadmap

What's shipped and what's next.

## Shipped

- [x] Email and password sign-up with a verification email
- [x] Google and Microsoft sign-in
- [x] Phone number verified by SMS at sign-up; password reset by email link or SMS code
- [x] Password check against known leaks (Have I Been Pwned)
- [x] Profiles with @username, photo and bio; in-browser avatar crop
- [x] Find people by name or @username; invite link and QR code
- [x] One-to-one chats over Firestore
- [x] Photos, videos and voice notes
- [x] Replies, reactions and edits
- [x] Delivered and read ticks
- [x] Typing and online status, with a switch to hide online status
- [x] Search inside a conversation, done on the device
- [x] Photos redrawn to WebP before upload; optional second metadata pass
- [x] Lightbox and media viewer with swipe; save to device
- [x] Voice and video calls, device to device (WebRTC)
- [x] Blocking, including calls
- [x] Collections, private or public
- [x] Installable PWA for Android, iPhone and desktop, with an offline app shell
- [x] Sign out of all devices; account deletion
- [x] Custom domain at nikkchat.com
- [x] November 2026: new brand and site, and security rules that limit every chat, its
      media, its typing state and its calls to the people in it
- [x] November 2026: end-to-end encryption for messages and media, with a recovery code,
      safety codes, and a mark on anything that arrives unencrypted
- [x] November 2026: group chats of up to 50, encrypted, with a new key when someone leaves
- [x] November 2026: a block list only its owner can read

## Built, switched on separately

Each needs a key from an outside service before it goes live.

- [ ] Notifications while the app is closed (web push through Firebase Cloud Messaging)
- [ ] A relay so calls connect on networks that block direct connections (Cloudflare TURN)

## Next

- [ ] Forward secrecy
- [ ] A notice in the conversation when someone is added to a group, and a safety code for
      groups
- [ ] Erase stored files when a message or conversation is deleted
- [ ] End-to-end encrypted collections
- [ ] Group calls, and typing indicators in groups

## Later

- [ ] Two-factor sign-in (TOTP)
- [ ] Disappearing messages
- [ ] Native wrapper for the app stores

## Won't do (on purpose)

- A public feed, discovery or an "explore" tab. NikkChat is for people you know.
- Follower counts.
- Ads, analytics SDKs, third-party trackers.
- Selling user data. Ever.
