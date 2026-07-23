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
# Currently all logic lives in header files; BankSystem.cpp is the only .cpp.
# As the refactor progresses and .cpp files are extracted, add them here.
set(BANKSYSTEM_SOURCES
    BankSystem.cpp
)

# ── Header include paths ───────────────────────────────────────────────────────
# Headers use relative paths like #include"clsLoginScreen.h".
# Add every directory that contains an included header.
set(BANKSYSTEM_INCLUDE_DIRS
    ${CMAKE_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/Models
    ${CMAKE_SOURCE_DIR}/Models/Bank/Client
    ${CMAKE_SOURCE_DIR}/Models/Bank/User
    ${CMAKE_SOURCE_DIR}/Models/Bank/Currency
    ${CMAKE_SOURCE_DIR}/Screens
    ${CMAKE_SOURCE_DIR}/Utility
)

# Screens sub-directories (spaces in folder names require explicit listing)
list(APPEND BANKSYSTEM_INCLUDE_DIRS
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Login"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Login Register"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Client"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Transactions"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Transactions/Transcations Menu"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/User"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Communications"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Communications/Communication Menu"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Currency"
    "${CMAKE_SOURCE_DIR}/Screens/Main Menu/Currency/Currency Menu"
)

# ── Executable ────────────────────────────────────────────────────────────────
add_executable(BankSystem ${BANKSYSTEM_SOURCES})

target_include_directories(BankSystem PRIVATE ${BANKSYSTEM_INCLUDE_DIRS})

# ── Compiler warnings ─────────────────────────────────────────────────────────
if(MSVC)
    target_compile_options(BankSystem PRIVATE
        /W4
        /wd4100   # unreferenced formal parameter
        /wd4996   # deprecated POSIX names
    )
    # Suppress MSVC security warnings from legacy CRT functions
    target_compile_definitions(BankSystem PRIVATE _CRT_SECURE_NO_WARNINGS)
else()
    target_compile_options(BankSystem PRIVATE
        -Wall
        -Wextra
        -Wpedantic
        -Wno-unused-parameter
    )
    # NOTE: GCC/Clang build will fail until Phase 1 refactor is complete:
    #   - __declspec(property) must be replaced with standard accessors
    #   - system("cls") / system("pause>0") must be guarded with #ifdef _WIN32
    #   - using namespace std; must be removed from headers
endif()

# ── Install ───────────────────────────────────────────────────────────────────
install(TARGETS BankSystem DESTINATION bin)

# ── Tests (uncomment when tests/ directory and Catch2 are added) ──────────────
# See docs/BUILD_MODERNIZATION.md Section 4 for instructions.
#
# include(FetchContent)
# FetchContent_Declare(
#     Catch2
#     GIT_REPOSITORY https://github.com/catchorg/Catch2.git
#     GIT_TAG        v3.5.2
# )
# FetchContent_MakeAvailable(Catch2)
#
# enable_testing()
# add_subdirectory(tests)
