# hash-based-intergrity-checker
# Expense Ledger Integrity Checker

A PHP web application that uses **HMAC-SHA256 hash chaining** to detect tampering in expense records. Any modification to a stored entry — amount, description, date, or category — is cryptographically detectable without requiring a blockchain or external service.

---

## How It Works

Each expense entry is hashed using HMAC-SHA256. The hash of each entry incorporates the hash of the previous entry, forming a chain. Modifying any entry breaks the chain from that point forward, making tampering immediately detectable on the next verification run.

```
Entry 1:  prev_hash = 000...000 (genesis sentinel)
          entry_hash = HMAC-SHA256(key, canonical(entry1) + prev_bytes)
          = "a3f8c2d1..."

Entry 2:  prev_hash = "a3f8c2d1..."  ← entry 1's hash
          entry_hash = HMAC-SHA256(key, canonical(entry2) + prev_bytes)
          = "9b7e41f3..."

Entry 3:  prev_hash = "9b7e41f3..."  ← entry 2's hash
          entry_hash = HMAC-SHA256(key, canonical(entry3) + prev_bytes)
          = "2c4a88d0..."
```

Changing entry 1's amount from `$349.99` to `$34.99` produces a different hash for entry 1, which the verifier detects instantly.

---

## Features

- **HMAC-SHA256 hash chaining** — tamper-evident ledger using a server-side secret key
- **MySQL storage** — durable, concurrent-safe persistence via WAMP/MySQL
- **Multi-user support** — each user sees only their own ledger; admins see all
- **Secure authentication** — bcrypt passwords, session fixation protection, CSRF tokens on all forms
- **Login rate limiting** — 5 failed attempts triggers a 60-second lockout
- **Tamper simulation** — demo button that mutates data without recomputing the hash, for testing detection
- **Audit log** — every verification run is recorded with timestamp and result
- **CSV export** — download the full ledger including hashes
- **Filter by category and date**
- **Spend breakdown** by category with progress bars
- **Property-based test suite** — 62 tests across 10 correctness properties (PHPUnit + eris)

---

## Requirements

- PHP 8.1+
- MySQL 5.7+ (WAMP, XAMPP, or standalone)
- Composer

---

## Installation

### 1. Clone or copy the project

```
C:\wamp64\www\web\hash based intergrity checker\
```

### 2. Install PHP dependencies

```bash
composer install
```

### 3. Configure the database

Edit `config/database.php`:

```php
return [
    'host'     => '127.0.0.1',
    'port'     => 3306,
    'dbname'   => 'expense_ledger',
    'username' => 'root',
    'password' => '',        // WAMP default
    'charset'  => 'utf8mb4',
];
```

### 4. Run the database migration

```bash
php scripts/migrate.php
```

This creates the `expense_ledger` database and all required tables:
- `users`
- `login_attempts`
- `ledger_entries`
- `audit_log`

### 5. Open in browser

Navigate to:
```
http://localhost/web/hash%20based%20intergrity%20checker/public/
```

---

## First Use

1. Go to `register.php` and create an account
2. Log in at `login.php`
3. Add expense entries using the form on the left
4. Click **Verify** to confirm the chain is intact
5. Use **Tamper Entry** (demo only) to corrupt an entry, then verify again to see detection in action

### Promoting a user to admin

Run this SQL in phpMyAdmin or MySQL CLI:

```sql
UPDATE users SET role = 'admin' WHERE username = 'yourusername';
```

Admins see all users' entries and a registered user list panel.

---

## Project Structure

```
├── config/
│   └── database.php          MySQL connection settings
├── data/
│   └── hmac.key              Auto-generated HMAC secret key (never commit this)
├── js/
│   ├── ledger.js             JavaScript JSON codec matching the PHP schema
│   └── ledger.test.js        Jest + fast-check property tests for the JS module
├── public/
│   ├── index.php             Main ledger dashboard (requires login)
│   ├── login.php             Login form
│   ├── register.php          Registration form
│   └── logout.php            Session destruction
├── scripts/
│   ├── migrate.php           Creates/updates MySQL tables
│   └── test_db.php           End-to-end MySQL + auth smoke test
├── src/
│   ├── Auth.php              Session helper (login, logout, CSRF)
│   ├── AuthStore.php         MySQL user registration and authentication
│   ├── Database.php          PDO connection factory
│   ├── DecodedLedger.php     Immutable deserialized ledger wrapper
│   ├── ExpenseEntry.php      Validated expense entry value object
│   ├── HashedEntry.php       Entry + entry_hash + prev_hash value object
│   ├── Hasher.php            Plain SHA-256 hasher (implements HasherInterface)
│   ├── HasherInterface.php   Common interface for Hasher and HmacHasher
│   ├── HmacHasher.php        HMAC-SHA256 hasher with secret key
│   ├── IntegrityReport.php   Verification result value object
│   ├── Ledger.php            Mutable append-only hash chain aggregate
│   ├── LedgerFactory.php     Wires components into ready-to-use instances
│   ├── MysqlLedgerStore.php  MySQL persistence for the ledger chain
│   ├── Serializer.php        Canonical byte serialization + JSON codec
│   ├── SqliteLedgerStore.php SQLite persistence (legacy, replaced by MySQL)
│   ├── TamperedEntryInfo.php Tamper detection detail value object
│   └── Verifier.php          Chain verification logic
│   └── Exception/
│       ├── NotFoundException.php
│       ├── ParseException.php
│       └── ValidationException.php
├── tests/
│   ├── Property/             Eris property-based tests (10 correctness properties)
│   └── Unit/                 PHPUnit unit tests
├── composer.json
└── phpunit.xml
```

---

## Running Tests

```bash
vendor/bin/phpunit
```

Expected output:
```
Tests: 62, Assertions: 3000+, Deprecations: 2 (from eris library, not project code)
```

The 2 deprecations are PHP 8.3 return type notices inside the `eris` library itself — they do not affect correctness.

### Test smoke test (MySQL + auth)

```bash
php scripts/test_db.php
```

---

## Security Notes

| Concern | Implementation |
|---|---|
| Password storage | bcrypt (cost 12) — never stored in plain text |
| Hash forgery | HMAC-SHA256 with key stored outside web root in `data/hmac.key` |
| CSRF | Per-session token verified on every POST using `hash_equals()` |
| Session fixation | `session_regenerate_id(true)` on login |
| Brute force | 5 failed logins → 60s lockout, tracked in `login_attempts` table |
| SQL injection | PDO prepared statements throughout — no string interpolation of user input |
| Cookie security | `HttpOnly`, `SameSite=Strict` session cookies |

### Important

- **Never commit `data/hmac.key`** — add `data/` to `.gitignore`
- The HMAC key is auto-generated on first run; losing it means existing hash chains cannot be re-verified
- `config/database.php` contains credentials — do not commit with real passwords

---

## Correctness Properties

The test suite verifies 10 formal correctness properties using property-based testing:

| # | Property |
|---|---|
| 1 | Canonical serialization is injective (no two entries produce the same bytes) |
| 2 | Hash chaining — each entry's prev_hash equals the preceding entry's entry_hash |
| 3 | Genesis entry always uses the zero hash as prev_hash |
| 4 | Ledger serialization round-trip — encode then decode produces identical data |
| 5 | Intact ledger always verifies cleanly |
| 6 | Any single-field mutation is detected by the verifier |
| 7 | Tamper detection cascades — the mutated position is always identified |
| 8 | Single-entry verification agrees with full-ledger verification |
| 9 | Empty ledger verifies as intact with zero entries |
| 10 | Entries with invalid/empty fields are rejected without modifying the ledger |

---

## License

MIT
