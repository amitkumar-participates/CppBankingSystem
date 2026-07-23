# C++ Banking System Review and Modernization Notes

## Executive Summary
This repository is a console-based banking application built in C++ around file-based persistence. The application starts at `BankSystem.cpp`, authenticates users through `clsLoginScreen`, and then routes users through menu-driven screens for client management, transactions, user administration, communication, and currency exchange.

From a modernization perspective, the codebase demonstrates strong functional decomposition by feature, but it is tightly coupled to:
- Visual Studio / MSVC-specific extensions such as `__declspec(property)`
- global mutable state via `LoginUser`
- direct file I/O inside domain models
- synchronous console UI and business logic mixed together
- plain-text / reversible credential handling

This document summarizes the **business flow**, **code flow**, **database/file schema**, and **C++ modernization recommendations**.

---

## 1. Repository Overview

### Primary runtime
- **Entry point:** `BankSystem.cpp`
- **Application type:** Console application
- **Build system:** Visual Studio solution/project (`BankSystem.sln`, `BankSystem.vcxproj`)
- **Primary language:** C++
- **Persistence model:** Flat files (`.txt`) instead of a relational database

### Major functional areas
- Secure-ish login and user access control
- Client onboarding and maintenance
- Deposit, withdrawal, total balance, transfer, transfer logs
- User management and permissions
- Internal mailbox system
- Currency listing, lookup, update, and conversion

---

## 2. Business Flow

### 2.1 User authentication flow
1. Application starts in `main()`.
2. `clsLoginScreen::ShowLoginScreen()` displays the login UI.
3. User enters username and password.
4. `clsUser::Find(username, password)` loads users from `MyUsers.txt` and validates the credentials.
5. If login succeeds:
   - global `LoginUser` is populated
   - login event is recorded in `LoginRegister.txt`
   - user is routed to `clsMainMenu::MainMenuScreen()`
6. If login fails 3 times, the system is locked for that run.

### 2.2 Main banking flow
After login, the user chooses one of these business functions:
- Show clients
- Add client
- Delete client
- Update client
- Find client
- Transactions
- Manage users
- Login register
- Communication
- Currency exchange
- Logout

Permission checks are enforced at screen level by `clsScreen::CheckPermissions(...)` using bitmask permissions from `clsUser`.

### 2.3 Client lifecycle flow
#### Add client
1. User selects **Add New Client**.
2. System validates account number uniqueness against `MyClients.txt`.
3. User enters personal and account data.
4. `clsBankClient::Save()` appends the record to `MyClients.txt`.

#### Update client
1. User selects **Update Client**.
2. System searches by account number.
3. Existing client is loaded from file.
4. User edits fields.
5. Entire client file is rewritten with updated data.

#### Delete client
1. User selects **Delete Client**.
2. Client is loaded by account number.
3. Record is marked for deletion in memory.
4. `MyClients.txt` is rewritten excluding deleted rows.

### 2.4 Transaction flow
#### Deposit
1. User chooses an account.
2. Deposit amount is validated.
3. In-memory balance is updated.
4. Full client file is rewritten.

#### Withdraw
1. User chooses an account.
2. Withdrawal amount is validated against available balance.
3. Balance is reduced.
4. Full client file is rewritten.

#### Transfer
1. Source account is selected.
2. Destination account is selected.
3. Amount is validated against source balance.
4. Source withdraws funds.
5. Destination deposits funds.
6. Transfer event is appended to `TransferRegister.txt`.

### 2.5 User management flow
Administrators or privileged users can:
- list users
- add new users
- update existing users
- delete users
- search for users

When a new user is created:
1. User record is stored in `MyUsers.txt`
2. personal mailbox files are created in:
   - `ReceivedMailBox/`
   - `SendedMailBox/`

### 2.6 Communication flow
1. Logged-in user selects communication menu.
2. User can:
   - view received mail
   - view sent mail
   - send email to another user
   - clear sent mailbox
   - clear received mailbox
3. Sending email writes one record to sender sent-box file and one record to receiver inbox file.

### 2.7 Currency exchange flow
1. User enters the currency exchange module.
2. User can:
   - list currencies
   - search by code or country
   - update a currency rate
   - convert one currency to another through USD normalization
3. Currency data is read from and written back to `Currencies.txt`.

---

## 3. Code Flow

## 3.1 Entry and navigation
### Entry point
- `BankSystem.cpp`
  - calls `clsLoginScreen::ShowLoginScreen()`

### Core navigation chain
```text
main()
  -> clsLoginScreen::ShowLoginScreen()
      -> clsLoginScreen::_login()
          -> clsUser::Find(username, password)
          -> LoginUser.SaveInRegisterLogin()
          -> clsMainMenu::MainMenuScreen()
```

### Main menu routing
`Screens/Main Menu/clsMainMenu.h` dispatches to feature-specific screens:
- `clsShowClients`
- `clsAddNewClient`
- `clsDeleteClient`
- `clsUpdateClient`
- `clsFindClient`
- `clsTransactionsMenu`
- `clsManageUsersScreen`
- `clsLoginRegisterScreen`
- `clsCommunicationScreen`
- `clsCurrencyExchangeScreen`

---

## 3.2 Layering pattern actually used
Although the project is not formally layered, the repository behaves like this:

### A. UI / Screen layer
Located under `Screens/`
- Responsible for console rendering, input prompts, confirmation dialogs, and menu routing.
- Example classes:
  - `clsLoginScreen`
  - `clsMainMenu`
  - `clsTransactionsMenu`
  - `clsCommunicationScreen`
  - `clsCurrencyExchangeScreen`

### B. Domain / data model layer
Located under `Models/`
- Encapsulates domain entities and persistence operations.
- Example classes:
  - `clsPerson`
  - `clsBankClient`
  - `clsUser`
  - `clsCurrency`

### C. Utility layer
Located under `Utility/`
- Common helpers for string manipulation, input validation, date handling, and simple encryption.
- Example files:
  - `clsInputValidate.h`
  - `clsString.h`
  - `clsDate.h`
  - `clsUtil.h`
  - `Global.h`

---

## 3.3 Important class responsibilities

### `clsPerson`
Base entity for people.
Fields:
- first name
- last name
- email
- phone

Used as a parent for:
- `clsUser`
- `clsBankClient`

### `clsUser`
Represents system users and combines:
- identity/profile data
- authentication data
- permissions
- login-register persistence
- mailbox persistence
- user CRUD persistence

Key responsibilities:
- parse and serialize `MyUsers.txt`
- encrypt/decrypt stored password text using `clsUtil::EncryptText(..., 3)`
- append login history to `LoginRegister.txt`
- create and manage mailbox files
- permission model via bit flags

### `clsBankClient`
Represents bank customer account data.
Responsibilities:
- parse and serialize `MyClients.txt`
- deposit / withdraw / transfer
- compute total balances
- append transfer log rows to `TransferRegister.txt`

### `clsCurrency`
Represents currency rows from `Currencies.txt`.
Responsibilities:
- find currency by country or code
- update exchange rates
- convert through USD

### `clsScreen`
Common base class for UI screens.
Responsibilities:
- draw header
- show current user and date
- enforce permission checks

---

## 3.4 Feature-specific code flow

### Login flow
```text
clsLoginScreen::ShowLoginScreen()
  -> _drawScreenHeader()
  -> _login()
     -> read username/password
     -> clsUser::Find(username, password)
        -> _fromFileToVUsers()
        -> _convertLinetoUserObject()
     -> SaveInRegisterLogin()
     -> clsMainMenu::MainMenuScreen()
```

### Client add flow
```text
clsAddNewClient::ShowAddNewClientScreen()
  -> check permission
  -> validate unique account number
  -> clsBankClient::GetAddNewClientObject()
  -> collect input
  -> clsBankClient::Save()
     -> _add()
     -> append to MyClients.txt
```

### Transfer flow
```text
clsTransferScreen::ShowTransferScreen()
  -> find source client
  -> find destination client
  -> validate transfer amount
  -> clsBankClient::Transfer(amount, destination)
     -> Withdraw(amount)
     -> Save()
     -> Destination.Deposit(amount)
     -> Save()
     -> _saveTransferInRegister(...)
        -> append to TransferRegister.txt
```

### Send email flow
```text
clsSendEmailScreen::ShowSendEmailScreen()
  -> validate receiver user
  -> LoginUser.SendMail(title, body, receiver)
     -> sender _sendMail()
        -> append to SendedMailBox/<User>SendedBox.txt
     -> receiver ReceiveMail(...)
        -> append to ReceivedMailBox/<User>ReceivedBox.txt
```

### Currency conversion flow
```text
clsCurrencyCalculator::ShowCurrencyCalculatorScreen()
  -> read source currency code
  -> read target currency code
  -> read amount
  -> CurrnecyConvertToUSD()
  -> CurrnecyConvertToAnotherCurrency()
```

---

## 4. Repository Structure

```text
BankSystem.cpp                         # main entry point
BankSystem.sln                         # Visual Studio solution
BankSystem.vcxproj                     # Visual Studio project file

Models/
  clsPerson.h                          # shared base class for person-like entities
  Bank/
    Client/
      clsBankClient.h                  # bank client entity + file persistence + transfers
    User/
      clsUser.h                        # user entity + auth + permissions + mailbox + logs
    Currency/
      clsCurrency.h                    # currency entity + file persistence + conversion

Screens/
  clsScreen.h                          # base screen and permission guard
  Main Menu/
    clsMainMenu.h                      # top-level feature navigation
    Login/
      clsLoginScreen.h                 # login and lockout behavior
    Login Register/
      clsLoginRegisterScreen.h         # login history screen
    Client/
      clsAddNewClient.h
      clsDeleteClient.h
      clsFindClient.h
      clsShowClients.h
      clsUpdateClient.h
    Transactions/
      clsTransactionsMenu.h
      Transcations Menu/
        clsDepositScreen.h
        clsWithdrawScreen.h
        clsTransferScreen.h
        clsTransferLogList.h
        clsTotalBalanceScreen.h
    User/
      clsManageUsers.h
      clsAddNewUser.h
      clsDeleteUser.h
      clsFindUser.h
      clsUpdateUser.h
      clsUserList.h
    Communications/
      clsCommunicationScreen.h
      Communication Menu/
        clsSendEmailScreen.h
        clsReceivedMailBox.h
        clsSendedMailBoxScreen.h
        clsClearEmailBox.h
    Currency/
      clsCurrencyExchangeScreen.h
      Currency Menu/
        clsListCurrencies.h
        clsFindCurrencyScreen.h
        clsUpdateCurrencyRate.h
        clsCurrencyCalculator.h

Utility/
  Global.h                             # global logged-in user instance
  clsDate.h                            # date/time helper
  clsInputValidate.h                   # validated console input helper
  clsString.h                          # string utility helper
  clsUtil.h                            # random, swap, simple encryption, misc helpers

Data files/
  MyUsers.txt                          # user records
  MyClients.txt                        # client records
  LoginRegister.txt                    # login audit log
  TransferRegister.txt                 # transfer audit log
  Currencies.txt                       # exchange rates
  ReceivedMailBox/                     # per-user inbox files
  SendedMailBox/                       # per-user sent-mail files
```

---

## 5. Database Design (Actual Storage Model)
This repository does **not** use a database engine such as SQLite, MySQL, or PostgreSQL.
Instead, it uses **flat-file persistence** with custom `#//#` delimiters.

You can think of each file as a table.

## 5.1 Logical schema

### Table: `MyUsers.txt`
Represents application users.

| Column | Type | Description |
|---|---|---|
| first_name | string | User first name |
| last_name | string | User last name |
| email | string | Email address |
| phone | string | Phone number |
| username | string | Login identifier |
| encrypted_password | string | Caesar-shift style encrypted password |
| permissions | int | Bitmask permission set |

#### Example row format
```text
FirstName#//#LastName#//#Email#//#Phone#//#Username#//#EncryptedPassword#//#Permissions
```

---

### Table: `MyClients.txt`
Represents bank clients/accounts.

| Column | Type | Description |
|---|---|---|
| first_name | string | Client first name |
| last_name | string | Client last name |
| email | string | Client email |
| phone | string | Client phone |
| account_number | string | Unique account number |
| pin_code | string | Client PIN |
| account_balance | float | Current balance |

#### Example row format
```text
FirstName#//#LastName#//#Email#//#Phone#//#AccountNumber#//#PinCode#//#Balance
```

---

### Table: `LoginRegister.txt`
Represents login audit events.

| Column | Type | Description |
|---|---|---|
| date | string | Login timestamp |
| username | string | User who logged in |
| encrypted_password | string | Encrypted password copy as written by app |
| permissions | int | Permission snapshot |

#### Example row format
```text
Date#//#Username#//#EncryptedPassword#//#Permissions
```

---

### Table: `TransferRegister.txt`
Represents transfer audit records.

| Column | Type | Description |
|---|---|---|
| date | string | Transfer timestamp |
| transferor_account | string | Source account |
| recipient_account | string | Destination account |
| transfer_amount | float | Amount transferred |
| transferor_balance | float | Source balance after operation |
| recipient_balance | float | Destination balance after operation |
| login_username | string | Logged-in user executing the transfer |

#### Example row format
```text
Date#//#Transferor#//#Recipient#//#Amount#//#TransferorBalance#//#RecipientBalance#//#LoginUser
```

---

### Table: `Currencies.txt`
Represents currency master data.

| Column | Type | Description |
|---|---|---|
| country_name | string | Country |
| currency_code | string | ISO-like code |
| currency_name | string | Currency display name |
| rate_to_usd_base | double | Exchange rate used by app |

#### Example row format
```text
CountryName#//#CurrencyCode#//#CurrencyName#//#CurrencyValue
```

---

### Table family: `ReceivedMailBox/<User>ReceivedBox.txt`
Represents received emails for a user.

| Column | Type | Description |
|---|---|---|
| date | string | Mail timestamp |
| dealer | string | Sender username |
| title | string | Subject |
| body | string | Message body |

#### Example row format
```text
Date#//#Sender#//#Title#//#Body
```

---

### Table family: `SendedMailBox/<User>SendedBox.txt`
Represents sent emails for a user.

| Column | Type | Description |
|---|---|---|
| date | string | Mail timestamp |
| dealer | string | Receiver username |
| title | string | Subject |
| body | string | Message body |

#### Example row format
```text
Date#//#Receiver#//#Title#//#Body
```

---

## 5.2 Entity relationship view

```text
User
 ├── logs many LoginRegister entries
 ├── sends many SentMailbox messages
 └── receives many ReceivedMailbox messages

Client
 ├── participates as source in many TransferRegister entries
 └── participates as target in many TransferRegister entries

Currency
 └── used by currency conversion workflows
```

---

## 6. Architectural Observations

### Strengths
- Clear feature-oriented menu structure
- CRUD logic exists for clients and users
- Consistent delimiter-based serialization format
- Basic audit trails exist for logins and transfers
- Permissions are modeled as bit flags, which is compact and extensible

### Weaknesses
- domain logic, UI, storage, and validation are tightly coupled
- heavy use of `static` methods prevents dependency injection and testability
- no transactional safety for multi-file operations
- no repository/service abstraction
- no automated tests
- file format is fragile and delimiter-sensitive
- global state makes behavior hard to reason about
- reversible password storage is insecure
- code depends on MSVC extensions and Windows shell commands (`system("cls")`, `system("pause>0")`)

---

## 7. C++ Modernization Review

## 7.1 Current legacy / non-portable patterns

### 1. `using namespace std;` in headers
This pollutes the global namespace for every translation unit including the header.

### 2. `__declspec(property)` in headers
This is MSVC-specific and prevents portability to standard C++ compilers.

### 3. Header-only implementation of everything
Almost all business logic is implemented in `.h` files. This increases compile time and coupling.

### 4. Global mutable singleton-style state
`Global.h` defines:
```cpp
clsUser LoginUser = clsUser::Find("", "");
```
This creates hidden dependencies across the whole program.

### 5. Direct file access inside entity classes
`clsUser`, `clsBankClient`, and `clsCurrency` perform persistence directly, mixing domain and infrastructure.

### 6. Plain reversible encryption for passwords
`clsUtil::EncryptText` is a Caesar-shift-like transformation, not secure password hashing.

### 7. Recursive menu navigation / screen chaining
Some screens call back into other screens directly, which can complicate control flow and stack behavior.

### 8. `system("cls")` and `system("pause>0")`
Platform-specific and discouraged.

### 9. Weak type modeling
- money stored as `float`
- permissions stored as `int`
- dates stored as free-form strings

### 10. Missing const-correctness and references
Many methods should be `const`, and many parameters should be `const std::string&` or references.

---

## 7.2 Recommended modernization roadmap

### Phase 1: Safe refactor without behavior changes
1. Remove `using namespace std;` from headers.
2. Replace `__declspec(property)` with standard getters/setters.
3. Move implementations from headers into `.cpp` files.
4. Introduce `const` correctness.
5. Replace C-style includes / functions where appropriate.
6. Replace global `LoginUser` with an application/session context object.

### Phase 2: Separate concerns
1. Create dedicated layers:
   - `ui/`
   - `domain/`
   - `persistence/`
   - `services/`
2. Move all file operations into repository classes such as:
   - `UserRepository`
   - `ClientRepository`
   - `CurrencyRepository`
   - `AuditRepository`
3. Move workflows into services such as:
   - `AuthenticationService`
   - `TransferService`
   - `MailService`

### Phase 3: Improve data safety
1. Replace password encryption with salted password hashing.
2. Use structured records or SQLite instead of custom text files.
3. Use a decimal-safe money representation instead of `float`.
4. Add input sanitization for delimiter collisions.
5. Add atomic file replacement for writes.

### Phase 4: Improve portability and maintainability
1. Add CMake support.
2. Remove Visual Studio-specific constructs.
3. Replace console shell commands with portable abstractions.
4. Add unit tests for parsing, validation, authentication, and transfer logic.

---

## 8. Suggested Target Architecture

```text
src/
  app/
    Application.cpp
    SessionContext.h
  domain/
    User.h
    Client.h
    Currency.h
    TransferLog.h
    MailMessage.h
  services/
    AuthenticationService.h
    ClientService.h
    TransferService.h
    CurrencyService.h
    MailService.h
  persistence/
    UserRepository.h
    ClientRepository.h
    CurrencyRepository.h
    MailRepository.h
    AuditRepository.h
  ui/
    ConsoleApp.cpp
    screens/
  util/
    DateTime.h
    StringUtils.h
    Validation.h
```

This would make business rules independent of UI and storage.

---

## 9. Highest-Priority Risks to Address

### Security
- Passwords are recoverable, not hashed.
- Login register stores encrypted passwords too.
- Communication and account data are stored in plain text files.

### Data integrity
- Updates rewrite full files without locking.
- Transfer is not atomic across source, destination, and register log.
- Partial failure can produce inconsistent state.

### Portability
- Project is tightly coupled to Visual Studio and Windows console behavior.

### Maintainability
- Circular-style include risk from cross-header dependencies.
- Large headers increase rebuild scope.
- Static-only architecture is hard to test.

---

## 10. Recommended Next Modernization Tasks

### Immediate
1. Replace reversible password logic with one-way hashing.
2. Remove password from `LoginRegister.txt` entirely.
3. Introduce `SessionContext` instead of `Global.h`.
4. Move persistence logic out of `clsUser` and `clsBankClient`.
5. Replace `float` balances with integer cents or decimal-safe type.

### Near term
1. Add `CMakeLists.txt`.
2. Introduce `.cpp` implementation files.
3. Add unit tests for parsers and transfer behavior.
4. Normalize naming (`SentMailbox` instead of `SendedMailBox`, `Transactions` spelling, etc.).

### Longer term
1. Replace flat files with SQLite.
2. Introduce structured DTOs and repository interfaces.
3. Add transaction boundaries and rollback-safe writes.

---

## 11. Conclusion
This repository is a feature-rich educational banking console application with a clear user-facing flow and a consistent file-backed domain model. Its biggest opportunity is not feature completion, but **architectural modernization**: portability, testability, security, separation of concerns, and safer persistence.

If modernized in phases, the project could evolve from a tightly coupled MSVC console app into a cleaner, cross-platform, maintainable C++ application with reusable business services.
