# C/C++ CODING CONVENTIONS — Master Reference

This document is the single coding standard for **every C/C++ project in this
workspace — present and future**. Read it before writing code. Follow it
unless §13 (waivers) applies.

It is project-agnostic by design: a new project inherits everything below and
only adds one row to the adaptation table (§1). Anyone (or any tool) reading
this file must be able to produce conforming code without other context.

Symbol prefix in examples: `<pfx>` in prose, `vx_` in code. Substitute the
project's own prefix from §1.

---

## 1. Project adaptation table

| Project | Purpose | Core language | Symbol prefix | Constraints |
|---|---|---|---|---|
| basalt | WinAPI-only foundation lib (crt, string, vector, json, socket, http) | C17 + optional C++ layer | `vx_` | no CRT dependency by design; prefix from the author's handle |
| sable | security research toolkit (api resolve, clr host, http) | C17 + C++ for clr module | `sb_` | builds on basalt |
| husk | modular shellcode kit | C17 freestanding | `hk_` | **no CRT**, §10 mandatory |

Rules for any **new** project:

- Choose a symbol prefix at repo creation: 2–4 lowercase letters + `_`,
  unique across the workspace, never changed afterwards (it is public ABI).
- The prefix does **not** have to derive from the project name
  (basalt → `vx_`, from the author's handle). It only has to be
  (a) unique per project — two libraries that can be linked into the same
  binary must never share one — and (b) traceable: record its origin in this
  table so `vx_socket_connect` can be traced back to basalt.
- One prefix per **project**, never per module, never per person. The module
  already lives in the second name segment (`vx_crt_memcpy`, `vx_str_dup`).
- Default core language: C17 (`/std:c17`). C++ is opt-in per layer (§9).
- Everything else in this document is inherited unchanged.

---

## 2. Principles

1. **Case carries meaning.** PascalCase = a type. snake_case = a function or
   a value. `Socket socket;` must be readable without an IDE.
2. **Prefixes encode what the compiler cannot check** — pointers, byte
   counts vs element counts, NUL-terminated strings vs fixed buffers,
   handles. Do not prefix what is obvious in a five-line scope (`i`, `tmp`).
3. **Consistency beats taste.** Never mix styles inside one module; never
   introduce a second local convention.
4. **Ownership is explicit.** Every resource has exactly one creator function
   and one destroyer. Whoever creates it, destroys it — unless the API
   documents a transfer.
5. **The public API is small, C-shaped, and stable.** Everything not public
   is `static`.

---

## 3. Files & organization

| Item | Rule | Example |
|---|---|---|
| Source files | lowercase snake_case; `.c`/`.h` for C, `.cpp`/`.hpp` for C++ | `http_client.c`, `socket.hpp` |
| Layout | one module = one folder under `src/`; public headers under `include/<lib>/` | `src/net/http_client.c` |
| Tests / examples | `tests/`, `examples/` at repo root | `tests/test_json.c` |
| Header guard | `<PREFIX>_<FILE>_H` | `VX_HTTP_H` |
| Include order | own header **first** (catches non-self-contained headers), blank line, system headers | see §15 |
| Repo / project / target / output | lowercase | `basalt`, `basalt.lib` |
| First line of every file | SPDX license identifier | `/* SPDX-License-Identifier: MIT */` |
| Braces | function: opening brace on its own line; control flow: same line; one-line function bodies and single guarded statements may collapse | see §15 |

Keep lines around 100 columns — prefer wrapping over horizontal scrolling.

Build settings, all configurations:

- C: `/std:c17`, `/W4`, warning-free. C++: `/std:c++20`, `/permissive-`, `/W4`.
- Ship a `.editorconfig` at repo root:

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

**Why the prefix exists.** It applies **only to symbols that cross the
module boundary** — `static` internals never carry it. For exported symbols
the prefix is not decoration: C has no namespaces, so the prefix *is* the
namespace at link time. A C++ `namespace` does not replace it, because
macros, C enum values and `extern "C"` exports all ignore namespaces —
the prefix stays mandatory at the ABI. The C++ wrapper layer re-exposes the
API under `namespace <project>` so application code never types the prefix.
The "no prefix for internals" rule is only safe because internals are
`static` (C) or in an anonymous namespace (C++) — those two rules travel
together. The prefix follows **C linkage, not file extension**: a `.cpp`
file that exports `extern "C"` functions is part of the C API and still
prefixes them (`vx_clr_run` lives in a `.cpp`); a C++ class or free
function inside `namespace basalt` never carries it.

| Category | Convention | Example |
|---|---|---|
| C public functions | `<pfx>_<module>_<verb>_<object>`, snake_case | `vx_http_request_send()` |
| C internal functions | snake_case, **no prefix**, always `static` | `parse_status_line()` |
| C structs / enums (public) | `<pfx>_<name>` lowercase typedef | `vx_http_response` |
| C++ classes / structs | PascalCase, no prefix | `HttpClient` |
| C++ methods & free functions | snake_case — methods follow free functions | `connect()`, `parse_url()` |
| C++ `enum class` values | `k` + PascalCase | `HttpMethod::kGet` |
| C enum values / macros / guards | `<PREFIX>_` + SCREAMING_SNAKE | `VX_HTTP_GET`, `VX_ERR_MEM` |
| Namespaces | lowercase, one word, = project name (not the prefix) | `namespace basalt` |
| Constants | `k` prefix: snake_case in C, PascalCase in C++; storage per §9 | `k_default_timeout_ms` (C) · `kMaxLen` (C++) |
| Predicates returning bool | prefix `is_` / `has_` / `can_` | `is_tls()`, `has_body()` |
| Lifecycle pairs | `create` / `destroy` only — never `init/free` mixed in | `vx_socket_create()` |

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
| `h` | WinAPI handle (`HANDLE`, `SOCKET`, `HINTERNET`) | `h_file` |
| `fn` | function pointer | `fn_on_data` |
| `e` | variable whose type is an enum — **not** the enum type itself, **not** its values (those follow §4) | `e_method` |
| struct instance | no prefix — a plain noun | `client`, `resp`, `addr` |

Application rules:

- The highest-value pair in the table is **`c` vs `cb`**: confusing element
  counts with byte counts is the classic C bug; the prefix makes it visible
  at every call site.
- An enum involves **three** names: the type (`vx_http_method` — §4 type
  rule), its values (`VX_HTTP_GET` — §4 constant rule), and variables of
  that type (`e_method` — this table). §15's status enum follows all three.
- Pointers to C handles take `p` + handle name: `p_sock` for
  `vx_socket*`. Inside C++ classes, members are the object — no `p` needed.
- Loop indices `i`, `j` and throwaway locals in short scopes need no prefix.
- When a variable's type changes, its prefix changes with it — same commit.

---

## 6. Scope & storage

| Marker | Meaning | Example |
|---|---|---|
| `g_` | global — must carry a comment justifying it | `g_heap` |
| `s_` | file-scope `static` | `s_init_done` |
| trailing `_` | C++ class member (default for this workspace) | `cb_len_` |
| `m_` | alternative member marker — pick ONE per project, never both | `m_cbLen` |
| `k` | constant (see §4) | `kMaxLen` |

Hard rules:

- **No exported mutable globals.** Public state goes through functions.
- A C++ member that already has a type prefix keeps it and appends the
  member marker: `u32_timeout_ms_`.

---

## 7. Functions

- Verb-first: `get_`, `set_`, `parse_`, `build_`, `send_`, `free_`.
- `out` parameters go last and are named `out` (C++) / `p_out` (C).
- Predicates return plain `bool` and are named `is_*` — not `get_*`.
- Functions owning two or more resources unwind through a single
  `goto cleanup` exit path.
- Runtime input validation returns an error code (§8). `assert` is only for
  caller bugs that should never ship.

---

## 8. Errors & status codes

- One status enum per library: `0` = success, negative = error.

```c
typedef enum <pfx>_status {
    <PFX>_OK              =  0,
    <PFX>_ERR_ARG         = -1,   /* invalid argument        */
    <PFX>_ERR_MEM         = -2,   /* allocation failure      */
    <PFX>_ERR_PARSE       = -3,   /* malformed input         */
    <PFX>_ERR_UNSUPPORTED = -4,   /* not supported here      */
    <PFX>_ERR_NETWORK     = -5,   /* transport-level failure */
} <pfx>_status;
```

- Never reuse `<errno.h>` names (`EINVAL`…) — they collide with macros.
- On any error path, output parameters are left fully zeroed/untouched —
  never half-initialized.
- No exceptions anywhere (see §9). No `setjmp/longjmp`.

---

## 9. C core / C++ layer split

- **C core**: opaque handles — `typedef struct <pfx>_socket <pfx>_socket;`
  in the header, definition hidden in the `.c`, `create`/`destroy` lifecycle.
- **Every public header compiles from both languages**: `extern "C"` guarded
  by `#ifdef __cplusplus`.
- **C++ layer: plain classes only.** The five prohibitions:
  1. no exceptions — return codes only;
  2. no STL members — own types or fixed arrays;
  3. no `virtual` unless polymorphism is the point;
  4. no global/static objects with constructors;
  5. copy deleted, move defined — or both deleted.
- C gotcha: a `static const` variable is **not** a constant expression in C
  and cannot size a file-scope array — use `enum { k_name_max = 64 };`.
- **Classes never carry `extern "C"`** — the standard ignores C linkage for
  class members; a member function with `extern "C"` is a compile error.
  Expose a class through its namespace (C++ consumers) or through
  `extern "C"` free functions taking an opaque pointer (C consumers). At a
  C-API boundary use `new (std::nothrow)` — plain `new` throws.
- **C/C++ callback bridge.** A C API taking a function pointer cannot accept
  a member function or a capturing lambda. Standard pattern: a C-linkage
  thunk with internal linkage forwards to the member through the context
  pointer the API provides:

  ```cpp
  extern "C" {
      static void on_data_thunk(void* p_ctx, const char* sz_chunk, size_t cb)
      {
          static_cast<Downloader*>(p_ctx)->on_data(sz_chunk, cb);
      }
  }
  vx_http_set_callback(p_client, on_data_thunk, this);   /* this rides p_ctx */
  ```

---

## 10. Shellcode / no-CRT readiness

Applies to projects marked no-CRT in §1 (and to any module meant to become
position-independent code).

- CRT is banned: no `malloc/printf/memcpy*` from the CRT unless the module's
  own `crt` layer provides it. Memory comes from the module's allocator
  (Heap/VirtualAlloc wrappers).
- Includes: WinAPI headers + own headers only.
- Build flags: `/GS-`, `/O1` or `/Os`, no SEH (`/EH-`); never `__try/__except`
  — x64 SEH is table-based unwinding and does not survive shellcode.
- No global/static objects with constructors; no TLS (`__declspec(thread)`).
- Position independence: no absolute data references or relocations — verify
  the artifact links with an empty relocation table.
- Bitness: **x64 is the default** (RIP-relative PIC, uniform `syscall`,
  matches modern processes). Build a second x86 artifact only for 32-bit
  targets; never mix bitness with the host process.
- One status-code family and one allocator family across the whole project —
  mixed heaps across module boundaries are undefined behavior.

---

## 11. API stability & versioning

- Version macros in the main header: `<PFX>_VERSION_MAJOR/MINOR/PATCH`.
- After first release the public C API is frozen: additions only;
  breaking changes require a major version and a `MIGRATION.md` note.
- Prefer opaque types; if a struct must be public, its fields are append-only
  and never reordered (ABI).

---

## 12. Comments & documentation

- `/** */` Doxygen on every public symbol — at minimum a one-line summary.
- Comments explain **why**, never restate what the code does.
- Every public header is self-contained: it compiles alone
  (guaranteed by including it first in its own `.c`).
- Code, comments, identifiers: English only.
- No commented-out code in committed files — delete it; history remembers.

---

## 13. Waivers

- **Mirroring WinAPI documentation**: keep MSDN parameter names verbatim
  (`dwFlags`, `lpSecurityAttributes`) so code and docs align line by line.
- Any other deviation requires a one-line comment at the site stating why.
  Silence is not a waiver.

---

## 14. Review checklist

Run on every new module and every PR:

- [ ] Header guard + `extern "C"` block present
- [ ] Every non-`static` symbol carries the project prefix
- [ ] Every internal helper is `static`
- [ ] Lifecycle via `create`/`destroy` only; ownership documented
- [ ] `out` parameters last, named `out`/`p_out`
- [ ] Error paths leave outputs untouched; status codes per §8
- [ ] Type prefixes applied where the compiler can't help (§5)
- [ ] No exported mutable globals; C++ layer respects the five prohibitions
- [ ] Compiles `/W4`-clean; header self-contained
- [ ] no-CRT project: all of §10 holds

---

## 15. Worked examples — full files

One module, three layers, complete enough to copy. This is the shape every
module follows. The consumer example at the end shows both doors of §4:
the namespaced C++ class and the prefixed C API. Note the deliberate split:
folder names use the project name (`basalt/`), macros and symbols use the
prefix (`VX_`, `vx_`).

### 15.1 `include/basalt/socket.h` — C API, both-language header

```c
/* SPDX-License-Identifier: MIT */
#ifndef VX_SOCKET_H
#define VX_SOCKET_H

#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef enum vx_status {
    VX_OK          =  0,
    VX_ERR_ARG     = -1,
    VX_ERR_NETWORK = -5,
} vx_status;

typedef struct vx_socket vx_socket;              /* opaque handle */

vx_socket* vx_socket_create(void);
void       vx_socket_destroy(vx_socket* p_sock);
vx_status  vx_socket_connect(vx_socket* p_sock,
                             const char* sz_host, uint16_t u16_port);
vx_status  vx_socket_send(vx_socket* p_sock,
                          const void* p_buf, size_t cb_len);

#ifdef __cplusplus
}
#endif
#endif /* VX_SOCKET_H */
```

### 15.2 `src/net/socket.c` — C implementation

```c
/* SPDX-License-Identifier: MIT */
#include "basalt/socket.h"                       /* own header first */

#include <winsock2.h>
#include <ws2tcpip.h>

#pragma comment(lib, "ws2_32.lib")

enum { k_default_timeout_ms = 30000 };           /* true C constant (§9) */

struct vx_socket {                               /* opaque half — this file only */
    SOCKET     h_sock;
    uint32_t   u32_timeout_ms;
    bool       b_connected;
};

static bool host_is_valid(const char* sz_host)   /* internal: static, no prefix */
{
    return sz_host != NULL && sz_host[0] != '\0';
}

static vx_status net_init_once(void)             /* WSAStartup exactly once */
{
    static bool s_started = false;               /* single-threaded for brevity */
    WSADATA     wsa;

    if (s_started)
        return VX_OK;
    if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
        return VX_ERR_NETWORK;
    s_started = true;
    return VX_OK;
}

vx_socket* vx_socket_create(void)
{
    /* HeapAlloc keeps the example CRT-free; real modules go through the
       project's own allocator (§10). */
    HANDLE      h_heap = GetProcessHeap();
    vx_socket*  p_sock = (h_heap != NULL)
        ? (vx_socket*)HeapAlloc(h_heap, HEAP_ZERO_MEMORY, sizeof(*p_sock))
        : NULL;

    if (p_sock == NULL)
        return NULL;
    p_sock->h_sock         = INVALID_SOCKET;
    p_sock->u32_timeout_ms = k_default_timeout_ms;
    return p_sock;
}

void vx_socket_destroy(vx_socket* p_sock)
{
    if (p_sock == NULL)
        return;
    if (p_sock->h_sock != INVALID_SOCKET)
        closesocket(p_sock->h_sock);
    HeapFree(GetProcessHeap(), 0, p_sock);
}

vx_status vx_socket_connect(vx_socket* p_sock,
                            const char* sz_host, uint16_t u16_port)
{
    if (p_sock == NULL || !host_is_valid(sz_host) || u16_port == 0)
        return VX_ERR_ARG;

    vx_status   e_status = VX_OK;                /* enum variable: e_ prefix */
    SOCKADDR_IN sa       = { 0 };                /* struct instance: noun    */

    sa.sin_family = AF_INET;
    sa.sin_port   = htons(u16_port);
    if (inet_pton(AF_INET, sz_host, &sa.sin_addr) != 1)
        return VX_ERR_ARG;
    if (net_init_once() != VX_OK)
        return VX_ERR_NETWORK;

    p_sock->h_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (p_sock->h_sock == INVALID_SOCKET) {
        e_status = VX_ERR_NETWORK;
        goto cleanup;                            /* single unwind path (§7) */
    }
    if (connect(p_sock->h_sock, (const SOCKADDR*)&sa, sizeof(sa)) != 0) {
        e_status = VX_ERR_NETWORK;
        goto cleanup;
    }
    p_sock->b_connected = true;

cleanup:
    if (e_status != VX_OK && p_sock->h_sock != INVALID_SOCKET) {
        closesocket(p_sock->h_sock);             /* never leak the handle */
        p_sock->h_sock = INVALID_SOCKET;
    }
    return e_status;
}

vx_status vx_socket_send(vx_socket* p_sock, const void* p_buf, size_t cb_len)
{
    if (p_sock == NULL || p_buf == NULL || !p_sock->b_connected || cb_len == 0)
        return VX_ERR_ARG;

    /* Single-shot for brevity — production code loops until cb_len. */
    int cb_sent = send(p_sock->h_sock, (const char*)p_buf, (int)cb_len, 0);
    if (cb_sent == SOCKET_ERROR || (size_t)cb_sent != cb_len)
        return VX_ERR_NETWORK;
    return VX_OK;
}
```

### 15.3 `include/basalt/socket_cpp.hpp` — C++ wrapper (plain class, §9 rules)

```cpp
/* SPDX-License-Identifier: MIT */
#ifndef VX_SOCKET_CPP_HPP
#define VX_SOCKET_CPP_HPP

#include "basalt/socket.h"
#include <winsock2.h>

namespace basalt {

class Socket {
public:
    Socket() = default;
    ~Socket() { close(); }

    Socket(const Socket&)            = delete;    /* §9 rule 5: no copy   */
    Socket& operator=(const Socket&) = delete;

    bool connect(const char* sz_host, uint16_t u16_port)
    {
        close();
        p_sock_ = vx_socket_create();
        if (p_sock_ == nullptr)
            return false;
        return vx_socket_connect(p_sock_, sz_host, u16_port) == VX_OK;
    }

    bool send(const void* p_buf, size_t cb_len)
    {
        return p_sock_ != nullptr
            && vx_socket_send(p_sock_, p_buf, cb_len) == VX_OK;
    }

    void close()
    {
        vx_socket_destroy(p_sock_);
        p_sock_ = nullptr;
    }

private:
    vx_socket* p_sock_ = nullptr;                /* type prefix + member marker */
};

} // namespace basalt
#endif /* VX_SOCKET_CPP_HPP */
```

### 15.4 `examples/echo.cpp` — consumer side: both doors of §4

```cpp
/* SPDX-License-Identifier: MIT */
#include "basalt/socket_cpp.hpp"

int main()
{
    /* C++ door — namespace + RAII; application code never types vx_ */
    basalt::Socket socket;                       /* PascalCase type, snake var */
    if (!socket.connect("127.0.0.1", 9000))
        return 1;
    socket.send("ping", 4);

    /* C door — prefixed API: what C / C# / Rust / shellcode callers see */
    vx_socket* p_sock = vx_socket_create();
    if (vx_socket_connect(p_sock, "127.0.0.1", 9000) == VX_OK)
        vx_socket_send(p_sock, "ping", 4);
    vx_socket_destroy(p_sock);
    return 0;
}
```
