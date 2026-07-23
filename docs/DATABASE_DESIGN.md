# Database Design — C++ Banking System

> **Part of the modernization documentation set.**
> See [MODERNIZATION_REVIEW.md](../MODERNIZATION_REVIEW.md) for the index.

---

## Table of Contents

1. [Current File-Based Schema](#1-current-file-based-schema)
2. [Entity Relationships (Current)](#2-entity-relationships-current)
3. [Known Problems with the Flat-File Model](#3-known-problems-with-the-flat-file-model)
4. [Proposed SQLite Schema](#4-proposed-sqlite-schema)
5. [Indexes and Constraints](#5-indexes-and-constraints)
6. [Migration Notes](#6-migration-notes)
7. [Security Improvements](#7-security-improvements)

---

## 1. Current File-Based Schema

All persistence is in `.txt` files located at the repository root. Fields are separated by `#//#`.
The parser in each entity class splits on this delimiter, so any user-supplied value containing `#//#` would corrupt the record.

---

### 1.1 `MyUsers.txt`

Stores application operator accounts.

| # | Field name | C++ type | Notes |
|---|---|---|---|
| 1 | `first_name` | `string` | |
| 2 | `last_name` | `string` | |
| 3 | `email` | `string` | Not validated for format |
| 4 | `phone` | `string` | |
| 5 | `username` | `string` | Login identifier; uniqueness enforced in memory |
| 6 | `encrypted_password` | `string` | Caesar-shift +3 encoding of plain-text password |
| 7 | `permissions` | `int` | Bitmask; `-1` = all permissions |

**Real sample row** (from `MyUsers.txt`):
```
Admin#//#Admin#//#NO EMAIL#//#00000000#//#Admin#//#4567#//#-1
```
> `4567` is `Admin` encrypted with Caesar +3 (`A`→`D`, `d`→`g`, etc.).

---

### 1.2 `MyClients.txt`

Stores bank customer account data.

| # | Field name | C++ type | Notes |
|---|---|---|---|
| 1 | `first_name` | `string` | |
| 2 | `last_name` | `string` | |
| 3 | `email` | `string` | |
| 4 | `phone` | `string` | |
| 5 | `account_number` | `string` | e.g. `A102`; treated as PK |
| 6 | `pin_code` | `string` | Stored in plain text |
| 7 | `account_balance` | `float` | Serialized as `%.6f` |

**Real sample row**:
```
Khaili#//#Ahmed#//#Khalil#//#8928982#//#A102#//#1234#//#748.000000
```

---

### 1.3 `LoginRegister.txt`

Append-only login audit trail.

| # | Field name | C++ type | Notes |
|---|---|---|---|
| 1 | `login_date` | `string` | e.g. `Sun - 6/8/2023 - 17:48:56` (non-ISO) |
| 2 | `username` | `string` | FK reference to `MyUsers.txt`.username |
| 3 | `encrypted_password` | `string` | Password copy **should not be stored** |
| 4 | `permissions` | `int` | Snapshot of permission bitmask at login time |

**Real sample row**:
```
Sun - 6/8/2023 - 17:48:56#//#Admin#//#4567#//#-1
```

---

### 1.4 `TransferRegister.txt`

Append-only transfer audit trail.

| # | Field name | C++ type | Notes |
|---|---|---|---|
| 1 | `transfer_date` | `string` | Non-ISO format, inconsistent (some rows missing day prefix) |
| 2 | `source_account` | `string` | FK reference to `MyClients.txt`.account_number |
| 3 | `dest_account` | `string` | FK reference to `MyClients.txt`.account_number |
| 4 | `amount` | `float` | |
| 5 | `source_balance_after` | `float` | Balance of source after transfer |
| 6 | `dest_balance_after` | `float` | Balance of destination after transfer |
| 7 | `operator_username` | `string` | FK to `MyUsers.txt`.username |

**Real sample row**:
```
6/8/2023 - 0:41:7#//#A104#//#A07#//#5000.000000#//#5000.000000#//#11123.000000#//#User1
```

---

### 1.5 `Currencies.txt`

Currency master data.

| # | Field name | C++ type | Notes |
|---|---|---|---|
| 1 | `country_name` | `string` | |
| 2 | `currency_code` | `string` | e.g. `USD`, `EUR`; treated as PK |
| 3 | `currency_name` | `string` | |
| 4 | `rate_to_usd` | `double` | USD = 1.0 base |

**Real sample rows**:
```
United States of America#//#USD#//#US Dollar#//#1.000000
France#//#EUR#//#Euro#//#0.900000
```

---

### 1.6 `ReceivedMailBox/<Username>ReceivedBox.txt`

Per-user inbox. One file per user, created when user account is created.

| # | Field name | C++ type | Notes |
|---|---|---|---|
| 1 | `sent_date` | `string` | |
| 2 | `sender_username` | `string` | FK to `MyUsers.txt`.username |
| 3 | `title` | `string` | Subject line |
| 4 | `body` | `string` | Message body |

---

### 1.7 `SendedMailBox/<Username>SendedBox.txt`

Per-user sent-mail box. Note: `Sended` is a naming error; should be `Sent`.

| # | Field name | C++ type | Notes |
|---|---|---|---|
| 1 | `sent_date` | `string` | |
| 2 | `receiver_username` | `string` | FK to `MyUsers.txt`.username |
| 3 | `title` | `string` | |
| 4 | `body` | `string` | |

---

## 2. Entity Relationships (Current)

```
users (MyUsers.txt)
  │  username  (logical PK)
  ├──< login_events  (LoginRegister.txt)   via username
  ├──< sent_mail     (SendedMailBox/)      via username
  └──< received_mail (ReceivedMailBox/)    via username

clients (MyClients.txt)
  │  account_number  (logical PK)
  ├──< transfer_source  (TransferRegister.txt)  via source_account
  └──< transfer_dest    (TransferRegister.txt)  via dest_account

currencies (Currencies.txt)
  │  currency_code  (logical PK)
  └─  (used only at runtime for conversion; no FK references)
```

---

## 3. Known Problems with the Flat-File Model

| Problem | Effect |
|---|---|
| No real primary key enforcement | Duplicate account numbers possible if two processes run simultaneously |
| No referential integrity | Deleting a user leaves orphan mailbox files and dangling audit log rows |
| Full file rewrite on every update | O(n) write cost; concurrent access causes corruption |
| `#//#` delimiter collision | User data containing `#//#` silently corrupts the record |
| Non-atomic transfer | 3 separate file writes; crash → inconsistent balances |
| `float` precision | `float` loses precision above ~16 million; balances in `Currencies.txt` use `double` but client balances use `float` |
| Password stored in audit log | `LoginRegister.txt` contains the encrypted (not hashed) password on every login row |
| No timestamp standard | Date strings use format `Sun - D/M/YYYY - H:MM:SS`; some rows lack the day-name prefix |

---

## 4. Proposed SQLite Schema

The following DDL creates a SQLite database (`bank.db`) that is functionally equivalent to the flat-file model while fixing all of the above problems.

```sql
-- ─────────────────────────────────────────────
-- bank.db — SQLite schema for CppBankingSystem
-- ─────────────────────────────────────────────

PRAGMA journal_mode = WAL;        -- safe concurrent reads while writing
PRAGMA foreign_keys = ON;         -- enforce FK constraints
PRAGMA strict = ON;               -- column type enforcement (SQLite 3.37+)

-- ─── Users ────────────────────────────────────
CREATE TABLE IF NOT EXISTS users (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    first_name   TEXT    NOT NULL,
    last_name    TEXT    NOT NULL,
    email        TEXT    NOT NULL DEFAULT '',
    phone        TEXT    NOT NULL DEFAULT '',
    username     TEXT    NOT NULL UNIQUE,
    password_hash TEXT   NOT NULL,   -- bcrypt / Argon2id hash; never plain text
    permissions  INTEGER NOT NULL DEFAULT 0,
    created_at   TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    updated_at   TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

-- ─── Clients / Accounts ───────────────────────
CREATE TABLE IF NOT EXISTS clients (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    first_name      TEXT    NOT NULL,
    last_name       TEXT    NOT NULL,
    email           TEXT    NOT NULL DEFAULT '',
    phone           TEXT    NOT NULL DEFAULT '',
    account_number  TEXT    NOT NULL UNIQUE,
    pin_hash        TEXT    NOT NULL,   -- hashed PIN; never plain text
    balance_cents   INTEGER NOT NULL DEFAULT 0,  -- store as integer cents
    created_at      TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    updated_at      TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

-- ─── Login Audit Log ──────────────────────────
CREATE TABLE IF NOT EXISTS login_events (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    username    TEXT    NOT NULL,   -- denormalized snapshot for audit immutability
    permissions INTEGER NOT NULL,  -- snapshot at time of login
    -- NOTE: password MUST NOT be stored here
    logged_at   TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

-- ─── Transfer Register ────────────────────────
CREATE TABLE IF NOT EXISTS transfer_events (
    id                     INTEGER PRIMARY KEY AUTOINCREMENT,
    source_account_number  TEXT    NOT NULL,   -- snapshot; client may be deleted later
    dest_account_number    TEXT    NOT NULL,
    amount_cents           INTEGER NOT NULL CHECK (amount_cents > 0),
    source_balance_after_cents INTEGER NOT NULL CHECK (source_balance_after_cents >= 0),
    dest_balance_after_cents   INTEGER NOT NULL CHECK (dest_balance_after_cents >= 0),
    operator_user_id       INTEGER REFERENCES users(id) ON DELETE SET NULL,
    operator_username      TEXT    NOT NULL,   -- snapshot for audit immutability
    transferred_at         TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

-- ─── Currencies ───────────────────────────────
CREATE TABLE IF NOT EXISTS currencies (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    country_name   TEXT    NOT NULL,
    currency_code  TEXT    NOT NULL UNIQUE,
    currency_name  TEXT    NOT NULL,
    rate_to_usd    REAL    NOT NULL CHECK (rate_to_usd > 0),
    updated_at     TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

-- ─── Mail Messages ────────────────────────────
-- Replaces per-user text files with a single normalized table.
CREATE TABLE IF NOT EXISTS mail_messages (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    sender_user_id  INTEGER REFERENCES users(id) ON DELETE SET NULL,
    sender_username TEXT    NOT NULL,   -- snapshot for immutability
    receiver_user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    receiver_username TEXT  NOT NULL,
    title           TEXT    NOT NULL DEFAULT '',
    body            TEXT    NOT NULL DEFAULT '',
    sent_at         TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    -- soft-delete flags replace "clear mailbox" file truncation
    deleted_by_sender   INTEGER NOT NULL DEFAULT 0 CHECK (deleted_by_sender IN (0,1)),
    deleted_by_receiver INTEGER NOT NULL DEFAULT 0 CHECK (deleted_by_receiver IN (0,1))
);

-- ─── Update triggers ──────────────────────────
-- Keep updated_at current when rows are modified.

CREATE TRIGGER IF NOT EXISTS users_updated_at
AFTER UPDATE ON users
BEGIN
    UPDATE users SET updated_at = strftime('%Y-%m-%dT%H:%M:%fZ', 'now')
    WHERE id = NEW.id;
END;

CREATE TRIGGER IF NOT EXISTS clients_updated_at
AFTER UPDATE ON clients
BEGIN
    UPDATE clients SET updated_at = strftime('%Y-%m-%dT%H:%M:%fZ', 'now')
    WHERE id = NEW.id;
END;

CREATE TRIGGER IF NOT EXISTS currencies_updated_at
AFTER UPDATE ON currencies
BEGIN
    UPDATE currencies SET updated_at = strftime('%Y-%m-%dT%H:%M:%fZ', 'now')
    WHERE id = NEW.id;
END;
```

---

## 5. Indexes and Constraints

```sql
-- Lookups by username are frequent (login, mail send).
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_username      ON users(username);

-- Lookups by account number drive every transaction and client find.
CREATE UNIQUE INDEX IF NOT EXISTS idx_clients_acct        ON clients(account_number);

-- Audit log queries by user and by date range.
CREATE INDEX IF NOT EXISTS idx_login_events_user_id       ON login_events(user_id);
CREATE INDEX IF NOT EXISTS idx_login_events_logged_at     ON login_events(logged_at);

-- Transfer log queries by account number and by date.
CREATE INDEX IF NOT EXISTS idx_transfers_source_acct      ON transfer_events(source_account_number);
CREATE INDEX IF NOT EXISTS idx_transfers_dest_acct        ON transfer_events(dest_account_number);
CREATE INDEX IF NOT EXISTS idx_transfers_transferred_at   ON transfer_events(transferred_at);

-- Currency lookup by code is the hot path for conversion.
CREATE UNIQUE INDEX IF NOT EXISTS idx_currencies_code     ON currencies(currency_code);

-- Mail queries: inbox for receiver, sent-box for sender.
CREATE INDEX IF NOT EXISTS idx_mail_receiver              ON mail_messages(receiver_user_id, deleted_by_receiver);
CREATE INDEX IF NOT EXISTS idx_mail_sender                ON mail_messages(sender_user_id, deleted_by_sender);
```

---

## 6. Migration Notes

### 6.1 One-time migration steps

The following describes how to migrate each flat file into the SQLite database. The migration should be done as a separate offline utility (`tools/migrate_files_to_sqlite.cpp`) before switching the application to use the new persistence layer.

#### Users (`MyUsers.txt` → `users`)
1. Read each `#//#`-delimited line.
2. Parse fields in order: first_name, last_name, email, phone, username, encrypted_password, permissions.
3. **Hash the password properly**: decrypt the Caesar-shift to recover plain text, then hash with bcrypt/Argon2id.
4. Insert into `users`.

> **Security note:** If the migration is done while the application is still in use, users will need to reset their passwords because the old Caesar-shift cipher is not recoverable as a hash. Plan for a forced password-reset flow or perform migration during a maintenance window.

#### Clients (`MyClients.txt` → `clients`)
1. Parse fields: first_name, last_name, email, phone, account_number, pin_code, account_balance.
2. Convert `account_balance` (float) to integer cents: `balance_cents = round(account_balance * 100)`.
3. **Hash the PIN**: store `pin_hash = hash(pin_code)`.
4. Insert into `clients`.

#### Login register (`LoginRegister.txt` → `login_events`)
1. Parse fields: login_date, username, encrypted_password, permissions.
2. Look up `user_id` from `users` where `username` matches.
3. Convert `login_date` string to ISO 8601 (`YYYY-MM-DDTHH:MM:SSZ`).
4. Insert into `login_events` — **do not migrate the encrypted_password field**.

#### Transfer register (`TransferRegister.txt` → `transfer_events`)
1. Parse fields: transfer_date, source_account, dest_account, amount, source_balance_after, dest_balance_after, operator_username.
2. Convert float amounts to integer cents.
3. Look up `operator_user_id` from `users` where username matches (may be NULL if user was deleted).
4. Convert date strings to ISO 8601.
5. Insert into `transfer_events`.

#### Currencies (`Currencies.txt` → `currencies`)
1. Parse fields: country_name, currency_code, currency_name, rate_to_usd.
2. Insert directly. Duplicate `currency_code` rows (e.g. USD appears twice in the data) should be deduplicated.

#### Mailbox files → `mail_messages`
1. For each file in `SendedMailBox/`: parse sender from filename, parse fields (sent_date, receiver, title, body).
2. Look up sender and receiver `user_id` values.
3. Insert into `mail_messages` with the appropriate deleted flags (default 0 for both).
4. Records in `ReceivedMailBox/` are duplicates of the sent records, so only one set needs to be migrated.

---

### 6.2 Transaction-safe transfer in SQLite

The current code performs a transfer as three independent file rewrites. With SQLite, the same operation becomes a single atomic transaction:

```sql
BEGIN IMMEDIATE;

UPDATE clients
SET balance_cents = balance_cents - :amount_cents,
    updated_at    = strftime('%Y-%m-%dT%H:%M:%fZ', 'now')
WHERE account_number = :source_acct
  AND balance_cents  >= :amount_cents;   -- guard: fail if insufficient funds

UPDATE clients
SET balance_cents = balance_cents + :amount_cents,
    updated_at    = strftime('%Y-%m-%dT%H:%M:%fZ', 'now')
WHERE account_number = :dest_acct;

INSERT INTO transfer_events
  (source_account_number, dest_account_number, amount_cents,
   source_balance_after_cents, dest_balance_after_cents,
   operator_user_id, operator_username)
VALUES
  (:source_acct, :dest_acct, :amount_cents,
   (SELECT balance_cents FROM clients WHERE account_number = :source_acct),
   (SELECT balance_cents FROM clients WHERE account_number = :dest_acct),
   :operator_id, :operator_username);

COMMIT;
```

If any step fails the `ROLLBACK` is automatic. This eliminates the data-corruption window that exists in the current file-based implementation.

---

### 6.3 Recommended SQLite C++ integration

| Library | Approach | Notes |
|---|---|---|
| [sqlite3.h](https://www.sqlite.org/amalgamation.html) (amalgamation) | Add `sqlite3.c` + `sqlite3.h` to the project | Zero external dependency; works on all platforms |
| [SQLiteCpp](https://github.com/SRombauts/SQLiteCpp) | Header-only C++ wrapper around sqlite3 | Cleaner C++ API; available via CMake FetchContent |
| [sqlpp11](https://github.com/rbock/sqlpp11) | Type-safe SQL in C++ | More complex; best suited for Phase 3+ |

For the initial migration the **sqlite3 amalgamation** is recommended because it requires no build system changes and can be dropped directly into the Visual Studio project.

---

## 7. Security Improvements

| Current state | Proposed state |
|---|---|
| Password stored as Caesar-shift +3 in `MyUsers.txt` | Password stored as bcrypt (cost 12) or Argon2id hash |
| PIN stored as plain text in `MyClients.txt` | PIN stored as hashed value |
| Encrypted password copied into every `LoginRegister.txt` row | No credential stored in audit log; `login_events` table has no password column |
| `#//#` delimiter allows injection if input is not sanitized | SQLite prepared statements with parameter binding eliminate injection entirely |
| Files are world-readable at the OS level | Single `bank.db` file with OS-level permissions (chmod 600 / ACL) |
| No record-level integrity check | SQLite `CHECK` constraints, `NOT NULL`, and `UNIQUE` enforce data integrity |

---

*Next: [Build Modernization →](BUILD_MODERNIZATION.md)*
