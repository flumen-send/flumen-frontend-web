# src/crypto/

Implements `flumen-e2ee-v1` exactly as written in `docs/E2EE.md`. This is the most security-critical code in the project. Mistakes here silently break confidentiality or integrity.

## Rules

- The spec is the authority. If code and `docs/E2EE.md` disagree, the code is wrong. Changing the format means a new version (`flumen-e2ee-v2`), new test vectors, and decrypt support for older versions.
- WebCrypto (`crypto.subtle`, `crypto.getRandomValues`) only. No third-party crypto, no `Math.random`, no hand-rolled primitives.
- Never reuse a nonce: nonces are `NONCE_BASE XOR be96(record_index)`; each file gets a fresh random salt and key.
- Decryption must fail closed on any authentication failure, a missing final (`0x02`) record, data after the final record, or a wrong record length. Never return partially decrypted data as success.
- Keys and plaintext never leave this module except as the decrypted stream handed to the caller. Don't log them, put them in errors, or store them anywhere except the IndexedDB key store.
- Tests (Vitest) must include: fixed test vectors (checked in under `test-vectors/`), round trips at record boundaries (0 bytes, `rs-18`, `rs-17`, `rs-16`, multi-record), and tamper tests (bit flip, truncation, reordering, appended data).
- Keep the code small and linear. No clever abstractions; a reviewer must be able to check it against the spec line by line.
