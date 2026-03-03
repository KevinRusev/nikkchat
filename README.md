# NikkChat

**Encrypted messaging, voice & video calls, and media sharing.**
Live at [nikkchat.com](https://nikkchat.com).

> NikkChat is a closed-source product. This repository is a public case study
> describing what was built and how. The source is private.

## What it is

NikkChat is a privacy-first messenger - a PWA that installs on any device,
runs without an app store, and treats media as end-to-end encrypted by default.
Every image, video, and voice message is encrypted client-side before it
ever touches a server.

## Tech stack

- React 19 + Vite 8
- Tailwind CSS 4
- Firebase (Auth, Firestore, Storage, Realtime DB, Functions)
- WebRTC for peer-to-peer voice and video calls
- Web Crypto API (AES-256-GCM)
- PWA with service worker

## Feature highlights

- End-to-end encrypted media (AES-256-GCM, per-blob ephemeral keys)
- Optional EXIF stripping
- Real-time messaging with presence and typing indicators
- WebRTC voice and video calls
- Email / Google / Apple / Microsoft / Phone OTP sign-in
- Public + private collections
- PWA install on Android, iPhone, and desktop

## Documentation

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) - system overview and data flow
- [`docs/SECURITY.md`](docs/SECURITY.md) - encryption model and threat model
- [`docs/ROADMAP.md`](docs/ROADMAP.md) - shipped and planned features
