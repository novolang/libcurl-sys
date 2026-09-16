# libcurl-sys

libcurl is a C library that transfers data to and from a server named
by a URL. It speaks HTTP, HTTPS, FTP, SFTP, SMTP, IMAP and about twenty
more protocols, and it handles the redirects, the authentication, the
cookies, the proxies and the TLS along the way. The library is
documented in [the curl manual](https://curl.se/libcurl/c/). This
package declares twenty-seven of its entry points to novo-lang, one
declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libcurl. The package contains no logic of
its own, and it does nothing without the C library installed. The
twenty-seven entry points are the ones a program needs to create a
handle, configure a transfer, run it, read the result and clean up; the
section "What is not included" says what a program still cannot do with
them alone.

## What it is

An **easy handle** is one transfer and the settings that describe it.
`curl_easy_init` creates one, `curl_easy_setopt` configures it,
`curl_easy_perform` runs it and waits for it to finish, and
`curl_easy_cleanup` releases it. A handle can be run again after a
transfer, and it keeps the connection open for the next one.

An **option** is one setting on a handle, named by a number. The URL,
the timeout, whether to follow a redirect and which certificate
authority file to trust are all options.

An **item of information** is one fact about a finished transfer, also
named by a number. The HTTP status, the final URL after redirects, the
number of bytes downloaded and the elapsed time are items of
information.

A **result code** is the value every call answers. Zero is `CURLE_OK`
and everything else is a failure that `curl_easy_strerror` describes. A
server that answers "404 Not Found" is a successful transfer: the
status is read back as an item of information, not as a result code.

A **string list** is libcurl's own linked list of strings, built with
`curl_slist_append`. It is how a program passes a set of HTTP request
headers.

libcurl writes a response body wherever the program tells it to. With
no instruction it writes to standard output, which is its documented
default.

## Install

```
novo pkg add libcurl-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its headers come from the system package
`libcurl4-openssl-dev`:

```
sudo apt install libcurl4-openssl-dev
```

On macOS libcurl ships with the operating system, and the Homebrew
formula is `curl`. On other systems curl builds from its own source.

## Example

Fetch a page and read the status back. The body goes to standard
output, which is libcurl's default with no write callback set:

```novo ignore
use libcurl

fn main() [io, ffi]
    let handle = libcurl.curl_easy_init()
    // 10002 is CURLOPT_URL.
    let _ = libcurl.curl_easy_setopt_str(handle, 10002, "https://example.com/")
    // 52 is CURLOPT_FOLLOWLOCATION and 13 is CURLOPT_TIMEOUT, in seconds.
    let _ = libcurl.curl_easy_setopt_long(handle, 52, 1)
    let _ = libcurl.curl_easy_setopt_long(handle, 13, 30)
    // One request header, in libcurl's own linked list.
    let headers = libcurl.curl_slist_append(0, "Accept: text/html")
    // 10023 is CURLOPT_HTTPHEADER.
    let _ = libcurl.curl_easy_setopt_ptr(handle, 10023, headers)

    let rc = libcurl.curl_easy_perform(handle)
    if rc != 0
        println("transfer failed: " + ptr.read_str(libcurl.curl_easy_strerror(rc)))
    else
        let slot = ptr.alloc_word()
        // 2097154 is CURLINFO_RESPONSE_CODE, a `long` item.
        let _ = libcurl.curl_easy_getinfo_long(handle, 2097154, slot)
        println("status ${ptr.read_word(slot)}")
        // 1048577 is CURLINFO_EFFECTIVE_URL, a string item.
        let _ = libcurl.curl_easy_getinfo_str(handle, 1048577, slot)
        println("final url " + ptr.read_str(ptr.read_word(slot)))
        ptr.free(slot)

    libcurl.curl_slist_free_all(headers)
    libcurl.curl_easy_cleanup(handle)
```

The example is fenced as an illustration rather than a compiled block
because it reaches a server over the network, which this repository
does not do.

## What the package contains

| Module | Contents |
| --- | --- |
| `libcurl` | Every entry point, in six groups: the library, the easy handle, the options, the transfer, the information, and the helpers. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Library | 4 | Initialises libcurl, reports its version and frees what it allocated for the caller. |
| Easy handle | 4 | Creates, copies, resets and releases the object one transfer is described by. |
| Options | 4 | Sets one option, in the four argument kinds an option can take. |
| Transfer | 5 | Runs the transfer, moves raw bytes on a connect-only handle, and pauses or resumes. |
| Information | 5 | Reads one item of information in four kinds, and names a result code. |
| Helpers | 5 | Builds and releases a header list, escapes and unescapes a URL, and parses a date. |

## How to choose an entry point

**The setopt family.** Four declarations, one symbol. The option number
says which one to use, and the "Option numbers" table below gives the
ranges: below 10000 is `curl_easy_setopt_long`, 10000 to 19999 is
`_str` for a string and `_ptr` for any other address, and 30000 and
above is `_off_t`.

**The getinfo family.** Four declarations, one symbol, and the
information number says which. The "Information numbers" table gives
the ranges.

**`curl_easy_perform` or `curl_easy_send` and `curl_easy_recv`.**
`curl_easy_perform` runs a whole protocol transfer. The send and
receive pair move raw bytes over a connection opened with
`CURLOPT_CONNECT_ONLY`, for a program that speaks the protocol itself.

**`curl_easy_reset` or a new handle.** `curl_easy_reset` forgets every
option but keeps the open connections and the cookies, which is the
cheaper way to make a second, unrelated request to the same server.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
2. **An entry point that answers a C `int` answers it in 32 bits.**
   Write `as i32` before comparing the answer with a negative number.
   Bind the cast to a local first: `novo fmt` removes the parentheses
   that would otherwise group it.
3. **The setopt and getinfo numbers are not checked.** In C they are
   variadic calls, so the compiler cannot check the argument's type
   either. Passing an option number to the wrong declaration of the
   family is undefined behaviour, and neither libcurl nor novo-lang
   will report it. curl manual, `curl_easy_setopt`.
4. **A failed HTTP status is not a failed transfer.**
   `curl_easy_perform` answers 0 for a 404 and for a 500. Read
   `CURLINFO_RESPONSE_CODE` to see the status. curl manual,
   `curl_easy_getinfo`.
5. **A string set through `curl_easy_setopt_str` is copied.** libcurl
   7.17.0 and later copy the bytes before the call returns, so the
   novo-lang string need not outlive it. A *pointer* set through
   `curl_easy_setopt_ptr` is not copied, and what it points at must
   outlive the transfer.
6. **`curl_slist_append` answers the list.** It does not modify the
   caller's variable. Assigning its answer back over the old head is
   the only correct use; discarding it leaks the list.
7. **A `curl_slist` node is two words the caller reads by hand.** Both
   `curl_slist_append` and the pointer items of `curl_easy_getinfo_ptr`
   answer the address of one.

   | Offset | Meaning |
   | --- | --- |
   | 0 | the address of this node's string |
   | 8 | the address of the next node, or 0 at the end |

   `ptr.read_word` walks it and `ptr.read_str` reads each string. The
   strings belong to the list, so `curl_slist_free_all` on the head
   releases all of them.
8. **The header list must outlive the transfer.** libcurl reads it when
   `curl_easy_perform` runs, not when the option is set. Release it
   after the transfer finishes.
9. **`curl_easy_escape` and `curl_easy_unescape` allocate.** Release
   the answer with `curl_free`, not with `ptr.free`.
10. **Unescaped bytes may contain a NUL.** `curl_easy_unescape` reports
    the decoded length in a slot for that reason. `ptr.read_str` stops
    at the first NUL; `ptr.read_bytes_n` with the reported length does
    not.
11. **With no write callback the body goes to standard output.** That
    is libcurl's documented default, and this package carries no way to
    change it. curl manual, `CURLOPT_WRITEFUNCTION`.
12. **A pointer item does not always hand over ownership.** Whether the
    caller releases what `curl_easy_getinfo_ptr` wrote depends on the
    item: `CURLINFO_COOKIELIST` is the caller's to free with
    `curl_slist_free_all`, and the reference says which of the others
    are. curl manual, `curl_easy_getinfo`.
13. **`curl_global_cleanup` ends the library.** No entry point here may
    be called after it until `curl_global_init` runs again, and every
    handle must be released first.

### Option numbers

The number encodes the argument's type. These are the options a first
program needs.

| Option | Number | Declaration |
| --- | --- | --- |
| `CURLOPT_PORT` | 3 | `curl_easy_setopt_long` |
| `CURLOPT_TIMEOUT` | 13 | `curl_easy_setopt_long` |
| `CURLOPT_VERBOSE` | 41 | `curl_easy_setopt_long` |
| `CURLOPT_NOBODY` | 44 | `curl_easy_setopt_long` |
| `CURLOPT_POST` | 47 | `curl_easy_setopt_long` |
| `CURLOPT_FOLLOWLOCATION` | 52 | `curl_easy_setopt_long` |
| `CURLOPT_SSL_VERIFYPEER` | 64 | `curl_easy_setopt_long` |
| `CURLOPT_MAXREDIRS` | 68 | `curl_easy_setopt_long` |
| `CURLOPT_CONNECTTIMEOUT` | 78 | `curl_easy_setopt_long` |
| `CURLOPT_SSL_VERIFYHOST` | 81 | `curl_easy_setopt_long` |
| `CURLOPT_TIMEOUT_MS` | 155 | `curl_easy_setopt_long` |
| `CURLOPT_URL` | 10002 | `curl_easy_setopt_str` |
| `CURLOPT_PROXY` | 10004 | `curl_easy_setopt_str` |
| `CURLOPT_USERPWD` | 10005 | `curl_easy_setopt_str` |
| `CURLOPT_USERAGENT` | 10018 | `curl_easy_setopt_str` |
| `CURLOPT_CUSTOMREQUEST` | 10036 | `curl_easy_setopt_str` |
| `CURLOPT_CAINFO` | 10065 | `curl_easy_setopt_str` |
| `CURLOPT_ACCEPT_ENCODING` | 10102 | `curl_easy_setopt_str` |
| `CURLOPT_COOKIELIST` | 10135 | `curl_easy_setopt_str` |
| `CURLOPT_WRITEDATA` | 10001 | `curl_easy_setopt_ptr` |
| `CURLOPT_ERRORBUFFER` | 10010 | `curl_easy_setopt_ptr` |
| `CURLOPT_POSTFIELDS` | 10015 | `curl_easy_setopt_ptr` |
| `CURLOPT_HTTPHEADER` | 10023 | `curl_easy_setopt_ptr` |
| `CURLOPT_POSTFIELDSIZE_LARGE` | 30115 | `curl_easy_setopt_off_t` |

### Information numbers

| Item | Number | Declaration |
| --- | --- | --- |
| `CURLINFO_EFFECTIVE_URL` | 1048577 | `curl_easy_getinfo_str` |
| `CURLINFO_CONTENT_TYPE` | 1048594 | `curl_easy_getinfo_str` |
| `CURLINFO_REDIRECT_URL` | 1048607 | `curl_easy_getinfo_str` |
| `CURLINFO_PRIMARY_IP` | 1048608 | `curl_easy_getinfo_str` |
| `CURLINFO_SCHEME` | 1048625 | `curl_easy_getinfo_str` |
| `CURLINFO_RESPONSE_CODE` | 2097154 | `curl_easy_getinfo_long` |
| `CURLINFO_REDIRECT_COUNT` | 2097172 | `curl_easy_getinfo_long` |
| `CURLINFO_PRIMARY_PORT` | 2097192 | `curl_easy_getinfo_long` |
| `CURLINFO_HTTP_VERSION` | 2097198 | `curl_easy_getinfo_long` |
| `CURLINFO_SSL_ENGINES` | 4194331 | `curl_easy_getinfo_ptr` |
| `CURLINFO_COOKIELIST` | 4194332 | `curl_easy_getinfo_ptr` |
| `CURLINFO_SIZE_DOWNLOAD_T` | 6291464 | `curl_easy_getinfo_off_t` |
| `CURLINFO_SPEED_DOWNLOAD_T` | 6291465 | `curl_easy_getinfo_off_t` |
| `CURLINFO_TOTAL_TIME_T` | 6291506 | `curl_easy_getinfo_off_t` |

## What is not included

- **Every entry point that takes a C function pointer.** The write, read,
  header, progress, seek, debug and SSL context callbacks are absent,
  because a novo-lang function is not a C function pointer. With no
  write callback libcurl writes the body to standard output.
- **Every entry point that passes or returns a structure by value.** The
  novo-lang foreign function interface passes integers, floats,
  strings and pointers, and nothing else. `curl_version_info` answers
  the address of a `curl_version_info_data` whose layout grows with
  every release, so reading its fields is not something a binding can
  offer safely; `curl_version` answers the same facts as a string and
  is here.
- **The multi interface.** `curl_multi_init`, `curl_multi_perform`,
  `curl_multi_poll` and their neighbours drive many transfers from one
  thread. They are left out of the first release.
- **The share interface.** `curl_share_init` and its neighbours let
  several handles share a cookie jar, a DNS cache and a connection
  cache. They are left out of the first release.
- **The mime interface.** `curl_mime_init`, `curl_mime_addpart` and
  their neighbours build a multipart form body. They are left out of
  the first release; a program that posts one builds the bytes itself
  and sets `CURLOPT_POSTFIELDS`.
- **The URL API.** `curl_url`, `curl_url_set` and `curl_url_get` parse
  and build a URL with no transfer. They are left out of the first
  release; `url-nv` does the same job in novo-lang.
- **The option and information names.** libcurl spells them as C
  macros and enumerators, which are not symbols a binding can resolve.
  The numbers are in the two tables above.

## Related packages

The native counterpart is **`std.http`**, the standard library's own
HTTP client, and it already ships. A program that makes an HTTP request
uses `std.http` and links no C library. Choose this package instead
when the program needs a protocol `std.http` does not speak, such as
FTP, SFTP or IMAP, or when it must reuse curl's own proxy
configuration, cookie jar or certificate trust store.

`url-nv` parses and builds URLs in novo-lang, which is what libcurl's
URL API does and this package leaves out.

`libssl-sys` binds OpenSSL's protocol library. A libcurl built against
OpenSSL already links it, and a program that needs to reach into the
TLS session directly takes that package as well.

## Tests

`tests/libcurl_tests.nv` holds thirteen tests written against the
signatures. They call the C library, so `novo test` needs libcurl
installed and linkable:

```
novo test tests/libcurl_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

No test opens a network connection. The suite asserts that the library
names its own version, that a result code has a description, that a
handle is created, copied, reset and released, that each of the four
option kinds is accepted and each of the four information kinds reads
back, that an option number this libcurl does not define answers
`CURLE_UNKNOWN_OPTION`, that a pointer item answers the cookie list a
seeded handle holds and that the caller is the one who frees it, that a
header list grows and is released, that `a b&c` escapes to `a%20b%26c`
and back, that an HTTP date parses to the second the reference gives
it, that a transfer with no URL fails with `CURLE_URL_MALFORMAT` before
any socket is opened, and that `curl_easy_upkeep` succeeds on a handle
with nothing to keep alive while `curl_easy_send` and `curl_easy_recv`
answer `CURLE_UNSUPPORTED_PROTOCOL` without a connect-only connection.

**`novo test` exits 23 on this suite even when every assertion passes.**
`ptr.read_str` copies a string the C library owns, and the toolchain's
default leak check counts every one of those copies as an object the
test leaked: the stdlib declares the call as returning nothing to
release and the runtime allocates anyway. The exit code is the leak
check's, not an assertion's; the output above it says how many
assertions passed. Running with `NOVO_LEAK_CHECK=0` set, or
`novo test --no-leak-check`, exits 0. It is a toolchain defect and it is
filed as one; nothing in this package can close it.

## Implementation status

| Group | State |
| --- | --- |
| Library | Complete. |
| Easy handle | Complete. |
| Options | Complete for the four argument kinds. The callback options are absent. |
| Transfer | Complete for a blocking transfer and for connect-only bytes. |
| Information | Complete for the four value kinds. |
| Helpers | Complete. |
| Callbacks | Absent. Every one takes a C function pointer. |
| Multi interface | Absent. Left out of the first release. |
| Share interface | Absent. Left out of the first release. |
| Mime interface | Absent. Left out of the first release. |
| URL API | Absent. Left out of the first release. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

curl itself is distributed under a licence derived from the MIT
licence, and installing it is the reader's own step.
