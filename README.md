# PPSSPP — Unofficial Xbox 360 Port

This is an unofficial port of [PPSSPP](https://www.ppsspp.org/) (a PlayStation Portable emulator) targeting **modified Xbox 360 consoles** (JTAG / RGH / RGH2 / RGH3). It is based on PPSSPP 0.9.5–0.9.6 (circa 2014) and is not affiliated with the official PPSSPP project.

This branch contains **only the files necessary to build the Xbox 360 `.xex`**. All other platform ports (Android, iOS, Windows desktop, Linux/SDL, BlackBerry, Qt) have been removed.

---

## Hardware Requirements

| Item | Requirement |
|------|-------------|
| Console | Xbox 360 (any SKU) with RGH / JTAG / RGH2 / RGH3 mod |
| Dashboard | Aurora, FreeStyle Dash, or any homebrew loader that launches `.xex` files |
| Storage | HDD or USB drive — ISO files can be large |
| Controller | Standard Xbox 360 controller (wired or wireless with receiver) |

> **Jasper / Slim / Trinity / Corona** boards all work. RGH3 on Jasper is well-supported. The ECC overclock present on some Jasper boards does not affect this build.

---

## Build Requirements (Windows PC)

| Tool | Version | Notes |
|------|---------|-------|
| Windows | 7 or later | x64 recommended |
| Visual Studio | 2010 (VS10) | The `.sln`/`.vcxproj` files target `ToolsVersion="4.0"` |
| Xbox 360 XDK | Any version compatible with VS 2010 | Provides the `Xbox 360` platform toolset, `xtl.h`, and all XDK libraries |
| Git | Any recent version | Required by the pre-build version script |

> **Important:** The Xbox 360 XDK is Microsoft proprietary software. You must obtain it through legitimate means (e.g., as a registered Xbox 360 developer or via archival sources). The XDK installs the `Xbox 360` platform target into Visual Studio automatically.

---

## Directory Structure

```
ppsspp-xbox-360/
├── Xbox/                   # Xbox 360 platform frontend
│   ├── PPSSPP.sln          # ← Open this in Visual Studio
│   ├── xx/
│   │   └── xx.vcxproj      # Main executable project (outputs xx.xex)
│   ├── Jit/
│   │   ├── bram.cpp/.h     # VRAM allocator
│   ├── XboxMain.cpp        # Entry point (main())
│   ├── XboxHost.cpp/.h     # Host interface: audio, DX swap, asset loading
│   ├── XboxOs.cpp          # Thread naming via XTL
│   ├── XboxOnScreenDisplay.cpp/.h
│   ├── display_xbox.cpp    # Resolution globals
│   ├── XaudioSound.cpp/.h  # XAudio2 backend
│   ├── XinputDevice.cpp/.h # Xbox 360 controller input
│   ├── InputDevice.cpp/.h
│   ├── Compare.cpp/.h
│   └── StubHost.h
│
├── Common/                 # Shared utilities + Xbox-specific files
│   ├── CommonXbox.vcxproj  # Xbox static library
│   ├── XboxCPUDetect.cpp   # Reports CPU as "Xenon"
│   ├── ppcEmitter.cpp/.h   # PowerPC code generation
│   └── ...
│
├── Core/                   # PSP CPU / HLE emulation core
│   ├── CoreXbox.vcxproj    # Xbox static library
│   ├── x360_compat.h       # XTL compatibility, barrier intrinsics
│   ├── MIPS/PPC/           # PowerPC JIT for Xenon
│   └── ...
│
├── GPU/                    # Graphics emulation
│   ├── GPUXbox.vcxproj     # Xbox static library
│   ├── Directx9/           # DirectX 9 backend (used by Xbox)
│   └── ...
│
├── ext/                    # Third-party libraries
│   ├── zlib/zlibXbox.vcxproj
│   ├── libkirk/libkirkXbox.vcxproj
│   └── xxhash.c
│
├── assets/                 # Runtime assets: shaders, UI atlas
├── Windows/
│   └── git-version-gen.cmd # Pre-build script: generates git-version.cpp
├── Globals.h
├── git-version.cmake
└── .gitmodules             # Submodules: native, ffmpeg, dx9sdk, lang, redist
```

---

## Submodules

The build depends on several git submodules. **These are not optional** — the build will fail without them.

| Submodule | Path | Purpose |
|-----------|------|---------|
| `native` | `native/` | Cross-platform base library (file I/O, threading, image loading, JSON, math) |
| `ffmpeg` | `ffmpeg/` | Media decoding (Atrac3+, PMF video) — Xbox PPC pre-built libs in `ffmpeg/Xbox/ppc/lib/` |
| `dx9sdk` | `dx9sdk/` | Minimal DirectX 9 headers/libs (supplements XDK) |
| `lang` | `lang/` | Localization strings |
| `redist` | `redist/` | Runtime files including `flash0/` PSP firmware fonts |

Initialize them all at once:

```cmd
git submodule update --init --recursive
```

> The `ffmpeg` submodule is the Ced2911 fork ([github.com/Ced2911/FFmpeg](https://github.com/Ced2911/FFmpeg)) with pre-compiled Xbox 360 PPC static libraries. These `.lib` files are what the linker actually uses — you do **not** need to compile FFmpeg yourself.

---

## Step-by-Step Build Guide

### 1. Clone the Repository

```cmd
git clone https://github.com/sourkyle/ppsspp-xbox-360-beta.git ppsspp-xbox360
cd ppsspp-xbox360
git checkout cursor/xbox360-clean-build-bf77
git submodule update --init --recursive
```

> If the `ffmpeg` or `native` submodule clone is slow, try cloning with `--depth 1`:
> ```cmd
> git submodule update --init --recursive --depth 1
> ```

### 2. Verify Submodule Contents

Make sure these directories are not empty before proceeding:

```cmd
dir native\base\
dir ffmpeg\Xbox\ppc\lib\
dir dx9sdk\
```

If any are empty, re-run `git submodule update --init --recursive`.

### 3. Verify XDK Installation

Open a new Visual Studio 2010 command prompt and confirm the `Xbox 360` platform is present:

- Launch Visual Studio 2010
- Go to **File → New → Project**
- In the project templates, look for **Xbox 360** under Visual C++

If the Xbox 360 platform is missing, re-run the XDK installer and select the Visual Studio 2010 integration option.

### 4. Open the Solution

```
Xbox\PPSSPP.sln
```

Visual Studio will load 6 projects:

| Project | Type | Purpose |
|---------|------|---------|
| `xx` | Application | Main executable → `xx.xex` |
| `CommonXbox` | Static lib | Shared utilities |
| `CoreXbox` | Static lib | PSP CPU + HLE emulation |
| `GPUXbox` | Static lib | DirectX 9 GPU backend |
| `libkirkXbox` | Static lib | PSP crypto (Kirk engine) |
| `zlibXbox` | Static lib | Compression |

### 5. Select Build Configuration

**Recommended for first build:** `Release | Xbox 360`

Available configurations:

| Configuration | Use Case |
|---------------|----------|
| `Debug` | Full debug symbols, no optimization — very slow on hardware |
| `Debug_Optimised` | Debug symbols + full optimization — useful for crash debugging |
| `Release` | Optimized release build — good balance |
| `Release_LTCG` | Link-time code generation — best runtime performance, slow to compile |
| `Profile` | Release + Callcap profiling support |
| `Profile_FastCap` | Release + Fastcap profiling |

Set the platform to **Xbox 360** in the toolbar dropdown (not Win32 or x64).

### 6. Build

Press **F7** (Build Solution) or go to **Build → Build Solution**.

Build order is automatic via project references:
1. `zlibXbox` → `libkirkXbox` → `CommonXbox` → `CoreXbox` → `GPUXbox` → `xx`

The pre-build event in `CoreXbox` runs `Windows\git-version-gen.cmd` to write `git-version.cpp`. This requires `git.exe` to be on your PATH. If `git` is not found, the script writes a fallback `"unknown"` version string and continues — the build still succeeds.

### 7. Output

The built executable is located at:

```
Xbox\bin\Xbox 360\Release\xx.xex
```

(Path varies by configuration: `Debug`, `Release`, `Release_LTCG`, etc.)

---

## Deploying to the Xbox 360

### Option A: XDK Neighborhood (Recommended for Testing)

If you have the full XDK installed with the Xbox 360 Neighborhood tool:

1. Connect your Xbox 360 to the same LAN as your PC
2. The project's **Deploy** action (`CopyToHardDrive`) will copy `xx.xex` directly to the console's HDD
3. In Visual Studio, use **Build → Deploy** or press **F5** to deploy and launch

### Option B: Manual Copy via FTP / XBDM

1. Use **XBDM** (Xbox Debug Monitor) or an FTP client (e.g., Xbox 360 FTP server homebrew) to connect to your console
2. Copy the following to a folder on the HDD, e.g., `HDD:\Games\PPSSPP\`:
   ```
   xx.xex
   assets\           (entire folder from repo root)
   flash0\           (from redist submodule — PSP system fonts)
   lang\             (from lang submodule — optional, for non-English UI)
   ```

### Option C: USB Drive

1. Format a USB drive as FAT32 (Xbox 360 compatible)
2. Copy the above files to the USB drive root or a subfolder
3. Plug into the Xbox 360 and launch `xx.xex` from Aurora / FSD

---

## Running Games

By default, `XboxMain.cpp` is hardcoded to boot from:

```cpp
bootFilename = "game:\\psp.iso";
```

`game:` maps to the directory where `xx.xex` is located. Place your PSP ISO or CSO in the same directory as `xx.xex` and name it `psp.iso`, or edit `XboxMain.cpp` to change the path before building.

Config and save data paths:

```
game:\ppsspp.ini       (main config, auto-generated on first run)
game:\controls.ini     (button mapping)
game:\memstick\        (PSP memory stick — saves, DLC, etc.)
game:\flash0\          (PSP system files — fonts required for some games)
```

---

## Key Preprocessor Defines

| Define | Meaning |
|--------|---------|
| `_XBOX` | Xbox 360 platform (XTL APIs) |
| `PPC` | PowerPC CPU target (Xenon is a tri-core PowerPC) |
| `BIG_ENDIAN` | Xenon is big-endian; affects PSP RAM byte-swap logic |
| `NO_JIT` | Disables the x86 JIT; the PowerPC JIT in `Core/MIPS/PPC/` is used instead |
| `USE_DIRECTX` | Selects DirectX 9 GPU backend over OpenGL ES |
| `NDEBUG` | Disables assertions (Release builds) |
| `NO_UNICODE` | Uses multi-byte character set (MBCS) instead of wide strings |

---

## Architecture Notes

### CPU Emulation (MIPS → PowerPC)

The PSP uses a 32-bit MIPS R4000 CPU. On Xbox 360, PPSSPP compiles MIPS guest code to native Xenon PowerPC instructions using the JIT in `Core/MIPS/PPC/`. The flag `NO_JIT` in the project file refers specifically to the **x86** JIT (used on Windows desktop), not the PPC JIT — this naming is confusing but intentional.

### GPU Emulation (PSP GPU → DirectX 9)

The PSP GPU command list is translated to DirectX 9 draw calls by `GPU/Directx9/`. The Xbox 360's `d3d9.lib` is the XDK's DirectX 9 implementation, not the PC version. Texture formats use the `D3DFMT(x)` linear format macro defined in `GPU/Directx9/helper/global.h` which accounts for the Xbox 360's tiled/linear texture layout difference.

### Audio (PSP Audio → XAudio2)

`Xbox/XaudioSound.cpp` implements a PCM mixer using XAudio2. The PSP's ATRAC3+ audio is decoded by FFmpeg (`avcodec.lib` etc.) before being fed to XAudio2.

### Memory Layout

The `.xex` is linked at base address `0x92000000` (`/BASE:0x92000000`). Stack size for Release builds is 1 MB. The `Xbox/Jit/bram.cpp` custom allocator (`balloc`/`bfree`) manages VRAM allocations separately from the main heap.

---

## Known Issues and Troubleshooting

### Build Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot open include file: 'xtl.h'` | XDK not installed or not integrated with VS 2010 | Reinstall XDK with VS 2010 integration |
| `Cannot open include file: 'native/base/NativeApp.h'` | `native` submodule not checked out | `git submodule update --init native` |
| `Cannot open file 'avcodec.lib'` | `ffmpeg` submodule not checked out | `git submodule update --init ffmpeg` |
| `'ppcAbi.cpp' missing` | Referenced in `CommonXbox.vcxproj` but file does not exist in repo | See note below |
| `LNK2019` unresolved externals | Usually means a dependent lib didn't build | Build each project individually to find root cause |
| `LNK1104: cannot open 'xapilib.lib'` | XDK not found by linker | Ensure XDK library paths are in `$(XDKPath)\lib\xbox` |

> **Note on `ppcAbi.cpp`:** `CommonXbox.vcxproj` references `ppcAbi.cpp`, which is missing from this repository. If you get an error about this file, either create a stub `Common/ppcAbi.cpp` with empty implementations, or remove the reference from `CommonXbox.vcxproj`. The file contained PowerPC ABI helper stubs that may not be strictly required if the PPC JIT is functioning correctly via `ppcEmitter.cpp`.

### Runtime Issues

| Symptom | Likely Cause |
|---------|-------------|
| Black screen on boot | `flash0/` fonts missing — copy from `redist` submodule |
| Crash immediately after launching | Missing `assets/` folder next to `xx.xex` |
| No sound | XAudio2 init failure — try a different audio config in `XaudioSound.cpp` |
| Game boots but runs very slowly | Try `Release_LTCG` build; check `g_Config.iLockedCPUSpeed` in `XboxMain.cpp` |
| Controller not responding | Verify `XinputDevice.cpp` button mapping; only player 1 controller is polled |
| E74 / hardware crash | Unrelated to PPSSPP — indicates hardware issue on the console |

### Jasper + RGH3 + ECC OC Notes

The Jasper variant with ECC overclock is one of the most stable RGH3 platforms. No special build changes are required — the overclock affects the CPU timing used for glitch injection, not the running CPU frequency. PPSSPP will run at standard Xenon speeds (3 × 3.2 GHz cores). The `g_Config.iLockedCPUSpeed = 111` line in `XboxMain.cpp` locks the *emulated* PSP CPU to 111 MHz (the PSP's battery-saving clock), not the Xenon clock.

---

## Improving the Port (Next Steps)

The following areas are good candidates for improvement:

1. **Dynamic ISO loading** — Replace the hardcoded `game:\\psp.iso` path with a file browser using Aurora's launch data or a simple D-pad menu.
2. **Config file UI** — The `ppsspp.ini` is currently edited by hand; a minimal settings screen would help.
3. **Frame pacing** — The main loop uses a fixed 60 Hz tick; variable frame skip (`g_Config.iFrameSkip`) should be tuned.
4. **Memory management** — The `bram` VRAM allocator is a custom heap; testing with more RAM-intensive games may reveal fragmentation issues.
5. **FFmpeg ATRAC3+ accuracy** — Some games use audio formats that the 2014-era FFmpeg may decode with artifacts.
6. **Missing `ppcAbi.cpp`** — Stub out or properly implement this file to ensure clean builds without workarounds.
7. **`ExtendedTrace.cpp` missing** — Also referenced in `CommonXbox.vcxproj`; stub with a no-op `ExtendedTrace()` if needed.
8. **GPU tiling** — Some games expose bugs in the `D3DFMT` linear/tiled format handling; the `helper/global.h` macro may need expansion for additional texture formats.

---

## License

PPSSPP is licensed under the **GNU General Public License v2.0** (or later). See `LICENSE.TXT`.

This Xbox 360 port is © its respective contributors and is similarly licensed under GPL v2.0+. It is intended for use on **hardware you own**, running legitimately obtained game backups or homebrew.
