# Build Modernization Plan — C++ Banking System

> **Part of the modernization documentation set.**
> See [MODERNIZATION_REVIEW.md](../MODERNIZATION_REVIEW.md) for the index.

---

## Table of Contents

1. [Current Build Setup](#1-current-build-setup)
2. [Motivation for CMake](#2-motivation-for-cmake)
3. [CMake Migration Plan](#3-cmake-migration-plan)
4. [Starter CMakeLists.txt](#4-starter-cmakeliststxt)
5. [Coexistence with Visual Studio](#5-coexistence-with-visual-studio)
6. [Cross-Platform Portability Checklist](#6-cross-platform-portability-checklist)
7. [Recommended Toolchain Targets](#7-recommended-toolchain-targets)
8. [CI Integration](#8-ci-integration)

---

## 1. Current Build Setup

| Item | Value |
|---|---|
| Build system | Visual Studio 2022 (`BankSystem.sln` / `BankSystem.vcxproj`) |
| Compiler | MSVC (`cl.exe`) |
| C++ standard | Not explicitly set in project; defaults to MSVC default (C++14) |
| Platform | Windows x64 only |
| Dependencies | None (standard library + `windows.h`) |
| Output | `BankSystem.exe` |
| Tests | None |

The project currently builds in a single step: open `BankSystem.sln` in Visual Studio and press **Build**. There is no `CMakeLists.txt`, no package manager configuration, and no CI workflow.

---

## 2. Motivation for CMake

| Goal | Why CMake helps |
|---|---|
| Cross-platform builds (Linux, macOS) | CMake generates native build files for any platform |
| Use GCC / Clang instead of MSVC only | Catches non-portable code that MSVC silently accepts |
| Dependency management | CMake `FetchContent` can pull in SQLiteCpp, Catch2, etc. |
| CI/CD pipeline | GitHub Actions / GitLab CI can use CMake directly |
| IDEs beyond Visual Studio | CLion, VS Code, Xcode all consume `CMakeLists.txt` natively |
| Unit test integration | CMake `CTest` and `Catch2` / `GoogleTest` plug in naturally |
| Static analysis | clang-tidy and cppcheck integrate with CMake compile-command export |

Adding `CMakeLists.txt` **does not remove or break** the existing `.sln` / `.vcxproj` files. Both can coexist.

---

## 3. CMake Migration Plan

The migration is organized into three phases that match the overall refactor roadmap.

### Phase A — Build-system parity (no code changes required)

**Goal:** CMake produces the same executable as the current `.sln`.

Steps:
1. Add `CMakeLists.txt` at the repository root (see [Section 4](#4-starter-cmakeliststxt)).
2. List all current `.cpp` and `.h` files.
3. Set `CMAKE_CXX_STANDARD 17`.
4. Add `/W4` (MSVC) or `-Wall -Wextra` (GCC/Clang) warning flags.
5. Verify build on Windows with MSVC via CMake (`cmake -G "Visual Studio 17 2022" ..`).
6. Verify build on Linux with GCC or Clang (optional, as this requires addressing MSVC-specific code first).

**Blockers for Linux/GCC build:**
- `__declspec(property)` must be removed or wrapped.
- `system("cls")`, `system("pause>0")` must be conditioned on `_WIN32`.
- Any Windows-only headers (e.g. `<windows.h>`) must be guarded.

---

### Phase B — Dependency management

**Goal:** Introduce SQLite and testing framework via CMake.

Steps:
1. Use `FetchContent` to pull in:
   - `sqlite3` amalgamation (or `SQLiteCpp`)
   - `Catch2` for unit tests
2. Add a `tests/` subdirectory with `CMakeLists.txt`.
3. Wire tests with `enable_testing()` and `add_test()`.

---

### Phase C — Full cross-platform build

**Goal:** Green build on Linux and macOS without modifications at configure time.

Steps:
1. All MSVC-specific constructs removed (see [Refactor Roadmap](REFACTOR_ROADMAP.md) Phase 1).
2. Replace `system("cls")` with a portable `clearScreen()` function.
3. Replace string date formatting with `std::chrono` / `std::format` (C++20).
4. Validate with GitHub Actions matrix: `windows-latest`, `ubuntu-latest`, `macos-latest`.

---

## 4. Starter CMakeLists.txt

The file below is **conservative**: it lists the current source files, sets C++17, and adds warning flags without touching the existing `.vcxproj`. Copy it to the repository root.

```cmake
cmake_minimum_required(VERSION 3.20)

project(BankSystem
    VERSION 1.0.0
    DESCRIPTION "Console banking system"
    LANGUAGES CXX
)

# ── C++ standard ──────────────────────────────────────────────────────────────
set(CMAKE_CXX_STANDARD          17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS        OFF)

# ── Source files ──────────────────────────────────────────────────────────────
# All current .cpp files in the project.  As the refactor progresses and .cpp
# files are split from headers, add them here.
set(BANKSYSTEM_SOURCES
    BankSystem.cpp
)

# ── Header include paths ───────────────────────────────────────────────────────
# The project uses #include"clsLoginScreen.h" with relative paths from within
# the source tree.  The paths below mirror the structure expected by headers.
set(BANKSYSTEM_INCLUDE_DIRS
    ${CMAKE_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/Models
    ${CMAKE_SOURCE_DIR}/Models/Bank/Client
    ${CMAKE_SOURCE_DIR}/Models/Bank/User
    ${CMAKE_SOURCE_DIR}/Models/Bank/Currency
    ${CMAKE_SOURCE_DIR}/Screens
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Login
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Login\ Register
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Client
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Transactions
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Transactions/Transcations\ Menu
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/User
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Communications
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Communications/Communication\ Menu
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Currency
    ${CMAKE_SOURCE_DIR}/Screens/Main\ Menu/Currency/Currency\ Menu
    ${CMAKE_SOURCE_DIR}/Utility
)

# ── Executable ────────────────────────────────────────────────────────────────
add_executable(BankSystem ${BANKSYSTEM_SOURCES})

target_include_directories(BankSystem PRIVATE ${BANKSYSTEM_INCLUDE_DIRS})

# ── Compiler warnings ─────────────────────────────────────────────────────────
if (MSVC)
    target_compile_options(BankSystem PRIVATE
        /W4
        /wd4100   # unreferenced formal parameter (common in event handlers)
        /wd4996   # 'deprecated' POSIX names used in legacy code
    )
else()
    target_compile_options(BankSystem PRIVATE
        -Wall
        -Wextra
        -Wpedantic
        -Wno-unused-parameter
    )
endif()

# ── Windows-specific: suppress security warnings from MSVC ────────────────────
if (MSVC)
    target_compile_definitions(BankSystem PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

# ── Install ───────────────────────────────────────────────────────────────────
install(TARGETS BankSystem DESTINATION bin)

# ── Tests (Phase B — uncomment when tests exist) ──────────────────────────────
# enable_testing()
# add_subdirectory(tests)
```

### How to use (command line)

**Windows with MSVC (Visual Studio generator):**
```
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

**Windows with MSVC (Ninja, faster incremental):**
```
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

**Linux / macOS with GCC or Clang:**
```
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

> **Note:** The Linux/macOS build will fail until Phase 1 of the refactor (removing `__declspec(property)` and Windows-specific calls) is complete. The `CMakeLists.txt` is added now so the build infrastructure is ready when the code is ready.

---

## 5. Coexistence with Visual Studio

The following files can exist side-by-side without conflict:

| File | Used by |
|---|---|
| `BankSystem.sln` | Visual Studio IDE (open solution) |
| `BankSystem.vcxproj` | MSBuild / Visual Studio |
| `CMakeLists.txt` | CMake, CLion, VS Code CMake Tools, CI |

Visual Studio 2019+ natively supports opening a folder with a `CMakeLists.txt` via **File → Open → Folder**. This gives a second build path without requiring the `.sln` file. Both approaches can coexist in the same repository.

Recommended `.gitignore` additions (so generated build artifacts are not committed):
```
# CMake build output
build/
cmake-build-*/
CMakeCache.txt
CMakeFiles/
*.cmake
!CMakeLists.txt

# Visual Studio generated output (if not already present)
*.user
x64/
Debug/
Release/
```

---

## 6. Cross-Platform Portability Checklist

The table below lists every known MSVC/Windows-specific construct in the codebase and the portable replacement:

| Code pattern | File(s) | Portable replacement |
|---|---|---|
| `__declspec(property(get=..., put=...))` | `clsBankClient.h`, `clsUser.h`, others | Standard getter/setter methods or C++17 `[[nodiscard]]` accessors |
| `system("cls")` | Multiple screen files | `#ifdef _WIN32` / ANSI escape sequence `"\033[2J\033[H"` or `ncurses` |
| `system("pause>0")` | Multiple screen files | `std::cin.get()` or a portable `waitForKeypress()` helper |
| `#include <windows.h>` | (implicit through MSVC) | Guard with `#ifdef _WIN32`; use `<cstdlib>` for portable equivalents |
| `using namespace std;` in headers | All model/utility headers | Qualify with `std::` everywhere; remove `using` from headers |
| Non-standard C++ pragmas (`#pragma once`) | All headers | `#pragma once` is widely supported; acceptable, or replace with include guards |
| `float` for money | `clsBankClient.h` | `int64_t` cents, or wrap in a `Money` value type |
| Date as free-form `string` | All audit files | `std::chrono::system_clock::time_point`, serialized as ISO 8601 |

---

## 7. Recommended Toolchain Targets

Once `CMakeLists.txt` is in place, the following toolchain matrix is recommended for CI:

| Platform | Compiler | C++ Standard | Notes |
|---|---|---|---|
| Windows | MSVC 19.x | C++17 | Primary; must always pass |
| Windows | Clang-cl | C++17 | Catches MSVC-specific code earlier |
| Linux (Ubuntu 22.04) | GCC 11 | C++17 | Portability baseline |
| Linux (Ubuntu 22.04) | Clang 14 | C++17 | Strict mode; enables sanitizers |
| macOS 13 | Apple Clang 14 | C++17 | Optional; good for developer machines |

**Sanitizers to enable on Linux/Clang builds:**
```cmake
# In CMakeLists.txt, for Debug builds with Clang/GCC:
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(BankSystem PRIVATE -fsanitize=address,undefined)
    target_link_options   (BankSystem PRIVATE -fsanitize=address,undefined)
endif()
```

---

## 8. CI Integration

A minimal GitHub Actions workflow that builds with CMake on Windows and Ubuntu:

```yaml
# .github/workflows/cmake-build.yml
name: CMake Build

on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        os: [windows-latest, ubuntu-latest]
        build_type: [Release]

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure CMake
        run: cmake -S . -B build -DCMAKE_BUILD_TYPE=${{ matrix.build_type }}

      - name: Build
        run: cmake --build build --config ${{ matrix.build_type }}

      - name: Test
        run: ctest --test-dir build --build-config ${{ matrix.build_type }} --output-on-failure
```

> **Current state:** The Ubuntu job will fail until Phase 1 of the refactor is done. Add the workflow file anyway so failures are visible and tracked.

---

*Next: [Refactor Roadmap →](REFACTOR_ROADMAP.md)*
