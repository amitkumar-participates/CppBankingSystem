# C++ Banking System — Modernization Documentation

> **Navigation**
>
> | Document | Purpose |
> |---|---|
> | **This file** | Overview, executive summary, and documentation index |
> | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Business flow, code flow, layering, target architecture |
> | [docs/DATABASE_DESIGN.md](docs/DATABASE_DESIGN.md) | Current file schema, proposed SQLite schema, migration guide |
> | [docs/BUILD_MODERNIZATION.md](docs/BUILD_MODERNIZATION.md) | CMake build plan, cross-platform portability |
> | [docs/REFACTOR_ROADMAP.md](docs/REFACTOR_ROADMAP.md) | Phased C++17/C++20 refactor roadmap, security and data risks |

---

## Executive Summary

This repository is a console-based banking application built in C++ around file-based persistence. The application starts at `BankSystem.cpp`, authenticates users through `clsLoginScreen`, and then routes users through menu-driven screens for client management, transactions, user administration, communication, and currency exchange.

From a modernization perspective, the codebase demonstrates strong functional decomposition by feature, but it is tightly coupled to:
- Visual Studio / MSVC-specific extensions such as `__declspec(property)`
- global mutable state via `LoginUser`
- direct file I/O inside domain models
- synchronous console UI and business logic mixed together
- plain-text / reversible credential handling

This document is the **top-level index** for the full modernization documentation set. Detailed analysis is organized across the linked documents above.

---

## 1. Repository Overview

### Primary runtime

| Property | Value |
|---|---|
| Entry point | `BankSystem.cpp` |
| Application type | Console application |
| Build system | Visual Studio (`BankSystem.sln`, `BankSystem.vcxproj`) |
| Primary language | C++ |
| Persistence model | Flat files (`.txt`) with `#//#` delimiter |
| C++ standard in use | C++14 / MSVC extensions |

### Major functional areas
- Secure-ish login and user access control (bitmask permissions)
- Client onboarding and account maintenance
- Deposit, withdrawal, total balance, fund transfer, transfer logs
- User management and permissions administration
- Internal per-user mailbox system
- Currency listing, lookup, rate update, and conversion

### Repository layout

```text
BankSystem.cpp                         # main entry point
BankSystem.sln / BankSystem.vcxproj    # Visual Studio build files

Models/
  clsPerson.h                          # shared base: first/last name, email, phone
  Bank/
    Client/  clsBankClient.h           # account entity + file I/O + transactions
    User/    clsUser.h                 # user entity + auth + permissions + mailbox
    Currency/clsCurrency.h             # currency entity + rate + conversion

Screens/
  clsScreen.h                          # base screen: header, permission guard
  Main Menu/
    clsMainMenu.h                      # top-level menu dispatcher
    Login/           clsLoginScreen.h
    Login Register/  clsLoginRegisterScreen.h
    Client/          clsAddNewClient, clsDeleteClient, clsFindClient,
                     clsShowClients, clsUpdateClient
    Transactions/    clsTransactionsMenu + Deposit/Withdraw/Transfer/Log/TotalBalance
    User/            clsManageUsers + Add/Delete/Find/Update/List
    Communications/  clsCommunicationScreen + Send/Received/Sended/Clear
    Currency/        clsCurrencyExchangeScreen + List/Find/Update/Calculator

Utility/
  Global.h           # global LoginUser singleton
  clsDate.h          # date/time helper
  clsInputValidate.h # validated console input
  clsString.h        # string utilities
  clsUtil.h          # encryption, random, misc helpers

Data files (at repo root):
  MyUsers.txt          TransferRegister.txt
  MyClients.txt        Currencies.txt
  LoginRegister.txt    ReceivedMailBox/  SendedMailBox/
```

---

## 2. High-Level Business Flow

The application follows a simple linear flow:

1. `main()` → `clsLoginScreen::ShowLoginScreen()` → authenticate via `MyUsers.txt`
2. On success: record login in `LoginRegister.txt`, populate global `LoginUser`, route to `clsMainMenu`
3. Main menu dispatches to one of ten feature areas:
   - Client CRUD (Show / Add / Delete / Update / Find)
   - Transactions (Deposit / Withdraw / Transfer / Transfer Log / Total Balance)
   - User management (List / Add / Delete / Update / Find)
   - Login register viewer
   - Communication (Send / Inbox / Sent / Clear)
   - Currency exchange (List / Find / Update Rate / Convert)
   - Logout

Permission checks are enforced per screen via `clsScreen::CheckPermissions(...)` and the bitmask stored in `clsUser`.

For full detail — including per-feature code flow, class responsibilities, and layering analysis — see **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

---

## 3. Current File-Based Storage

Six logical "tables" are stored as delimited text files using `#//#` as the field separator:

| File | Content | Key |
|---|---|---|
| `MyUsers.txt` | System users, credentials, permissions | `username` |
| `MyClients.txt` | Bank clients and account balances | `account_number` |
| `LoginRegister.txt` | Login audit trail | (append-only) |
| `TransferRegister.txt` | Transfer audit trail | (append-only) |
| `Currencies.txt` | Exchange rates | `currency_code` |
| `ReceivedMailBox/<U>ReceivedBox.txt` | Per-user inbox | (append-only) |
| `SendedMailBox/<U>SendedBox.txt` | Per-user sent mail | (append-only) |

For exact column definitions, real sample data, entity relationships, and the proposed SQLite replacement schema, see **[docs/DATABASE_DESIGN.md](docs/DATABASE_DESIGN.md)**.

---

## 4. Top Modernization Issues

| # | Issue | Severity |
|---|---|---|
| 1 | Reversible Caesar-shift password storage | 🔴 Critical |
| 2 | No transactional safety — transfer touches 4 files non-atomically | 🔴 Critical |
| 3 | `Global.h` mutable singleton `LoginUser` | 🟠 High |
| 4 | `__declspec(property)` — MSVC-only, non-portable | 🟠 High |
| 5 | `using namespace std;` in every header | 🟡 Medium |
| 6 | Domain persistence mixed directly into entity classes | 🟡 Medium |
| 7 | `float` for monetary amounts (precision loss) | 🟡 Medium |
| 8 | `system("cls")` / `system("pause>0")` — Windows-only | 🟡 Medium |
| 9 | Header-only implementation (slow builds, high coupling) | 🟡 Medium |
| 10 | No automated tests | 🟡 Medium |
| 11 | No CMake / cross-platform build support | 🟢 Low-Medium |

---

## 5. Documentation Index

| Document | Contents |
|---|---|
| **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** | Full business flow, per-feature code flow, class responsibilities, layering analysis, and proposed target architecture |
| **[docs/DATABASE_DESIGN.md](docs/DATABASE_DESIGN.md)** | Current file schema, proposed SQLite schema with DDL, indexes, migration notes, and security improvements |
| **[docs/BUILD_MODERNIZATION.md](docs/BUILD_MODERNIZATION.md)** | CMake build plan, directory layout, cross-platform portability, and starter `CMakeLists.txt` |
| **[docs/REFACTOR_ROADMAP.md](docs/REFACTOR_ROADMAP.md)** | Four-phase C++17/C++20 refactor roadmap with per-phase goals, risks, and acceptance criteria |

---

## 6. Conclusion

This repository is a feature-rich educational banking console application with a clear user-facing flow and a consistent file-backed domain model. Its biggest opportunity is not feature completion but **architectural modernization**: portability, testability, security, separation of concerns, and safer persistence.

Modernized in phases (as detailed in the linked documents), the project can evolve from a tightly coupled MSVC console app into a clean, cross-platform, testable C++ application with reusable business services.
