# Building on Linux with MSVC targets: Rust and rust9x

Note that only the last step is rust9x-specific.

## 1: Obtain MSVC Build tools

The simplest way to get a full MSVC toolchain is to use the [Enterprise WDK][ewdk]. It's a
positively huge download, but it contains an effectively-portable MSVC toolchain, together with the
Windows WDK and SDK headers/libs.

[ewdk]:
    https://learn.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk#download-icon-for-ewdk-enterprise-wdk-ewdk

Alternatively install the VS Build tools on a Windows system. Make sure to select the MSVC toolchain
and Windows SDK at the minimum. Copy the appropriate folders (see below) over.

## 2: (Optional?) Set up a partition with case-insensitive filesystem support

Personally, I set up an ext4 partition with the `-O casefold` option, which allows `chattr +F
msvc-toolchains` to mark any path in that folder as case-insensitive. This is necessary because
different deps might try to import the import libraries with different case (e.g. `KERNEL32.LIB` vs
`kernel32.lib`), which need to resolve correctly.

Of course you can also use an NTFS partition directly.

## 3: Set up `LIB` environment variable

The linker needs to know where to find the "system" import libraries. The paths are roughly as
follows (tweak for your own version and target architecture):

```plain
/home/seri/mnt/ext4/msvc-toolchains/ewdk-25h2/Program Files/Windows Kits/10/Lib/10.0.26100.0/um/x86/
/home/seri/mnt/ext4/msvc-toolchains/ewdk-25h2/Program Files/Windows Kits/10/Lib/10.0.26100.0/ucrt/x86/
/home/seri/mnt/ext4/msvc-toolchains/ewdk-25h2/Program Files/Microsoft Visual Studio/2022/BuildTools/VC/Tools/MSVC/14.44.35207/lib/x86/
```

`LIB` is semicolon-separated:

```bash
LIB="/home/seri/mnt/ext4/msvc-toolchains/ewdk-25h2/Program Files/Windows Kits/10/Lib/10.0.26100.0/um/x86/;/home/seri/mnt/ext4/msvc-toolchains/ewdk-25h2/Program Files/Windows Kits/10/Lib/10.0.26100.0/ucrt/x86/;/home/seri/mnt/ext4/msvc-toolchains/ewdk-25h2/Program Files/Microsoft Visual Studio/2022/BuildTools/VC/Tools/MSVC/14.44.35207/lib/x86/"
```

## 4: Use the `rust-lld` linker

For rust9x, the `rust-lld` linker is used by default.

For regular Rust, set it in `.cargo/config.toml` in your project:

```toml
[target.'cfg(all(target_env = "msvc"))']
linker = "rust-lld"
```

If you also want to compile c code, you'll likely have to set the `CC` env var to `clang-cl` as
well.

## 5: Done

Building for your MSVC target should now work. You might additionally want to set the [target
runner][runner] to `wine` so you can just `cargo run` directly on linux:

```toml
[target.'cfg(all(target_env = "msvc"))']
linker = "rust-lld"
runner = "wine"
```

[runner]: https://doc.rust-lang.org/cargo/reference/config.html#targettriplerunner

## 6: Building rust9x itself on Linux

Follow steps 1-3, then use
[`bootstrap.rust9x.linuxmsvc.toml`][bootstrap-toml]
in the repo as base for your build configuration. In particular, it sets these:

[bootstrap-toml]: https://github.com/rust9x/rust/blob/rust9x/bootstrap.rust9x.linuxmsvc.toml

```toml
[target.i586-rust9x-windows-msvc]
cc = "clang-cl"
ar = "llvm-lib"
[target.i686-rust9x-windows-msvc]
cc = "clang-cl"
ar = "llvm-lib"
[target.x86_64-rust9x-windows-msvc]
cc = "clang-cl"
ar = "llvm-lib"
[target.x86_64-pc-windows-msvc]
cc = "clang-cl"
ar = "llvm-lib"

# ...

[rust]
lld = true
bootstrap-override-lld = "self-contained"
```

Note that you can't build x86 and x86_64 targets in the same build, because you'll have to switch
the `LIB` env var between them. If you figure out a way to do that automatically, please let me
know!
