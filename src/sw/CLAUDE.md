# src/sw/

Service worker for streaming downloads on the download domain: intercept `/_dl/<claim_id>`, fetch the signed URL, decrypt through a `TransformStream` (via `src/crypto/`), stream to disk, resume with `Range` after re-signing, and call the claim `complete` endpoint at the end.

## Rules

- Call `complete` only after the last byte is written **and** (for encrypted files) the final record authenticated. Never call it on error, cancel, or partial download.
- Resume uses the claim's re-sign endpoint and HTTP `Range`. Never start a new claim to resume; claims are persisted in IndexedDB.
- The worker must not cache file contents or signed URLs (no Cache API for `/f/*` or `/_dl/*`).
- The service worker script is served with `Cache-Control: no-cache`. Keep it small, and bump its version constant on every change so browsers pick it up.
- Test with Vitest using a stubbed `fetch` for the state machine (re-sign, resume, complete), plus a Playwright test for a real interrupted download.
