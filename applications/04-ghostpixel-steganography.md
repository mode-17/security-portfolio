# 04 — GhostPixel: encrypted steganography in the browser

**Type:** security application · **My role:** design and implementation · **Source:** private
**Stack:** vanilla HTML/CSS/ES modules, Web Crypto API, Canvas

---

## The problem

Hiding a message inside an image is easy to do badly. The common LSB tutorial ships plaintext
in the pixels, uses no integrity check, and stores the extraction key next to the payload. The
goal was a tool that is offline, client-side only, and treats the hidden payload as an actual
secret: encrypted, authenticated, bounded by lifetime, and removed from the DOM after use.

## Design

### Carrier

1 bit per RGB channel in a lossless PNG. Capacity is `(width × height × 3) / 8` bytes, shown
live as a capacity bar so a message cannot silently overrun the carrier.

PNG is a hard requirement: JPEG recompression destroys LSB data. The tool says so rather than
producing an image that fails to decode later.

### Payload

```
plaintext
  → AES-256-GCM (random 12-byte IV, 128-bit tag)
  → base64
  → JSON envelope { data, created_at, expires_in, sha256 }
  → LSB embed
```

- **Key derivation:** PBKDF2, SHA-256, 100,000 iterations, random 16-byte salt.
- **Integrity:** SHA-256 over the ciphertext, creation time and expiry. A single changed pixel
  fails verification before decryption is attempted.
- **Expiry:** 1 h / 12 h / 24 h / 7 d. Decryption is blocked after the deadline, checked
  against the envelope rather than trusted from the UI.

### Handling the plaintext

The decoded message is held in module scope, rendered once, then overwritten with filler and
nulled, and the DOM node is removed after five seconds.

This is a mitigation, not a guarantee. A browser is not a secure memory environment: strings
are immutable, the GC may have already copied them, and a hostile extension or a compromised
page can read the DOM before removal. It raises the cost of casual recovery; it does not make
the plaintext unrecoverable.

### No network

No upload, no server, no telemetry. The application is static files and runs from
`file://` or any static host. That removes the entire class of "the steganography service kept
your message" failures.

## Threat model

**Covers:** someone who obtains the image but not the passphrase; casual inspection of the
file; modification of the image in transit (integrity check fails); reading the message after
its intended lifetime.

**Does not cover:** a compromised endpoint, a keylogger, a malicious browser extension, an
attacker who can observe the encode operation, or traffic analysis that detects that *a*
steganographic image was sent. Steganography hides the payload, not the fact that two parties
are communicating.

## What I would improve

- Add a decoy passphrase path, so a coerced user can reveal a plausible message.
- Use a memory-hard KDF (Argon2id via WASM) instead of PBKDF2 if the browser target allows.
- Add a statistical detector to the test page that estimates whether an image looks
  stego-modified, so the tool can be honest about its own detectability.

## What this demonstrates

Applied cryptography with the failure modes named: authenticated encryption, a real KDF,
integrity before decryption, and explicit limits on the self-destruct and memory-wiping
claims. The interesting part of a security tool is not that it encrypts — it is that its
author can say what it does not protect.
