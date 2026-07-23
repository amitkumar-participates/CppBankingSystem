# Architecture Specification — C++ Banking System

> **Part of the modernization documentation set.**
> See [MODERNIZATION_REVIEW.md](../MODERNIZATION_REVIEW.md) for the index.

---

## Table of Contents

1. [Business Flow](#1-business-flow)
2. [Code Flow](#2-code-flow)
3. [Layering Analysis](#3-layering-analysis)
4. [Class Responsibilities](#4-class-responsibilities)
5. [Feature-Specific Code Flows](#5-feature-specific-code-flows)
6. [Architectural Observations](#6-architectural-observations)
7. [Proposed Target Architecture](#7-proposed-target-architecture)

---

## 1. Business Flow

### 1.1 User Authentication

```
start
  └─ main()
       └─ clsLoginScreen::ShowLoginScreen()
            ├─ prompt: username / password
            ├─ clsUser::Find(username, password)
            │    └─ scan MyUsers.txt, Caesar-decrypt, compare
            ├─ [success] populate global LoginUser
            │            record LoginRegister.txt row
            │            route → clsMainMenu::MainMenuScreen()
            └─ [failure × 3] lock session (no exit loop)
```

**Lock behavior:** After three failed login attempts the application enters an infinite do/while loop and cannot be escaped without killing the process. This is a hardcoded limit in `clsLoginScreen`.

---

### 1.2 Main Menu Navigation

After a successful login the operator selects one of ten numbered options:

| # | Menu item | Screen class | Permission bit |
|---|---|---|---|
| 1 | Show all clients | `clsShowClients` | `eShowClients` (bit 0) |
| 2 | Add new client | `clsAddNewClient` | `eAddNewClient` (bit 1) |
| 3 | Delete client | `clsDeleteClient` | `eDeleteClient` (bit 2) |
| 4 | Update client | `clsUpdateClient` | `eUpdateClient` (bit 3) |
| 5 | Find client | `clsFindClient` | `eFindClient` (bit 4) |
| 6 | Transactions | `clsTransactionsMenu` | `eTransactions` (bit 5) |
| 7 | Manage users | `clsManageUsersScreen` | `eManageUsers` (bit 6) |
| 8 | Login register | `clsLoginRegisterScreen` | `eLoginRegister` (bit 7) |
| 9 | Communication | `clsCommunicationScreen` | (no separate bit) |
| 10 | Currency exchange | `clsCurrencyExchangeScreen` | (no separate bit) |
| 11 | Logout | returns to login screen | — |

Permission `-1` (all bits set) is granted to the built-in `Admin` user.

---

### 1.3 Client Lifecycle

#### Add client
1. Operator enters account number; system checks uniqueness against `MyClients.txt`.
2. If duplicate: reject and re-prompt.
3. Operator fills all fields (name, email, phone, PIN, opening balance).
4. `clsBankClient::Save()` → `_add()` → appends one delimited line to `MyClients.txt`.

#### Update client
1. Look up account number → load client from file into memory.
2. Present current values; operator edits individual fields.
3. `Save()` → `_update()` → rewrites entire `MyClients.txt` with updated row.

#### Delete client
1. Look up account number → load client.
2. Operator confirms.
3. `MarkForDelete()` sets `_markForDelete = true`.
4. `_perfromDelete()` (note typo in source) rewrites `MyClients.txt` excluding the marked row.

---

### 1.4 Transaction Flow

#### Deposit
1. Look up account → load client.
2. Validate amount > 0.
3. `Deposit(amount)` → `_accountBalance += amount` → `Save()` (full file rewrite).

#### Withdraw
1. Look up account → load client.
2. Validate amount ≤ balance.
3. `Withdraw(amount)` → `_accountBalance -= amount` → `Save()`.

#### Transfer (most complex flow)
1. Look up source account.
2. Look up destination account.
3. Validate amount ≤ source balance.
4. Source: `Withdraw(amount)` → `Save()`.
5. Destination: `Deposit(amount)` → `Save()`.
6. `_saveTransferInRegister(...)` → appends row to `TransferRegister.txt`.

> ⚠️ **No atomicity.** Steps 4, 5, and 6 each independently rewrite files. A crash between any two steps leaves data in an inconsistent state.

---

### 1.5 User Management

Privileged operators (those with `eManageUsers` bit set) can perform full CRUD on `MyUsers.txt`. When a new user is created the system also creates two mailbox files:
- `ReceivedMailBox/<Username>ReceivedBox.txt`
- `SendedMailBox/<Username>SendedBox.txt`

---

### 1.6 Communication (Mailbox)

| Action | Files touched |
|---|---|
| Send email | Appends to `SendedMailBox/<Sender>SendedBox.txt` and `ReceivedMailBox/<Receiver>ReceivedBox.txt` |
| View inbox | Reads `ReceivedMailBox/<LoginUser>ReceivedBox.txt` |
| View sent | Reads `SendedMailBox/<LoginUser>SendedBox.txt` |
| Clear inbox | Truncates / deletes `ReceivedMailBox/<LoginUser>ReceivedBox.txt` |
| Clear sent | Truncates / deletes `SendedMailBox/<LoginUser>SendedBox.txt` |

---

### 1.7 Currency Exchange

All currency conversion is USD-normalised:

```
target_amount = source_amount / source_rate_to_USD * target_rate_to_USD
```

Rates are read from `Currencies.txt`. An operator with update permission can overwrite the rate for any currency row (full file rewrite).

---

## 2. Code Flow

### 2.1 Entry and top-level call chain

```
BankSystem.cpp: main()
  └─ clsLoginScreen::ShowLoginScreen()          [Screens/Main Menu/Login/]
       └─ _login()
            ├─ clsUser::Find(user, pass)         [Models/Bank/User/]
            │    ├─ _fromFileToVUsers()           reads MyUsers.txt
            │    └─ _convertLinetoUserObject()    splits on #//#, decrypts password
            ├─ LoginUser = <found user>           Global.h — global mutable state
            ├─ LoginUser.SaveInRegisterLogin()    appends to LoginRegister.txt
            └─ clsMainMenu::MainMenuScreen()      [Screens/Main Menu/]
```

### 2.2 Client add call chain

```
clsAddNewClient::ShowAddNewClientScreen()
  ├─ clsScreen::CheckPermissions(eAddNewClient)   permission guard
  ├─ clsBankClient::GetAddNewClientObject(acctNum)
  │    └─ returns clsBankClient in eAddMode
  ├─ _readClientInfo(client)                      console prompts
  └─ client.Save()
       └─ _add()
            ├─ IsClientExist(acctNum) → re-check uniqueness
            └─ _addDataToFile(line, "MyClients.txt")
```

### 2.3 Transfer call chain

```
clsTransferScreen::ShowTransferScreen()
  ├─ clsBankClient::Find(sourceAcct)
  ├─ clsBankClient::Find(destAcct)
  ├─ validate amount <= source.AccountBalance
  └─ source.Transfer(amount, dest)
       ├─ Withdraw(amount) → Save()    rewrite MyClients.txt
       ├─ dest.Deposit(amount) → dest.Save()   rewrite MyClients.txt again
       └─ _saveTransferInRegister(date, src, dst, amt, srcBal, dstBal, LoginUser)
            └─ _addDataToFile(line, "TransferRegister.txt")
```

### 2.4 Send email call chain

```
clsSendEmailScreen::ShowSendEmailScreen()
  ├─ clsUser::Find(receiverUsername)
  ├─ build stMailMessage {date, title, body}
  └─ LoginUser.SendMail(title, body, receiverUsername)
       ├─ _sendMail(...)    append to SendedMailBox/<Sender>SendedBox.txt
       └─ receiver.ReceiveMail(...)
            └─ append to ReceivedMailBox/<Receiver>ReceivedBox.txt
```

---

## 3. Layering Analysis

### 3.1 De-facto layers (current state)

Although no formal architectural boundaries exist, the code naturally groups into three layers:

```
┌────────────────────────────────────────────────────┐
│  UI / Screen Layer      (Screens/)                 │
│  Console rendering, input collection, menu routing │
│  clsScreen base, clsMainMenu, feature screens      │
├────────────────────────────────────────────────────┤
│  Domain + Persistence Layer   (Models/)            │
│  Entity classes that ALSO perform all file I/O     │
│  clsPerson, clsBankClient, clsUser, clsCurrency    │
├────────────────────────────────────────────────────┤
│  Utility Layer          (Utility/)                 │
│  String, date, input validation, simple encryption │
│  clsString, clsDate, clsInputValidate, clsUtil     │
└────────────────────────────────────────────────────┘
```

> **Problem:** The Domain and Persistence layers are merged. `clsBankClient`, `clsUser`, and `clsCurrency` each contain both domain rules and direct `fstream` file access. This prevents unit testing and makes it impossible to swap the storage engine.

### 3.2 Cross-cutting concerns

| Concern | Where handled | Problem |
|---|---|---|
| Authentication state | `Global.h` — `LoginUser` global | Hidden dependency; not thread-safe |
| Permissions | `clsScreen::CheckPermissions(...)` at screen level | Not enforced below the UI |
| Timestamps | `clsDate::GetSystemDateTimeString()` at call site | Inconsistent formats across files |
| Error display | `cout << "Error..."` inline everywhere | No centralized error handling |

---

## 4. Class Responsibilities

### `clsPerson` (`Models/clsPerson.h`)
Abstract base for person-like entities. Holds first name, last name, email, and phone. Has no persistence — purely a data holder. Inherited by `clsUser` and `clsBankClient`.

### `clsUser` (`Models/Bank/User/clsUser.h`)
One of the heaviest classes in the project. Combines **six** distinct responsibilities:
1. Person identity data (inherited from `clsPerson`)
2. Authentication: username, encrypted password
3. Permission bitmask
4. User-record CRUD: serialise/deserialise `MyUsers.txt`
5. Login-register persistence: append to `LoginRegister.txt`
6. Mailbox management: create, read, append, clear mail files

### `clsBankClient` (`Models/Bank/Client/clsBankClient.h`)
Combines **five** responsibilities:
1. Account identity data (inherited from `clsPerson`)
2. Account data: account number, PIN, balance
3. Account-record CRUD: serialise/deserialise `MyClients.txt`
4. Transaction logic: Deposit, Withdraw, Transfer
5. Transfer-log persistence: append to `TransferRegister.txt`

### `clsCurrency` (`Models/Bank/Currency/clsCurrency.h`)
Combines **three** responsibilities:
1. Currency data: country, code, name, USD rate
2. Currency-record CRUD: serialise/deserialise `Currencies.txt`
3. Conversion logic: convert through USD base

### `clsScreen` (`Screens/clsScreen.h`)
Base class for all console screens. Draws the header banner, prints the current date/time and logged-in username, and gates access via `CheckPermissions(enPermissions)`. Derived classes call `_drawScreenHeader(title)` in their show methods.

### `clsUtil` (`Utility/clsUtil.h`)
Provides `EncryptText(text, key)` (Caesar-shift), `DecryptText(text, key)`, number-to-words, random helpers, and generic swap. The Caesar-shift key used for passwords is `3`.

### `Global.h` (`Utility/Global.h`)
Defines a single global variable:
```cpp
clsUser LoginUser = clsUser::Find("", "");
```
This is included by nearly every file. Any code that reads or modifies `LoginUser` creates a hidden global dependency.

---

## 5. Feature-Specific Code Flows

### 5.1 Login register (audit viewer)

```
clsLoginRegisterScreen::ShowLoginRegisterScreen()
  └─ clsUser::GetLoginRegisterList()
       └─ read all lines from LoginRegister.txt
            └─ _convertLinetoLoginRegisterObject()
  └─ display table
```

### 5.2 Total balance

```
clsTotalBalanceScreen::ShowTotalBalancesScreen()
  └─ clsBankClient::GetClientsList()
       └─ _fromFileToVectorOfClient()   reads all of MyClients.txt
  └─ sum all _accountBalance fields
  └─ display
```

### 5.3 Currency conversion

```
clsCurrencyCalculator::ShowCurrencyCalculatorScreen()
  ├─ read source currency code → clsCurrency::FindByCode(code)
  ├─ read target currency code → clsCurrency::FindByCode(code)
  ├─ read amount
  ├─ usd_amount = amount / source.RateToUSD
  └─ result = usd_amount * target.RateToUSD
```

---

## 6. Architectural Observations

### Strengths
- Clear feature-oriented menu structure; each feature has its own screen class.
- Consistent `#//#` delimiter-based serialization across all files.
- Basic audit trails exist for logins and transfers.
- Permission bitmask model is compact and extensible.
- CRUD exists for both clients and users.

### Weaknesses

| Issue | Impact |
|---|---|
| Domain + persistence in same class | Cannot test domain logic without touching files |
| All-static methods | Cannot inject dependencies or mock storage |
| No transactional safety for multi-file writes | Data corruption on crash during transfer |
| Global mutable `LoginUser` | Hidden state; order-of-initialization hazard |
| Reversible password storage | Passwords recoverable from `MyUsers.txt` |
| `__declspec(property)` MSVC extension | Breaks all non-MSVC compilers |
| `using namespace std;` in headers | Namespace pollution for every includer |
| `system("cls")` / `system("pause>0")` | Windows-only; security risk |
| `float` for money | Precision loss on large balances |
| No unit tests | No safety net for any refactor |

---

## 7. Proposed Target Architecture

The goal of the target architecture is to separate the five concerns that are currently mixed together:

```
┌──────────────────────────────────────────────────────────────────────┐
│  UI Layer  (src/ui/)                                                 │
│  Console rendering only. No business logic, no file access.          │
│  ConsoleApp, LoginView, MainMenuView, ClientViews, etc.              │
├──────────────────────────────────────────────────────────────────────┤
│  Application / Service Layer  (src/services/)                        │
│  Orchestrates use cases. Calls domain and repository.                │
│  AuthService, ClientService, TransferService, MailService,           │
│  CurrencyService                                                     │
├──────────────────────────────────────────────────────────────────────┤
│  Domain Layer  (src/domain/)                                         │
│  Pure C++ value types and business rules. No I/O.                    │
│  User, Client, Currency, TransferLog, MailMessage                    │
├──────────────────────────────────────────────────────────────────────┤
│  Persistence Layer  (src/persistence/)                               │
│  Repository interfaces + implementations (file or SQLite).           │
│  IUserRepo, IClientRepo, ICurrencyRepo, IAuditRepo, IMailRepo        │
│  FileUserRepo, SqliteClientRepo, etc.                                │
├──────────────────────────────────────────────────────────────────────┤
│  Infrastructure / Utility  (src/util/)                               │
│  DateTime, StringUtils, Validation, PasswordHash, Console           │
└──────────────────────────────────────────────────────────────────────┘
```

### Proposed directory layout

```text
src/
  app/
    main.cpp
    Application.h / Application.cpp      # startup, wiring, session context
    SessionContext.h                     # replaces Global.h LoginUser
  domain/
    User.h
    Client.h
    Currency.h
    TransferLog.h
    MailMessage.h
  services/
    AuthService.h / AuthService.cpp
    ClientService.h / ClientService.cpp
    TransferService.h / TransferService.cpp
    CurrencyService.h / CurrencyService.cpp
    MailService.h / MailService.cpp
  persistence/
    interfaces/
      IUserRepository.h
      IClientRepository.h
      ICurrencyRepository.h
      IAuditRepository.h
      IMailRepository.h
    file/
      FileUserRepository.h / .cpp        # backward-compat flat-file impl
      FileClientRepository.h / .cpp
    sqlite/
      SqliteUserRepository.h / .cpp      # future SQLite impl
      SqliteClientRepository.h / .cpp
  ui/
    ConsoleHelper.h / .cpp               # portable cls, pause replacements
    screens/
      LoginScreen.h / .cpp
      MainMenu.h / .cpp
      ClientScreens.h / .cpp
      TransactionScreens.h / .cpp
      UserScreens.h / .cpp
      CommunicationScreens.h / .cpp
      CurrencyScreens.h / .cpp
  util/
    DateTime.h / .cpp
    StringUtils.h / .cpp
    Validation.h / .cpp
    PasswordHash.h / .cpp               # replaces clsUtil::EncryptText

tests/
  domain/
  services/
  persistence/
```

### Key design decisions for the target

| Decision | Rationale |
|---|---|
| Replace `Global.h LoginUser` with `SessionContext` passed by reference | Eliminates hidden global state; enables unit testing |
| Repository interfaces (`IClientRepository`) | Enables swapping file → SQLite without touching services |
| `PasswordHash` using bcrypt/SHA-256 | Eliminates reversible Caesar-shift credential storage |
| `int64_t` cents or `Decimal` type for balances | Eliminates `float` rounding errors on financial data |
| `std::chrono::system_clock::time_point` for timestamps | Replaces free-form string date storage |
| Per-feature `.cpp` files | Reduces build times and coupling |
| CMake as primary build system | Cross-platform; does not remove existing `.vcxproj` |

---

*Next: [Database Design →](DATABASE_DESIGN.md)*
