# Building bookokrat on Windows (with PDF support)

This guide walks you through setting up a Windows environment to build
bookokrat from source, including the PDF rendering feature powered by
mupdf.

## Overview

bookokrat uses the **GNU** Rust toolchain (`x86_64-pc-windows-gnu`) on
Windows rather than the default MSVC toolchain. This is because the
mupdf PDF library requires `clang` and `bindgen`, which integrate more
cleanly with the GNU toolchain. All of this tooling comes from MSYS2.

The build produces `bookokrat.exe` — a single binary that reads both
EPUB and PDF files in the terminal.

---

## Step 1: Install Rust

Download and run the installer from [rustup.rs](https://rustup.rs).

During installation:
- Choose **"Customize installation"** when prompted.
- On the host-triple page, select **`x86_64-pc-windows-gnu`** instead of
  the default `x86_64-pc-windows-msvc`.  (If you already have MSVC
  installed, that is fine — the steps below work with either default.)

Verify the installation:

```powershell
rustc --version    # should show 1.86 or later
cargo --version
```

---

## Step 2: Install MSYS2

MSYS2 provides the C/C++ toolchain (clang, make, pkg-config) needed to
compile mupdf.

Install MSYS2 via winget (easiest):

```powershell
winget install --id MSYS2.MSYS2 --accept-source-agreements --accept-package-agreements
```

Or download manually from [msys2.org](https://www.msys2.org/) and run
the installer.  Accept the default install location (`C:\msys64`).

---

## Step 3: Install MSYS2 packages

Open a PowerShell terminal and install the required packages:

```powershell
& "C:\msys64\msys2_shell.cmd" -defterm -no-start -mingw64 -c "pacman -S --needed --noconfirm mingw-w64-x86_64-clang mingw-w64-x86_64-pkgconf make"
```

This installs:
- `mingw-w64-x86_64-clang`  — C/C++ compiler + libclang (needed by bindgen)
- `mingw-w64-x86_64-pkgconf` — pkg-config replacement (finds library paths)
- `make` — GNU Make (mupdf build system)

After installation, verify the tools exist:

```powershell
Test-Path "C:\msys64\mingw64\bin\clang.exe"
Test-Path "C:\msys64\mingw64\bin\libclang.dll"
Test-Path "C:\msys64\mingw64\bin\pkgconf.exe"
Test-Path "C:\msys64\usr\bin\make.exe"
```

All four should return `True`.

---

## Step 4: Add the GNU Rust target (if needed)

If your default Rust toolchain is MSVC, add the GNU target:

```powershell
rustup target add x86_64-pc-windows-gnu
```

If you installed with the GNU default in Step 1, you can skip this.

---

## Step 5: Build

Set the environment variables, then run `cargo build` with the GNU
toolchain and the `pdf` feature flag.

### Debug build (faster compilation, for development)

```powershell
$env:PATH   = "C:\msys64\mingw64\bin;C:\msys64\usr\bin;$env:PATH"
$env:LIBCLANG_PATH = "C:/msys64/mingw64/bin"
cargo +stable-gnu build --features pdf
```

The `+stable-gnu` override ensures both the host toolchain (build
scripts, proc macros) and target toolchain use the same GNU compiler —
avoiding MSVC/GNU mixing issues.

### Release build (optimized, for actual use)

```powershell
$env:PATH   = "C:\msys64\mingw64\bin;C:\msys64\usr\bin;$env:PATH"
$env:LIBCLANG_PATH = "C:/msys64/mingw64/bin"
cargo +stable-gnu build --release --features pdf
```

The release binary lands at `target\release\bookokrat.exe` and is
stripped + LTO'd (~15-20 MB).  The first build takes 15-30 minutes
depending on your machine; subsequent incremental builds are much
faster.

---

## Step 6: Run

```powershell
.\target\debug\bookokrat.exe
```

Or with a specific book:

```powershell
.\target\debug\bookokrat.exe path\to\book.epub
.\target\debug\bookokrat.exe path\to\document.pdf
```

---

## Troubleshooting

### `error: linking with link.exe failed`

You have MSYS2's GCC tools in PATH while using the MSVC toolchain.
Use `cargo +stable-gnu` (see Step 5) so everything uses the GNU
toolchain consistently.

### `make: program not found`

Run the command in Step 3 again — `make` was not installed.

### `clang not found` / bindgen errors

Ensure `C:\msys64\mingw64\bin` is in PATH and `LIBCLANG_PATH` is set
(see the `$env:` lines in Step 5).

### Build is extremely slow (>30 min)

The first build compiles ~200 crates including mupdf from source.  This
is normal.  Subsequent builds only recompile changed code.

If you need faster iteration, use the debug profile (skip `--release`)
and set `codegen-units = 16` in `Cargo.toml` — the existing
`release-fast` profile already does this:

```powershell
cargo +stable-gnu build --profile release-fast --features pdf
```

### `error: failed to run custom build command for mupdf-sys`

Make sure all four tools from Step 3 verify correctly.  Also confirm
`C:\msys64\usr\bin` is in PATH (it provides `make`).

---

## Environment variable reference

| Variable | Value | Purpose |
|---|---|---|
| `PATH` | prepend `C:\msys64\mingw64\bin` and `C:\msys64\usr\bin` | Makes clang, pkgconf, make available |
| `LIBCLANG_PATH` | `C:/msys64/mingw64/bin` | Points bindgen to `libclang.dll` |
| `BINDGEN_EXTRA_CLANG_ARGS` | not needed with `+stable-gnu` | Only required when cross-compiling MSVC→GNU |
