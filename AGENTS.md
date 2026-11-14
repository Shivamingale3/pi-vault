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

### Don't

- Commit env secrets
- Change any setting or app configuration without permission even if auto mode is on stop the execution with warning let me do that or command you to do it.
- Commit or push code without my approval
- Send any API response without the security headers, or add inline scripts, `eval`/`new Function`, or third-party hosts to the PWA (see "Security headers")

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
- **Login (not built yet) MUST refuse users whose `emailVerifiedAt` is null.**
- The password policy has one backend source, `backend/src/modules/user/password.schema.ts`, mirrored in `pwa/src/utils/password-rules.utils.ts`. Do not duplicate it elsewhere.
- The PWA strips `?key=` from the address bar right after reading it and keeps the token in memory only. Never store it in storage or caches.
- Rate limits: `PASSWORD_LINK_VALIDATE` and `PASSWORD_LINK_COMPLETE` in `ratelimits.constants.ts`. New public paths belong in `security-headers.test.ts`.
- Expired tokens are swept whenever a new token is issued (no scheduler, by design).
- The Pi nginx must serve `/reset-password` with the SPA fallback and must not block `/api/auth/*` (only `/api/internal/*` is blocked). Verify after deploy; changing it is a Pi change, ask first.

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
