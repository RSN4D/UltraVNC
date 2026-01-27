# RSN Study Notes

Technical study notes for UltraVNC internals and comparisons.

---

## Desktop Capture Methods

### UltraVNC Capture Architecture

UltraVNC uses a **layered capture architecture** with multiple methods depending on Windows version and configuration:

#### 1. DXGI Desktop Duplication (Windows 8+) - Primary Method

**Files:** `DeskdupEngine.cpp/.h`, external `ddengine.dll`/`ddengine64.dll`

**How it works:**
- Loads `ddengine.dll` which uses the Windows **DXGI Desktop Duplication API**
- Desktop Duplication API provides GPU-accelerated screen capture
- Framebuffer and change list shared via memory-mapped files
- Events signal when screen/pointer changes occur
- Captures directly from GPU compositor - efficient for modern Windows

```cpp
// Shared memory for framebuffer and change tracking
hFileMapBitmap = OpenFileMapping(..., g_szIPCSharedMMFBitmap);
pFramebuffer = (PCHAR)MapViewOfFile(hFileMapBitmap, ...);
pChangebuf = (CHANGES_BUF*)MapViewOfFile(hFileMap, ...);  // Changed rectangles
```

#### 2. VNC Hooks DLL - Change Detection

**Files:** `winvnc/vnchooks/VNCHooks.cpp`

**How it works:**
- Uses Windows hooks (`SetWindowsHookEx`) to intercept system messages
- Hooks `WH_CALLWNDPROC`, `WH_GETMESSAGE`, `WH_CBT` (dialog)
- Detects when windows are painted/moved/resized
- Sends `UpdateRectMessage` to server with changed rectangle coordinates
- Does NOT capture pixels - only detects what regions changed

```cpp
// Hook types used:
HHOOK hCallWndHook;    // Intercepts window messages
HHOOK hGetMsgHook;     // Intercepts GetMessage() calls
HHOOK hDialogMsgHook;  // Intercepts dialog creation
```

#### 3. GDI BitBlt - Pixel Capture Fallback

**Files:** `vncdesktop.cpp` - `PixelCaptureEngine` class

**How it works:**
- Gets device context for desktop: `GetDC(NULL)`
- Uses `BitBlt()` to copy pixels from screen to memory bitmap
- Used as fallback when Desktop Duplication unavailable
- Also used to capture specific rectangles identified by hooks

```cpp
// Capture a rectangle from screen
BOOL blitok = BitBlt(m_hmemdc,           // Destination memory DC
                     0, 0,                // Dest position
                     rect.width(), rect.height(),
                     m_hrootdc_Desktop,   // Source = desktop DC
                     rect.tl.x, rect.tl.y,
                     SRCCOPY);
```

#### Capture Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Windows 8+                              │
│  ┌──────────────────┐                                       │
│  │ Desktop Duplication│ <── GPU provides framebuffer        │
│  │ (ddengine.dll)    │      + dirty rectangles              │
│  └────────┬─────────┘                                       │
│           v                                                 │
│  ┌──────────────────┐                                       │
│  │ Shared Memory     │ <── Framebuffer + CHANGES_BUF        │
│  │ (memory-mapped)   │                                      │
│  └────────┬─────────┘                                       │
└───────────┼─────────────────────────────────────────────────┘
            v
┌───────────────────────────────────────────────────────────────┐
│                    vncDesktop                                 │
│  ┌──────────────────┐    ┌──────────────────┐                │
│  │ m_screenCapture   │    │ vnchooks.dll     │                │
│  │ (DeskDupEngine)   │    │ (change detect)  │                │
│  └────────┬─────────┘    └────────┬─────────┘                │
│           v                       v                          │
│  ┌──────────────────────────────────────────┐                │
│  │         PixelCaptureEngine               │                │
│  │         (GDI BitBlt fallback)            │                │
│  └────────┬─────────────────────────────────┘                │
│           v                                                  │
│  ┌──────────────────┐                                        │
│  │ m_membitmap       │ <── Local framebuffer copy            │
│  │ (DIB Section)     │                                       │
│  └────────┬─────────┘                                        │
└───────────┼──────────────────────────────────────────────────┘
            v
     Encode & Send to Clients
```

#### Configuration Options

From settings, users can choose:
- **Hook dll** (`m_hookdll`) - Use vnchooks.dll for change detection
- **Hook driver** (`m_hookdriver`) - Use Desktop Duplication / video driver
- Fallback to polling + GDI capture if hooks unavailable

#### Key Classes

| Class | Purpose |
|-------|---------|
| `ScreenCapture` | Abstract base class for capture engines |
| `DeskDupEngine` | DXGI Desktop Duplication implementation |
| `PixelCaptureEngine` | GDI BitBlt capture (in vncdesktop.cpp) |
| `vncDesktop` | Orchestrates capture, owns framebuffer |
| `vnchooks.dll` | Separate DLL for system-wide hooks |

---

## TightVNC Comparison

TightVNC uses **similar but simpler** capture methods:

### Capture Method Comparison

| Method | TightVNC | UltraVNC |
|--------|----------|----------|
| **GDI BitBlt** | Yes - primary method | Yes - fallback |
| **Windows Hooks** | Yes - `WH_CALLWNDPROC` etc. | Yes - vnchooks.dll |
| **DXGI Desktop Duplication** | No (not in open source version) | Yes - ddengine.dll |
| **Mirror Driver** | Yes (legacy, pre-Win8) | Yes (legacy) |
| **Polling** | Yes - fallback | Yes - fallback |

### Key Differences

**TightVNC:**
- Primarily relies on **GDI capture + hooks** for change detection
- Simpler architecture - capture code integrated in main executable
- No separate capture DLL like UltraVNC's ddengine.dll
- Open source version lacks modern DXGI Desktop Duplication
- Commercial "TightVNC for Windows" may have additional capture methods

**UltraVNC:**
- More sophisticated with **layered capture engines**
- Desktop Duplication API for Windows 8+ (GPU-accelerated)
- Separate vnchooks.dll and ddengine.dll modules
- Better performance on modern Windows due to DXGI

### Shared Concepts

Both use the same fundamental approach:
1. **Detect changes** - via Windows hooks or polling
2. **Capture pixels** - via GDI BitBlt from desktop DC
3. **Encode & send** - RFB protocol with Tight/ZRLE/etc. encodings

The main innovation in UltraVNC is the **DXGI Desktop Duplication** path which captures directly from the GPU compositor, avoiding the CPU overhead of GDI BitBlt on modern systems.

---

## Windows Pre-Login Remote Access

UltraVNC can capture and control the Windows login screen, allowing remote users to log into a machine before anyone is physically logged in.

### Architecture Overview

#### 1. Windows Service (Session 0)

UltraVNC runs as a **Windows service** in Session 0 (isolated from user sessions). This is critical because:
- Services start before any user logs in
- Session 0 isolation (Vista+) means the service can't directly interact with user desktops
- The service monitors console sessions and spawns winvnc.exe instances

**Files:** `winvnc/winvnc/UltraVNCService.cpp`

```cpp
// Service monitors for active console session
DWORD dwSessionId = WTSGetActiveConsoleSessionId();

// Launches winvnc.exe in the target session using the session token
CreateProcessAsUser(hPToken, NULL, app_path, ...)
```

#### 2. Desktop Switching

Windows has multiple desktops that UltraVNC must follow:
- **Default** - Normal user desktop
- **Winlogon** - Login screen (Ctrl+Alt+Del prompt, password entry)
- **Screensaver** - Screen saver desktop

**Files:** `winvnc/winvnc/HelperFunctions.cpp` (desktopSelector namespace)

```cpp
// Check if thread is on the current input desktop
int InputDesktopSelected();

// Switch thread to follow the active input desktop
BOOL SelectDesktop(char* name, HDESK* new_desktop);
```

The server periodically calls `OpenInputDesktop()` and `SetThreadDesktop()` to follow whatever desktop is currently active, including the Winlogon desktop.

#### 3. Secure Attention Sequence (SAS) - Ctrl+Alt+Del

On Vista+ systems, sending Ctrl+Alt+Del to reach the login screen requires special handling.

**Files:** `winvnc/winvnc/cadthread.cpp`

```cpp
// Enable software SAS via registry
void Enable_softwareCAD() {
    // Sets HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
    //      SoftwareSASGeneration = 1
    RegSetValueEx(hkLocalKey, "SoftwareSASGeneration", 0, REG_DWORD, ...);
}

// Signal the service to call SendSAS()
HANDLE hShutdownEventcad = OpenEvent(EVENT_MODIFY_STATE, FALSE,
                                      "Global\\SessionEventUltraCad");
SetEvent(hShutdownEventcad);
```

The service receives this signal and calls the Windows `SendSAS()` API to trigger the actual Ctrl+Alt+Del sequence.

### Pre-Login Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Windows Service (Session 0)                   │
│                                                                  │
│  ┌──────────────────┐     ┌──────────────────────┐              │
│  │ monitorSessions()│     │ Handles SAS Event     │              │
│  │ - WTSGetActive   │     │ "SessionEventUltraCad"│              │
│  │   ConsoleSession │     │ - Calls SendSAS()     │              │
│  └────────┬─────────┘     └──────────────────────┘              │
└───────────┼──────────────────────────────────────────────────────┘
            │ CreateProcessAsUser()
            v
┌───────────────────────────────────────────────────────────────────┐
│                    winvnc.exe (User Session)                      │
│                                                                   │
│  ┌──────────────────┐     ┌──────────────────────┐               │
│  │ vncDesktop       │     │ desktopSelector      │               │
│  │ - Screen capture │     │ - OpenInputDesktop() │               │
│  │ - Follows active │<--->│ - SetThreadDesktop() │               │
│  │   input desktop  │     │ - Detects: Default,  │               │
│  └──────────────────┘     │   Winlogon, Screensaver│             │
│                           └──────────────────────┘               │
└───────────────────────────────────────────────────────────────────┘
            │
            v
     ┌──────────────────────┐
     │ Desktop States:       │
     │ - Default (logged in) │
     │ - Winlogon (login)    │
     │ - Screensaver         │
     └──────────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `UltraVNCService.cpp` | Service in Session 0, monitors sessions, spawns winvnc.exe |
| `HelperFunctions.cpp` | desktopSelector namespace for desktop switching |
| `cadthread.cpp` | Ctrl+Alt+Del / SAS handling |
| `credentials.h` | DesktopUsersToken for session token management |

### How Remote Login Works

This architecture allows a remote user to:
1. **Connect before anyone is logged in** - Service is always running
2. **See the Windows login screen** - Desktop switching follows Winlogon
3. **Send Ctrl+Alt+Del** - SAS registry key + SendSAS() API
4. **Type credentials** - Keyboard input works on all desktops
5. **Log in remotely** - Full access once authenticated

### Requirements

- UltraVNC must be installed as a **Windows service**
- **SoftwareSASGeneration** registry key must be enabled (for Ctrl+Alt+Del)
- Service needs appropriate permissions for CreateProcessAsUser

---

## References

### Desktop Capture
- `winvnc/winvnc/vncdesktop.cpp` - Main desktop capture orchestration
- `winvnc/winvnc/DeskdupEngine.cpp` - DXGI Desktop Duplication wrapper
- `winvnc/winvnc/ScreenCapture.h` - Abstract capture interface
- `winvnc/vnchooks/VNCHooks.cpp` - Windows hooks for change detection

### Pre-Login / Service
- `winvnc/winvnc/UltraVNCService.cpp` - Windows service, session monitoring
- `winvnc/winvnc/HelperFunctions.cpp` - Desktop switching (desktopSelector)
- `winvnc/winvnc/cadthread.cpp` - Ctrl+Alt+Del / SAS handling
- `winvnc/winvnc/credentials.h` - Session token management
