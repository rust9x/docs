# Building with rust9x

## Build requirements

Have a look at the [sample application](https://github.com/rust9x/rust9x-sample) for an example of
how to set up a build environment for rust9x.

### Windows (Platform) SDK

The only parts required from the Platform SDK are the import libraries (and potentially headers for
C deps).

For the rust9x standard library, basically any version of the Platform SDK will work, as only APIs
available in all Windows versions are directly linked.

The two exceptions are: WinSock 2, which is required for networking, and `unicows.lib` (see below).

The last version of the Platform SDK that fully supports Windows 95 is 5.2 Build 3790 (Server 2003).

Tested versions:

Version | Path | Notes
------- | ---- | -------------------------
VC6 integrated | `VC98/Lib` | does not include `ws2_32.lib` or `unicows.lib`, need to additionally link that from another version
VC7.1 (2003) integrated | `Vc7/PlatformSDK/Lib` | works
VC8 (2005) integrated | `VC/PlatformSDK/Lib` | works
Platform SDK 5.2 3790 (Server 2003) | `Lib` | works
Windows 7 Platform SDK | `Microsoft SDKs/Windows/v7.1A/Lib` | works. also installed with the VS2017 Windows XP platform support
Enterprise WDK 25H2 | `Program Files/Windows Kits/10/Lib/10.0.26100.0/um/x86/` | works

### MSVC runtime

Unlike the Platform SDK, the choice of MSVC runtime is much more important. Your choice decides
which Windows versions are supported, and which additional features are available.

Of course, if you have any C or C++ dependencies that need modern APIs, you'll need to use an
appropriate version of the runtime.

Version | static (libcmt.lib) | dynamic (msvcrt.lib) | OS support | Panic unwinding | Path | Notes
------- | ------------------- | -------------------- | ---------- | --------------- | ---- | -------
[r9xrsrt] | yes | n/a | all | no | - | Minimal rust9x runtime, no C support, no unwinding, no float, needs std recompilation
VC 4.1 | no | ? | ? | ? | `MSDEV/Lib` | static results in a broken executable (broken import table), didn't test dynamic
VC 6 | yes | yes | 95+, NT 3.5+ | no | `VC98/lib` | no `/SAFESEH` support, dynamic links against `msvcrt.dll`, which is not present on all versions of Windows 95/NT
VC 7.1 (2003) | yes | yes | 95+, NT 4+, maybe earlier? | no | `Vc7/lib` | `rust-lld` fails linking debug information with this version, needs `/DEBUG:NONE`.
VC 8 (2005) | yes | yes | 98+, NT 4+ | yes | `VC/lib` | fully working
VC 9 (2008) | ? | ? | 2000+ | ? | | likely fully working
... | | | | | | likely fully working, with various minimum OS requirements
VC 2017 XP toolset | no* | yes | XP+ | yes | `BuildTools/VC/Tools/MSVC/14.16.27023/lib/x86` | dynamic linking is fine; static linking includes unsupported APIs and will fail to run on XP

[r9xrsrt]: https://github.com/rust9x/r9xrsrt

When linking dynamically, you of course have to make sure the target system has the appropriate
version of the runtime redistributable installed.

### Microsoft Layer for Unicode (aka MSLU, unicows)

If you don't want to target Windows 9x/Me, pass `-Zunicows=no` as a rustc flag to disable the MSLU
support.

If you like to target Windows 9x/Me, you will also need to link against the Microsoft Layer for
Unicode (MSLU), also known by its library name `unicows` 🐄.

The latest version of `unicows.lib` that fully supports Windows 95 is provided with the Windows
Platform SDK 5.2 Build 3790 (Server 2003).

Unicows works by injecting its static library before other libraries in the linker search logic.
rust9x automatically injects `unicows.lib` in the right places for any Rust usage.

See the link arg configuration in [rust9x-sample][r9x-sample-config-toml] for an example.

[r9x-sample-config-toml]: https://github.com/rust9x/rust9x-sample/blob/main/.cargo/config.toml

## Runtime requirements

Beyond the runtime libraries' requirements, the following additional requirements apply to rust9x
executables:

### Network APIs: WinSock 2

The network APIs require WinSock 2. WinSock 2 is supported on Windows 95+ and NT 4.0+.

Windows 95 RTM does not include WinSock 2 by default, but Microsoft released a WinSock 2 update
package for Windows 95. It is available for example on [WinWorldPC][win95winsock2].

[win95winsock2]: https://winworldpc.com/product/windows-95/patches

### MSLU/unicows DLL

Again, if you like to target Windows 9x/Me, you'll also need to provide the DLL `unicows.dll` with
your executable. The latest version of `unicows.dll`, 1.1.3790.0, is available [in the
redistributable][unicows-redist], courtesy of the Internet Archive.

[unicows-redist]:
    https://web.archive.org/web/20041210000000id_/https://download.microsoft.com/download/b/7/5/b75eace3-00e2-4aa0-9a6f-0b6882c71642/unicows.exe

Windows 95 needs the "net" packages installed (`mpr.dll`) as the Microsoft Layer for Unicode
requires this. Information about this and other known issues can be found [on this
website][unicows-issues].

[unicows-issues]:
    https://web.archive.org/web/20061110213523/http://www.trigeminal.com/usenet/usenet035.asp?00100001

### `dbghelp.dll` for backtrace support

If backtraces are enabled and you want symbolization to work, `dbghelp.dll` must be provided.
Generally the backtrace support in rust9x works with XP's `dbghelp.dll` version 5.1.2600.x.

A missing/non-working `dbghelp.dll` just disables backtrace support.

The "best"/most compatible version so far seems to be `6.5.3.7`. The [Debugging Tools for Windows
6.5] ship that version. Alternatively it can be found as part of an (updated?) VS2005 installation,
in the path `Common7\IDE\dbghelp.dll`.

[Debugging Tools for Windows 6.5]: https://web.archive.org/web/20130416014739/https://msdl.microsoft.com/download/symbols/debuggers/dbg_x86_6.5.3.8.exe

At least Windows 95+ and NT 4+ should be supported with a provided `dbghelp.dll`.
