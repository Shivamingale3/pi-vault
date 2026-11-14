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

### Don't

- Commit env secrets
- Change any setting or app configuration without permission even if auto mode is on stop the execution with warning let me do that or command you to do it.
- Commit or push code without my approval

### Features

@FEATURES.md

## Architecture(BACKEND)

- No cache, intentional since 3 users max. Sessions and reset tokens are read from Postgres.
- No queue, intentional since 3 users max. Emails are sent in the request itself.

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
