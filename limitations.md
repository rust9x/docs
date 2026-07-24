rust9x only alters the standard library. Third-party crates that directly interact with the Windows
API may need additional changes.

A lot of APIs have limitations on older systems - refer to old MSDN Libraries (e.g. from VS 2005,
the last one to have full information about 9x/ME compatibility and proper system requirements per
API) and various old KB articles for more information.

## Compiling, Linking, Runtime

In order to support linking the [Microsoft Layer for
Unicode](https://en.wikipedia.org/wiki/Microsoft_Layer_for_Unicode) (also known as
`unicows.lib/.dll`) correctly, that means before supported/wrapped system libraries, the code that
emits linker arguments is modified **not** to emit native libraries that are supported by Unicows.
You'll have to emit those yourself, after `unicows.lib`.

Note that `raw-dylib`-linked imports [cannot be
overridden](https://github.com/rust9x/rust/issues/15) this way at the moment.

List of skipped libraries:

```plain
kernel32
advapi32
user32
gdi32
shell32
comdlg32
version
mpr
rasapi32
winmm
winspool
vfw32
secur32
oleacc
oledlg
sensapi
```

See the rust9x sample project for a possible setup of this.

### Floating point intrinsics

FP intrinsics have to be provided by the linked runtime. MS didn't bother implementing a lot of the
C99 ones until 2013, so there's [hacky workarounds to be
done](https://github.com/rust9x/rust/issues/36), linking modern parts of the MSVC runtime.

### Console support

#### Windows NT-based systems

Please note that while all NT-based Windows versions support Unicode, the `cmd.exe` and
`conhost.exe` of older Windows versions have a lot of limitations. It mostly behaves like the
[Legacy Console mode](https://learn.microsoft.com/en-us/windows/console/legacymode) in modern
Windows versions.

As far as I can tell it does not support font fallback for international scripts (e.g. chinese
characters), so you'll still have to select an appropriate font and/or non-unicode codepage to
display characters correctly. This is unrelated to rust9x and a legacy console limitation.

#### Windows 9x/ME

Even with unicows.dll/lib linked correctly, characters outside of the current codepage are replaced
by '?'.

### Panic handling

- Panic unwinding only works when linking against the VC8 (Visual C++ 2005) toolset or newer. Note
  that this toolset raises the minimum supported Windows version to Windows 98 (unclear about the
  minimum NT version, but NT 4.0 at the very least, because of the `IsDebuggerPresent` call). For
  pre-98 support you have to build with `panic = "abort"`.
- Backtraces are available. Backtrace support needs `dbghelp.dll`. On pre-XP systems you have to
  provide it yourself. A missing `dbghelp.dll` just silently disables backtraces. An old/unsupported
  one might crash, though.
  - "Best"/most compatible version so far seems to be `6.5.3.7` (part of an updated VS2005
    installation, in the path `Common7\IDE\dbghelp.dll`).
  - The [Debugging Tools for Windows
    6.5](https://web.archive.org/web/20130416014739/https://msdl.microsoft.com/download/symbols/debuggers/dbg_x86_6.5.3.8.exe)
    also ship that version.
  - Basic support for Windows XP's built-in dbghelp.dll `5.1.2600.x` was added as well, removing the
    need to ship it for Windows XP.
    - There _are_ slight differences in how some of the symbols are displayed, so using a newer
      version is still recommended.

## Standard library

### File handling

For NT-based systems: Support for functions that need `SetFileInformationByHandle` and
`GetFileInformationByHandleEx` has been added since
[rust9x-1.84-beta-v3](https://github.com/rust9x/rust/releases/tag/rust9x-1.84-beta-v3).

- `std::fs::canonicalize` uses `GetFinalPathNameByHandleW`, which is only supported starting with
  Vista/Server 2008. Systems that don't support this function also don't support symlinks, so a
  fallback to `GetFullPathName` is used. Note that this **changes the documented behavior** of this
  function (does not check for existence, does not convert into extended length syntax).
- `std::fs::soft_link` uses `CreateSymbolicLinkW` and will `Err` on systems before Vista/Server
  2008.
- `std::fs::hard_link` uses `CreateHardLinkW` and will `Err` on systems before Windows 2000.
- `std::fs::File::set_permissions` will currently `Err` on systems before Vista/Server 2008.
  - The easy workaround is to use `std::fs::set_permissions` instead.
- 9x/ME: The directory removal implementation falls back to the old simple recursive one since the
  necessary APIs for the modern one are not available.
  - **This might cause security issues**, see https://github.com/rust-lang/rust/pull/93112
- `OpenOptions`:
  - 9x/ME only supports a limited number of access rights. This means that we can't use the special
    append-only behavior (atomic appends). Additionally, files opened with `append(true)` *will* be
    able to seek back and overwrite existing data on these systems.
    - The fallback impl will seek to the end of the file on open.
  - 9x/ME does not support `FILE_SHARE_DELETE`. Opened files cannot be opened to request a delete at
    the same time (if that even existed back then).
- 9x/ME does not support `FILE_FLAG_BACKUP_SEMANTICS` and symbolic links, so `stat`/`fs::metatada`,
  `try_exists`, and `remove_dir_all` have various fallback implementations, working around the
  limitation of not being able to open folders as files.
- `Dir::open`, `Dir::open_file` ([nightly](https://github.com/rust-lang/rust/issues/120426)): These
  will fail on 9x/ME, as those systems don't support the necessary `FILE_FLAG_BACKUP_SEMANTICS`.

### Processes/Environment

- TLS: The number of TLS indices is limited:
  - Windows 2000 and newer: 1088 indices per process
  - Windows Me/98: 80 indices per process
  - Windows NT and Windows 95: 64 indices per process
- TLS destructors do NOT run on 9x/Me. They are tested to be working on NT4 RTM and newer.
- TLS on Windows 95 RTM: The loader only aligns to 4-byte-boundaries
- If `CompareStringOrdinal` is not available (before Vista/Server 2008) the impl falls back to old
  env arg behavior
  - See https://github.com/rust-lang/rust/pull/85270 and
    https://github.com/rust-lang/rust/pull/87863
- `FreeEnvironmentStringsW` doesn't exist on NT 3.1, so `env::vars` and `env::vars_os` leak the
  OS-allocated buffers.
- Added raw attributes to `Command`s are skipped on systems that don't support them.
- Stack size: NT 3.51 (and likely below) is picky about the stack size (`/STACK` linker flag). 1 MiB
  is too big and will be silently ignored, instead falling back down to some default.

### Networking

- As WinSock 2 is not supported on NT 3.51 and below, networking is entirely unsupported on those
  systems.
  - Socket handles would be inherited on NT versions before 3.51, as neither
    `WSA_FLAG_NO_HANDLE_INHERIT` nor `SetHandleInformation` are supported.
  - On 9X/ME, socket handles are [non-inheritable by
      default](https://www.betaarchive.com/wiki/index.php/Microsoft_KB_Archive/150523#MORE_INFORMATION).
- `SockAddr` resolution is limited to IPv4 on systems older than Windows XP/Server 2003. If you have
  installed the "IPv6 Technology Preview for Windows 2000", it should also be supported
  (`wship6.dll`; untested).

### Synchronization

`Mutex`, `CondVar` and `RwLock`, `Once`, and the threadparker have fallback implementations for
every windows version.

- `CondVar` Limitation: the `CreateEvent`-based fallback implementation always wakes up all waiting
  threads.
  - **SAFETY LIMITATION**: The `CreateEvent`-based fallback implementation uses `PulseEvent`, which
    may cause deadlocks. See [Old New
    Thing](https://devblogs.microsoft.com/oldnewthing/20050105-00/?p=36803) and the [MSDN
    docs](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-pulseevent) for more
    information.
    - If anyone can think of how to improve the fallback implementation while still being compatible
      with all mutex fallbacks on their respective systems, please reach out/create an issue!

### Other

- `hash_map::RandomState` initialization: If `RtlGenRandom`/`SystemFunction036` (>= XP/Server 2003)
  is not available, rust9x will fall back to `CryptGenRandom`, which is available on NT 4.0 /
  Windows 95 OSR2 / Windows 95 RTM with IE 3.02 and higher. If that one isn't available, a pretty
  simple PRNG (`xoroshiro64**`) is used instead, seeded with a stack address and `GetTickCount`.

## Other crates

### windows-rs, windows-sys

If you don't need `unicows` support, the official release of windows-rs 0.53.0 or higher should
work.

For 9x/Me support you *will* need a `-9xme` branch. It does not automatically link any additional
input libraries (esp. the "god import" lib shipped with windows-rs). This allows you to link
`unicows.lib` before any other libraries, but means that you have to specify all required libraries
via `link-args` yourself.

Check out the fork and different branches of the `windows-rs` repo here:
https://github.com/rust9x/windows-rs/branches
