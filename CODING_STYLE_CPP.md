# C++ NAMING CONVENTIONS — CODING_STYLE_CPP

How things are **named** in every C++ project in this workspace — present
and future: variables, functions, classes, enums, macros, files. Formatting,
architecture and build rules are outside this document.

Macro prefix in examples: `VX_` (from project basalt). A new project picks
its own prefix (2–4 letters + `_`, unique, never changed) and one namespace
named after the project.

---

## 1. Identifier categories

Case carries meaning — `Socket socket;` must be readable without an IDE:
PascalCase = a type, snake_case = a function or a value. Consistency beats
taste: never mix styles inside one module.

**The namespace is mandatory.** Every public type, function and variable
lives inside the project namespace — nothing public ever sits at global
scope, where WinAPI's own identifiers (`socket`, `connect`, `send`) are
waiting to collide. The only exemptions are `extern "C"` entry points and
`main()`.

**Why macros carry the prefix.** The namespace protects every type, function
and variable — that is its whole job. **Macros are the one thing it cannot
protect**: the preprocessor runs before namespaces exist, so a `#define` is
global no matter where it sits. Therefore the `VX_` prefix belongs to macros
only; classes, functions and variables never carry it. Prefer `constexpr`
constants and inline functions over macros; macros survive only as include
guards and conditional-compilation flags.

| Category | Convention | Example |
|---|---|---|
| Namespace | lowercase snake_case, = project name | `namespace basalt`, `namespace api_resolve` |
| Classes / structs / enums | PascalCase, no prefix | `HttpClient`, `HttpResponse` |
| Methods & free functions | snake_case — both alike | `connect()`, `parse_url()` |
| `enum class` values | SCREAMING_SNAKE — no type-name stutter, `enum class` already scopes | `HttpMethod::GET`, `Status::ERR_NETWORK` |
| Constants | SCREAMING_SNAKE; `constexpr`/`const`, never `#define` | `MAX_URL` |
| Macros / include guards | `VX_` + SCREAMING_SNAKE | `VX_HTTP_HPP`, `VX_HAS_TLS` |
| Predicates | `is_` / `has_` / `can_`, return plain `bool` | `is_open()`, `has_body()` |

SCREAMING_SNAKE is shared by constants, enum values and macros — the `VX_`
prefix is what tells them apart: `MAX_URL` is a constant, `VX_MAX_URL`
would be a macro.

---

## 2. Variable type prefixes

Prefix the type, then continue in snake_case: `u32_timeout_ms`, `sz_host`.
Prefixes encode what the compiler cannot check — byte counts vs element
counts, NUL-terminated strings vs fixed buffers, handles.

| Prefix | Type / meaning | Example |
|---|---|---|
| `b` | bool | `b_connected` |
| `ch` | single char | `ch_delim` |
| `sz` | NUL-terminated string (`char*`) | `sz_host` |
| `wsz` | NUL-terminated wide string (`wchar_t*`) | `wsz_path` |
| `rg` | fixed-size array ("range") | `rg_ports[16]` |
| `c` | **element** count | `c_entries` |
| `cb` | **byte** count (count-of-bytes) | `cb_body` |
| `i` | index | `i_row` |
| `u8` `u16` `u32` `u64` | unsigned fixed-width integer | `u32_timeout_ms` |
| `i32` `i64` | signed fixed-width integer | `i64_offset` |
| `f` | float | `f_ratio` |
| `d` | double | `d_progress` |
| `p` | pointer to object | `p_next` |
| `pp` | pointer to pointer | `pp_env` |
| `h` | WinAPI handle (`HANDLE`, `SOCKET`, `HINTERNET`) | `h_sock` |
| `fn` | function pointer | `fn_on_data` |
| `e` | variable whose type is an enum — **not** the enum type, **not** its values | `e_method` |
| struct/class instance | no prefix — a plain noun | `client`, `resp`, `addr` |

Application rules:

- The highest-value pair is **`c` vs `cb`**: confusing element counts with
  byte counts is the classic memory bug; the prefix makes it visible at
  every call site. Count prefixes describe meaning and may stand alone even
  when the underlying type is a plain `int` — `cb_sent` for what `send()`
  returns is correct.
- An enum involves **three** names: the type (`HttpMethod` — PascalCase),
  its values (`GET` — SCREAMING rule, §1), and variables of that type
  (`e_method` — this table).
- Loop indices `i`, `j` and throwaway locals in short scopes need no prefix.
- When a variable's type changes, its prefix changes with it — same commit.

---

## 3. WinAPI types

In public headers of WinAPI-only projects use the WinAPI vocabulary — one
vocabulary, no casts between twins: `SIZE_T` over `std::size_t`,
`DWORD`/`DWORD64` over `uint32_t`/`uint64_t`, `LONGLONG` over
`std::int64_t` (`Windows.h` already pulls `size_t` in through
`vcruntime.h`; `SIZE_T` is the same underlying type). Prefixes follow width
and meaning, never the typedef name; the prefix set in §2 is closed: no
`dw_`, no per-typedef inventions.

| WinAPI type | Prefix |
|---|---|
| `DWORD` | `u32_` |
| `DWORD64` | `u64_` |
| `WORD` | `u16_` |
| `LONGLONG` | `i64_` |
| `SIZE_T` | `cb_` / `c_` — by meaning |
| `BOOL` | `b_` — a boolean in disguise |

---

## 4. Scope & storage

| Marker | Meaning | Example |
|---|---|---|
| `g_` | global — must carry a comment justifying it | `g_heap` |
| anonymous `namespace {}` | file-local, preferred over `static` | `namespace { … }` |
| `s_` | function-local `static` state | `s_started` |
| trailing `_` | class member — the one member marker | `cb_len_` |
| SCREAMING_SNAKE | constant (§1) | `MAX_LEN` |

Members combine the §2 type prefix with the trailing underscore — the two
markers travel together, always: `u32_timeout_ms_`, `h_sock_`,
`b_connected_`. Never drop the type prefix (`timeout_ms_` alone says
nothing about what it counts) and never introduce a second member flavor
such as `m_`.

---

## 5. Function naming

- Verb-first: `get_`, `set_`, `parse_`, `build_`, `send_`.
- `out` parameters go last and are named `out`.
- Predicates return plain `bool` and are named `is_*` — not `get_*`.

---

## 6. File naming

| Item | Rule | Example |
|---|---|---|
| Source files | lowercase snake_case; headers `.hpp`, sources `.cpp` | `http_client.hpp` / `http_client.cpp` |
| Header guard | `VX_<FILE>_HPP` | `VX_HTTP_HPP` |
| Tests | one `test_<module>.cpp` per module | `tests/test_json.cpp` |
| Repo / project / target / output | lowercase | `basalt`, `basalt.lib` |

---

## 7. Waivers — naming exceptions

- **Mirroring WinAPI documentation**: keep MSDN parameter names verbatim
  (`dwFlags`, `lpSecurityAttributes`) so code and docs align line by line.
- **Mirroring CRT documentation**: CRT-replacement declarations keep the
  standard's parameter names verbatim (`dest`, `src`, `count`) for the same
  reason. Implementation bodies use §2-prefixed names for their own locals,
  and definition parameter names must match the declaration.
- Any other deviation requires a one-line comment at the site stating why.
  Silence is not a waiver.

---

## 8. Example — every rule in one place

`include/basalt/status.hpp` — enum type PascalCase, values SCREAMING_SNAKE:

```cpp
/* SPDX-License-Identifier: MIT */
#ifndef VX_STATUS_HPP
#define VX_STATUS_HPP

namespace basalt {

enum class Status {
    OK           =  0,
    ERR_ARG      = -1,
    ERR_NETWORK  = -5,
};

} // namespace basalt
#endif /* VX_STATUS_HPP */
```

`include/basalt/socket.hpp` — class, methods, parameters, members, guard:

```cpp
/* SPDX-License-Identifier: MIT */
#ifndef VX_SOCKET_HPP
#define VX_SOCKET_HPP

#include "basalt/status.hpp"

#include <winsock2.h>                              /* SIZE_T/DWORD/WORD via Windows headers */

namespace basalt {

class Socket {
public:
    Socket() = default;
    ~Socket();

    Status connect(const char* sz_host, WORD u16_port);
    Status send(const void* p_buf, SIZE_T cb_len);
    void close();

private:
    SOCKET      h_sock_         = INVALID_SOCKET; /* type prefix + member marker */
    DWORD       u32_timeout_ms_ = 30000;
    bool        b_connected_    = false;
};

} // namespace basalt
#endif /* VX_SOCKET_HPP */
```
