# Changelog

All notable changes to libcurl-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-15

The first release: twenty-seven entry points of the libcurl C API, one
`@ffi` declaration each, and no logic.

### Added

- `libcurl` — the whole surface, in six groups.
  - The library: `curl_global_init`, `curl_global_cleanup`,
    `curl_version` and `curl_free`.
  - The easy handle: `curl_easy_init`, `curl_easy_cleanup`,
    `curl_easy_duphandle` and `curl_easy_reset`.
  - The options: the four typed declarations of `curl_easy_setopt`.
  - The transfer: `curl_easy_perform`, `curl_easy_send`,
    `curl_easy_recv`, `curl_easy_upkeep` and `curl_easy_pause`.
  - The information: the four typed declarations of
    `curl_easy_getinfo`, and `curl_easy_strerror`.
  - The helpers: `curl_slist_append`, `curl_slist_free_all`,
    `curl_easy_escape`, `curl_easy_unescape` and `curl_getdate`.
- `tests/libcurl_tests.nv` — thirteen tests over the signatures. They
  call the C library, so they need libcurl installed. No test opens a
  network connection: the two that reach the transfer group assert what
  `curl_easy_upkeep`, `curl_easy_send` and `curl_easy_recv` answer
  before any connection exists, and the pointer item is read through a
  cookie the suite seeds itself. All thirteen pass. The run then exits
  23 from the default leak check, which counts the copy `ptr.read_str`
  makes of what the C library owns; that is a toolchain defect, it is
  filed as one, and the README's "Tests" section says how to run
  without it.

### The variadic calls are declared once per argument kind

`curl_easy_setopt` and `curl_easy_getinfo` are variadic in C: the type
of the third argument is decided by the option or information number,
and the compiler cannot check it. A variadic call has no single
novo-lang signature, so each argument kind gets its own declaration and
all of them resolve to the same symbol —
`curl_easy_setopt_long`, `_str`, `_ptr` and `_off_t`, and
`curl_easy_getinfo_long`, `_str`, `_off_t` and `_ptr`. The caller picks
the declaration that matches the number, and the README's tables say
which number is which kind.

This is the shape libssl-sys uses for `SSL_ctrl`, with one difference.
`SSL_ctrl` is a fixed-arity C function that the OpenSSL headers wrap in
macros, so one declaration covers it. `curl_easy_setopt` is variadic
itself, so one declaration cannot.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**Everything that takes a C function pointer.** `CURLOPT_WRITEFUNCTION`,
`CURLOPT_READFUNCTION`, `CURLOPT_HEADERFUNCTION`,
`CURLOPT_PROGRESSFUNCTION` and their neighbours are absent, because a
novo-lang function is not a C function pointer. Without a write
callback libcurl writes a response body to standard output, which is
its default. A program that needs the bytes writes a small C function
of its own, or uses `CURLOPT_CONNECT_ONLY` with `curl_easy_send` and
`curl_easy_recv`, which are here.

**The multi and share interfaces.** `curl_multi_*` drives many
transfers from one thread and `curl_share_*` shares a cookie jar and a
connection cache between handles. Both are left out of the first
release.

**The mime interface.** `curl_mime_init` and its neighbours build a
multipart body. They are left out of the first release; a program that
posts one builds the body itself and sets `CURLOPT_POSTFIELDS`.

**The URL API.** `curl_url`, `curl_url_set` and `curl_url_get` parse
and build a URL without performing a transfer. They are left out of the
first release.

**Everything that passes or returns a structure by value.**
`curl_version_info` answers the address of a `curl_version_info_data`,
and reading its fields means knowing the layout of a structure that
grows with every release. `curl_version` answers the same facts as a
string and is here instead.
