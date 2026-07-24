<img width="640" alt="rust9x logo" src="https://user-images.githubusercontent.com/5844066/164355665-eef4ba70-0c6d-47c2-b520-e0a89b56742a.svg">

_Blazingly fast! Y2k compliant! Works everywhere!_

----------------------------------------------------------------------------------------------------

rust9x brings the Rust standard library to legacy 32-bit Windows versions (9x/Me/NT/2000/XP/Vista).
Note that this only includes these systems as _target_.

[Sample application](https://github.com/rust9x/rust9x-sample)

Depending on how far back you'd like to go, there are **[lots of limitations][limitations]** to keep
in mind. Please have a read!

Please note that this is not stable software at all. Do not expect all rust crates to work without
issues on Windows 95! In particular, _only_ the standard library is ported. Crates that directly use
Windows APIs _will_ need to be patched.

Only minimal testing has been done, but please feel free to file
[issues](https://github.com/rust9x/rust/issues) or PRs if you're crazy enough to test this :)

## Installation

### Compiling rust9x yourself

[How to build and run the compiler][rustc-dev-guide]

NOTE: There currently is a bug with the VS 2026 toolchain. Make sure to use VS 2022 for the time
being: [Github Issue](https://github.com/rust-lang/rust/issues/150280#issuecomment-3695099864)

1. Clone this repo and make sure that the `rust9x` branch is checked out.
2. Copy `bootstrap.rust9x.toml` to `bootstrap.toml`, edit to suit your needs. Or, if you just want
   to build it, pass `--config bootstrap.rust9x.toml` to the `x.py` calls below
3. `python x.py build` and grab a tea or coffee.
   - If you're changing only the stdlib, you can use `python x.py build -i library/std --stage 1
     --keep-stage 0` in future builds to only rebuild the standard library.
4. (Only once) Link the built toolchain to the name `rust9x`, e.g. for a stage 1 build: `rustup
   toolchain link rust9x D:\your\path\to\rust9x\rust\build\x86_64-pc-windows-msvc\stage1`
5. You can now use `cargo +rust9x <command>`/`rustc +rust9x <...>` to use the new toolchain with the
   new targets:
   - `i586-rust9x-windows-msvc`: default target-cpu: `pentium`
   - `i686-rust9x-windows-msvc`: default target-cpu: `pentium4` (SSE2)
   - `x86_64-rust9x-windows-msvc`: should work for x64 builds of Windows XP and up (untested)
   - Alternatively, you can `rustup override set rust9x` in your workspace folder. You might need to
     build the other tooling (rust-analyzer, rustdoc, clippy, ...) or set up your workflow to
     instead use another toolchain's tools, e.g. in the VSCode config for the rust-analyzer
     extension.
6. Have a look at how the [sample application](https://github.com/rust9x/rust9x-sample) configures
   search paths and linker arguments to bring everything together (see `Cargo.toml` and
   `.cargo/config.toml`)

## Requirements

In general, any platform libraries/SDKs that has `.lib` files that are compatible with the modern
MSVC toolsets' linkers should be fine. Static linking should always work, dynamic linking might need
additional steps.

Note that **panic unwinding needs at least the VC8** (Visual C++ 2005) libraries.

The VC7.1 toolset (Visual C++ 2003) is the last version to officially support Windows 95. Visual C++
6.0 SP6 also works fine after adding `unicows.lib` from a later toolset/platform SDK.

### Building programs for 9x/ME _and_ NT-based systems

As Rust follows Unicode Everywhere, a compatible `unicows.lib` has to be linked in order to enable
support for `W`ide APIs on these systems. See the [sample
application](https://github.com/rust9x/rust9x-sample) on how to do this correctly - the `README.md`
 and the `.cargo/config.toml` all have information/comments inside!

The last `unicows.lib` version that officially supports Windows 95 is from "Microsoft Platform SDK
February 2003" (download link [on Wikipedia](https://en.wikipedia.org/wiki/Microsoft_Windows_SDK))

### Building programs for NT-based systems only

Same thing as 9x/ME, but you can skip the lines regarding `unicows.lib`.

## Runtime requirements

A rust9x executable has the following minimum requirements:

- Windows 95 or Windows NT 3.51*
  - \* I've only tested Windows 95 and up, and NT 3.51 and up, though there should not be any static
    API dependencies left to prevent earlier NT versions from running. Just make sure to lower the
    subsystem version to `3.10` (**not `3.1`**) if you'd like to target NT 3.1 or 3.5.
  - Windows 95 needs the "net" packages installed (`mpr.dll`) as the Microsoft Layer for Unicode
    [requires
    this](https://web.archive.org/web/20061110213523/http://www.trigeminal.com/usenet/usenet035.asp?00100001).
  - Make sure to link against an MSVC toolset that is compatible with your target system. If you're
    linking the CRT dynamically, keep in mind that you might have to provide the runtime DLLs as
    well.
- If network API is used: Windows 95 with the [WinSock 2
  update](https://winworldpc.com/product/windows-95/patches) or Windows NT 4.0 (Microsoft never
  released WS2 for 3.51 or earlier)
- For 9x/ME: As Rust follows Unicode Everywhere, the Microsoft Layer for Unicode runtime DLL
  `unicows.dll` should be supplied alongside the executable.
- For backtrace support, `dbghelp.dll` must be provided (see [limitations])

While the executable and Rust standard library mostly work fine, there are some APIs are not
available on older Windows versions - see the [limitations] page for all the details.

## rust9x-sample in action (rust9x-1.61.0-beta)

All these run _the same_ binary (except NT3.51). The non-network binary runs on all of them, of
course.

### Windows NT Workstation 3.51 (VM, with `network` feature disabled as there is no WinSock2 available)

[<img width="640" alt="win351"
src="https://user-images.githubusercontent.com/5844066/164351469-86433cda-6fc0-4e7b-b75e-b42cd727935d.png">](https://user-images.githubusercontent.com/5844066/164351469-86433cda-6fc0-4e7b-b75e-b42cd727935d.png)

### Windows 95 B (real hardware, my Pentium MMX 233MHz machine <3)

[<img width="640" alt="win95"
src="https://user-images.githubusercontent.com/5844066/164351479-ac7bf75d-d6fe-4897-8be0-bab726c24906.png">](https://user-images.githubusercontent.com/5844066/164351479-ac7bf75d-d6fe-4897-8be0-bab726c24906.png)

### Windows XP SP3 (real hardware)

[<img width="640" alt="winxp"
src="https://user-images.githubusercontent.com/5844066/164351581-61f0cea0-ba94-49f6-b5eb-1f07412659b2.PNG">](https://user-images.githubusercontent.com/5844066/164351581-61f0cea0-ba94-49f6-b5eb-1f07412659b2.PNG)

### Windows 11 Pro Insider (real hardware)

[<img width="640" alt="win11"
src="https://user-images.githubusercontent.com/5844066/164351670-3d13e58c-fde6-40e8-b5fb-22a6aabb9f86.png">](https://user-images.githubusercontent.com/5844066/164351670-3d13e58c-fde6-40e8-b5fb-22a6aabb9f86.png)

[rustc-dev-guide]: https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html
[limitations]: limitations.md