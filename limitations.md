# Limitations

rust9x _only alters the standard library_.

Third-party crates that directly interact with the
Windows API may need additional changes. This is explicitly out of scope for rust9x, as is host
support for rust9x targets (running the rust toolchain itself on legacy Windows).

A lot of APIs have limitations on older systems - refer to old MSDN Libraries and various old KB
articles for more information. The VS2005 MSDN library is the last one to have information about
pre-2000 Windows versions and compatibility.

## Floating point intrinsics

Some of the floating point intrinsics have to be provided by the linked runtime. Microsoft didn't
bother implementing a lot of the C99 ones until 2013, so you'll have to do some [hacky workarounds],
linking parts of a modern runtime.

[hacky workarounds]: https://github.com/rust9x/rust/issues/36

Floating point support compiled for targets without SSE2 (i.e. by default, `i586` targets), is
[generally unsound](https://github.com/rust-lang/rust/issues/114479).

## Console support

Please note that while all NT-based Windows versions support Unicode, the `cmd.exe` and
`conhost.exe` of older Windows versions have a lot of limitations. It mostly behaves like the
[Legacy Console mode](https://learn.microsoft.com/en-us/windows/console/legacymode) in modern
Windows versions. As far as I can tell it does not support font fallback for international scripts
(e.g. chinese characters), so you'll still have to select an appropriate font and/or non-unicode
codepage to display characters correctly. This is unrelated to rust9x and a legacy console
limitation.

On Windows 9x/Me, even with `unicows.dll/lib` linked correctly, characters outside of the current
codepage are replaced by `?`.

## File handling

- For NT-based systems: Support for functions that need `SetFileInformationByHandle` and
`GetFileInformationByHandleEx` has been added since rust9x `1.84-beta-v3`.
- `std::fs::canonicalize` uses `GetFinalPathNameByHandleW`, which is only supported starting with
  Vista/Server 2008. Systems that don't support this function also don't support symlinks, so a
  fallback to `GetFullPathName` is used. Note that this _changes the documented behavior_ of this
  function (does not check for existence, does not convert into extended length syntax).
- `std::fs::soft_link` uses `CreateSymbolicLinkW` and will `Err` on systems before Vista/Server 2008.
- `std::fs::hard_link` uses `CreateHardLinkW` and will `Err` on systems before Windows 2000.
- `std::fs::File::set_permissions` will currently `Err` on systems before Vista/Server 2008.
  - The easy workaround is to use `std::fs::set_permissions` instead.
- 9x/ME: The directory removal implementation falls back to the old simple recursive one since the
  necessary APIs for the modern one are not available. See
  [#93112](https://github.com/rust-lang/rust/pull/93112) for a discussion of the implications of
  this.
- `OpenOptions`
  - 9x/Me only supports a limited number of access rights. This means that we can't use the special
    append-only behavior (atomic appends).
  - On 9x/Me and NT3.5 and before, files opened with `append(true)` _will_ be able to seek back and
    overwrite existing data. The fallback impl will only seek to the end of the file when opening.
  - 9x/Me does not support `FILE_SHARE_DELETE`. Opened files cannot be opened to request a delete at
    the same time (if that even existed back then).
- 9x/Me does not support `FILE_FLAG_BACKUP_SEMANTICS` and symbolic links, so `stat`/`fs::metatada`,
  `try_exists`, and `remove_dir_all` have various fallback implementations, working around the
  limitation of not being able to open folders as files.
  - `Dir::open`, `Dir::open_file` ([#120426](https://github.com/rust-lang/rust/issues/120426)):
    These will fail on 9x/Me, as those systems don't support the necessary
    `FILE_FLAG_BACKUP_SEMANTICS`.

## Processes/Environment

- TLS: The number of TLS indices is limited:
  - Windows 2000 and newer: 1088 indices per process
  - Windows Me/98: 80 indices per process
  - Windows NT and Windows 95: 64 indices per process
- TLS destructors do NOT run on 9x/Me. They work on _all_ NT-based systems.
- If `CompareStringOrdinal` is not available (before Vista/Server 2008) the impl falls back to old
  env arg behavior
  - See [#85270](https://github.com/rust-lang/rust/pull/85270) and
    [#87863](https://github.com/rust-lang/rust/pull/87863)
- Added raw attributes to `Command`s are skipped on systems that don't support them.
- Stack size: NT 3.51 (and likely below) is picky about the startup thread stack size (`/STACK`
  linker flag). 1 MiB is too big and will be silently ignored, instead falling back down to some
  tiny default, small enough to fail writing to the console, even. 128 KiB seems to be fine.

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

`Mutex`, `CondVar`, and the thread parker have fallback implementations for every windows version.

- `RwLock`, `Once` are implemented using the thread parker in a system-agnostic way.
- `CondVar` Limitation: the `CreateEvent`-based fallback implementation always wakes up all waiting
  threads.

### Randomness (used for `std::random::random` and `hash_map::RandomState`)

If `RtlGenRandom`/`SystemFunction036` (>= XP/Server 2003) is not available, rust9x will fall back to
`CryptGenRandom`, which is available on NT 4.0 / Windows 95 OSR2 / Windows 95 RTM with IE 3.02 and
higher.

If _that_ one isn't available, a pretty simple PRNG (`xoroshiro64**`) is used instead, seeded with a
stack address and `GetTickCount`.

## Other crates

### windows-rs, windows-sys, windows-link

If you don't need `unicows` support, any recent release of windows-rs should work.

For 9x/Me support, if you want to access any unicows-wrapped API, you will need a patched
`windows-link` crate, switching it from `raw-dylib` to regular linking.
