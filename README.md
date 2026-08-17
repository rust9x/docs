<img width="640" alt="rust9x logo"
src="https://user-images.githubusercontent.com/5844066/164355665-eef4ba70-0c6d-47c2-b520-e0a89b56742a.svg">

_Blazingly fast! Y2k compliant! Works everywhere!_

---

rust9x brings the Rust standard library to legacy Windows versions (9x, Me, NT3.1 SP2+). Note that
this only includes these systems as _target_.

[rust9x-sample: Sample application](https://github.com/rust9x/rust9x-sample)

Depending on how far back you'd like to go, there are lots of **[limitations]** to keep in mind.

Please note that this is not stable software at all, and does not claim to ever be. _Only_ the
standard library is ported. Crates that directly use Windows APIs _will_ need to be patched.

## Precompiled toolchain

Starting from `rust9x-1.98.0-beta-v4`, you can download precompiled toolchains from the [releases
page](https://github.com/rust9x/rust/releases).

Just download the file for your platform, extract it, and link it via `rustup`:

```shell
rustup toolchain link rust9x /path/to/extracted/rust9x
```

The folder to specify is the one that contains `bin`, `lib`, etc. Uninstall via `rustup toolchain
remove rust9x`.

Afterwards, you can use `+rust9x` after a tool name (e.g. `cargo +rust9x build`) to use the new
toolchain, or you can run `rustup override set rust9x` in a project folder to set it as default in
that folder.

For Linux, additionally see how to [build on Linux with MSVC targets](linux-msvc.md)
for how to set up the build environment.

## Compiling rust9x yourself

[How to build and run the compiler][rustc-dev-guide]

[Building on Linux with MSVC targets: Rust and rust9x](linux-msvc.md)

[rustc-dev-guide]: https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html

NOTE: There currently is a bug with the VS 2026 toolchain. Make sure to use VS 2022 for the time
being: [Github Issue](https://github.com/rust-lang/rust/issues/150280#issuecomment-3695099864)

1. Clone this repo and make sure that your preferred version's branch is checked out.
2. Copy `bootstrap.rust9x.toml` to `bootstrap.toml`, edit to suit your needs. Or, if you just want
   to build it, pass `--config bootstrap.rust9x.toml` to the `x.py` calls below
3. `python x.py build --stage 1`
4. (Only once) Link the built toolchain to the name `rust9x`, e.g. for a stage 1 build:

   `rustup toolchain link rust9x D:\your\path\to\rust9x\rust\build\x86_64-pc-windows-msvc\stage1`

5. You can now use `cargo +rust9x <command>`/`rustc +rust9x <...>` to use the new toolchain with the
   new targets:
   - `i586-rust9x-windows-msvc`: default target-cpu: `pentium` (see [limitations] for floating point
     support)
   - `i686-rust9x-windows-msvc`: default target-cpu: `pentium4` (SSE2)
   - `x86_64-rust9x-windows-msvc`: should work for x64 builds of Windows XP and up (untested)
   - Alternatively, you can `rustup override set rust9x` in your workspace folder. You might need to
     build the other tooling (rust-analyzer, rustdoc, clippy, ...) or set up your workflow to
     instead use another toolchain's tools, e.g. in the VSCode config for the rust-analyzer
     extension.

6. Have a look at how the [sample application](https://github.com/rust9x/rust9x-sample) configures
   search paths and linker arguments to bring everything together (see `Cargo.toml` and
   `.cargo/config.toml`)

[limitations]: limitations.md

## Building with rust9x

[Building with rust9x](building-with-r9x.md)

## `rust9x-sample` in action (various versoins)

### Windows 95 RTM with WinSock2 update (VM)

<img width="1024" height="768" alt="image" src="https://github.com/user-attachments/assets/e3aa48c3-9b50-427f-81b5-493cc0a76542" />

### Windows 95 B (real hardware, my Pentium MMX 233MHz machine)

[<img width="640" alt="win95"
src="https://user-images.githubusercontent.com/5844066/164351479-ac7bf75d-d6fe-4897-8be0-bab726c24906.png">](https://user-images.githubusercontent.com/5844066/164351479-ac7bf75d-d6fe-4897-8be0-bab726c24906.png)

### Windows NT Workstation 3.1 SP3 (VM) (with [r9xrsrt](https://github.com/rust9x/r9xrsrt) only)

<img width="800" height="600" alt="rust9x on WinNT 3.1 SP3" src="https://github.com/user-attachments/assets/72d60543-74de-4183-a541-15b2825a19ab" />

### Windows NT Workstation 3.5 (VM)

<img width="800" height="600" alt="image" src="https://github.com/user-attachments/assets/3198700a-cc0a-4435-8d4b-d304d543ea9c" />

### Windows NT Workstation 3.51 (VM)

<img width="1024" height="768" alt="image" src="https://github.com/user-attachments/assets/435e0843-8e04-4dbf-bcb5-1b9bb84c87b0" />

### Windows NT Workstation 4.0 RTM/SP0 (VM)

<img width="800" height="600" alt="image" src="https://github.com/user-attachments/assets/c6a47771-d562-424e-a872-e4cb4d4ea106" />

### Windows XP SP3 (real hardware)

[<img width="640" alt="winxp"
src="https://user-images.githubusercontent.com/5844066/164351581-61f0cea0-ba94-49f6-b5eb-1f07412659b2.PNG">](https://user-images.githubusercontent.com/5844066/164351581-61f0cea0-ba94-49f6-b5eb-1f07412659b2.PNG)

### Windows 11 Pro Insider (real hardware)

[<img width="640" alt="win11"
src="https://user-images.githubusercontent.com/5844066/164351670-3d13e58c-fde6-40e8-b5fb-22a6aabb9f86.png">](https://user-images.githubusercontent.com/5844066/164351670-3d13e58c-fde6-40e8-b5fb-22a6aabb9f86.png)

### Linux via wine

<img width="1626" height="1802" alt="image" src="https://github.com/user-attachments/assets/23ac2ead-595f-4e06-b8dd-3dc936d3ca5f" />

