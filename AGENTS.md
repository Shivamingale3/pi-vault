# PiVault
- Personal Vault application to save password and notes encrypted and fully secure.
- An application which is created to be fully supported for all devices, screen sizes and operating system using the PWA(Progressive Web Application) system. 
- Offline first, sync to server support when online. 
- ReactJS frontend and a NodeJs backend which will be hosted on my pi server.
- Designs - https://claude.ai/artifact/6n9ZGZdMXnQm8DzNVnV1MZ#artboard-a76f8dd437c2


# Architecture (FRONTEND)
- Layered architecture, a folder per layer inside the src folder(types, constants, enums, interfaces, hooks, providers, contexts, pages, components, components/modules, routes, services, utils, db(RxDB)).
- Proper unit tests & integration tests in Vitest.
- ShadCn components
- Since the app is offline first whatever we write goes first into the client DB(RxDB) and then on the server when sync. 
- ULID everywhere


# Coding Practices
- No file bigger than 200 LOC
- Segregation of concerns, separate the UI & logic and call them in a file to complete a page/module.
- Follow theme, token, proper design conventions to maintain consistency across components and pages.
- Strictly follow the designs unless explicitly told to ignore or modify - https://claude.ai/artifact/6n9ZGZdMXnQm8DzNVnV1MZ#artboard-a76f8dd437c2
- 1 thing at a time, while executing we are going to focus on one work/module/part unless multiple required.
- Reuse - whatever's possible
- Every screen/page just be full height and width, no manual height, width, padding, margin, spacing unless required. Always distribute space.
- Do not use tailwind classes that give yellow squiggles "deprecated" or "can be written as" if it can be written as then already write it that way.
- App is a vault made for storing passwords and secrets so security is the first angle to think from.
- Every write goes into client side DB first this helps with offline first architecture.
- Every exception must be caught/thrown/handled and shown toast wherever required.
- For UI never write components manually, always check if relevant shadcn component for it is available, if yes then use it.
- Once you finish writing code, always write its tests, then check for type errors and lint errors and do not quit just after writing code, always write tests, fix them fix type errors and lint errors.

# Do
- Follow the coding style already being used in the project
- Always research on the version of package working on do not blindly just code.
- Follow Designs strictly for UI.

# Don't
- Commit env secrets
- Change any setting or app configuration without permission even if auto mode is on stop the execution with warning let me do that or command you to do it.
- Commit or push code without my approval 

# Features
@FEATURES.md

# Tech stack

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