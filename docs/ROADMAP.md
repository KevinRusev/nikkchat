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
- [x] Photos redrawn to WebP before upload; optional second metadata pass
- [x] Lightbox and media viewer with swipe; save to device
- [x] Voice and video calls, device to device (WebRTC)
- [x] Blocking, including calls
- [x] Collections, private or public
- [x] Installable PWA for Android, iPhone and desktop, with an offline app shell
- [x] Sign out of all devices; account deletion
- [x] Custom domain at nikkchat.com
- [x] November 2026: new brand and site, and security rules that limit every chat, its
      media, its typing state and its calls to the two people in it

## Next

- [ ] End-to-end encryption for messages and media
- [ ] Notifications while the app is closed
- [ ] A relay (TURN) server so calls connect on networks that block direct connections
- [ ] Lock the block list away from other accounts
- [ ] Erase stored files when a message or conversation is deleted
- [ ] Group chats
- [ ] Search inside a chat

## Later

- [ ] Two-factor sign-in (TOTP)
- [ ] Disappearing messages
- [ ] Native wrapper for the app stores

## Won't do (on purpose)

- A public feed, discovery or an "explore" tab. NikkChat is for people you know.
- Follower counts.
- Ads, analytics SDKs, third-party trackers.
- Selling user data. Ever.
