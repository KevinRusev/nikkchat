# Security

This document describes NikkChat's security model honestly: what it protects
against, what it doesn't, and the trade-offs I made.

## Threat model

**Who NikkChat tries to protect users from:**
- A passive attacker on the network (TLS handles this).
- An attacker who breaches Cloud Storage and dumps the bucket.
- A casual insider at the storage layer.

**Who NikkChat does NOT fully protect users from:**
- An attacker with full Firebase project access.
- An attacker who compromises the user's device.
- A malicious recipient.

## Crypto primitives

| Primitive | Choice |
|---|---|
| Symmetric cipher | AES-256-GCM via Web Crypto API |
| IV size | 96 bits, randomly generated per encryption |
| Key generation | `crypto.subtle.generateKey` |
| RNG | `crypto.getRandomValues` |

No third-party crypto libraries.
