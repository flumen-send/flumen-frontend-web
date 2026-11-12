# Flumen end-to-end encryption

Files marked "End-to-End Encrypt" are encrypted in the browser before upload. The decryption key is placed only in the share link's URL fragment (`#…`), which browsers never send to servers, so Flumen's servers store ciphertext they cannot read.

This document is the normative spec for the format. `src/crypto/` implements it; any change to the format requires a new version identifier (e.g. `flumen-e2ee-v2`) and new test vectors.

## Format `flumen-e2ee-v1`

- **Key**: random 32-byte `K` generated with `crypto.getRandomValues`, encoded base64url (43 chars) in the share URL fragment: `https://<host>/s/<token>#<K>`.
- **Per-file salt**: random 16 bytes, stored server-side as `e2ee_salt` (tampering only causes decryption failure, never disclosure).
- **Key derivation** (HKDF-SHA256, `ikm = K`, `salt = e2ee_salt`):
  - `CEK = HKDF(info = "flumen-e2ee-v1 content", L = 32)` → AES-256-GCM key
  - `NONCE_BASE = HKDF(info = "flumen-e2ee-v1 nonce", L = 12)`
  - `MEK = HKDF(info = "flumen-e2ee-v1 metadata", L = 32)`
- **Records** (RFC 8188-style framing, no inline header): ciphertext record size `rs = e2ee_record_size` (default `65536`). Each record's plaintext is `data || delimiter`, where `delimiter = 0x01` for non-final records (data length exactly `rs - 17`) and `0x02` for the final record (data length `0..rs-17`). Record `i` is encrypted with `nonce_i = NONCE_BASE XOR be96(i)`, producing `len(data) + 1 + 16` bytes.
- **Ciphertext size**: `n = max(1, ceil(P / (rs - 17)))`; `size_bytes = P + 17 × n` where `P` is plaintext length. Computed client-side before requesting the ticket.
- **Multipart alignment**: `part_size_bytes` is a multiple of `rs`, so every part is a whole number of records and can be (re-)encrypted independently for retries.
- **Integrity**: decryption fails on any modified, reordered, or truncated stream (missing `0x02` final record → error; data after final record → error).
- **Metadata**: `encrypted_metadata = base64url(iv[12] || AES-256-GCM(MEK, iv, JSON{ name, type, plaintext_size }))`. The server stores `filename = "encrypted"` and `content_type = "application/octet-stream"` for E2EE files.
- **Owner key storage**: file keys are kept in the uploading browser's IndexedDB. On other devices, the dashboard shows encrypted files as "Encrypted — key on another device" and cannot create copyable share links for them. (Future: wrap keys with a user-held passphrase-derived key.)
- **Download**: a Service Worker intercepts a same-origin `/_dl/<claim_id>` navigation, fetches the signed URL (re-signing + `Range` on failure), decrypts through a `TransformStream`, and streams to disk with the decrypted filename. File System Access API (`showSaveFilePicker`) used when available. After the final record authenticates and the last byte is written, it calls the claim `complete` endpoint.
