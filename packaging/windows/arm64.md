# Native Windows ARM64 build

herdr builds natively for `aarch64-pc-windows-msvc` (PE `0xAA64`) on Windows
ARM64 hardware. Upstream release manifests only ship `windows-x86_64`, which
runs under Prism emulation — this document covers building the native binary
and packaging it in the installer layout.

## Toolchain

| Piece | Notes |
|---|---|
| Rust | pinned by `rust-toolchain.toml` (1.96.1, clippy + rustfmt) |
| MSVC | VS with `Hostarm64\arm64\link.exe` |
| Windows SDK | 10.0.26100+ (any recent SDK with ARM64 libs) |
| Zig 0.16.0 | **x86_64 build** — the native ARM64 Zig binary crashes; x86_64 runs under emulation |
| LLVM/libclang | aarch64 Windows build; provides `libclang.dll` (bindgen) and MSVC-aware `clang` (ring build.rs) |

Environment (PowerShell):

```powershell
$env:ZIG = "C:\path\to\zig-x86_64\zig.exe"
$env:LIBCLANG_PATH = "C:\path\to\llvm\bin"
$env:RUSTFLAGS = "-C target-feature=+crt-static"
$env:PATH = "C:\path\to\llvm\bin;$env:PATH"
```

`+crt-static` makes the exe static-CRT so no VC++ redistributable is needed.

## Build

```powershell
cargo build --release    # → target\release\herdr.exe
```

On ARM64, pane terminals use the OS system ConPTY — the app-local ConPTY
loader in `vendor/portable-pty` is compiled x86_64-only upstream, so the bare
exe is sufficient to run.

## Packaging (installer layout)

To install via `distribution/install.ps1 -LocalPackagePath`, zip:

```
herdr.exe
conpty\conpty.dll                 # ARM64, from the pinned ConPTY nupkg
conpty\arm64\OpenConsole.exe      # ARM64
conpty\x64\OpenConsole.exe        # x64 (layout compatibility)
herdr-conpty.json                 # marker w/ per-file sha256
THIRD-PARTY-NOTICES\
```

The pinned package is `Microsoft.Windows.Console.ConPTY.1.24.260710001.nupkg`
(see `packaging/windows/conpty.json` for URL + sha256). It ships
`runtimes/win-arm64/native/conpty.dll` and
`build/native/runtimes/arm64/OpenConsole.exe` — extract those plus the x64
`OpenConsole.exe` and licenses.

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File ".\distribution\install.ps1" `
  -LocalPackagePath ".\herdr-windows-aarch64-vX.Y.Z.zip" `
  -LocalPackageFormat zip -LocalPackageIdentity "X.Y.Z" `
  -LocalPackageSha256 "<sha256 of zip>"
```

Note: on ARM64 hosts the installer labels the release dir
`*-x86_64-pc-windows-msvc` (hardcoded target triple) — cosmetic only.

## Limitations

- `herdr update` cannot work: manifests have no `windows-aarch64` asset. The
  local-package install mode above is the update path.
- Notification click-to-focus uses `conpty/x64/OpenConsole.exe` under
  emulation; the ARM64 `conpty.dll` is not loaded by the ARM64 CLI build.
