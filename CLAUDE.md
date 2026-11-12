# flumen-frontend-web

React 19 + TypeScript + Vite single-page app for **Flumen**: the uploader dashboard, the public share/download pages (served from a separate download domain), and the admin app (a separate Vite entry on its own origin). Files are optionally end-to-end encrypted in the browser; the format is specified in `docs/E2EE.md`.

**This repository is public.** Never commit secrets, API keys (even "test" ones other than Cloudflare's documented Turnstile test keys), internal abuse thresholds or limits, infrastructure details beyond what the browser can already see, or the private project plan. If a task needs one of those, ask instead of adding it.

## Layout (created in Phase 4)

| Path | Role |
| :--- | :--- |
| `src/crypto/` | `flumen-e2ee-v1` implementation (WebCrypto only). Security-critical. |
| `src/sw/` | Service worker: streaming download, resume, decrypt, `complete` call. |
| `src/app/` | Dashboard (app domain). |
| `src/share/` | Public `/s/:token` page (download domain). |
| `src/admin/` | Admin app entry (admin origin). Never imported by `app/` or `share/`. |
| `docs/E2EE.md` | Normative encryption spec. |

`src/crypto/` and `src/sw/` have their own `CLAUDE.md`. Read it before editing there.

## Commands

```bash
npm ci --ignore-scripts
npm run lint          # ESLint (react/no-danger is an error)
npm run typecheck     # tsc --noEmit, strict
npm test              # Vitest
npm run build         # deterministic Vite build for app/ and admin/
npm run e2e           # Playwright against the dev stack
```

## Rules

**KISS (`karpathy-guidelines` skill):**
- Plain React function components and TanStack Query. No global state library, no CSS-in-JS beyond Tailwind, no component library unless it removes substantial code.
- No abstraction until there are two real uses.
- Every new dependency is a supply-chain risk for an E2EE app (people trust the JavaScript we serve). Add one only when it's clearly worth it, and say why in the commit body. Never add crypto libraries: WebCrypto only.

**Security invariants:**
- Never send `location.hash`, keys, or decrypted metadata to any API, log, analytics, or error reporter.
- Never use `dangerouslySetInnerHTML` or `innerHTML`. Render filenames, report text, and anything from the API as text.
- The API client is generated from the backend's `openapi.yaml`. Don't hand-write request types that duplicate it.
- The admin app uses its own Cognito app client and origin. Shared code must not import from `src/admin/`.
- Before downloading an executable or double-extension file, show the warning interstitial (list comes from the backend; don't fork it).

**License headers:** every new source file (`.ts`, `.tsx`, `.js`, `.css`, `.html`) starts with the MPL 2.0 notice:

```ts
/* This Source Code Form is subject to the terms of the Mozilla Public
 * License, v. 2.0. If a copy of the MPL was not distributed with this
 * file, You can obtain one at https://mozilla.org/MPL/2.0/. */
```

Don't copy code from projects whose licenses aren't compatible with MPL 2.0.

**TypeScript:** `strict: true`, no `any` (use `unknown` and narrow), no non-null `!` assertions on API data.

## Workflow

- Write a failing test before code (`test-driven-development` skill). Crypto and service-worker logic get Vitest unit tests; user flows get Playwright tests.
- Investigate failures before fixing (`systematic-debugging` skill).
- After a feature works, consider a scoped simplification pass (`code-simplification` skill).
- Before saying anything is done, run lint, typecheck, tests, and build, and quote the results (`verification-before-completion` skill).
- Commits: Conventional Commits. One logical change per commit.
