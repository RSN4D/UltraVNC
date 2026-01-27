# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UltraVNC is a Windows-only remote desktop tool using the RFB (Remote Frame Buffer) protocol. Both server (winvnc) and viewer (vncviewer) are built for Windows. The viewer can connect to any VNC server on any platform.

## Build Commands

### Prerequisites
- Visual Studio 2022 with MFC components
- NASM (Netwide Assembler): https://nasm.us/
- vcpkg at `D:\rsn\vcpkg`

### Windows Build (CMake + vcpkg)

```cmd
set VCPKG_ROOT=D:\rsn\vcpkg

# Build
mkdir build && cd build
cmake -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static ..\cmake
cmake --build . --parallel --config=RelWithDebInfo
```

### Dependencies (vcpkg)

Required packages (x64-windows-static triplet): zlib, zstd, libjpeg-turbo, liblzma, libsodium

**Check installed packages:**
```cmd
vcpkg list | grep -iE "zlib|zstd|libjpeg|liblzma|libsodium"
```

**Install missing packages:**
```cmd
vcpkg install zlib:x64-windows-static zstd:x64-windows-static libjpeg-turbo:x64-windows-static liblzma:x64-windows-static libsodium:x64-windows-static
```

When adding new external dependencies, first check if they exist in the local vcpkg. If not present, add them using `vcpkg install <package>:x64-windows-static`.

### Legacy Visual Studio Build (without CMake)

```cmd
msbuild /p:Platform=x64 /p:Configuration=Release winvnc\winvnc.sln
msbuild /p:Platform=x64 /p:Configuration=Release vncviewer\vncviewer.sln
```

### Build Variants

**Ninja + Address Sanitizer:**
```cmd
set VCPKG_ROOT=D:\rsn\vcpkg
cmake -G Ninja -Dasan=TRUE -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static ..\cmake
```

**LLVM/Clang:**
```cmd
set VCPKG_ROOT=D:\rsn\vcpkg
cmake -T ClangCL -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static ..\cmake
```

## Architecture

### Core Components

| Component | Directory | Description |
|-----------|-----------|-------------|
| winvnc | `winvnc/winvnc/` | VNC Server - handles desktop capture, client management, input injection |
| vncviewer | `vncviewer/vncviewer/` | VNC Viewer - RFB client with multiple encoding decoders |
| repeater | `repeater/` | Connection relay for NAT traversal |

### Key Libraries

| Library | Directory | Purpose |
|---------|-----------|---------|
| librdr | `rdr/` | Data stream I/O with compression (zlib, zstd, LZMA) |
| libomnithread | `omnithread/` | Threading abstraction for Windows NT |
| vnchooks | `winvnc/vnchooks/` | Desktop capture hooks DLL |

### Authentication Modules (`addon/ms-logon/`)

- **authSSP** - Windows Security Support Provider authentication
- **authadmin** - Admin authentication
- **ldapauth/ldapauth9x/ldapauthnt4** - LDAP variants for different Windows versions

### Server Architecture (winvnc)

Key classes in `winvnc/winvnc/`:
- **vncServer** (`vncserver.h`) - Main coordination, manages up to 128 client connections
- **vncDesktop** (`vncdesktop.h`) - Desktop capture, update tracking, multiple monitor support
- **vncClient** (`vncclient.h`) - Per-connection handler, encoding, file transfer

### Viewer Architecture (vncviewer)

- **ClientConnection** (`ClientConnection.h`) - Main protocol handler, encoding decoders, DirectX rendering

### Protocol & Encodings

- **Protocol:** RFB (Remote Frame Buffer)
- **Encodings:** Raw, RRE, CoRRE, Hextile, ZRLE, Tight, Ultra, Ultra2, ZlibHex, XZ
- **Security:** VNC auth, TLS, RSA-AES, SSP, MS-Logon (LDAP)

## Build Configuration

- **C++ Standard:** C++14
- **Architectures:** x64 (`_X64` defined) and x86
- **Output:** `${CMAKE_BINARY_DIR}/ultravnc`

## Important Notes

- **Windows-only:** This project builds and runs only on Windows. Do not attempt cross-compilation.
- **Cloud/UDT feature is non-functional:** The cloud relay source code (libudt4, libudtcloud) is not in this repository. Do not attempt to enable or develop this feature - it will not compile. Code paths are guarded by `_CLOUD` preprocessor define which must remain undefined. See `cmake/RSN_IMPORTANT.txt` for details.
- **Default ports:** 5900 (VNC), 5800 (HTTP/Java viewer)
- **vnchooks.dll** must be in same directory as winvnc.exe for efficient desktop capture
- **Server requires admin privileges** for full functionality
