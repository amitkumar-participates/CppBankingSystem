# Refactor Roadmap — C++ Banking System

> **Part of the modernization documentation set.**
> See [MODERNIZATION_REVIEW.md](../MODERNIZATION_REVIEW.md) for the index.

---

## Table of Contents

1. [Overview and Guiding Principles](#1-overview-and-guiding-principles)
2. [Phase 1 — Safe Cleanup (No Behavior Changes)](#2-phase-1--safe-cleanup-no-behavior-changes)
3. [Phase 2 — Separate Concerns](#3-phase-2--separate-concerns)
4. [Phase 3 — Safe Data and Security](#4-phase-3--safe-data-and-security)
5. [Phase 4 — Modern C++17/C++20 and Portability](#5-phase-4--modern-c17c20-and-portability)
6. [Security and Data Integrity Risks](#6-security-and-data-integrity-risks)
7. [Modernization Priority Matrix](#7-modernization-priority-matrix)

---

## 1. Overview and Guiding Principles

This roadmap converts the project from a tightly coupled MSVC console application into a clean, cross-platform, testable C++ codebase in four sequential phases. Each phase has:
- A stated **goal**.
- A list of **concrete tasks** with acceptance criteria.
- An **estimated effort** (developer days, relative).
- Notes on what can be done **without breaking the existing build**.

### Guiding principles

1. **Never break the existing build** during any phase. Each phase must leave the Visual Studio project in a buildable state.
2. **Add tests before refactoring logic.** Introduce the testing framework in Phase 1 and add tests for any module before changing it.
3. **Move in one direction.** Each phase builds on the previous. Do not attempt Phase 3 changes (SQLite) before Phase 2 (repository abstraction) is complete.
4. **Name things accurately.** Fix typos and misleading names (e.g. `SendedMailBox` → `SentMailBox`, `Transcations` → `Transactions`, `_perfromDelete` → `performDelete`) as you touch each file.
5. **Preserve audit trails.** `LoginRegister.txt` and `TransferRegister.txt` data must be retained and migrated, not deleted.

---

## 2. Phase 1 — Safe Cleanup (No Behavior Changes)

**Goal:** Remove non-portable and code-quality issues without changing any observable behavior.
**Effort:** ~5 developer days
**Prerequisite:** None. Can start immediately.

---

### Task 1.1 — Add CMake build file

**What:** Add `CMakeLists.txt` to the repository root (see [BUILD_MODERNIZATION.md](BUILD_MODERNIZATION.md) Section 4).
**Acceptance:** `cmake -S . -B build && cmake --build build` produces a working `BankSystem.exe` on Windows.
**Risk:** Low. Does not touch any `.cpp` or `.h` files.

---

### Task 1.2 — Remove `using namespace std;` from all headers

**Affected files:** Every `.h` in `Models/`, `Screens/`, `Utility/`.

**What to do:**
- Remove the `using namespace std;` line.
- Add `std::` prefix wherever `std` types are used (`std::string`, `std::vector`, `std::cout`, etc.).

**Before:**
```cpp
// clsBankClient.h
#include <string>
using namespace std;

class clsBankClient : public clsPerson {
    string _accountNumber;
    vector<clsBankClient> _fromFileToVectorOfClient();
};
```

**After:**
```cpp
// clsBankClient.h
#include <string>
#include <vector>

class clsBankClient : public clsPerson {
    std::string _accountNumber;
    std::vector<clsBankClient> _fromFileToVectorOfClient();
};
```

**Acceptance:** Project compiles without `using namespace std;` in any header.
**Risk:** Low. No logic changes.

---

### Task 1.3 — Replace `__declspec(property)` with standard getters/setters

**What it is:** MSVC-only syntax that exposes `get`/`set` as a property:
```cpp
__declspec(property(get=GetFirstName, put=SetFirstName)) string FirstName;
```

**What to do:** Replace each `__declspec(property(...))` declaration with an inline accessor pair:
```cpp
// Before:
__declspec(property(get=GetFirstName, put=SetFirstName)) std::string FirstName;

// After: remove the __declspec line; the getter/setter methods already exist.
// Callers that used  obj.FirstName  become  obj.GetFirstName()  /  obj.SetFirstName(v)
```

**Note:** All call sites that used `object.PropertyName` must be updated to `object.GetPropertyName()` or `object.SetPropertyName(value)`. This is a mechanical find-and-replace task.

**Acceptance:** Project compiles with MSVC `/permissive-` (strict conformance) and with GCC/Clang.
**Risk:** Medium. Requires updating all call sites, but logic does not change.

---

### Task 1.4 — Introduce `const` correctness

**What to do:**
- Mark read-only methods `const`: `bool IsEmpty() const`, `std::string GetAccountNumber() const`, etc.
- Change pass-by-value `string` parameters to `const std::string&`.
- Mark getters in `clsPerson`, `clsBankClient`, `clsUser`, `clsCurrency` as `const`.

**Before:**
```cpp
string GetFirstName() { return _firstName; }
static bool IsClientExist(string AccountNumber) { ... }
```

**After:**
```cpp
const std::string& GetFirstName() const { return _firstName; }
static bool IsClientExist(const std::string& accountNumber);
```

**Acceptance:** No `const`-correctness warnings with `-Wall -Wextra`.
**Risk:** Low. Purely additive type information.

---

### Task 1.5 — Replace `system("cls")` and `system("pause>0")`

**What to do:** Add a portable utility header `Utility/ConsoleHelper.h`:
```cpp
// ConsoleHelper.h
#pragma once

namespace ConsoleHelper {

inline void clearScreen() {
#ifdef _WIN32
    std::system("cls");
#else
    // ANSI escape: move cursor home, clear screen
    std::cout << "\033[2J\033[H" << std::flush;
#endif
}

inline void waitForKey() {
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    std::cin.get();
}

} // namespace ConsoleHelper
```

Replace all `system("cls")` calls with `ConsoleHelper::clearScreen()` and `system("pause>0")` / `system("pause")` calls with `ConsoleHelper::waitForKey()`.

**Acceptance:** No direct `system()` calls remain in the codebase.
**Risk:** Low. Behavior is identical on Windows; Linux/macOS now has a working implementation.

---

### Task 1.6 — Replace global `LoginUser` singleton

**What it is:** `Utility/Global.h` defines:
```cpp
clsUser LoginUser = clsUser::Find("", "");
```
This is a global variable with default initialization at program startup, included in many files.

**What to do:** Introduce a `SessionContext` struct and pass it by reference or pointer:
```cpp
// SessionContext.h
#pragma once
#include "clsUser.h"

struct SessionContext {
    clsUser currentUser;
    bool    isLoggedIn = false;

    explicit SessionContext(clsUser user)
        : currentUser(std::move(user)), isLoggedIn(true) {}
};
```

- `clsLoginScreen::ShowLoginScreen()` creates a `SessionContext` on login success.
- Pass `SessionContext&` down to `clsMainMenu` and all feature screens.
- Remove `Global.h` and all `#include "Global.h"` directives.

**Acceptance:** No global variables remain. `LoginUser` is passed explicitly.
**Risk:** Medium. Touches many files, but the change is mechanical (add parameter, remove `#include "Global.h"`).

---

### Task 1.7 — Split implementations into `.cpp` files

**What to do:** For each heavily used model file, create a matching `.cpp`:
- `Models/Bank/Client/clsBankClient.cpp`
- `Models/Bank/User/clsUser.cpp`
- `Models/Bank/Currency/clsCurrency.cpp`

Move all method bodies (especially the large static helpers) from the `.h` into the `.cpp`. Keep only declarations in the `.h`.

**Acceptance:** Each class `.h` contains only declarations. Incremental recompile after changing one `.cpp` does not rebuild the whole project.
**Risk:** Medium. Must update `CMakeLists.txt` and `.vcxproj` with new `.cpp` files. No logic changes.

---

## 3. Phase 2 — Separate Concerns

**Goal:** Decouple persistence, domain logic, and UI into distinct layers.
**Effort:** ~10 developer days
**Prerequisite:** Phase 1 complete.

---

### Task 2.1 — Add unit test framework

**What to do:**
1. Add `Catch2` via CMake `FetchContent`:
```cmake
include(FetchContent)
FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG        v3.5.2
)
FetchContent_MakeAvailable(Catch2)

enable_testing()
add_subdirectory(tests)
```
2. Create `tests/CMakeLists.txt` and a `tests/test_parsing.cpp` that tests the delimiter-splitting logic.
3. Start writing tests before refactoring each class.

**Acceptance:** `ctest` runs and reports pass/fail.
**Risk:** Low. Test infrastructure is additive.

---

### Task 2.2 — Extract repository interfaces

**What to do:** Create pure-virtual repository interfaces in `src/persistence/interfaces/`:

```cpp
// IClientRepository.h
#pragma once
#include <optional>
#include <vector>
#include "domain/Client.h"

struct IClientRepository {
    virtual ~IClientRepository() = default;

    virtual std::optional<Client> findByAccountNumber(const std::string& acctNo) const = 0;
    virtual std::vector<Client>   findAll()                                             const = 0;
    virtual bool                  save(const Client& client)                                  = 0;
    virtual bool                  remove(const std::string& acctNo)                          = 0;
};
```

Create similar interfaces for `IUserRepository`, `ICurrencyRepository`, `IAuditRepository`, `IMailRepository`.

**Acceptance:** Interfaces exist and are implemented by both the old `File*Repository` (backward-compat) and the new `Sqlite*Repository` (Phase 3).
**Risk:** Low. Purely additive.

---

### Task 2.3 — Extract file repository implementations

**What to do:** Move the flat-file parsing/serializing logic out of `clsBankClient`, `clsUser`, and `clsCurrency` into dedicated repository classes that implement the interfaces:

```cpp
// FileClientRepository.h / .cpp
#pragma once
#include "IClientRepository.h"

class FileClientRepository : public IClientRepository {
public:
    explicit FileClientRepository(std::string filePath);

    std::optional<Client> findByAccountNumber(const std::string& acctNo) const override;
    std::vector<Client>   findAll()                                       const override;
    bool                  save(const Client& client)                            override;
    bool                  remove(const std::string& acctNo)                    override;

private:
    std::string _filePath;
    static Client   _parseLine(const std::string& line);
    static std::string _serializeLine(const Client& client);
};
```

The entity classes (`clsBankClient`, etc.) then become **pure domain objects** with no `fstream` dependencies.

**Acceptance:** `clsBankClient.h` contains no `#include <fstream>`. All file I/O is in `FileClientRepository.cpp`.
**Risk:** High. This is the most invasive change in Phase 2. Must be done per-entity, one at a time, with tests at each step.

---

### Task 2.4 — Extract service layer

**What to do:** Move multi-step workflows out of screen classes and into services:

```cpp
// TransferService.h
#pragma once
#include "persistence/IClientRepository.h"
#include "persistence/IAuditRepository.h"

class TransferService {
public:
    TransferService(IClientRepository& clients, IAuditRepository& audit);

    enum class TransferResult { Success, InsufficientFunds, SourceNotFound, DestNotFound };
    TransferResult transfer(const std::string& sourceAcct,
                            const std::string& destAcct,
                            int64_t             amountCents,
                            const std::string& operatorUsername);
private:
    IClientRepository& _clients;
    IAuditRepository&  _audit;
};
```

Screen classes become thin: they collect input, call the service, and display the result.

**Acceptance:** Screen classes contain no direct file access and no business rules beyond input routing.
**Risk:** Medium. Mechanical extraction, but many call sites.

---

## 4. Phase 3 — Safe Data and Security

**Goal:** Fix the security and data integrity problems identified in the risk section.
**Effort:** ~8 developer days
**Prerequisite:** Phase 2 complete (repository interfaces in place).

---

### Task 3.1 — Replace reversible password with a real hash

**What to do:**
1. Add a `PasswordHash` utility (using `bcrypt` or a SHA-256 + salt wrapper from OpenSSL, or a single-header library like [`bcrypt.h`](https://github.com/rg3/bcrypt)):
```cpp
namespace PasswordHash {
    std::string hash(const std::string& plainText);
    bool        verify(const std::string& plainText, const std::string& hash);
}
```
2. Replace all calls to `clsUtil::EncryptText(password, 3)` with `PasswordHash::hash(password)`.
3. Replace all calls to `clsUtil::DecryptText(stored, 3)` with `PasswordHash::verify(plain, stored)`.
4. Run the file migration utility to re-hash all stored passwords (see [DATABASE_DESIGN.md](DATABASE_DESIGN.md) Section 6).
5. Remove the `encrypted_password` field from `LoginRegister.txt` and from the `login_events` SQLite table.

**Acceptance:** `MyUsers.txt` no longer contains recoverable passwords. `LoginRegister.txt` no longer contains any credential field.
**Risk:** High (security-critical). Requires a migration step. Users will need to change passwords or a migration password is derived.

---

### Task 3.2 — Replace `float` balances with integer cents

**What to do:**
- Change `_accountBalance` from `float` to `int64_t` (representing cents).
- All deposit/withdraw/transfer amounts become `int64_t`.
- Display layer divides by 100 for rendering.
- Serialization writes the integer directly.

**Before:**
```cpp
float _accountBalance;
void Deposit(float Amount) { _accountBalance += Amount; }
```

**After:**
```cpp
int64_t _balanceCents = 0;
void deposit(int64_t amountCents) {
    if (amountCents <= 0) throw std::invalid_argument("amount must be positive");
    _balanceCents += amountCents;
}
```

**Acceptance:** No `float` or `double` used for monetary values. All arithmetic is integer.
**Risk:** Medium. Affects every code path that touches balances. Must update display formatting.

---

### Task 3.3 — Add SQLite persistence layer

**What to do:**
1. Add `sqlite3` amalgamation to the project (copy `sqlite3.c` and `sqlite3.h` into `vendor/sqlite3/`).
2. Implement `SqliteClientRepository`, `SqliteUserRepository`, etc., following the interfaces defined in Phase 2.
3. Implement the transfer transaction as a single `BEGIN IMMEDIATE ... COMMIT` block (see [DATABASE_DESIGN.md](DATABASE_DESIGN.md) Section 6.2).
4. Run the migration utility to populate `bank.db` from existing `.txt` files.
5. Wire the application to use SQLite repositories by default; keep file repositories as a `--legacy-files` fallback.

**Acceptance:** All operations work via SQLite. Transfer is atomic. Password not stored in audit table.
**Risk:** High (most complex task). Must be done behind the repository interface so the rest of the code does not change.

---

### Task 3.4 — Add input sanitization

**What to do:**
- Before any value is serialized to a flat file, validate it does not contain `#//#`.
- For SQLite, use prepared statements exclusively (parameter binding, not string concatenation).

**Acceptance:** No SQL string concatenation with user input. No `#//#` in serialized output.
**Risk:** Low. Additive validation logic.

---

## 5. Phase 4 — Modern C++17/C++20 and Portability

**Goal:** Adopt modern C++ idioms, achieve a green cross-platform build.
**Effort:** ~5 developer days
**Prerequisite:** Phases 1–3 complete.

---

### Task 4.1 — Use `std::optional` and `std::variant` for error propagation

Replace the current pattern of returning "empty" objects to signal failure:

**Before:**
```cpp
// Returns an "empty" client if not found
clsBankClient Client = clsBankClient::Find(acctNo);
if (Client.IsEmpty()) { ... }
```

**After:**
```cpp
std::optional<Client> result = clientRepo.findByAccountNumber(acctNo);
if (!result) { ... }
```

---

### Task 4.2 — Use `std::string_view` for read-only string parameters

Replace `const std::string&` parameters in parsing and lookup functions with `std::string_view` where the function does not store the string:

```cpp
// Before
static bool IsClientExist(const std::string& AccountNumber);

// After
static bool isClientExist(std::string_view accountNumber);
```

---

### Task 4.3 — Use `std::chrono` and `std::format` for dates

Replace the custom `clsDate` string-based date formatting with standard types:

```cpp
// Before (clsDate.h)
string GetSystemDateTimeString();   // returns "Sun - 6/8/2023 - 17:48:56"

// After (C++20)
#include <chrono>
#include <format>

auto now = std::chrono::system_clock::now();
std::string timestamp = std::format("{:%Y-%m-%dT%H:%M:%SZ}", now);
```

---

### Task 4.4 — Use `enum class` for permission flags

**Before:**
```cpp
enum enPermissions {
    eAll = -1, eShowClients = 1, eAddNewClient = 2, ...
};
```

**After:**
```cpp
enum class Permission : int {
    None          = 0,
    ShowClients   = 1 << 0,
    AddNewClient  = 1 << 1,
    DeleteClient  = 1 << 2,
    UpdateClient  = 1 << 3,
    FindClient    = 1 << 4,
    Transactions  = 1 << 5,
    ManageUsers   = 1 << 6,
    LoginRegister = 1 << 7,
    All           = ~0
};

// Enable bitwise operators for Permission
inline Permission operator|(Permission a, Permission b) {
    return static_cast<Permission>(static_cast<int>(a) | static_cast<int>(b));
}
inline bool hasPermission(Permission set, Permission required) {
    return (static_cast<int>(set) & static_cast<int>(required)) != 0;
}
```

---

### Task 4.5 — Validate cross-platform build in CI

Wire the GitHub Actions workflow defined in [BUILD_MODERNIZATION.md](BUILD_MODERNIZATION.md) Section 8 and ensure:
- `windows-latest` (MSVC) passes.
- `ubuntu-latest` (GCC/Clang) passes.
- All `ctest` unit tests pass on both platforms.

---

## 6. Security and Data Integrity Risks

### 6.1 Security risks

| Risk | Current severity | Recommended fix | Phase |
|---|---|---|---|
| Reversible Caesar-shift password storage | 🔴 Critical | Replace with bcrypt/Argon2id | Phase 3 |
| Plain-text PIN in `MyClients.txt` | 🔴 Critical | Hash PINs before storage | Phase 3 |
| Password copied into `LoginRegister.txt` | 🔴 Critical | Remove password from audit log immediately | Phase 1* |
| Data files world-readable on disk | 🟠 High | Set OS file permissions (chmod 600 / ACL); use SQLite | Phase 3 |
| No input sanitization (delimiter injection) | 🟠 High | Validate `#//#` not in input; use SQLite parameterized queries | Phase 3 |
| `system("cls")` command injection (low risk) | 🟡 Medium | Replace with portable helper | Phase 1 |
| No session expiry or re-authentication | 🟡 Medium | Add timeout or re-auth after inactivity | Phase 4 |

> *The password should be removed from `LoginRegister.txt` as a standalone hotfix **before** the full Phase 3 work, because it is the single highest-risk item in the codebase.

---

### 6.2 Data integrity risks

| Risk | Current severity | Recommended fix | Phase |
|---|---|---|---|
| Non-atomic transfer (3 separate file rewrites) | 🔴 Critical | Single SQLite transaction | Phase 3 |
| Full file rewrite on every update — concurrent access corrupts file | 🟠 High | File locking (short term); SQLite WAL (long term) | Phase 2/3 |
| `float` balance precision loss | 🟡 Medium | Integer cents | Phase 3 |
| No timestamp standard in audit files | 🟡 Medium | ISO 8601 across all files | Phase 1 |
| Delimiter collision in data | 🟡 Medium | Input validation; SQLite | Phase 3 |
| No FK enforcement between files | 🟡 Medium | SQLite FK constraints | Phase 3 |
| Orphaned mailbox files when user deleted | 🟢 Low | Cascade delete in SQLite | Phase 3 |

---

### 6.3 Maintainability risks

| Risk | Current severity | Recommended fix | Phase |
|---|---|---|---|
| No unit tests | 🟠 High | Add Catch2; start with parsing and transfer tests | Phase 2 |
| All logic in headers — full recompile on any change | 🟡 Medium | Split into `.cpp` files | Phase 1 |
| Static-only architecture — cannot mock or inject | 🟡 Medium | Repository interfaces + constructor injection | Phase 2 |
| Circular-style include risk | 🟡 Medium | Forward declarations; split into `.cpp` | Phase 1/2 |
| Misleading / incorrect names (`SendedMailBox`, `Transcations`) | 🟢 Low | Rename as you touch each file | Phase 1 |

---

## 7. Modernization Priority Matrix

The table below orders all tasks by **value delivered** against **effort required**, to help prioritize when resources are limited.

| Priority | Task | Value | Effort | Phase |
|---|---|---|---|---|
| 1 | Remove password from `LoginRegister.txt` | 🔴 Security fix | 1 hour | 1* |
| 2 | Hash passwords with bcrypt/Argon2id | 🔴 Security fix | 2 days | 3 |
| 3 | Atomic transfer via SQLite transaction | 🔴 Data safety | 3 days | 3 |
| 4 | Remove `__declspec(property)` | 🟠 Portability | 1 day | 1 |
| 5 | Remove `using namespace std;` from headers | 🟡 Code quality | 0.5 days | 1 |
| 6 | Add `CMakeLists.txt` | 🟡 Build parity | 0.5 days | 1 |
| 7 | Replace `float` balances with integer cents | 🟡 Precision | 2 days | 3 |
| 8 | Add unit tests (Catch2) | 🟡 Safety net | 2 days | 2 |
| 9 | Replace `system("cls")` / `system("pause")` | 🟡 Portability | 0.5 days | 1 |
| 10 | Extract repository interfaces | 🟡 Architecture | 3 days | 2 |
| 11 | Replace `Global.h` with `SessionContext` | 🟡 Architecture | 2 days | 1 |
| 12 | Split headers into `.cpp` files | 🟢 Build speed | 2 days | 1 |
| 13 | SQLite persistence layer | 🟢 Long-term | 5 days | 3 |
| 14 | `std::optional` / `std::variant` | 🟢 Modern C++ | 2 days | 4 |
| 15 | `std::chrono` / `std::format` for dates | 🟢 Modern C++ | 1 day | 4 |
| 16 | `enum class` for permissions | 🟢 Type safety | 1 day | 4 |
| 17 | Cross-platform CI pipeline | 🟢 Quality | 1 day | 4 |

---

*Back to: [Architecture →](ARCHITECTURE.md) | [Database Design →](DATABASE_DESIGN.md) | [Build Modernization →](BUILD_MODERNIZATION.md)*
