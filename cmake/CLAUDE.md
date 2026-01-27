# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UltraVNC is a Windows-only remote desktop tool using the RFB (Remote Frame Buffer) protocol. This `cmake/` directory contains the CMake-based build system.

## Build Commands

### Prerequisites
- Visual Studio 2022 with MFC components
- NASM (Netwide Assembler): https://nasm.us/
- vcpkg for dependency management

### Windows Build with Visual Studio (recommended)

```cmd
# Set up vcpkg (one-time)
git clone https://github.com/microsoft/vcpkg.git c:\source\vcpkg
cd c:\source\vcpkg
bootstrap-vcpkg.bat -disableMetrics
set VCPKG_ROOT=c:\source\vcpkg
set PATH=%VCPKG_ROOT%;%PATH%

# Install dependencies
vcpkg install zlib:x64-windows-static zstd:x64-windows-static libjpeg-turbo:x64-windows-static liblzma:x64-windows-static libsodium:x64-windows-static
vcpkg integrate install

# Build
mkdir obj && cd obj
cmake -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static ..\UltraVNC\cmake
set CL=/MP
cmake --build . --parallel --config=RelWithDebInfo
```

### Build with Ninja and Address Sanitizer

```cmd
cmake -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static -G Ninja -Dasan=TRUE ..\UltraVNC\cmake
cmake --build . --parallel --config=RelWithDebInfo
cmake --build . --target install --config=RelWithDebInfo
```

### Build with LLVM/Clang

```cmd
cmake -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static -T ClangCL ..\UltraVNC\cmake
cmake --build . --parallel --config=RelWithDebInfo
```

### Legacy Visual Studio Build (without CMake)

```cmd
msbuild /p:Platform=x64 /p:Configuration=Release winvnc\winvnc.sln
msbuild /p:Platform=x64 /p:Configuration=Release vncviewer\vncviewer.sln
```

## Architecture

### Core Components

| Component | Type | Description |
|-----------|------|-------------|
| **vncviewer** | Executable | UltraVNC Viewer (RFB client) |
| **winvnc** | Executable | UltraVNC Server (without cloud feature - see RSN_IMPORTANT.txt) |
| **repeater** | Executable | VNC connection relay service |

### Shared Libraries (DLLs)

| Library | Purpose |
|---------|---------|
| **vnchooks** | Desktop capture hooks |
| **authSSP** | Windows Security Support Provider authentication |
| **authadmin** | Admin authentication |
| **ldapauth/ldapauth9x/ldapauthnt4** | LDAP authentication for different Windows versions |
| **logging** | Logging functionality |

### Static Libraries

| Library | Purpose |
|---------|---------|
| **librdr** | RFB data stream I/O (zlib, zstd, LZMA compression) |
| **libomnithread** | Multi-threading abstraction (Windows NT) |
| **libzip32/libzipunzip** | ZIP archive handling |

### Utilities

| Utility | Purpose |
|---------|---------|
| **uvnc_settings** | Configuration GUI |
| **setpasswd/createpassword** | Password management |
| **MSLogonACL** | MS-Logon ACL management |
| **setcad** | CAD configuration |

## Key Source Directories (relative to repository root)

- `winvnc/` - Server source code
- `vncviewer/` - Viewer source code
- `repeater/` - Repeater source code
- `rfb/` - RFB protocol implementation
- `rdr/` - Data stream handling with compression
- `omnithread/` - Threading library
- `addon/ms-logon/` - MS-Logon authentication modules
- `common/` - Shared utilities (includes inifile.cpp used by winvnc)

## Build Configuration

- **C++ Standard:** C++14
- **Architectures:** x64 and x86 (x64 sets `_X64` define)
- **Dependencies:** zlib, zstd, libjpeg-turbo, liblzma, libsodium (managed via vcpkg)
- **Output directory:** `${CMAKE_BINARY_DIR}/ultravnc`

## Notes

- **Windows-only:** This project builds and runs only on Windows. Cross-compilation is not supported.
- **Cloud/UDT feature is non-functional:** The cloud relay source code (libudt4, libudtcloud) is not in this repository. Do not attempt to enable or develop this feature - it will not compile. Code paths are guarded by `_CLOUD` preprocessor define which must remain undefined. See RSN_IMPORTANT.txt for details.
- SecureVNCPlugin is not publicly available
- Windows system libraries required: comctl32, gdi32, ws2_32, wtsapi32, etc.
