# Security

## Crypto primitives

| Primitive | Choice |
|---|---|
| Symmetric cipher | AES-256-GCM via Web Crypto API |
| IV size | 96 bits, randomly generated per encryption |
| Key generation | `crypto.subtle.generateKey` |
| RNG | `crypto.getRandomValues` |

No third-party crypto libraries.
