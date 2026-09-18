# Changelog

All notable changes to libcurl-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

### Corrected against the libcurl reference

- `CURLOPT_POSTFIELDSIZE_LARGE` is 30120. 30115 is
  `CURLOPT_INFILESIZE_LARGE`.
- `curl_easy_pause` takes 0 `CURLPAUSE_CONT`, 1 `CURLPAUSE_RECV`, 4
  `CURLPAUSE_SEND` and 5 `CURLPAUSE_ALL`.
- The options that take a `curl_off_t` run from 30000 to 39999.
  `CURLOPTTYPE_BLOB` begins at 40000.
- A string from `curl_easy_getinfo` is valid until
  `curl_easy_cleanup` releases the handle. A further transfer may
  replace what it points at.
- `curl_easy_init` is what initialises the library implicitly.
  libcurl 7.84 made `curl_global_init` thread-safe where
  `curl_version_info` reports `CURL_VERSION_THREADSAFE`.
- `curl_easy_reset` keeps the name resolution cache as well as the
  other state named there.
- The suite's constant for `CURLOPT_POSTFIELDSIZE_LARGE` held 30115,
  so it named one option and exercised another. It now holds 30120.

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
- `tests/libcurl_tests.nv` — thirteen tests over the twenty-seven entry
  points. They call the C library, so they need libcurl installed. No
  test opens a network connection. The two that reach the transfer
  group assert what `curl_easy_upkeep`, `curl_easy_send` and
  `curl_easy_recv` answer before any connection exists, and the pointer
  item is read through a cookie the suite seeds itself. All thirteen
  pass.

### The variadic calls are declared once per argument kind

`curl_easy_setopt` and `curl_easy_getinfo` are variadic in C. The type
of the third argument is decided by the option or information number,
and the compiler cannot check it. A variadic call has no single
novo-lang signature, so each argument kind gets its own declaration and
all of them resolve to the same symbol: `curl_easy_setopt_long`,
`_str`, `_ptr` and `_off_t`, and `curl_easy_getinfo_long`, `_str`,
`_off_t` and `_ptr`. The caller picks the declaration that matches the
number, and the README's tables say which number is which kind.

This is the shape libssl-sys uses for `SSL_ctrl`, with one difference.
`SSL_ctrl` is a fixed-arity C function that the OpenSSL headers wrap in
macros, so one declaration covers it. `curl_easy_setopt` is variadic
itself, so one declaration cannot.

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
