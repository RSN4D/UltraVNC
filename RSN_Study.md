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

## Minimal VNC Implementation Analysis

Analysis of what's required to build a minimal ImGui-based VNC viewer/server with only core features: image, connection, login, and input handling.

### Feature Scope

| Feature | Include | Exclude |
|---------|---------|---------|
| **Image/Framebuffer** | Yes | - |
| **Connection/Protocol** | Yes | - |
| **Authentication** | VNC password only | MS Logon, RSA-AES, TLS |
| **Input (mouse/keyboard)** | Yes | Touch/Gii |
| **File Transfer** | No | All |
| **Text Chat** | No | All |
| **Clipboard** | Basic only | Extended formats |
| **Recording/Snapshot** | No | All |

### Core Components Required

#### Viewer (~15,000 lines minimal)

| Component | Files | Lines | Notes |
|-----------|-------|-------|-------|
| Protocol/Connection | `ClientConnection.cpp/h` | ~10,400 | **Monolithic - needs refactoring** |
| Encoders | `ClientConnection*.cpp` (Raw, Hextile, CoRRE, Zlib) | ~1,400 | Well separated |
| Network I/O | `rdr/*` (streams library) | ~1,000 | Clean abstraction |
| Input/Keyboard | `KeyMap.cpp/h` | ~2,000 | Mostly mapping tables |
| Auth Crypto | `vncauth.h`, `d3des.h` | ~500 | VNC password only |

#### Server (~20,000 lines minimal)

| Component | Files | Lines | Notes |
|-----------|-------|-------|-------|
| Server Core | `vncserver.cpp/h` | ~7,000 | Client management |
| Client Handler | `vncclient.cpp/h` | ~6,800 | **Monolithic - needs refactoring** |
| Desktop Capture | `vncdesktop.cpp/h`, `DeskdupEngine.cpp` | ~5,000 | Clean `ScreenCapture` interface |
| Encoders | `vncencoder*.cpp` (basic set) | ~3,000 | Modular |
| Input Injection | `MouseSimulator.cpp`, `vnckeymap.cpp` | ~2,000 | Uses Windows `SendInput()` |
| Socket | `vncsockconnect.cpp/h` | ~500 | Clean |

### Code Separation Analysis

#### The Problem: Two "God Objects"

```
Viewer: ClientConnection.cpp (10,426 lines)
        └── Protocol + Auth + Rendering + Input + FileTransfer + Chat + Clipboard
            ALL IN ONE CLASS - 87 private member variables

Server: vncclient.cpp (6,840 lines)
        └── Protocol + Encoding + Input + FileTransfer + Chat + Clipboard
            ALL IN ONE CLASS
```

#### Separation Difficulty by Feature

| Feature | Current State | Extraction Effort |
|---------|---------------|-------------------|
| Encoders | Separate files per encoding | **Easy** - already modular |
| Desktop Capture | `ScreenCapture` interface exists | **Easy** - clean abstraction |
| Authentication | Mixed in main classes | **Medium** - extract methods |
| Input Handling | Methods in main classes | **Medium** - extract to handler |
| File Transfer | Woven into vncclient | **Hard** - deeply embedded |
| Text Chat | Mixed in multiple classes | **Hard** - shared message loop |

**Overall Separation Difficulty: 7/10**

### Files to Keep vs. Exclude

#### KEEP (Core)

**Viewer:**
```
vncviewer/
  ClientConnection.cpp/h     (refactor to remove optional features)
  ClientConnectionRaw.cpp    (Raw encoding)
  ClientConnectionHextile.cpp
  ClientConnectionCoRRE.cpp
  ClientConnectionZlib.cpp
  KeyMap.cpp/h
  rdr/*                      (network streams)

common/
  vncauth.h, d3des.h         (VNC password crypto)
  rfb.h                      (protocol definitions)
```

**Server:**
```
winvnc/winvnc/
  vncserver.cpp/h            (refactor)
  vncclient.cpp/h            (refactor)
  vncdesktop.cpp/h
  vncdesktopthread.cpp/h
  DeskdupEngine.cpp/h
  ScreenCapture.cpp/h
  vncencoder.cpp/h           (base encoder)
  vncencodehext.cpp/h        (Hextile)
  vncencoderre.cpp/h         (RRE)
  vncencodecorre.cpp/h       (CoRRE)
  MouseSimulator.cpp/h
  vnckeymap.cpp/h
  vncsockconnect.cpp/h
  vncbuffer.cpp/h
```

#### EXCLUDE (Optional Features)

**Both:**
```
FileTransfer.cpp/h           (~5,000 lines)
TextChat.cpp/h               (~25,000 lines server, ~800 lines viewer)
All UI dialogs               (SessionDialog, PropertiesDialog, vncmenu, etc.)
Cloud/UDT files              (disabled, non-functional)
Extended clipboard
```

**Viewer-specific:**
```
Snapshot.cpp/h
Daemon.cpp/h
VNCviewerApp.cpp/h           (MFC framework)
directx/*                    (optional renderer)
```

**Server-specific:**
```
vncmenu.cpp/h                (tray icon UI)
PropertiesDialog.cpp/h       (~77,000 lines!)
videodriver.cpp/h            (legacy capture)
UltraVNCService.cpp          (if not running as service)
All *Dialog.cpp files
```

### Recommended Implementation Approach

#### Option A: Wrapper Approach (Faster, Recommended for Prototype)

Keep existing code, add thin wrapper for ImGui:

```cpp
// Minimal viewer wrapper
class ImGuiVncViewer {
    ClientConnection* m_conn;  // Use existing monolith
public:
    bool Connect(const char* host, int port, const char* password);
    void Disconnect();
    bool IsConnected();

    // Framebuffer for ImGui texture
    void* GetFramebufferBits();
    int GetWidth();
    int GetHeight();

    // Input forwarding
    void SendKeyEvent(int keysym, bool down);
    void SendPointerEvent(int x, int y, int buttonMask);

    // Call in render loop
    void ProcessUpdates();
};
```

```cpp
// Minimal server wrapper
class ImGuiVncServer {
    vncServer* m_server;
public:
    bool Start(int port, const char* password);
    void Stop();

    int GetClientCount();
    void DisconnectAllClients();

    // Status for UI
    bool IsRunning();
    std::vector<ClientInfo> GetClients();
};
```

#### Option B: Extract & Refactor (Cleaner, More Work)

Create new abstraction layer:

```
┌─────────────────────────────────────────────────────────┐
│                    ImGui Application                     │
├─────────────────────────────────────────────────────────┤
│  IVncViewer              │  IVncServer                  │
│  - Connect()             │  - Start()                   │
│  - GetFramebuffer()      │  - GetClientCount()          │
│  - SendKeyEvent()        │  - OnClientConnect callback  │
│  - SendPointerEvent()    │                              │
├─────────────────────────────────────────────────────────┤
│  Extracted Core Classes                                  │
│  - RfbProtocol (from ClientConnection)                  │
│  - FramebufferManager                                    │
│  - InputHandler                                          │
│  - AuthManager (VNC password only)                       │
└─────────────────────────────────────────────────────────┘
```

### Prototype Development Path

1. **Start with Viewer** - Less complex than server
2. **Use existing `ClientConnection`** - Bypass Win32 UI, not replace it yet
3. **Create `ImGuiVncViewer` wrapper** that:
   - Calls `ClientConnection::Connect()`
   - Gets framebuffer pointer → uploads to ImGui texture
   - Forwards ImGui input → `SendKeyEvent()`/`SendPointerEvent()`
4. **Compile out optional features** via preprocessor defines
5. **Test with existing UltraVNC server** first
6. **Then tackle server wrapper** once viewer works

### Encoding Recommendations

For minimal implementation, support these encodings:

| Encoding | Priority | Complexity | Notes |
|----------|----------|------------|-------|
| Raw | Required | Low | Baseline, always works |
| Hextile | Required | Medium | Good compression, widely supported |
| CoRRE | Optional | Low | Simple RLE variant |
| Zlib | Optional | Medium | Better compression |
| Tight | Skip initially | High | Complex, JPEG support needed |
| Ultra | Skip | High | Proprietary |

### Threading Considerations

```
Viewer Threading:
┌─────────────────┐     ┌─────────────────┐
│ Protocol Thread │     │  ImGui Thread   │
│ (existing)      │     │  (new)          │
│                 │     │                 │
│ - Receive data  │     │ - Render UI     │
│ - Decode frames │     │ - Handle input  │
│ - Update buffer │────>│ - Upload texture│
│                 │     │ - Display frame │
└─────────────────┘     └─────────────────┘
        │                       │
        └───── Mutex on ────────┘
              framebuffer

Server Threading:
┌─────────────────┐     ┌─────────────────┐
│ Server Threads  │     │  ImGui Thread   │
│ (existing)      │     │  (new)          │
│                 │     │                 │
│ - Capture       │     │ - Status UI     │
│ - Encode        │────>│ - Client list   │
│ - Send          │     │ - Settings      │
└─────────────────┘     └─────────────────┘
        │                       │
        └─── Event queue ───────┘
```

---

## Build vs. Refactor Decision

**The Question:** Should we refactor existing UltraVNC or build a new implementation using UltraVNC as reference?

### Option 1: Refactor Existing UltraVNC

**Pros:**
- Already works - 20+ years of edge cases handled
- Battle-tested on real Windows systems
- DXGI capture, pre-login, UAC all working
- Encoders are already modular

**Cons:**
- Two 10,000+ line "god objects" to untangle
- No tests - refactoring blind is dangerous
- Old C++ style (no RAII, raw pointers, manual memory)
- Win32 assumptions baked deep
- Risk of breaking working code
- Inherit all the technical debt

**Estimated effort:** 9-14 weeks
**Risk level:** HIGH - surgery on working code without tests

### Option 2: Build New Using UltraVNC as Reference

**Pros:**
- Clean architecture from day one
- Modern C++ (smart pointers, std::thread, etc.)
- ImGui designed in, not bolted on
- Only implement what you need
- Can write tests as you go
- Cherry-pick UltraVNC's clean parts
- Easier to understand and maintain long-term

**Cons:**
- More initial work
- May miss edge cases UltraVNC handles
- Need to relearn some Windows quirks

**Estimated effort:** 11-17 weeks
**Risk level:** MEDIUM - new code but controlled

### Recommendation: Build New, Cherry-Pick the Good Parts

#### What to BUILD FRESH

The RFB protocol is well-documented (RFC 6143) and not complex:

```
New Core:
├── RfbProtocol.cpp/h        # Fresh implementation from RFC 6143
├── Framebuffer.cpp/h        # Simple pixel buffer
├── VncViewer.cpp/h          # Clean viewer class
├── VncServer.cpp/h          # Clean server class
└── ImGuiApp.cpp/h           # Native ImGui integration
```

UltraVNC's complexity comes from 20 years of feature creep, not protocol complexity.

#### What to COPY/ADAPT from UltraVNC

These components are already clean and modular:

| Component | UltraVNC Source | Why It's Clean |
|-----------|-----------------|----------------|
| **DXGI Capture** | `DeskdupEngine.cpp`, `ScreenCapture.h` | Already abstracted, interface-based |
| **Encoders** | `vncencoder*.cpp` | Modular, one file per encoding |
| **Input Injection** | `MouseSimulator.cpp` | Fairly standalone |
| **Keyboard Maps** | `vnckeymap.cpp`, `KeyMap.cpp` | Data tables, easy to extract |
| **DES Crypto** | `d3des.c/h` | Standard algorithm, standalone |
| **Desktop Switching** | `HelperFunctions.cpp` | desktopSelector is isolated |

#### What to REFERENCE but REWRITE

| Component | Why Rewrite |
|-----------|-------------|
| Protocol handling | Clean slate, modern C++ |
| Authentication flow | Simpler if VNC-password only |
| Client/Server classes | Design for your needs |
| Threading model | Design for ImGui from start |

### Comparison Summary

| Approach | Effort | Risk | Maintainability | Recommendation |
|----------|--------|------|-----------------|----------------|
| Refactor UltraVNC | 9-14 weeks | HIGH | Poor | No |
| Build New + Cherry-pick | 11-17 weeks | MEDIUM | Good | **Yes** |

### Suggested Development Phases

```
Phase 1: Minimal Viewer (4-6 weeks)
├── RFB protocol client (handshake, auth, framebuffer updates)
├── Raw + Hextile encoding (copy from UltraVNC)
├── Basic input sending
├── ImGui window with framebuffer texture
└── Test against existing UltraVNC server

Phase 2: Minimal Server (4-6 weeks)
├── RFB protocol server
├── Copy DeskdupEngine + ScreenCapture from UltraVNC
├── Copy encoders from UltraVNC
├── Copy MouseSimulator from UltraVNC
└── Test with standard VNC viewers

Phase 3: Windows Integration (2-3 weeks)
├── Service wrapper (reference UltraVNCService.cpp)
├── Pre-login support (reference HelperFunctions.cpp)
├── SAS/Ctrl+Alt+Del (reference cadthread.cpp)
└── Polish and testing
```

### Why This Approach

1. **Similar effort, lower risk** - Estimates are close (9-14 vs 11-17 weeks), but refactoring without tests is gambling

2. **The protocol is the easy part** - RFB is well-documented. UltraVNC's complexity is accidental, not essential

3. **The hard parts are extractable** - DXGI capture, encoders, input injection are already modular in UltraVNC

4. **Future-proof** - Clean codebase you fully understand vs. inherited complexity

5. **Incremental development** - Build viewer first (connect to existing UltraVNC server), then server

---

## Viewer Abstraction Layer & Browser Delivery

Design for a flexible viewer that can output to multiple targets: ImGui (desktop), browser (WebRTC/WebSocket), or other backends.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      VNC Protocol Core                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ RfbClient   │  │ Decoders    │  │ Auth        │             │
│  │ (network)   │  │ (Hextile..) │  │ (VNC pass)  │             │
│  └──────┬──────┘  └──────┬──────┘  └─────────────┘             │
│         │                │                                      │
│         └────────┬───────┘                                      │
│                  ▼                                              │
│         ┌─────────────────┐                                     │
│         │  Framebuffer    │  ← Raw pixels live here             │
│         └────────┬────────┘                                     │
└──────────────────┼──────────────────────────────────────────────┘
                   │
          ┌────────▼────────┐
          │ IFrameConsumer  │  ← Abstract interface (output)
          │ IInputSource    │  ← Abstract interface (input)
          └────────┬────────┘
                   │
    ┌──────────────┼──────────────┬──────────────────┐
    │              │              │                  │
┌───▼───┐    ┌─────▼─────┐  ┌─────▼─────┐    ┌──────▼──────┐
│ ImGui │    │  WebRTC   │  │ WebSocket │    │    SDL2     │
│Backend│    │  Backend  │  │ + Canvas  │    │   Backend   │
└───────┘    └───────────┘  └───────────┘    └─────────────┘
 Desktop       Browser        Browser          Cross-platform
              (low latency)   (simple)          Desktop
```

### Interface Design

```cpp
// Output: How frames get displayed
class IFrameConsumer {
public:
    virtual ~IFrameConsumer() = default;

    // Called when framebuffer size changes
    virtual void OnResize(int width, int height, PixelFormat format) = 0;

    // Called when a region is updated (dirty rect)
    virtual void OnFrameUpdate(int x, int y, int w, int h,
                               const void* pixels, int stride) = 0;

    // Called when cursor changes (optional)
    virtual void OnCursorUpdate(const CursorInfo& cursor) = 0;

    // Called each frame (for streaming backends)
    virtual void Flush() = 0;
};

// Input: How mouse/keyboard events are received
class IInputSource {
public:
    virtual ~IInputSource() = default;

    // Poll for pending input events
    virtual bool PollEvent(InputEvent& event) = 0;

    // Or callback-based
    virtual void SetEventCallback(std::function<void(const InputEvent&)> cb) = 0;
};

// Combined for convenience
class IViewerBackend : public IFrameConsumer, public IInputSource {
public:
    virtual bool Initialize(int width, int height) = 0;
    virtual void Shutdown() = 0;
    virtual bool IsRunning() = 0;
};
```

### Usage Example

```cpp
// Core viewer - backend agnostic
class VncViewer {
public:
    VncViewer(std::unique_ptr<IViewerBackend> backend);

    bool Connect(const std::string& host, int port, const std::string& password);
    void Disconnect();
    void Run();  // Main loop - calls backend

private:
    std::unique_ptr<IViewerBackend> m_backend;
    std::unique_ptr<RfbClient> m_protocol;
    Framebuffer m_framebuffer;
};

// Usage - ImGui desktop app
auto viewer = VncViewer(std::make_unique<ImGuiBackend>());
viewer.Connect("192.168.1.100", 5900, "secret");
viewer.Run();

// Usage - WebSocket bridge for browser
auto viewer = VncViewer(std::make_unique<WebSocketBackend>(8080));
viewer.Connect("192.168.1.100", 5900, "secret");
viewer.Run();  // Serves frames to browser clients
```

### Browser Delivery Options

#### Option 1: WebRTC (Lowest Latency)

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│ VNC Server  │────▶│  VNC-to-WebRTC  │────▶│   Browser   │
│             │ RFB │  Bridge Server  │WebRTC│  (JS client)│
└─────────────┘     └─────────────────┘     └─────────────┘
                           │
                    Encodes to H.264/VP8
                    Handles STUN/TURN
```

| Aspect | Details |
|--------|---------|
| **Latency** | Lowest (sub-100ms possible) |
| **Pros** | Hardware decode in browser, P2P capable, congestion control |
| **Cons** | Complex (STUN/TURN servers), encoding overhead |
| **Use when** | Interactive remote desktop, latency-critical |

#### Option 2: WebSocket + Canvas (Simple, like noVNC)

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│ VNC Server  │────▶│  WebSocket      │────▶│   Browser   │
│             │ RFB │  Proxy/Bridge   │ WS  │  Canvas API │
└─────────────┘     └─────────────────┘     └─────────────┘
                           │
                    Forwards RFB or
                    converts to images
```

| Aspect | Details |
|--------|---------|
| **Latency** | Medium (100-300ms typical) |
| **Pros** | Simple, noVNC proves it works, no special infrastructure |
| **Cons** | No hardware acceleration, more bandwidth |
| **Use when** | Admin panels, casual access, simplicity preferred |

#### Option 3: WebSocket + WebCodecs (Modern)

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│ VNC Server  │────▶│  Encoding       │────▶│   Browser   │
│             │ RFB │  Bridge (H.264) │ WS  │  WebCodecs  │
└─────────────┘     └─────────────────┘     └─────────────┘
                           │
                    Encodes to H.264
                    Sends over WebSocket
```

| Aspect | Details |
|--------|---------|
| **Latency** | Low-Medium |
| **Pros** | Hardware decode, simpler than WebRTC |
| **Cons** | Limited browser support (Chrome/Edge mainly) |
| **Use when** | Modern browsers only, good performance without WebRTC complexity |

#### Option 4: Use Existing noVNC

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│ Your VNC    │────▶│  websockify     │────▶│   noVNC     │
│ Server      │ RFB │  (proxy)        │ WS  │  (existing) │
└─────────────┘     └─────────────────┘     └─────────────┘
```

| Aspect | Details |
|--------|---------|
| **Latency** | Medium |
| **Pros** | Already exists, zero browser development, well-tested |
| **Cons** | Less control over UI/UX |
| **Use when** | Quick solution, don't need custom browser UI |

### Browser Delivery Comparison

| Option | Latency | Complexity | Browser Support | Recommendation |
|--------|---------|------------|-----------------|----------------|
| WebRTC | Best | High | Good | For interactive use |
| WebSocket + Canvas | Medium | Low | Universal | **Start here** |
| WebSocket + WebCodecs | Good | Medium | Limited | Future option |
| noVNC (existing) | Medium | None | Universal | Quick prototype |

### WebSocket Frame Protocol

Simple binary protocol for WebSocket communication:

```
Server → Browser (Frame Updates):
  0x01 - Resize     : u8 type, u16 width, u16 height, u8 format
  0x02 - FrameUpdate: u8 type, u16 x, u16 y, u16 w, u16 h, bytes pixels
  0x03 - Cursor     : u8 type, u16 hotX, u16 hotY, u8 w, u8 h, bytes pixels

Browser → Server (Input Events):
  0x10 - KeyEvent    : u8 type, u32 keysym, u8 down
  0x11 - PointerEvent: u8 type, u16 x, u16 y, u8 buttons
```

### Browser Client Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     C++ Server Process                          │
│                                                                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐    │
│  │ RfbClient   │───▶│ Framebuffer │───▶│ WebSocketBackend│    │
│  │ (to VNC srv)│    │             │    │ (serves browser)│    │
│  └─────────────┘    └─────────────┘    └────────┬────────┘    │
│                                                  │             │
└──────────────────────────────────────────────────┼─────────────┘
                                                   │ WebSocket
                                                   ▼
┌────────────────────────────────────────────────────────────────┐
│                        Browser                                  │
│  ┌─────────────────────────────────────────────────────┐      │
│  │                    JavaScript                        │      │
│  │  - Connect to WebSocket server                      │      │
│  │  - Receive frame updates (binary ArrayBuffer)       │      │
│  │  - Draw to <canvas> using putImageData()            │      │
│  │  - Capture mouse events (mousemove, mousedown, etc) │      │
│  │  - Capture keyboard events (keydown, keyup)         │      │
│  │  - Send input events back over WebSocket            │      │
│  └─────────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────────┘
```

### Complexity Assessment

| Component | Lines (est.) | Complexity |
|-----------|--------------|------------|
| IViewerBackend interface | ~100 | Low |
| ImGuiBackend implementation | ~300 | Low |
| WebSocketBackend implementation | ~500 | Medium |
| Browser JavaScript client | ~400 | Medium |
| **Total abstraction overhead** | **~1,300** | **Low-Medium** |

### Recommendation

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Add abstraction layer?** | Yes | Minimal overhead (~1,000 lines), high flexibility |
| **Primary backend** | ImGui | Develop and test first |
| **Browser backend** | WebSocket + Canvas | Start simple, proven approach |
| **Future option** | WebRTC | Add later if latency becomes critical |

### Development Order

```
Phase 1: Core + ImGui Backend
├── Implement IViewerBackend interface
├── Implement ImGuiBackend
├── Test with VNC server
└── Validate abstraction works

Phase 2: WebSocket Backend
├── Implement WebSocketBackend
├── Create minimal browser JS client
├── Test frame streaming
└── Test input handling

Phase 3: Optimization (if needed)
├── Add WebCodecs support
├── Consider WebRTC for low-latency needs
└── Optimize frame encoding/compression
```

---

## File Transfer Implementation

Analysis of how UltraVNC implements bidirectional file transfer.

### Protocol Overview

File transfer uses **RFB message type 7** (`rfbFileTransfer`) with 17 different subtypes. It runs over the **same socket** as VNC traffic - not a separate channel.

```
┌─────────────────────────────────────────────────────────────────┐
│                    RFB Message Type 7 Header                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ type (1B) │ contentType (1B) │ contentParam (2B) │      │   │
│  │ size (4B) │ length (4B)      │ data[length]...   │      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                    12 bytes header + variable payload           │
└─────────────────────────────────────────────────────────────────┘
```

### Message Types

| Value | Name | Direction | Purpose |
|-------|------|-----------|---------|
| 1 | `rfbDirContentRequest` | C→S | Request directory listing |
| 2 | `rfbDirPacket` | S→C | Directory/file entry |
| 3 | `rfbFileTransferRequest` | C→S | Request file download |
| 4 | `rfbFileHeader` | Both | File metadata (size, name) |
| 5 | `rfbFilePacket` | Both | File chunk (8KB default) |
| 6 | `rfbEndOfFile` | Both | Transfer complete |
| 7 | `rfbAbortFileTransfer` | Both | Cancel transfer |
| 8 | `rfbFileTransferOffer` | C→S | Offer file for upload |
| 9 | `rfbFileAcceptHeader` | S→C | Accept/reject upload |
| 10 | `rfbCommand` | C→S | Directory ops (create, delete, rename) |
| 11 | `rfbCommandReturn` | S→C | Command response |
| 12 | `rfbFileChecksums` | Both | Delta transfer checksums |
| 14 | `rfbFileTransferAccess` | S→C | Permission grant/deny |
| 15 | `rfbFileTransferSessionStart` | C→S | FT GUI opened |
| 16 | `rfbFileTransferSessionEnd` | C→S | FT GUI closed |
| 17 | `rfbFileTransferProtocolVersion` | S→C | Protocol version negotiation |

### Transfer Sequences

#### Upload (Client → Server)

```
Client                              Server
   │                                   │
   │── rfbFileTransferOffer ──────────▶│ (filename, size, timestamp)
   │                                   │ [check permissions, disk space]
   │◀── rfbFileChecksums ─────────────│ (if delta transfer enabled)
   │◀── rfbFileAcceptHeader ──────────│ (0=ok, -1=error)
   │                                   │
   │── rfbFilePacket ─────────────────▶│ × N chunks (8KB each)
   │── rfbFilePacket ─────────────────▶│
   │── ...                             │
   │── rfbEndOfFile ──────────────────▶│
   │                                   │ [finalize, rename temp file]
```

#### Download (Server → Client)

```
Client                              Server
   │                                   │
   │── rfbFileTransferRequest ────────▶│ (filename)
   │                                   │ [open file, check permissions]
   │◀── rfbFileHeader ────────────────│ (size, or -1 if error)
   │                                   │
   │◀── rfbFilePacket ────────────────│ × N chunks
   │◀── rfbFilePacket ────────────────│
   │◀── ...                            │
   │◀── rfbEndOfFile ─────────────────│
   │ [finalize file]                   │
```

### Key Features

#### 1. Zlib Compression

```cpp
// Compression decision based on network speed
if (networkSpeed > 2048)  // Kbit/s threshold
    m_fCompressionEnabled = false;

// Per-packet indicator in size field:
//   0 = uncompressed raw data
//   1 = zlib compressed
//   2 = delta skip (data already exists at destination)
```

#### 2. Delta Transfer (Adler-32 Checksums)

Skips transferring blocks that already exist at destination:

```cpp
// Server generates checksums for existing destination file
for each 8KB chunk in destination_file:
    checksum = adler32(chunk_data)
    append to rfbFileChecksums message

// During transfer, compare source vs destination checksums
if (source_chunk_checksum == dest_chunk_checksum)
    send empty packet with size=2  // Skip this chunk
    // Client seeks forward instead of writing
```

#### 3. Directory Transfer

- Directories are **zipped** before transfer (prefix: `!UVNCDIR-`)
- Uses `ZipUnZip32` library
- Automatically **unzipped** on receiving side

#### 4. Temporary Files

- Uploads use temp prefix: `!UVNCPFT-`
- Renamed to final name only on successful completion
- Allows detection of partial/failed transfers

#### 5. Timestamp Preservation

- Embedded in offer message: `filename,MM-DD-YYYY,HH:MM`
- Restored on destination file after transfer completes

### Code Structure

#### Viewer Side (`vncviewer/FileTransfer.cpp`)

```cpp
class FileTransfer {
    ClientConnection* m_pCC;      // Connection reference
    CZipUnZip32* m_pZipUnZip;    // Directory compression

    // Upload state
    HANDLE m_hSrcFile;
    char m_szSrcFileName[MAX_PATH + 32];
    bool m_fFileUploadRunning;
    bool m_fCompress;

    // Download state
    HANDLE m_hDestFile;
    char m_szDestFileName[MAX_PATH + 32];
    bool m_fFileDownloadRunning;

    // Delta transfer
    char* m_lpCSBuffer;          // Checksum buffer
    int m_nCSBufferSize;

    // Key methods
    void ProcessFileTransferMsg();     // Message dispatcher
    bool SendFileChunk();              // Send upload chunk
    bool ReceiveFileChunk();           // Receive download chunk
    void SendFiles();                  // Initiate upload
    void ReceiveFiles();               // Initiate download
    void GenerateFileChecksums();      // Delta transfer
};
```

#### Server Side (`winvnc/winvnc/vncclient.cpp`)

File transfer state is **embedded in vncClient class** (not separate):

```cpp
class vncClient {
    // Mixed with all other client state...

    // Upload reception (client sending to server)
    HANDLE m_hDestFile;
    char m_szFullDestName[MAX_PATH];
    char* m_pBuff;               // 8KB chunk buffer
    char* m_pCompBuff;           // Compression buffer
    bool m_fFileDownloadRunning;

    // Download sending (server sending to client)
    HANDLE m_hSrcFile;
    bool m_fFileUploadRunning;
    bool m_fCompressionEnabled;

    // Delta transfer
    char* m_lpCSBuffer;
    int m_nCSBufferSize;

    // Key methods
    bool ReceiveFileChunk(int nLen, int nSize);
    bool SendFileChunk();
    void FinishFileReception();
    void FinishFileSending();
    int GenerateFileChecksums(HANDLE hFile, char* lpCSBuffer, int nCSBufferSize);
};
```

### File Locations

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| Viewer FT class | `vncviewer/FileTransfer.cpp` | ~3,000 | Complete viewer-side implementation |
| Viewer FT header | `vncviewer/FileTransfer.h` | ~220 | Class definition |
| Server FT handler | `vncclient.cpp:3688` | embedded | Message switch handler |
| Server receive chunk | `vncclient.cpp:5946` | ~80 | `ReceiveFileChunk()` |
| Server send chunk | `vncclient.cpp:6129` | ~120 | `SendFileChunk()` |
| Server checksums | `vncclient.cpp:5870` | ~50 | `GenerateFileChecksums()` |
| Protocol defs | `rfb/rfbproto.h:1119` | ~50 | Message type constants |

### Permission and Security

```cpp
// Server-side permission check (vncclient.cpp:3691)
bool fUserOk = true;
if (settings->getFTUserImpersonation()) {
    fUserOk = m_client->DoFTUserImpersonation();
}
if (!m_client->m_keyboardenabled || !m_client->m_pointerenabled) {
    fUserOk = false;  // No FT if input disabled
}
// Also checks: settings->getEnableFileTransfer()
```

**User Impersonation** (Windows-specific):
- When enabled, file operations run under logged-in user's token
- Allows access to UNC paths and mapped network drives

### Why File Transfer is Hard to Separate

| Issue | Details |
|-------|---------|
| **Embedded in vncclient** | 20+ member variables mixed with other client state |
| **Shared buffers** | Uses same compression buffers as screen encoding |
| **Same socket** | No separate channel - interleaved with framebuffer updates |
| **Windows-specific** | User impersonation, FILETIME handling |
| **State machine** | Complex multi-message sequences with error handling |

### Summary

| Aspect | Implementation |
|--------|----------------|
| **Protocol** | Custom RFB type 7, 17 subtypes |
| **Channel** | Same socket as VNC (not separate) |
| **Chunk size** | 8KB (`sz_rfbBlockSize`) |
| **Compression** | Zlib (optional, speed-based threshold) |
| **Delta transfer** | Adler-32 checksums per 8KB chunk |
| **Directories** | Auto zip/unzip via ZipUnZip32 |
| **Timestamps** | Preserved via embedded metadata |
| **Temp files** | `!UVNCPFT-` prefix until complete |
| **Code coupling** | HIGH - deeply embedded in vncclient |
| **Separation effort** | HARD - would require significant refactoring |

---

## High-Performance File Transfer Redesign

Analysis of rebuilding file transfer with modern techniques for secure, high-speed transfers (10Gbps+).

### Current UltraVNC Limitations

| Limitation | Impact |
|------------|--------|
| **Same socket as VNC** | Competes with screen updates, can't parallelize |
| **Single-threaded** | One chunk at a time, sequential |
| **8KB chunks** | Far too small for high-speed networks |
| **Zlib compression** | Slow compared to modern algorithms |
| **No encryption** | Unless VNC connection itself is encrypted |
| **Synchronous I/O** | Blocking reads/writes |
| **Single TCP stream** | Can't saturate high-bandwidth links |

**Theoretical max with current design:** ~100-200 MB/s

### Requirements for 10Gbps+ Speeds

| Technique | Why Needed |
|-----------|------------|
| **Parallel TCP streams** | Single TCP tops out ~2-5 Gbps due to congestion control |
| **Large buffers** | 1-4 MB chunks instead of 8KB |
| **Async I/O** | IOCP (Windows) or io_uring (Linux) |
| **Zero-copy** | Memory-mapped files, scatter/gather |
| **Modern compression** | LZ4/Zstd (10-50x faster than zlib) |
| **Separate channel** | Don't compete with VNC traffic |

### Proposed Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        Control Channel                              │
│                    (WebSocket or TCP + TLS)                        │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • File/directory listings                                     │ │
│  │ • Transfer requests (with file metadata)                      │ │
│  │ • Progress updates                                            │ │
│  │ • Checksums for delta/resume                                  │ │
│  │ • Error handling                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│                     Data Channel Pool                               │
│                   (Multiple parallel streams)                       │
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Stream 1 │  │ Stream 2 │  │ Stream 3 │  │ Stream N │          │
│  │ TLS 1.3  │  │ TLS 1.3  │  │ TLS 1.3  │  │ TLS 1.3  │          │
│  │ 1MB bufs │  │ 1MB bufs │  │ 1MB bufs │  │ 1MB bufs │          │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │
│       │              │              │              │               │
│       └──────────────┴──────────────┴──────────────┘               │
│                              │                                     │
│                    Async I/O (IOCP/io_uring)                      │
└────────────────────────────────────────────────────────────────────┘
```

### Protocol Design

#### Control Messages (JSON over WebSocket)

```json
// Request file listing
{ "type": "list", "path": "/home/user/documents" }

// Start transfer
{
    "type": "transfer_start",
    "id": "uuid",
    "files": [
        { "path": "/file1.zip", "size": 1073741824, "mtime": 1699999999 }
    ],
    "options": {
        "compression": "lz4",
        "encryption": "aes-gcm",
        "delta": true,
        "streams": 8,
        "chunk_size": 1048576
    }
}

// Progress update
{ "type": "progress", "id": "uuid", "bytes": 536870912, "speed": 1250000000 }

// Checksums for delta (xxHash3)
{
    "type": "checksums",
    "file": "/file1.zip",
    "chunk_size": 1048576,
    "hashes": ["abc123...", "def456...", ...]
}
```

#### Data Stream Packet Format

```
┌─────────────────────────────────────────────────────────────┐
│                    Data Packet Header (34 bytes)             │
├─────────────────────────────────────────────────────────────┤
│ transfer_id (16B) │ file_index (4B) │ chunk_index (4B)      │
│ flags (2B)        │ compressed_len (4B) │ original_len (4B) │
├─────────────────────────────────────────────────────────────┤
│                    Payload (up to 4MB)                       │
│              [compressed/encrypted data]                     │
└─────────────────────────────────────────────────────────────┘

Flags:
  0x01 = compressed (LZ4/Zstd)
  0x02 = last chunk of file
  0x04 = delta skip (chunk unchanged)
  0x08 = encrypted
```

### Core Implementation

```cpp
// Transfer manager - main API
class FileTransferManager {
public:
    // Configuration
    void SetParallelStreams(int count);      // Default: 4-8
    void SetChunkSize(size_t bytes);         // Default: 1MB
    void SetCompression(Compression type);   // None, LZ4, Zstd
    void SetEncryption(Encryption type);     // None, AES-GCM, ChaCha20

    // Operations
    TransferId StartUpload(const std::vector<FileInfo>& files,
                          const std::string& destPath,
                          TransferOptions options);

    TransferId StartDownload(const std::vector<std::string>& remotePaths,
                            const std::string& localPath,
                            TransferOptions options);

    void Pause(TransferId id);
    void Resume(TransferId id);
    void Cancel(TransferId id);

    // Progress
    TransferProgress GetProgress(TransferId id);
    void SetProgressCallback(std::function<void(TransferProgress)> cb);

private:
    std::unique_ptr<ControlChannel> m_control;
    std::vector<std::unique_ptr<DataStream>> m_streams;
    std::unique_ptr<ChunkScheduler> m_scheduler;
};

// Async data stream (one of N parallel)
class DataStream {
public:
    DataStream(const std::string& host, int port, const TlsConfig& tls);

    void SendChunkAsync(const Chunk& chunk,
                       std::function<void(bool)> callback);
    void ReceiveChunkAsync(std::function<void(Chunk)> callback);

private:
    std::unique_ptr<AsyncSocket> m_socket;  // IOCP-based
    std::unique_ptr<TlsContext> m_tls;
    RingBuffer m_sendBuffer;
    RingBuffer m_recvBuffer;
};

// Chunk scheduler - distributes work across streams
class ChunkScheduler {
public:
    void AddFile(const FileInfo& file);
    Chunk GetNextChunk();           // Thread-safe
    void MarkComplete(ChunkId id);
    void MarkFailed(ChunkId id);    // Will retry

private:
    std::atomic<uint64_t> m_nextChunk;
    ConcurrentQueue<Chunk> m_pending;
    ConcurrentMap<ChunkId, ChunkState> m_state;
};
```

### Compression Comparison

| Algorithm | Compress | Decompress | Ratio | Best For |
|-----------|----------|------------|-------|----------|
| **None** | ∞ | ∞ | 1.0x | Already compressed |
| **LZ4** | 780 MB/s | 4000 MB/s | 2.1x | Speed priority (default) |
| **LZ4 HC** | 40 MB/s | 4000 MB/s | 2.7x | Better ratio |
| **Zstd** | 500 MB/s | 1700 MB/s | 2.9x | Good balance |
| **Zstd -3** | 350 MB/s | 1700 MB/s | 3.1x | Higher ratio |
| **Zlib** | 50 MB/s | 300 MB/s | 2.7x | Legacy (current) |

**Recommendation:** LZ4 default, Zstd for slow networks.

### Encryption Options

| Algorithm | Speed | Security | Notes |
|-----------|-------|----------|-------|
| **AES-256-GCM** | 3-5 GB/s | Excellent | Hardware accelerated (AES-NI) |
| **ChaCha20-Poly1305** | 1-2 GB/s | Excellent | Good without AES-NI |
| **TLS 1.3** | ~same | Excellent | Use for control channel |

**Recommendation:** AES-256-GCM for data streams (hardware accelerated).

### Delta Transfer Improvements

| Aspect | Current (UltraVNC) | Improved |
|--------|-------------------|----------|
| **Checksum** | Adler-32 | xxHash3 (10x faster) |
| **Block size** | 8KB | 1MB (fewer checksums) |
| **Delivery** | Full list upfront | Streaming/rolling |
| **Resume** | None | From last good chunk |

```cpp
// Fast delta using xxHash3
class DeltaCalculator {
public:
    std::vector<uint64_t> GenerateChecksums(
        const std::string& path,
        size_t chunkSize = 1048576)
    {
        std::vector<uint64_t> checksums;
        MemoryMappedFile file(path);

        for (size_t offset = 0; offset < file.Size(); offset += chunkSize) {
            size_t len = std::min(chunkSize, file.Size() - offset);
            uint64_t hash = XXH3_64bits(file.Data() + offset, len);
            checksums.push_back(hash);
        }
        return checksums;
    }

    bool ChunkMatches(const void* data, size_t len, uint64_t expected) {
        return XXH3_64bits(data, len) == expected;
    }
};
```

### Performance Targets

| Network | Streams | Chunk Size | Expected Speed |
|---------|---------|------------|----------------|
| 1 Gbps | 2-4 | 256KB | ~110 MB/s |
| 10 Gbps | 4-8 | 1MB | ~1.1 GB/s |
| 25 Gbps | 8-16 | 2MB | ~2.8 GB/s |
| 100 Gbps | 16-32 | 4MB | ~10+ GB/s |

**Key insight:** Single TCP stream maxes out around 2-5 Gbps. Multiple parallel streams are essential for 10Gbps+.

### Feasibility Assessment

| Aspect | Difficulty | Notes |
|--------|------------|-------|
| **Basic transfer** | Easy | Simple protocol, well-understood |
| **Parallel streams** | Medium | Thread pool + async I/O |
| **TLS encryption** | Easy | Use OpenSSL/BoringSSL |
| **LZ4/Zstd compression** | Easy | Drop-in libraries |
| **Delta transfer** | Medium | xxHash3 + chunk tracking |
| **Resume support** | Medium | Persist chunk state |
| **10Gbps saturation** | Medium | Tuning, large buffers, IOCP |
| **100Gbps saturation** | Hard | Kernel bypass (DPDK), zero-copy |

### Development Estimate

| Component | Time |
|-----------|------|
| Core protocol + control channel | 1-2 weeks |
| Single-stream transfer | 1 week |
| Parallel streams + scheduling | 1-2 weeks |
| Compression (LZ4/Zstd) | 2-3 days |
| Encryption (TLS) | 3-5 days |
| Delta transfer | 1 week |
| Resume support | 3-5 days |
| UI integration | 1 week |
| **Total** | **6-9 weeks** |

### Recommended Libraries

| Purpose | Library | Notes |
|---------|---------|-------|
| Async I/O | libuv or Asio | Cross-platform |
| TLS | OpenSSL or BoringSSL | AES-NI support |
| Compression | lz4, zstd | C APIs, very fast |
| Hashing | xxHash | Extremely fast checksums |
| WebSocket | libwebsockets or Beast | Control channel |
| JSON | nlohmann/json or simdjson | Message parsing |

### Development Phases

```
Phase 1: Basic Transfer (2-3 weeks)
├── Control channel (WebSocket + JSON)
├── Single data stream with TLS
├── 1MB chunks
├── LZ4 compression
└── Target: 1 Gbps

Phase 2: High Performance (2-3 weeks)
├── Parallel streams (4-8)
├── Async I/O (IOCP on Windows)
├── Large buffers, zero-copy
└── Target: 10 Gbps

Phase 3: Advanced Features (2-3 weeks)
├── Delta transfer (xxHash3)
├── Resume support
├── Directory transfer
├── Progress/cancel/pause
└── Integration with viewer/server
```

### Recommendation

**Rebuild file transfer from scratch.** Rationale:

1. **Simpler than VNC** - File transfer is a well-understood problem
2. **Huge improvement potential** - Current implementation is 20+ years old
3. **Clean separation** - Can be completely independent module
4. **Reusable** - Same system works for VNC, standalone, browser
5. **Modern security** - TLS 1.3, AES-GCM encryption built-in
6. **High performance** - 10-100x faster than current implementation

---

## Existing C++ Libraries for File Transfer

Analysis of existing libraries that could accelerate development.

### Facebook WDT (Warp speed Data Transfer) ⭐ Best Match

**Repository:** [github.com/facebook/wdt](https://github.com/facebook/wdt)

Facebook built WDT for transferring RocksDB snapshots at scale. It's almost exactly what we need.

#### Features

| Feature | WDT Support |
|---------|-------------|
| **Parallel TCP streams** | ✅ Native - arbitrary number of threads/connections |
| **Encryption** | ✅ AES-GCM (OpenSSL) - default |
| **Compression** | ✅ LZ4 built-in |
| **Resume** | ✅ `-enable_download_resumption` |
| **High speed** | ✅ **40+ Gbps** demonstrated |
| **Embeddable** | ✅ Library API (Wdt.h) |
| **Delta transfer** | ❌ Not built-in |

#### Performance

- **600 MB/s** across Sweden → Oregon (high latency links)
- **4+ GB/s** on 40 Gbps NIC (near theoretical line rate)
- 3x faster than Facebook's previous HTTP-based solution

#### Dependencies

- C++11 compiler
- glog (Google logging)
- Parts of Facebook Folly
- OpenSSL (for encryption)

#### Example Usage

```cpp
#include "Wdt.h"

Wdt& wdt = Wdt::initializeWdt("app_name");
WdtTransferRequest req;
req.directory = "/source/path";
req.hostName = "destination.host";
req.destDirectory = "/dest/path";
req.transferId = GenerateUuid();

// Enable encryption and compression
req.encryptionType = EncryptionType::AES_GCM;
req.enableCompression = true;

// Start transfer
ErrorCode code = wdt.wdtSend(req);
```

---

### librsync - Delta Transfer Algorithm

**Repository:** [github.com/librsync/librsync](https://github.com/librsync/librsync)
**Documentation:** [librsync.github.io](https://librsync.github.io/)

Implements the rsync remote-delta algorithm for efficient file synchronization.

#### Features

| Feature | librsync Support |
|---------|------------------|
| **Delta transfer** | ✅ rsync algorithm |
| **Streaming API** | ✅ Similar to zlib |
| **Parallel streams** | ❌ Single-threaded |
| **Encryption** | ❌ Not included |
| **Compression** | ❌ Not included |

#### Used By

Dropbox, rdiff-backup, Duplicity

#### How It Works

```cpp
// 1. Receiver generates signature of existing file
rs_signature_t *sig;
rs_sig_file(old_file, sig_file, RS_DEFAULT_BLOCK_LEN, 0, RS_BLAKE2_SIG_MAGIC, NULL);
rs_build_hash_table(sig);

// 2. Sender computes delta between new file and signature
rs_delta_file(sig, new_file, delta_file, NULL);

// 3. Receiver applies delta to recreate new file
rs_patch_file(old_file, delta_file, new_file, NULL);
```

#### Best For

Adding delta/differential capability to another transfer solution.

---

### libtorrent - BitTorrent Implementation

**Repository:** [github.com/arvidn/libtorrent](https://github.com/arvidn/libtorrent)
**Documentation:** [libtorrent.org](https://libtorrent.org/features.html)

Feature-complete C++ BitTorrent implementation with high-performance design.

#### Features

| Feature | libtorrent Support |
|---------|-------------------|
| **Parallel streams** | ✅ Multiple connections |
| **Encryption** | ✅ Protocol encryption |
| **Checksum verification** | ✅ SHA-1 (multi-threaded) |
| **Resume** | ✅ Fast resume data |
| **Piece-level transfer** | ✅ Block-level picking |
| **Disk I/O** | ✅ Separate thread, ARC cache |

#### Performance Features

- Separate disk I/O thread (non-blocking)
- Adjustable read/write disk cache
- Uses Boost.Asio (IOCP on Windows, epoll on Linux)
- Multi-threaded piece hash verification

#### Limitations

Designed for P2P/swarm transfers - overkill for direct point-to-point.

---

### Other Libraries

| Library | Strengths | Limitations | Link |
|---------|-----------|-------------|------|
| **Poco C++** | NetSSL, Zip, HTTP, cross-platform | Not speed-optimized | [pocoproject.org](https://pocoproject.org/) |
| **libcurl** | HTTP/FTP, resume, mature | Single-threaded per transfer | [curl.se/libcurl](https://curl.se/libcurl/) |
| **Asio** | Async I/O foundation, portable | No file transfer logic | [think-async.com](https://think-async.com/Asio/) |
| **gRPC** | Streaming, TLS, cross-language | Protocol overhead | [grpc.io](https://grpc.io/) |
| **evpp** | High-perf TCP/UDP/HTTP | Chinese docs, less known | [github.com/Qihoo360/evpp](https://github.com/Qihoo360/evpp) |

---

### Comparison Matrix

| Feature | WDT | librsync | libtorrent | Custom Build |
|---------|-----|----------|------------|--------------|
| **Parallel streams** | ✅ | ❌ | ✅ | ✅ |
| **Encryption** | ✅ AES-GCM | ❌ | ✅ | ✅ |
| **Compression** | ✅ LZ4 | ❌ | ❌ | ✅ LZ4/Zstd |
| **Delta transfer** | ❌ | ✅ | ❌ | ✅ |
| **Resume** | ✅ | ❌ | ✅ | ✅ |
| **Max speed** | 40+ Gbps | N/A | 10+ Gbps | 10-40 Gbps |
| **Windows support** | Partial | ✅ | ✅ | ✅ |
| **Dependencies** | Folly, glog | Minimal | Boost | Configurable |
| **Integration effort** | 1-2 weeks | 1 week | 2-3 weeks | 6-9 weeks |

---

### Recommended Approach: WDT + librsync

Combine WDT's high-speed parallel transfer with librsync's delta algorithm.

```
┌─────────────────────────────────────────────────────────────┐
│                    Our File Transfer                         │
├─────────────────────────────────────────────────────────────┤
│  Control Layer (our code)                                    │
│  - File listings, permissions                               │
│  - Delta decision logic                                      │
│  - Progress/cancel/pause                                     │
│  - UI integration                                            │
├──────────────────────────────┬──────────────────────────────┤
│  Delta Layer (librsync)      │  Transfer Layer (WDT)        │
│  - Signature generation      │  - Parallel TCP streams      │
│  - Delta computation         │  - AES-GCM encryption        │
│  - Patch application         │  - LZ4 compression           │
│                              │  - Resume support            │
└──────────────────────────────┴──────────────────────────────┘
```

#### Integration Example

```cpp
class HighSpeedTransfer {
public:
    HighSpeedTransfer() {
        wdt_ = &Wdt::initializeWdt("vnc_transfer");
    }

    TransferResult SendFile(const std::string& localPath,
                           const std::string& remoteHost,
                           const std::string& remotePath,
                           bool enableDelta = true) {
        if (enableDelta) {
            // 1. Request signature from remote via control channel
            auto signature = RequestRemoteSignature(remoteHost, remotePath);

            if (signature) {
                // 2. Compute delta locally using librsync
                std::string deltaPath = ComputeDelta(localPath, *signature);

                // 3. Send only the delta via WDT (much smaller)
                auto result = SendViaWdt(deltaPath, remoteHost,
                                        remotePath + ".delta");

                // 4. Remote applies delta
                NotifyApplyDelta(remoteHost, remotePath);
                return result;
            }
        }

        // Full file transfer via WDT
        return SendViaWdt(localPath, remoteHost, remotePath);
    }

private:
    TransferResult SendViaWdt(const std::string& path,
                              const std::string& host,
                              const std::string& dest) {
        WdtTransferRequest req;
        req.directory = GetDirectory(path);
        req.fileInfo = {{ GetFilename(path) }};
        req.hostName = host;
        req.destDirectory = GetDirectory(dest);
        req.encryptionType = EncryptionType::AES_GCM;

        return wdt_->wdtSend(req);
    }

    std::string ComputeDelta(const std::string& newFile,
                            const std::string& signaturePath) {
        std::string deltaPath = newFile + ".delta";

        // Load signature
        FILE* sigFile = fopen(signaturePath.c_str(), "rb");
        rs_signature_t* sig;
        rs_loadsig_file(sigFile, &sig, NULL);
        rs_build_hash_table(sig);
        fclose(sigFile);

        // Generate delta
        FILE* newF = fopen(newFile.c_str(), "rb");
        FILE* deltaF = fopen(deltaPath.c_str(), "wb");
        rs_delta_file(sig, newF, deltaF, NULL);

        fclose(newF);
        fclose(deltaF);
        rs_free_sumset(sig);

        return deltaPath;
    }

    Wdt* wdt_;
};
```

---

### Development Time Comparison

| Approach | Time | Risk | Performance |
|----------|------|------|-------------|
| **WDT only** | 1-2 weeks | Low | 40+ Gbps, no delta |
| **WDT + librsync** | 3-4 weeks | Low | 40+ Gbps + delta |
| **Custom from scratch** | 6-9 weeks | Medium | 10-40 Gbps |
| **Adapt UltraVNC** | 4-6 weeks | High | ~1-2 Gbps max |

### Final Recommendation

**Use WDT + librsync hybrid approach:**

1. **Week 1-2:** Integrate WDT
   - Build for Windows (may need patches)
   - Create wrapper API
   - Test basic high-speed transfers

2. **Week 3:** Add librsync delta
   - Signature generation on receiver
   - Delta computation on sender
   - Delta application on receiver

3. **Week 4:** Integration
   - Control channel for coordination
   - Progress callbacks
   - UI integration
   - Testing

**Result:** 40+ Gbps capable file transfer with delta support in ~4 weeks.

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

### Core Protocol (Viewer)
- `vncviewer/ClientConnection.cpp` - Main protocol handler (monolithic)
- `vncviewer/ClientConnection*.cpp` - Encoding decoders
- `vncviewer/KeyMap.cpp` - Keyboard mapping
- `vncviewer/rdr/*` - Network I/O streams

### Core Protocol (Server)
- `winvnc/winvnc/vncserver.cpp` - Server coordinator
- `winvnc/winvnc/vncclient.cpp` - Per-client protocol handler
- `winvnc/winvnc/vncencoder*.cpp` - Encoding implementations
- `winvnc/winvnc/MouseSimulator.cpp` - Input injection
- `winvnc/winvnc/vncsockconnect.cpp` - Socket listener

### File Transfer
- `vncviewer/FileTransfer.cpp` - Viewer-side file transfer (~3,000 lines)
- `vncviewer/FileTransfer.h` - FileTransfer class definition
- `winvnc/winvnc/vncclient.cpp:3688` - Server-side message handler
- `winvnc/winvnc/vncclient.cpp:5946` - Server `ReceiveFileChunk()`
- `winvnc/winvnc/vncclient.cpp:6129` - Server `SendFileChunk()`
- `rfb/rfbproto.h:1119` - File transfer message type definitions

### External References
- [RFC 6143 - The Remote Framebuffer Protocol](https://datatracker.ietf.org/doc/html/rfc6143) - Official RFB specification
- [DXGI Desktop Duplication API](https://docs.microsoft.com/en-us/windows/win32/direct3ddxgi/desktop-dup-api) - Windows screen capture
- [SendSAS function](https://docs.microsoft.com/en-us/windows/win32/api/sas/nf-sas-sendsas) - Secure Attention Sequence API
- [noVNC](https://github.com/novnc/noVNC) - HTML5 VNC client (WebSocket + Canvas approach)
- [websockify](https://github.com/novnc/websockify) - WebSocket to TCP proxy
- [WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) - Real-time browser communication
- [WebCodecs API](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API) - Low-level video encoding/decoding

### High-Performance File Transfer Libraries
- [Facebook WDT](https://github.com/facebook/wdt) - Warp speed Data Transfer (40+ Gbps)
- [librsync](https://github.com/librsync/librsync) - Delta compression library
- [librsync Documentation](https://librsync.github.io/) - API reference
- [libtorrent](https://github.com/arvidn/libtorrent) - BitTorrent implementation
- [libtorrent Features](https://libtorrent.org/features.html) - Feature documentation

### Compression & Hashing
- [LZ4](https://github.com/lz4/lz4) - Extremely fast compression
- [Zstd](https://github.com/facebook/zstd) - Fast compression with good ratios
- [xxHash](https://github.com/Cyan4973/xxHash) - Extremely fast hash algorithm

### Async I/O & Networking
- [libuv](https://github.com/libuv/libuv) - Cross-platform async I/O
- [Asio](https://think-async.com/Asio/) - C++ async I/O library
- [OpenSSL](https://www.openssl.org/) - TLS and crypto
- [libwebsockets](https://libwebsockets.org/) - WebSocket library
