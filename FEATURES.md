# Features

### No register UI, I'll directly register user via a local register api.
- Flow -> First Name, Last Name, Mobile, Email, Password -> Check duplicate -> Hash password & store in DB -> Send Welcome Email.
- Email validation, mobile validation, password (8 to 16 characters, A-Z, a-z, 0-9, at least one special character).
- Email & mobile both unique
- Online feature requires internet

### Login
- On the client side we create a public/private key pair and send public key with email & password to the backend if credentials are verified then create session and store public key on backend
- We will use Web Crypto API to generate Public/Private pair for which public will be exportable but private will will not be exportable and persisted to IndexedDB. refer to @SECURITY.md for details.
- There won't be multiple sessions 1 user 1 session. Once logged in to another device previous session along with public key is removed from the server.

### Post Login App Setup
- Once login is done we'll show the app lock setup screen, a 6 digit app lock PIN & option to enable biometric unlock, so on each app open we have biometric & PIN both app unlock options.  
- During first login  we'll check if the recovery key is created (meaning first Login?) if not then we create a key on frontend let the user save it. Then create another private non extractable key from that recovery key and store that object into the indexedDB.
- Then on each app open the device will always be logged in but the app will be locked using a pin/biometric on successful app unlock we'll unwrap the vault key stored and decrypt the whole vault notes and passwords and load them. 

### Forgot password
- Email if not exists(throw error) & if exists then the request is processed right away, the UI waits for it.
- Backend will create a temp random token(some random string can be even a uuid/ulid), save it to DB with an expiry of 15 mins and send the link in email.
- Email will have a link to platform /reset-password?key="random token". Page will take key from url and send to backend.
- Backend will validate the key(if not throw and show error) and then if valid send 200 to frontend which will show new password inputs and on submit send password and token the password will be set for the user for whom the token is stored against the token 
- The key in this context is the token itself just referenced as key on the frontend.

### Vault
- Once app lock is opened then we extract the key from indexedDB and then the vault has the key in memory and every read/write happens using that key
- The vault will consist of 2 types of content notes and passwords.
- Every entry will be saved to device's RxDB first, if the internet connection is available it is immediately synced to the server.
- Since there can't be multiple sessions if a session has unsynced changes they are lost.
- Note or Password both if created, updated and deleted must be in the RxDB first then synced to DB.

### Settings
- This is the whole link between the client and server.
- Will show logged in account
- Will show sync status details how many synced (created, updated, deleted)
