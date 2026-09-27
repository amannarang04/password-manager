# SecureVault — Local-First, Zero-Knowledge Password Manager

A password manager built to demonstrate real cryptographic engineering, not
just CRUD-with-a-lock-icon. The database is designed so that anyone who
steals `securevault.db` gets nothing usable without the master password.

## Why this design

| Requirement | How it's met |
|---|---|
| Master password never stored | Only an Argon2id salt + an encrypted verifier blob are stored. The password itself, and the key derived from it, exist only in server RAM for the life of an unlocked session. |
| Passwords unreadable at rest | Every sensitive field (site, username, password, notes) is individually encrypted with AES-256-GCM before it touches SQLite. |
| Tamper detection | AES-GCM's authentication tag means a modified ciphertext fails to decrypt instead of silently returning garbage. |
| No cross-record substitution | Each ciphertext is bound to `"vault-entry"` as associated data, so a ciphertext can't be copied from one row into another undetected. |
| Brute-force resistance | Argon2id (memory-hard, 64MB/3 iterations) makes offline guessing of the master password expensive; failed-login delay adds exponential backoff online. |
| Auto-lock | Sessions expire after 5 minutes of inactivity; a "Lock Now" button ends the session immediately. |
| Secure sharing without a trusted server | Each user has an X25519 keypair. Sharing uses a sealed-box construction (ephemeral ECDH + HKDF + AES-256-GCM) so the server stores ciphertext it cannot decrypt, and only the recipient's private key can open it. |
| Breach checking without leaking passwords | Uses the HIBP Pwned Passwords k-anonymity API: only the first 5 hex chars of a SHA-1 hash are ever sent over the network. |

## Architecture

```
Frontend (vanilla JS SPA)
        │  HTTPS / fetch
        ▼
FastAPI backend
   ├── crypto.py     Argon2id KDF + AES-256-GCM (the trust boundary)
   ├── sessions.py   in-memory session store, auto-lock timer
   ├── sharing.py    X25519 sealed-box credential sharing
   ├── generator.py  CSPRNG password generation + entropy estimation
   ├── health.py     weak / reused / old password detection
   ├── breach.py     HIBP k-anonymity breach checking
   └── database.py   SQLAlchemy models — every sensitive column is *_enc
        │
        ▼
   SQLite (ciphertext only)
```

Data flow for one saved password:

```
Master Password
      │  Argon2id(salt)      ← salt stored, password is not
      ▼
Encryption Key (RAM only, never persisted)
      │  AES-256-GCM(nonce, plaintext, aad="vault-entry")
      ▼
Ciphertext  ──────────────────────────────►  SQLite row
```

## Running it

```bash
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

Then open `frontend/index.html` directly in a browser (no build step —
it's a single self-contained page that calls the API at `127.0.0.1:8000`).

Run the security self-tests (proves the claims above, not just that the
code executes):

```bash
cd backend
python test_security.py
```

## What's implemented vs. roadmap

**Implemented and tested:**
- Argon2id key derivation + AES-256-GCM vault encryption
- Full vault CRUD, all fields encrypted at rest
- Secure password generator (CSPRNG-based) with entropy scoring
- Password health dashboard (weak / reused / old detection)
- Breach checking via HIBP k-anonymity
- Auto-lock on inactivity + manual lock
- Failed-login exponential backoff
- Secure credential sharing via X25519 sealed boxes (server never sees plaintext)

**Roadmap (documented, not built — good "what's next" talking points in an interview):**
- Browser extension for autofill (would reuse the same API; the hard part —
  encryption — is already solved. The extension is mostly a content-script +
  messaging-passing project on top of this backend.)
- Multi-device sync with conflict resolution
- Hardware-key (WebAuthn/FIDO2) as a second unlock factor
- Rate-limiting/backoff backed by Redis instead of in-process memory, for
  multi-instance deployment

## Resume bullets

```
SecureVault — Zero-Knowledge Password Manager
Python · FastAPI · SQLite · AES-256-GCM · Argon2id · X25519

• Built a local-first password manager where sensitive data is unreadable
  at rest: every field is individually encrypted with AES-256-GCM, keyed by
  an Argon2id-derived key that is never persisted to disk.
• Implemented secure credential sharing using X25519 sealed-box encryption,
  so shared passwords are never visible to the server in plaintext.
• Added breach checking via the HIBP k-anonymity API, ensuring full
  passwords never leave the device.
• Designed session handling with auto-lock on inactivity and exponential
  backoff on failed logins to mitigate brute-force attacks.
• Wrote a security-focused test suite validating tamper detection, wrong-
  password rejection, and encryption correctness — not just functional
  correctness.
```

This story is much stronger in an interview than "I made a CRUD app,"
because you can talk through *why* each design decision was made — that's
what turns a portfolio project into a real signal of engineering judgment.
