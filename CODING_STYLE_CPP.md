# C++ CODING CONVENTIONS — CODING_STYLE_CPP

The single C++ standard for **every C++ project in this workspace — present
and future**. Read it before writing code. Follow it unless §14 (waivers)
applies.

Project-agnostic by design: a new project inherits everything below and only
adds one row to the adaptation table (§1). Anyone (or any tool) reading this
file must be able to produce conforming code without other context.

Macro prefix in examples: `VX_` (from project basalt). Substitute the
project's own macro prefix from §1.

---

## 1. Project adaptation table

| Project | Purpose | Language | Macro prefix | Namespace |
|---|---|---|---|---|
| basalt | WinAPI-only foundation lib (string, vector, json, socket, http) | C++20 | `VX_` | `basalt` |

- C-only projects (sable, husk) follow a separate C standard — outside this
  document.
- **New project**: add a row here. Pick a macro prefix (2–4 letters + `_`,
  unique, never changed) and set the namespace = project name.
- Default standard: `/std:c++20`, `/permissive-`, `/W4`, warning-free.

---

## 2. Principles

1. **Case carries meaning.** PascalCase = a type. snake_case = a function or
   a value. `Socket socket;` must be readable without an IDE.
2. **The namespace is the prefix of the C++ world.** One namespace per
   project; no name folds the prefix into itself.
3. **Prefixes (§5) encode what the compiler cannot check** — byte counts vs
   element counts, NUL-terminated strings vs fixed buffers, handles.
4. **Ownership is explicit.** Every resource lives in a class with a
   destructor (RAII). No naked `new`/`delete` outside owner classes.
5. **Consistency beats taste.** Never mix styles inside one module.

---

## 3. Files & organization

| Item | Rule | Example |
|---|---|---|
| Source files | lowercase snake_case; headers `.hpp`, sources `.cpp` | `http_client.hpp` / `http_client.cpp` |
| Module shape | **one `.hpp` + one `.cpp` pair** per module — classes and free-function families alike; declaration in the header, bodies in the source | §16 |
| Layout | one module = one folder under `src/`; public headers under `include/<lib>/` | `src/net/http_client.cpp` |
| Tests / examples | `tests/`, `examples/` at repo root | `tests/test_json.cpp` |
| Header guard | `VX_<FILE>_HPP` | `VX_HTTP_HPP` |
| Include order | own header **first**, blank line, then std headers, then others | see §15 |
| Repo / project / target / output | lowercase | `basalt`, `basalt.lib` |
| First line of every file | SPDX license identifier | `/* SPDX-License-Identifier: MIT */` |
| Braces | function: opening brace on its own line; control flow: same line; one-line bodies and single guarded statements may collapse | see §16 |

Repository layout — one solution, one static-lib project, modules as folders.
The nested project folder is the VS default and is fine; the only invariant
is that `include/` and `src/` stay **siblings of the `.vcxproj`**, so every
path inside it keeps working:

```text
basalt/                          # repo root — the solution sits here (VS default)
├── basalt.slnx                  # references the projects by relative path
├── basalt.props                 # shared settings: /std:c++20 /permissive- /W4
└── basalt/                      # project folder — one folder per vcxproj
    ├── basalt.vcxproj           # ONE static-lib project
    ├── include/
    │   └── basalt/              # public headers, one per module
    │       ├── crt.hpp
    │       ├── socket.hpp
    │       ├── http.hpp
    │       └── json.hpp
    ├── src/                     # NO main() here — this builds into basalt.lib
    │   ├── crt/                 # depends on nothing
    │   ├── socket/              # depends on crt
    │   ├── json/                # depends on crt
    │   └── http/                # depends on socket (+ json as needed)
    └── tests/
        └── tests.vcxproj        # exe project — main() lives here (test_*.cpp files)
```

Split into per-module projects (`basalt_core.lib`, `basalt_net.lib`) only when
build times or partial linking demand it — the source layout stays identical.

Keep lines around 100 columns — prefer wrapping over horizontal scrolling.

```ini
root = true

[*]
charset = utf-8
end_of_line = crlf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4
```

---

## 4. Identifiers by category

**Why the macro prefix exists.** The namespace protects every type, function
and variable — that is its whole job. **Macros are the one thing it cannot
protect**: the preprocessor runs before namespaces exist, so a `#define` is
global no matter where it sits. Therefore the `VX_` prefix belongs to macros
only; classes, functions and variables never carry it.

| Category | Convention | Example |
|---|---|---|
| Namespace | lowercase, one word, = project name | `namespace basalt` |
| Classes / structs / enums | PascalCase, no prefix | `HttpClient`, `HttpResponse` |
| Methods & free functions | snake_case — methods follow free functions | `connect()`, `parse_url()` |
| `enum class` values | `k` + PascalCase | `HttpMethod::kGet` |
| Macros / include guards | `VX_` + SCREAMING_SNAKE | `VX_HTTP_HPP`, `VX_HAS_TLS` |
| Constants | `k` + PascalCase; `constexpr` or `const` — never `#define` | `kMaxUrl` |
| Predicates returning bool | prefix `is_` / `has_` / `can_` | `is_open()`, `has_body()` |
| Resource lifecycle | RAII — constructor acquires, destructor releases; no create/destroy pairs | `Socket s; s.connect(...)` |
| Prefer over `#define` | `constexpr` constant / `constexpr` function / inline function | `constexpr uint32_t kMaxUrl = 2048;` |

**Macros ignore namespaces** — `#define` inside `namespace basalt` is still a
global macro. In C++ code prefer `constexpr` and inline functions; macros
survive only as include guards and conditional-compilation flags.

---

## 5. Variable type prefixes

Prefix the type, then continue in snake_case: `u32_timeout_ms`, `sz_host`.

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
  every call site.
- An enum involves **three** names: the type (`HttpMethod` — PascalCase),
  its values (`kGet` — `k` rule), and variables of that type (`e_method` —
  this table).
- Loop indices `i`, `j` and throwaway locals in short scopes need no prefix.
- When a variable's type changes, its prefix changes with it — same commit.

---

## 6. Scope & storage

| Marker | Meaning | Example |
|---|---|---|
| `g_` | global — must carry a comment justifying it | `g_heap` |
| anonymous `namespace {}` | file-local (preferred over `static` in C++) | helpers in §16 |
| `s_` | function-local `static` state | `s_started` |
| trailing `_` | class member marker (default flavor) | `cb_len_` |
| `m_` | alternative member marker — pick ONE per project, never both | `m_cbLen` |
| `k` | constant (see §4) | `kMaxLen` |

**Members** combine the §5 type prefix with the member marker — the two
markers travel together, always:

| Flavor | Formula | Examples |
|---|---|---|
| trailing `_` (default) | §5 prefix + name + `_` | `u32_timeout_ms_`, `h_sock_`, `b_connected_`, `p_sock_` |
| `m_` (MSVC flavor) | `m_` + §5 prefix + camelCase | `m_u32TimeoutMs`, `m_cbLen`, `m_hSock` |

Never double-mark (`m_u32_timeout_ms_`) and never drop the type prefix
(`timeout_ms_` alone says nothing about what it counts).

Hard rules:

- **No exported mutable globals.** Public state goes through classes or
  functions.

---

## 7. Functions

- Verb-first: `get_`, `set_`, `parse_`, `build_`, `send_`.
- `out` parameters go last and are named `out`.
- Predicates return plain `bool` and are named `is_*` — not `get_*`.
- Cleanup belongs to destructors (RAII) — no `goto cleanup` chains.
- Runtime input validation returns an error code (§8); `assert` is only for
  caller bugs that should never ship.
- Error-returning functions are marked `[[nodiscard]]`.

---

## 8. Errors & status codes

- One `enum class status` per library, inside the namespace:
  `0` = success, negative = error.

```cpp
namespace basalt {
enum class status {
    kOk          =  0,
    kErrArg      = -1,   /* invalid argument        */
    kErrMem      = -2,   /* allocation failure      */
    kErrParse    = -3,   /* malformed input         */
    kErrNetwork  = -5,   /* transport-level failure */
};
} // namespace basalt
```

- Never reuse `<cerrno>` names (`EINVAL`…).
- On any error path, output parameters are left fully zeroed/untouched —
  never half-initialized.
- **No exceptions anywhere** (§9 rule 1). No `setjmp/longjmp`. Status codes
  travel with `[[nodiscard]]`.

---

## 9. Core rules

**The five prohibitions** (what keeps binaries small, deterministic and
shellcode-compatible):

1. no exceptions — status codes only;
2. no STL members in classes — own types, `std::span`/`std::string_view`, or
   fixed arrays;
3. no `virtual` unless polymorphism is the point;
4. no global/static objects with constructors;
5. copy deleted, move defined — or both deleted (rule of five complete).

**Ownership**: a resource owner class implements the full rule of five;
everything else holds no resources and stays copyable.

**Layout**: one namespace per project wraps every public type
(`basalt::Socket`, `basalt::HttpClient`); never fold the macro prefix into
class names. Stateless helpers stay free functions in the namespace — never
a class of static methods; one level of nested namespaces may group families
(`basalt::mem::copy`, `basalt::str::dup`).

**Header/source split**: the `.hpp` holds the class declaration —
signatures, `= default`/`= delete`, member initializers, nothing with
logic; every body lives in the `.cpp`, qualified (`Socket::connect`) with
the namespace already opened. The `.cpp` includes its own header first.
`using namespace` never appears in a header; in a `.cpp`, open the
namespace block instead.

**Casts by name**: `static_cast`, `reinterpret_cast` — never the C-style
`(T)x`. Plain `new` throws; use `new (std::nothrow)` or the project
allocator.

---

## 10. Free-function modules (no class)

Some modules are pure operations — `crt`, string helpers, small math. They
hold no state, so they get no class: the module is a **family of free
functions** inside the project namespace.

1. **Never a class of static methods.** `class Mem { static void* copy(); };`
   is a namespace with extra keystrokes.
2. **Group by one nested namespace**: `basalt::mem`, `basalt::str`,
   `basalt::math`. Do not name a function exactly like a global you did not
   write — `basalt::memcpy` shadows `::memcpy` on every unqualified lookup
   inside the namespace; `basalt::mem::copy` says the same thing safely.
   **Exception — CRT replacements**: the `crt` module may keep the CRT name
   inside the namespace (`basalt::memcpy`, `basalt::memzero`) because
   replacing those functions is its purpose. Conditions: same signature and
   semantics as the standard, call sites always qualified
   (`basalt::memcpy(...)`), and the implementation reaches the CRT through
   `::memcpy`/intrinsics (in no-CRT builds it is written by hand). Never
   define these at global scope — replacing `::memcpy` itself is undefined
   behavior.
3. **Stateless by law.** No mutable file-scope variables, no mutable
   function-local `static`. Constants and init guards (like §16.2's
   `net_init_once`) are fine — anything else is hidden global state and
   belongs in a class.
4. **File shape is unchanged**: one `.hpp` + one `.cpp` pair, own header
   first, file-local helpers in an anonymous namespace.
5. **Mark what the compiler can enforce**: `noexcept` on non-allocating
   operations, `constexpr` where the body qualifies, `[[nodiscard]]` when
   discarding the result is certainly a bug.
6. **Promote to a class the moment state appears** — shared resource,
   lifecycle, cached values. That need is an object asking to exist.

```cpp
/* SPDX-License-Identifier: MIT */
/* include/basalt/mem.hpp */
#ifndef VX_MEM_HPP
#define VX_MEM_HPP

#include <cstddef>

namespace basalt::mem {

/// Copy cb_len bytes. Regions must not overlap.
[[nodiscard]] void* copy(void* p_dst, const void* p_src, std::size_t cb_len) noexcept;

/// Zero cb_len bytes.
void zero(void* p_dst, std::size_t cb_len) noexcept;

} // namespace basalt::mem
#endif /* VX_MEM_HPP */
```

```cpp
/* SPDX-License-Identifier: MIT */
/* src/core/mem.cpp */
#include "basalt/mem.hpp"                         /* own header first */

#include <cstring>

namespace basalt::mem {

void* copy(void* p_dst, const void* p_src, std::size_t cb_len) noexcept
{
    if (cb_len == 0 || p_dst == p_src)
        return p_dst;
    return std::memcpy(p_dst, p_src, cb_len);     /* ::memcpy — global, on purpose */
}

void zero(void* p_dst, std::size_t cb_len) noexcept
{
    if (p_dst != nullptr && cb_len != 0)
        std::memset(p_dst, 0, cb_len);
}

} // namespace basalt::mem
```

---

## 11. Freestanding / no-CRT readiness

Applies to modules meant to become position-independent code.

- CRT is banned: memory comes from the project allocator (Heap/VirtualAlloc
  wrappers); no `std::string`/`std::vector`/iostreams.
- C++20 subset that is safe here: `std::span`, `std::string_view`,
  `std::exchange`, concepts, `constexpr`, structured bindings — nothing
  allocated, nothing thrown.
- Build flags: `/GS-`, `/O1` or `/Os`, no exceptions (`/EHs-c-`); never
  `__try/__except` — x64 SEH is table-based unwinding and does not survive
  shellcode.
- No global/static objects with constructors; no TLS (`thread_local`).
- Position independence: verify the artifact links with an empty relocation
  table.
- Bitness: **x64 is the default**; a second x86 artifact only for 32-bit
  targets; never mix bitness with the host process.
- One status family and one allocator family across the whole project.

---

## 12. API stability & versioning

- Version macros in the main header: `VX_VERSION_MAJOR/MINOR/PATCH`.
- After first release the public API is frozen: additions only; breaking
  changes require a major version and a `MIGRATION.md` note.
- Prefer hiding implementation: PIMPL or forward-declared privates keep
  headers light and ABI stable.

---

## 13. Comments & documentation

- `///` Doxygen on every public symbol — at minimum a one-line summary.
- Comments explain **why**, never restate what the code does.
- Code, comments, identifiers: English only.
- No commented-out code in committed files — delete it; history remembers.

---

## 14. Waivers

- **Mirroring WinAPI documentation**: keep MSDN parameter names verbatim
  (`dwFlags`, `lpSecurityAttributes`) so code and docs align line by line.
- Any other deviation requires a one-line comment at the site stating why.
  Silence is not a waiver.

---

## 15. Review checklist

Run on every new module and every PR:

- [ ] Everything public sits inside the project namespace
- [ ] Rule of five complete on every resource owner; five prohibitions hold
- [ ] `[[nodiscard]]` on all status-returning functions
- [ ] `out` parameters last, named `out`; error paths leave outputs untouched
- [ ] Type prefixes applied where the compiler can't help (§5)
- [ ] No `#define` constants — `constexpr` instead; macros are `VX_`-prefixed
- [ ] Named casts only; no plain `new` outside owners
- [ ] Bodies in the `.cpp`, declaration in the `.hpp`; no `using namespace`
      in headers
- [ ] Compiles `/W4`-clean
- [ ] no-CRT module: all of §11 holds

---

## 16. Worked example — full files

One module = two files: `socket.hpp` (declaration) + `socket.cpp`
(implementation), plus a consumer.

### 16.1 `include/basalt/socket.hpp`

```cpp
/* SPDX-License-Identifier: MIT */
#ifndef VX_SOCKET_HPP
#define VX_SOCKET_HPP

#include <cstddef>
#include <cstdint>
#include <winsock2.h>

namespace basalt {

enum class status {
    kOk          =  0,
    kErrArg      = -1,
    kErrNetwork  = -5,
};

class Socket {
public:
    Socket() = default;
    ~Socket();                                    /* body in the .cpp */

    Socket(const Socket&)            = delete;    /* rule of five */
    Socket& operator=(const Socket&) = delete;

    Socket(Socket&& other) noexcept;
    Socket& operator=(Socket&& other) noexcept;

    [[nodiscard]] status connect(const char* sz_host, uint16_t u16_port);
    [[nodiscard]] status send(const void* p_buf, std::size_t cb_len);
    void close();

private:
    SOCKET   h_sock_        = INVALID_SOCKET;     /* type prefix + member marker */
    uint32_t u32_timeout_ms_ = 30000;
    bool     b_connected_   = false;
};

} // namespace basalt
#endif /* VX_SOCKET_HPP */
```

### 16.2 `src/net/socket.cpp`

```cpp
/* SPDX-License-Identifier: MIT */
#include "basalt/socket.hpp"                      /* own header first */

#include <utility>

#include <ws2tcpip.h>

#pragma comment(lib, "ws2_32.lib")

namespace basalt {

namespace {                                       /* file-local: anonymous namespace */
constexpr uint32_t kDefaultTimeoutMs = 30000;

bool host_is_valid(const char* sz_host) noexcept
{
    return sz_host != nullptr && sz_host[0] != '\0';
}

bool net_init_once()                              /* WSAStartup exactly once */
{
    static bool s_started = false;                /* single-threaded for brevity */
    WSADATA     wsa {};
    if (s_started)
        return true;
    if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
        return false;
    s_started = true;
    return true;
}
} // anonymous namespace

Socket::~Socket() { close(); }

Socket::Socket(Socket&& other) noexcept
    : h_sock_(std::exchange(other.h_sock_, INVALID_SOCKET)),
      u32_timeout_ms_(std::exchange(other.u32_timeout_ms_, kDefaultTimeoutMs)),
      b_connected_(std::exchange(other.b_connected_, false)) {}

Socket& Socket::operator=(Socket&& other) noexcept
{
    if (this != &other) {
        close();
        h_sock_       = std::exchange(other.h_sock_, INVALID_SOCKET);
        u32_timeout_ms_ = other.u32_timeout_ms_;
        b_connected_  = std::exchange(other.b_connected_, false);
    }
    return *this;
}

status Socket::connect(const char* sz_host, uint16_t u16_port)
{
    if (h_sock_ != INVALID_SOCKET)
        close();
    if (!host_is_valid(sz_host) || u16_port == 0)
        return status::kErrArg;
    if (!net_init_once())
        return status::kErrNetwork;

    SOCKADDR_IN sa {};
    sa.sin_family = AF_INET;
    sa.sin_port   = htons(u16_port);
    if (inet_pton(AF_INET, sz_host, &sa.sin_addr) != 1)
        return status::kErrArg;

    h_sock_ = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (h_sock_ == INVALID_SOCKET)
        return status::kErrNetwork;
    if (connect(h_sock_, reinterpret_cast<const SOCKADDR*>(&sa), sizeof(sa)) != 0) {
        closesocket(h_sock_);                     /* never leak the handle */
        h_sock_ = INVALID_SOCKET;
        return status::kErrNetwork;
    }
    b_connected_ = true;
    return status::kOk;
}

status Socket::send(const void* p_buf, std::size_t cb_len)
{
    if (p_buf == nullptr || !b_connected_ || cb_len == 0)
        return status::kErrArg;

    /* Single-shot for brevity — production code loops until cb_len. */
    const int cb_sent = send(h_sock_,
                             static_cast<const char*>(p_buf),
                             static_cast<int>(cb_len), 0);
    if (cb_sent == SOCKET_ERROR || static_cast<std::size_t>(cb_sent) != cb_len)
        return status::kErrNetwork;
    return status::kOk;
}

void Socket::close()
{
    if (h_sock_ != INVALID_SOCKET)
        closesocket(h_sock_);
    h_sock_ = INVALID_SOCKET;
    b_connected_ = false;
}

} // namespace basalt
```

### 16.3 `examples/echo.cpp` — consumer side

```cpp
/* SPDX-License-Identifier: MIT */
#include "basalt/socket.hpp"

int main()
{
    basalt::Socket socket;                        /* PascalCase type, snake var */
    if (socket.connect("127.0.0.1", 9000) != basalt::status::kOk)
        return 1;
    socket.send("ping", 4);
    /* RAII: destructor closes — no manual close needed */
    return 0;
}
```
