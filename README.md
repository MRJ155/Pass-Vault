# pwvault: encrypted command-line password manager

A zero-dependency-on-the-cloud password vault in Python that demonstrates core security engineering concepts: memory-hard key derivation, authenticated encryption, tamper detection, secure randomness, and breach checking with k-anonymity. Built as a learning project; **not** a replacement for an audited manager like Bitwarden or 1Password.

## Quick start

```bash
pip install -r requirements.txt
python -m pwvault init                          # create vault (~/.pwvault/vault.json)
python -m pwvault add github myuser --generate  # generate a strong password and store it
python -m pwvault get github                    # copies to clipboard, auto-clears after 20s
python -m pwvault list
python -m pwvault audit --online                # weak / reused / breached passwords
python -m pwvault passwd                        # change master password (new salt, re-encrypt)
pytest                                          # 16 tests
```

## Security design

| Concept | Where | What it does |
|---|---|---|
| **Memory-hard KDF (Argon2id)** | `crypto.py` | Turns the master password into a 256-bit key using 64 MiB / 3 passes / 4 lanes. Makes offline brute force costly on GPUs/ASICs. |
| **Per-vault random salt** | `crypto.py` | 16-byte CSPRNG salt defeats rainbow tables and means two users with the same password get different keys. |
| **Authenticated encryption (AES-256-GCM)** | `crypto.py` | Confidentiality *and* integrity. Any bit-flip in the file is detected, so the vault can't be silently modified. |
| **AAD header binding** | `vault.py` | The KDF parameters and salt are authenticated as associated data, so an attacker can't downgrade the KDF settings without breaking decryption (covered by a test). |
| **Unique nonce per save** | `vault.py` | A fresh 96-bit random nonce on every write. GCM nonce reuse under one key is catastrophic, so this is tested. |
| **Input validation on untrusted files** | `vault.py` | KDF parameters read from disk are range-checked *before* deriving a key, preventing a crafted file from forcing a huge memory allocation (DoS). |
| **Atomic writes** | `vault.py` | Write to temp file, `fsync`, then `os.replace`, so a crash can never leave a corrupt vault. |
| **Least privilege** | `vault.py` | Vault file is `chmod 600` (owner only). |
| **CSPRNG** | `generator.py` | Uses `secrets`, not `random`. Rejection sampling guarantees all character classes. |
| **k-anonymity breach check** | `breach.py` | Sends only the first 5 hex chars of the SHA-1 hash to Have I Been Pwned and matches the rest locally. The password never leaves the machine. |
| **Clipboard hygiene** | `cli.py` | Passwords are copied rather than printed, then cleared after 20s (only if the clipboard still holds them). |
| **No secrets in args** | `cli.py` | Passwords come from `getpass` prompts (no echo, no shell history). `--password` exists but is documented as discouraged. |
| **Password audit** | `cli.py` | Flags weak (entropy estimate), reused, and (optionally) breached passwords. |

## Threat model

**In scope**
- Attacker steals the vault file (laptop theft, backup leak, cloud-sync exposure) and tries offline guessing. *Mitigated by Argon2id + a strong master password.*
- Attacker modifies the vault file (including KDF parameters). *Detected by GCM + AAD.*
- Shoulder-surfing / shell-history leakage. *Mitigated by no-echo prompts and clipboard clearing.*
- Malicious or corrupted vault file used to crash the tool. *Parameter validation.*

**Out of scope (honest limitations)**
- **Malware or a keylogger on the machine.** Once the vault is unlocked, an attacker with local code execution wins.
- **Memory forensics.** Python strings are immutable and cannot be reliably zeroed, so secrets may linger in RAM. A production tool would use a memory-safe language with explicit zeroization (e.g. Rust + `zeroize`) and `mlock`.
- **Weak master passwords.** No KDF can save `password123`. The tool enforces a 12-character minimum only.
- **No recovery.** Forget the master password and the data is unrecoverable. That's a deliberate trade-off.
- **Entropy estimate is crude.** It overestimates human-chosen passwords. A real implementation would use zxcvbn.
- **Clipboard managers / other apps** may read the clipboard during the 20s window.
- Not independently audited.

## Project layout

```
pwvault/
  crypto.py     # Argon2id KDF, AES-256-GCM encrypt/decrypt
  vault.py      # file format, atomic save, entry CRUD, master password rotation
  generator.py  # CSPRNG password generator and strength estimate
  breach.py     # HIBP k-anonymity client
  cli.py        # argparse commands
tests/test_pwvault.py
```

## Ideas to extend (good interview talking points)
1. Swap the entropy estimate for **zxcvbn** and show how scores differ.
2. Add **TOTP** (RFC 6238) storage and code generation.
3. Add an **auto-lock/idle timeout** and an interactive shell session mode.
4. Add **key wrapping**: a random data-encryption key wrapped by the password-derived key, so changing the master password doesn't require re-encrypting the data, and enabling a recovery key.
5. Add **keyfile / hardware key** as a second factor into the KDF.
6. Write a short **attack demo**: crack a weak master password with a wordlist against your own vault, and show how Argon2 cost parameters change the guess rate.
7. Run **Bandit** and **pip-audit** in a GitHub Actions workflow (SAST + dependency scanning).

---

## Resume bullets (pick 3-4)

**Project title line:** *PwVault: Encrypted Password Manager* | Python, Argon2id, AES-256-GCM, pytest | github.com/&lt;you&gt;/pwvault

- Designed and built a command-line password manager that encrypts credentials with **AES-256-GCM** using keys derived via memory-hard **Argon2id**, with per-vault salts and unique per-write nonces.
- Bound KDF parameters and salt to the ciphertext as **authenticated associated data** and validated untrusted file inputs, defeating parameter-downgrade and resource-exhaustion attacks against a tampered vault.
- Integrated the **Have I Been Pwned k-anonymity API** to audit passwords for breach exposure without transmitting passwords or full hashes; added detection of weak and reused credentials.
- Applied secure-coding practices: CSPRNG-based generation, atomic writes, `0600` file permissions, no-echo prompts, and auto-clearing clipboard.
- Wrote a **16-test pytest suite** covering tamper detection, wrong-key rejection, nonce uniqueness, and key rotation; authored a **threat model** documenting in-scope attacks and known limitations.

*Shorter version (one line):* Built an encrypted CLI password vault (Argon2id + AES-256-GCM) with tamper detection, HIBP k-anonymity breach auditing, and a documented threat model; 16 automated tests.

## Interview talking points
- **Why Argon2id over SHA-256 or bcrypt?** Fast hashes are trivially GPU-cracked; Argon2id is memory-hard and the current OWASP recommendation.
- **Why GCM instead of plain AES-CBC?** CBC gives no integrity, so a modified file could decrypt to garbage or enable padding-oracle attacks. GCM authenticates.
- **What happens if the nonce is reused?** GCM leaks the XOR of plaintexts and the authentication key. Hence a fresh random nonce per save.
- **Why bind the header as AAD?** Without it an attacker could lower the KDF cost in the file, and you'd derive a weaker key.
- **What would you do differently in production?** Memory zeroization, key wrapping, audited libraries only, secure enclave or keychain integration, and zxcvbn.
