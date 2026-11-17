# PiVault

- Personal Vault application to save password and notes encrypted and fully secure.
- An application which is created to be fully supported for all devices, screen sizes and operating system using the PWA(Progressive Web Application) system.
- Offline first, sync to server support when online.
- ReactJS frontend and a NodeJs backend which will be hosted on my pi server.
- Designs - <https://claude.ai/artifact/6n9ZGZdMXnQm8DzNVnV1MZ#artboard-a76f8dd437c2>

## Architecture(FRONTEND)

- Layered architecture, a folder per layer inside the src folder(types, constants, enums, interfaces, hooks, providers, contexts, pages, components, components/modules, routes, services, utils, db(RxDB)).
- Proper unit tests & integration tests in Vitest.
- ShadCn components
- Since the app is offline first whatever we write goes first into the client DB(RxDB) and then on the server when sync.
- ULID everywhere

### Coding Practices

- No file bigger than 200 LOC
- Segregation of concerns, separate the UI & logic and call them in a file to complete a page/module.
- Follow theme, token, proper design conventions to maintain consistency across components and pages.
- Strictly follow the designs unless explicitly told to ignore or modify - <https://claude.ai/artifact/6n9ZGZdMXnQm8DzNVnV1MZ###artboard-a76f8dd437c2>
- 1 thing at a time, while executing we are going to focus on one work/module/part unless multiple required.
- Reuse - whatever's possible
- Every screen/page just be full height and width, no manual height, width, padding, margin, spacing unless required. Always distribute space.
- Do not use tailwind classes that give yellow squiggles "deprecated" or "can be written as" if it can be written as then already write it that way.
- App is a vault made for storing passwords and secrets so security is the first angle to think from.
- Every write goes into client side DB first this helps with offline first architecture.
- Every exception must be caught/thrown/handled and shown toast wherever required.
- For UI never write components manually, always check if relevant shadcn component for it is available, if yes then use it.
- Once you finish writing code, always write its tests, then check for type errors and lint errors and do not quit just after writing code, always write tests, fix them fix type errors and lint errors.

### Deployment (how to ship a change)

- `backend/` (pivault-backend) and `pwa/` (pivault-pwa) are separate git repos. Each one deploys itself when its `live` branch is pushed. The parent repo has no CI and never deploys.
- Live site: <https://pivault.shivamingale.com> (API under `/api`, health at `/api/health/ready`).

## Steps, done inside the submodule that changed

1. Work on `main` in that submodule.
2. Run the same checks CI runs and fix everything before going further:
   - backend: `npm run check && npm test`
   - pwa: `npm run check`
3. Commit and `git push origin main`. This only runs the checks on GitHub, nothing deploys.
4. **(approval)** Ask me before deploying. Only after I say yes:
   `git push origin main:live`
5. Watch it: `gh run watch -R Shivamingale3/<pivault-backend|pivault-pwa>`. After the run goes green, Jenkins on the Pi deploys it within a minute and Telegram shows ✅ or ❌.
6. Verify: `curl -s https://pivault.shivamingale.com/api/health/ready`, then open the site.

## What happens after a push to live

GitHub Actions tests it, builds an ARM64 image tagged `sha-<first 7 chars of the commit>`, and pushes it to GHCR. Jenkins on the Pi then pulls it, runs migrations (backend), switches to the new version and checks health. If the check fails, it rolls back to the last working version automatically.

## Rules

- Never push to `live` with failing checks.
- If both backend and pwa changed and the pwa needs the new API, deploy backend first and wait for its ✅.
- Prisma migrations must be additive only (add tables and columns, never rename or drop in the same release), because rollback restores the old code but not the old schema.
- Never commit `.env` files or secrets. Server config lives on the Pi, not in these repos.
- Do not change CI workflows, Jenkinsfiles, Dockerfiles or anything on the Pi without asking me first.
- The parent repo only records submodule commits. Bumping it (`git add backend pwa && git commit`) is optional and does not deploy anything.

### Do

- Follow the coding style already being used in the project
- Always research on the version of package working on do not blindly just code.
- Follow Designs strictly for UI.
- Keep the "Security headers" section below and its tests up to date whenever a change touches headers, CSP, CORS, proxying or new response paths.
- If any change requires special consideration for further changes by coding AI agents then modify this file's sections Do and Don't etc accordingly so that it preserves consistency and avoids further issues.
- **Git workflow: commit every time.** Work on a feature branch (never directly on `main`) and commit after every logical unit of work (a service, a screen, a fix, a test file group), not one big commit at the end, so anything can be reverted on its own. Do this inside each submodule that changed. Use conventional messages (`feat:`, `fix:`, `test:`, `chore:`, `docs:`), keep each commit to one concern, run the checks before committing when the change is more than trivial, and never commit secrets or `.env` files. Push the branch and open a PR when the work is ready or when asked; merging and deploying to `live` stay with the user.

### Don't

- Commit env secrets
- Change any setting or app configuration without permission even if auto mode is on stop the execution with warning let me do that or command you to do it.
- Push to `live`, merge a PR, force-push, or commit straight onto `main` without my approval. (Feature branches and commits are expected, see "Git workflow" under Do.)
- Send any API response without the security headers, or add inline scripts, `eval`/`new Function`, or third-party hosts to the PWA (see "Security headers")

### GitGuardian

- Each submodule has a `.gitguardian.yaml` that skips test files only (`tests/**`, `*.test.ts(x)`, `src/test/**`). App code, config and docs are always scanned.
- The path ignore is not reliable on the GitHub PR check (it flagged dummy passwords in backend tests anyway). In NEW tests, do not write password-shaped literals (`password: 'Something1!'`, `const PASSWORD = '...'`): build them at runtime, e.g. `fakePassword()` in `backend/tests/helpers/passwords.ts`. Short throwaway values like `'x'` for "wrong input" cases are fine. Never put a real secret in a test: the path ignore would hide it.
- Incidents GitGuardian already opened for dummy values must be resolved as false positives in its dashboard by the owner; code changes can't close them.
- A fake secret outside tests (e.g. `.env.example` placeholders) gets `# ggignore` on the same line. Do not widen the ignored paths to silence a finding in real code: fix or rotate it.

### Features

@FEATURES.md

## Architecture(BACKEND)

- No cache, intentional since 3 users max. Sessions and reset tokens are read from Postgres.
- No queue, intentional since 3 users max. Emails are sent in the request itself.
- Account password is 8 to 16 characters (upper, lower, digit, special), intentional: the users must be able to remember it. Do not raise the maximum. The vault itself is protected by the separate recovery key (see SECURITY.md), not by this password.

### Security headers (API and static site)

Both must stay in place. Treat a missing header as a bug.

- **API (`/api`)**: `helmet()` is installed in `backend/src/core/app/App.ts` before any route and before the rate limiter, so every response (success, validation error, 404, 429, 500) carries the headers. Do not mount anything ahead of it, disable it, or override its headers per route. A new kind of response path (file or stream responses, a different middleware chain, another app mounted beside it) must be added to `backend/tests/integration/security-headers.test.ts`.
- **Static PWA**: nginx on the Pi sends the headers, from `/etc/nginx/snippets/pivault-security-headers.conf`, included in the `/`, `/index.html` and `/sw.js` locations of `/etc/nginx/sites-available/pivault`. It is not in any repo and is not deployed by CI. The policy is:
  - CSP: `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self'; connect-src 'self'; worker-src 'self'; manifest-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'`
  - `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `X-Frame-Options: DENY`, `Cross-Origin-Opener-Policy: same-origin`, `Permissions-Policy` (camera, microphone, geolocation, payment off).
- **CORS**: only the origin of `APP_URL` may read API responses (`allowedOrigins` in `App.ts`, tested in `backend/tests/integration/cors.test.ts`). Never use `cors()` with no options or a wildcard origin. Production is same-origin, so the PWA must call `/api` on its own origin.
- **Frontend rules that follow from the CSP**: no inline `<script>` or event-handler attributes, no `eval`/`new Function` in app code, no third-party scripts, fonts, images or API hosts, and the API is only called on the same origin (`/api`). If a feature genuinely needs a new source, stop and ask, because the change is made on the Pi.
- Changing the header set means editing the Pi snippet (ask first) and updating this section in the same change.

### Emailed links: set password + verify email

- The welcome email (and later forgot-password) carries `APP_URL/reset-password?key=<token>`. The PWA page `/reset-password` opens in a plain browser tab (it is in `PWA_BYPASS_PATHS`, keep it there; `ROUTES.resetPassword` is tested against it).
- API (public, not under `/internal`): `POST /api/auth/password-links/validate` and `/complete`, bodies `{ token }` and `{ token, password }`. The token travels in the body only, never in a URL or query string, and is never logged. Responses are `Cache-Control: no-store`.
- Every unknown, expired, spent or malformed token gets the same HTTP 400 with `INVALID_LINK_MESSAGE` ("This link is invalid or has expired"). The PWA matches that exact message (`pwa/src/constants/password-link.constants.ts`); change both sides together.
- `complete` checks the token before hashing (a bad token must never cost an argon2 hash), then in ONE transaction deletes the token (compare-and-delete, so it is single-use even under races), sets the password hash and `passwordChangedAt`, sets `emailVerifiedAt` if empty, and deletes the user's session. Keep that atomic: it lives in `PrismaPasswordResetRepository.consumeToSetPassword`.
- Login refuses users whose `emailVerifiedAt` is null (403, only after the password was right). Keep it that way.
- The password policy has one backend source, `backend/src/modules/user/password.schema.ts`, mirrored in `pwa/src/utils/password-rules.utils.ts`. Do not duplicate it elsewhere.
- The PWA strips `?key=` from the address bar right after reading it and keeps the token in memory only. Never store it in storage or caches.
- Rate limits: `PASSWORD_LINK_VALIDATE` and `PASSWORD_LINK_COMPLETE` in `ratelimits.constants.ts`. New public paths belong in `security-headers.test.ts`.
- Expired tokens are swept whenever a new token is issued (no scheduler, by design).
- The Pi nginx must serve `/reset-password` with the SPA fallback and must not block `/api/auth/*` (only `/api/internal/*` is blocked). Verify after deploy; changing it is a Pi change, ask first.

### Login, sessions and signed requests

- Public: `POST /api/auth/login` `{ email, password, publicKey }` and `POST /api/auth/forgot-password` `{ email }`. Signed: `POST /api/auth/logout`, `POST /api/auth/recovery-key-configured` (idempotent; records that the device made its recovery key, which is what makes a later login report `recoveryKeyConfigured: true`; optional body `{ keyCheck }`) and `POST /api/auth/recovery-key-check` (returns `{ keyCheck }` or null). All `no-store`, rate limited (`LOGIN`, `FORGOT_PASSWORD`), listed in `security-headers.test.ts`.
- `publicKey` is the device's ECDSA P-256 public key as **SPKI DER, base64url** (the PWA's `exportPublicKey` must emit base64url, not plain base64). Anything else is a 400. Login returns `{ token, expiresAt, user, recoveryKeyConfigured }`; the raw token is shown once and only its SHA-256 is stored. One user, one session: a new login replaces the old one.
- Wrong password and unknown email give the same 401 (`INVALID_CREDENTIALS_MESSAGE`) and both pay one argon2 verify. Per-account brake: 5 failures per 15 min then 429 (`LoginThrottle`, in memory by design, on top of the per-IP limit). Never log passwords or tokens.
- Authenticated requests send `Authorization: Bearer <token>`, `X-Timestamp` (ms epoch, within 60s of the server) and `X-Signature`: base64url raw r||s ECDSA-SHA256 over `"<timestamp>.<METHOD>.<path without query>.<bodyHash>"` (`signaturePayload` in `session.constants.ts`), signed with the non-extractable device key. `bodyHash` is the SHA-256 of the exact body bytes (base64url), also sent as `X-Body-SHA256`. Guard: `SessionAuthGuard`; every failure is the same 401. New protected routes use `requireSession` (signature check plus `verifyBody`) and read the user with `authOf(res)`. Every `express.json()` must pass `verify: recordBodyHash`, and a router that parses its own body (vault) must run `session.handle`, then its parser, then `session.verifyBody`. `ACCEPT_LEGACY_SIGNATURES` still lets older apps sign without the body; turn it off once every installed app has updated.
- PWA side: signed calls go through `pwa/src/services/signed-api.client.ts` (`postSigned`); its payload format is mirrored in `pwa/src/constants/auth.constants.ts`, so change both sides together. Auth state is the `AuthStatus` machine (`logged-out -> needs-setup -> locked -> unlocked`); every screen sits behind a `ZoneGuard` zone in `routes/app.routes.tsx`.
- Device storage is plain IndexedDB (`pwa/src/db/idb.ts`, stores `keys` and `auth`), because `CryptoKey`s need structured clone. Keys are always non-extractable; never store a recovery key, raw key bytes or the PIN. The PIN is a salted PBKDF2 hash with a lockout, and is only a UI gate (it does not encrypt anything).
- Sign out wipes everything on the device (session, device key, vault key, PIN) after a best-effort server logout, so the next login asks for the recovery key again. Setup order on first login: PIN, then recovery key; the server is told about the key before it is stored locally.
- Lock screen: the PIN unlocks on the sixth digit (`unlockWithPin`), biometric unlock is a WebAuthn `get()` with user verification (`unlockWithBiometric`), and both only load the saved non-extractable vault key into `secure-memory.service` (memory only). 5 wrong PINs start a growing lockout. A missing vault key or PIN record signs the user out with a message. The unlocked zone locks itself after 60s in the background (`useAutoLock`). A session past `expiresAt` is wiped at startup.
- Recovery-key check: the server stores `users.recoveryKeyCheck`, a fixed text encrypted on the device with the vault key (opaque to the server, first one wins). A recovery key typed on a new device must decrypt it (`key-check.service.ts`, `verifyAgainstServer`), even when the vault is empty. Accounts from before the check fall back to decrypting a few entries, and get their check stored after the next unlock (`ensureKeyCheckStored`). Never send the key itself.
- Forgot-password intentionally tells the caller when the email is unknown (FEATURES.md, a handful of known users). Do not "fix" that without asking.

### Vault data (notes and passwords)

- Flow: type -> zod validate -> encrypt -> RxDB on the device -> (later) sync. Code: `pwa/src/services/vault.service.ts` (seal/open, CRUD, `watchEntries`), `pwa/src/db/` (RxDB, Dexie storage), `VaultProvider` (decrypted entries, in memory only, inside the unlocked zone).
- **One key, one blob per entry** (AES-256-GCM, fresh 12-byte IV per write). SECURITY.md's "one encrypted payload" is read as "one key, no hierarchy": per-entry blobs are needed for sync and per-item counts. Do not add per-record keys or a key hierarchy.
- A stored document is ONLY `{ id (ULID), updatedAt, envelope }`. Title, username, password, url, notes and the entry `kind` live inside the ciphertext. Never add a plaintext field that carries user content (it would also reach the server). `updatedAt` and the id are the only metadata.
- The id is bound to its blob as AES-GCM additional data (`ENTRY_AAD_PREFIX`), so a blob moved to another record fails to decrypt. Keep passing `aad` for every entry encrypt/decrypt. Entries that fail to decrypt are hidden and counted, never shown.
- Decrypted data must not outlive the unlocked zone: do not copy entries into storage, caches, logs, URLs or global state outside `VaultProvider`. Sign out (and an expired session) delete the whole vault database (`removeVaultDb` in `wipeDevice`).
- RxDB: Dexie storage only, `rxdb/plugins/core` + `storage-dexie`. Do NOT import the pipeline plugin or schema validators (`validate-ajv`, `validate-z-schema`): they use `eval` / `new Function`, which the CSP forbids. Validation is zod. Check that the production bundle has no `eval(` / `new Function(` after any RxDB change.
- UI rules for secrets: passwords hidden until asked, copy clears the clipboard after 30s (`copySecret`), the website is only linked when `safeHttpUrl` accepts it, search never matches passwords.
- Server side (next): the server stores and returns the encrypted envelopes only. It must never receive a key, a recovery key or plaintext, and must not parse envelope contents.

### Dependencies and the runtime image

- `npm audit --omit=dev` must stay at 0 in both `backend/` and `pwa/`. Dev-only findings whose only suggested fix is a downgrade (typescript-eslint/minimatch, workbox-build, the shadcn CLI) are accepted.
- The backend image ships production dependencies only (`npm prune --omit=dev` in the Dockerfile builder stage). Anything needed at runtime or at deploy time must be in `dependencies`. `prisma` is one on purpose: Jenkins runs `prisma migrate deploy` inside that image. Never move it back to `devDependencies`.
- Bump `prisma`, `@prisma/client` and `@prisma/adapter-pg` together, to the same version. After a Prisma bump, check `uniqueViolationFields` in `backend/src/core/database/prismaErrors.ts`: Prisma 7.10 changed how unique violations are reported, which made a duplicate mobile look like a duplicate email. `backend/tests/unit/core/prisma-errors.test.ts` and the Postgres repository tests cover it.
- The `shadcn` CLI is a `devDependency` in `pwa/` (its CSS is imported at build time only).

### Client IP and rate limiting

- The API runs behind Cloudflare, cloudflared and nginx, so the TCP peer is always loopback. Rate limits must be keyed on the real client IP.
- nginx (`/etc/nginx/sites-available/pivault`, `/api/` location) sets `X-Forwarded-For` to exactly one value: Cloudflare's `CF-Connecting-IP` when it is a valid IP, otherwise the peer address. It must never append (`$proxy_add_x_forwarded_for` makes every visitor `127.0.0.1`, one shared bucket) and never be empty (`LocalOnlyGuard` treats a missing `X-Forwarded-For` as "not proxied" and would let `/api/internal/*` through).
- Express uses `trust proxy` 1, so `req.ip` is the last `X-Forwarded-For` entry. Do not change either side without the other. `backend/tests/integration/rate-limit-client-ip.test.ts` enforces the backend half.
- Per-user limits (e.g. login attempts per account) are separate from this and must not rely on IP alone.

### Tech stack

### Frontend (/pwa)

- Vite 8
- React 19
- Typescript 6
- Tailwind CSS 4
- Zod 4
- ShadCn 4

### Backend

- Node Js 24 LTS
- Typescript 6
- Express 5
- Express Rate Limiter 8
- Prisma 7
- Vitest 4
