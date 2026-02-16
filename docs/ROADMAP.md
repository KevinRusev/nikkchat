# Roadmap

A living list of what's shipped and what's next.

## Shipped

- [x] Email/password auth + verification email
- [x] Google / Apple / Microsoft OAuth sign-in
- [x] Phone OTP sign-in
- [x] User profiles with @tag, avatar, bio
- [x] In-browser avatar cropping
- [x] One-to-one chats over Firestore
- [x] Real-time presence + typing indicators (RTDB)
- [x] Read receipts
- [x] Image / video / voice message uploads
- [x] Client-side AES-256-GCM encryption for all media
- [x] Optional EXIF stripping
- [x] Lightbox + full media viewer with swipe
- [x] Save-to-device for received media
- [x] WebRTC peer-to-peer voice calls
- [x] WebRTC peer-to-peer video calls
- [x] Public + private collections
- [x] Share collections via link or QR
- [x] PWA install for Android, iOS, desktop
- [x] Service worker with offline fallback
- [x] Custom domain at nikkchat.com

## In progress

- [ ] Group chats (3-50 members)
- [ ] Message search across own chats
- [ ] Reply / quote a specific message
- [ ] Reactions on messages

## Planned

- [ ] True end-to-end encryption for text (Signal-style key exchange)
- [ ] TURN relay so calls work behind symmetric NATs
- [ ] TOTP 2FA
- [ ] Native iOS app (Capacitor wrapper around the PWA)
- [ ] Disappearing messages (per-chat retention setting)
- [ ] Voice rooms (multi-party audio)

## Won't do (on purpose)

- Public feed / discovery / "explore" tab. NikkChat is for people you know.
- Follower counts / social graph virality.
- Ads, analytics SDKs, third-party trackers.
- Selling user data. Ever.
