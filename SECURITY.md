# Vault Encryption Architecture — Implementation Specification

Implement a client-side encryption system for an offline-first React + TypeScript PWA.

## 1. Core security model

There are two completely separate mechanisms:

**A. Account Authentication**
- Users register and log in using email and password.
- This is only for backend account authentication and authorization.
- The account password must NOT be used to derive the vault encryption key.
- The backend stores an appropriately salted password hash.

**B. Vault Encryption**
- Vault encryption and decryption happen entirely on the client.
- The backend must never receive or store the plaintext recovery key, encryption key, or decrypted vault contents.
- The backend only stores and serves encrypted vault data.

## 2. Recovery Key Generation

During initial account setup:

1. Generate 32 cryptographically secure random bytes using `crypto.getRandomValues()`.
2. This represents a 256-bit random Recovery Key.
3. Encode it as 64 hexadecimal characters.
4. Display it in a UUID-style grouped format: 8 groups of 8 hexadecimal characters separated by hyphens.
5. Require the user to copy and securely save it.

The recovery key must be shown during initial setup and must never be transmitted to the backend.

This is a machine-generated, high-entropy secret, not a human-created password. Do not apply SHA-256, Argon2id, or another password hashing function to it.

## 3. Import Recovery Key as a Non-Extractable CryptoKey

Convert the hexadecimal recovery code back into its original 32-byte representation.

Import it using the Web Crypto API:

- Algorithm: AES-GCM
- Key length: 256 bits
- Extractable: false
- Key usages: encrypt, decrypt

Use `crypto.subtle.importKey()`.

The resulting object is a non-extractable `CryptoKey`.

Do not retain or persist the original plaintext recovery code.

## 4. Persist CryptoKey in IndexedDB

Persist the imported `CryptoKey` directly in IndexedDB.

Requirements:
- Use IndexedDB, not localStorage or sessionStorage.
- Store the CryptoKey object directly through IndexedDB structured cloning.
- Do not export the key into raw bytes or a string for persistence.
- Do not encrypt the CryptoKey again with another locally stored key.
- Retrieve the existing CryptoKey from IndexedDB on subsequent visits.

The intention is to reuse the same non-extractable CryptoKey across browser sessions without asking the user for the recovery code every time.

## 5. Vault Encryption

The entire vault is treated as one encrypted payload.

When saving the vault:

1. Serialize vault data into JSON.
2. Convert the JSON into UTF-8 bytes using TextEncoder.
3. Generate a fresh cryptographically random 12-byte IV for every encryption operation.
4. Encrypt using `crypto.subtle.encrypt()` with AES-256-GCM and the persisted CryptoKey.
5. Store the IV and ciphertext together.

Use a versioned encrypted envelope format containing:
- `version`
- `algorithm`
- `iv`
- `ciphertext`

The IV is not secret and can be stored alongside the ciphertext.

Never reuse an IV with the same AES-GCM key.

## 6. Normal Application Startup

After successful email/password authentication:

1. Request the encrypted vault blob from the backend.
2. Look for the persisted CryptoKey in IndexedDB.
3. If found, decrypt the vault locally.
4. If not found, ask the user for their Recovery Key.
5. Import the Recovery Key as a non-extractable CryptoKey.
6. Persist it in IndexedDB.
7. Decrypt the downloaded vault.

Once unlocked, keep decrypted data in application memory as needed for normal application functionality.

## 7. Recovery / New Device

When a user opens their account on a new device or browser:

1. Authenticate using email and password.
2. Download the encrypted vault.
3. Detect that no local CryptoKey exists.
4. Ask the user to enter their Recovery Key.
5. Import the key using `crypto.subtle.importKey()` with `extractable: false`.
6. Persist the resulting CryptoKey in IndexedDB.
7. Decrypt the vault locally.

The same Recovery Key must work across all authorized devices.

## 8. Backend Responsibilities

The backend must only handle:
- Account registration and authentication.
- Authorization to access the user's encrypted vault.
- Storage and retrieval of the encrypted vault blob.
- Encrypted blob synchronization.

The backend must never receive:
- Recovery Key.
- AES CryptoKey.
- Plaintext vault data.

## 9. Security and Implementation Constraints

- Use the browser's native Web Crypto API.
- Use AES-256-GCM with a 96-bit random IV and 128-bit authentication tag.
- Never use Math.random() for cryptographic randomness.
- Never store the recovery code in browser storage.
- Never make the CryptoKey extractable.
- Never silently fall back to storing raw key material if IndexedDB persistence fails.
- Handle missing or corrupted IndexedDB data.
- Handle AES-GCM authentication failures without exposing sensitive details.
- Require HTTPS in production (Web Crypto secure context).
- Remember that non-extractable does not prevent malicious same-origin JavaScript from using the key. Protect the application against XSS.

## 10. Scope Restrictions

Do not introduce:
- JWT as a vault encryption mechanism.
- A master-key / key-encryption-key hierarchy.
- HKDF-derived per-record encryption keys.
- Separate data encryption keys.
- An asymmetric key pair for vault encryption.
- Server-side encryption or decryption of vault contents.

Keep the architecture minimal: one Recovery Key, one AES-256-GCM CryptoKey, one encrypted vault payload.

Before implementation, inspect the existing project and present the proposed changes, affected modules, and persistence strategy. Do not change unrelated application functionality.