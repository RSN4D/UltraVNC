# GUI Separation Analysis

This document analyzes the separation between GUI elements and core functionality in UltraVNC, and assesses the feasibility of replacing the GUI with ImGui.

## Overview

| Component | GUI Framework | Coupling Level | ImGui Feasibility |
|-----------|---------------|----------------|-------------------|
| Viewer (vncviewer) | Win32 API | HIGH (8/10) | Moderate - 8-12 weeks |
| Server (winvnc) | Win32 API | MODERATE-HIGH (6-7/10) | Moderate - 3-5 weeks |

---

## Viewer (vncviewer)

### Current Architecture

**GUI Framework:** Pure Win32 API (no MFC)

**Coupling Level:** HIGH (8/10)

The viewer has **tight coupling** between GUI and protocol:
- `ClientConnection` class is 10,426 lines mixing GUI + protocol + rendering
- Win32 message handlers (`WndProc`) directly trigger RFB protocol operations
- Framebuffer writes directly to GDI bitmap DC during protocol parsing
- No abstraction layers exist

### Key Classes

| Type | Classes | Notes |
|------|---------|-------|
| Mixed (problem) | `ClientConnection` | Does everything - needs splitting |
| GUI | `SessionDialog`, `AuthDialog`, `FileTransfer`, `TextChat` | Win32 dialogs |
| Rendering | `ViewerDirectxClass` | Optional DirectX9 path |

### Rendering Paths

1. **GDI Path:** `BitBlt`/`StretchBlt` to window DC
2. **DirectX9 Path:** Hardware-accelerated via `ViewerDirectxClass`

### Input Handling

- Mouse/keyboard captured in `WndProc`
- Immediately converted to RFB messages and sent
- No input abstraction layer

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `ClientConnection.cpp` | ~6000 | Core protocol + GUI + rendering (needs refactoring) |
| `ClientConnectionTight.cpp` | ~800 | Tight encoding decoder |
| `ClientConnectionHextile.cpp` | ~400 | Hextile encoding decoder |
| `SessionDialog.cpp` | ~500 | Connection settings dialog |
| `AuthDialog.cpp` | ~300 | Authentication dialogs |
| `vncviewer.cpp` | ~400 | Application entry point |

---

## Server (winvnc)

### Current Architecture

**GUI Framework:** Pure Win32 API

**Coupling Level:** MODERATE-HIGH (6-7/10)

Better separation than viewer, but still issues:
- Core classes (`vncServer`, `vncDesktop`, `vncClient`) are mostly GUI-free
- `vncMenu` (tray icon) holds direct `vncServer*` pointer, makes 50+ direct calls
- Settings dialog directly manipulates server state
- Uses Win32 message notifications for client connect/disconnect events

### Key Classes

| Type | Classes | Notes |
|------|---------|-------|
| Core (clean) | `vncDesktop`, `vncClient`, `vncSockConnect` | Protocol/capture logic |
| Core (some coupling) | `vncServer` | Has notification HWND list |
| GUI | `vncMenu`, `PropertiesDialog`, `vncAbout`, `vncListDlg` | Win32 dialogs |

### System Tray

- `vncMenu` creates hidden window for tray icon messages
- Context menu built with `CreatePopupMenu()`
- Direct calls to `m_server->` methods on menu clicks

### Settings Dialogs

- `PropertiesDialog` - 2,316 lines, 12 tabs
- Directly reads/writes `SettingsManager` singleton
- Calls `UpdateServer()` to push changes

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `vncserver.cpp` | ~2100 | Server core, client management |
| `vncdesktop.cpp` | ~2600 | Desktop capture, screen updates |
| `vncclient.cpp` | ~6800 | Per-client protocol handler |
| `vncmenu.cpp` | ~2100 | System tray icon and menu |
| `PropertiesDialog.cpp` | ~2300 | Settings dialog (12 tabs) |

---

## ImGui Migration Strategy

### Proposed Architecture

```
Current:
  [Win32 WndProc] <-> [ClientConnection/vncMenu] <-> [Protocol/Server]
                      (tightly coupled)

Target:
  [ImGui Loop] <-> [GUI Layer] <-> [Interface] <-> [Protocol/Server]
                                        |
                                 Abstract APIs:
                                 - IRenderer
                                 - IFramebuffer
                                 - IRfbClient (viewer)
                                 - IServerControl (server)
```

### Required Abstractions

#### Viewer - Renderer Interface

```cpp
class IRenderer {
public:
    virtual ~IRenderer() = default;
    virtual void BeginFrame() = 0;
    virtual void UpdateRect(int x, int y, int w, int h,
                           PixelFormat fmt, void* data) = 0;
    virtual void Present() = 0;
    virtual void Resize(int width, int height) = 0;
};
```

#### Viewer - Framebuffer Interface

```cpp
class IFramebuffer {
public:
    virtual ~IFramebuffer() = default;
    virtual void UpdateRect(Rect r, void* data, int stride) = 0;
    virtual void Clear() = 0;
    virtual void* GetBits() = 0;
    virtual int GetWidth() = 0;
    virtual int GetHeight() = 0;
};
```

#### Viewer - Protocol Interface

```cpp
class IRfbClient {
public:
    virtual ~IRfbClient() = default;
    virtual bool Connect(const char* host, int port) = 0;
    virtual void Disconnect() = 0;
    virtual bool IsConnected() = 0;
    virtual void ProcessUpdates() = 0;
    virtual void SendKeyEvent(int keysym, bool down) = 0;
    virtual void SendPointerEvent(int x, int y, int buttonMask) = 0;
    virtual void SendClipboard(const char* text) = 0;
    virtual IFramebuffer* GetFramebuffer() = 0;
};
```

#### Server - Control Interface

```cpp
class IServerControl {
public:
    virtual ~IServerControl() = default;
    virtual void EnableConnections(bool enable) = 0;
    virtual bool ConnectionsEnabled() = 0;
    virtual int GetClientCount() = 0;
    virtual void KillAllClients() = 0;
    virtual void KillClient(int id) = 0;
    virtual std::vector<ClientInfo> GetClientList() = 0;
};
```

#### Event Abstraction

```cpp
struct InputEvent {
    enum Type {
        KeyDown,
        KeyUp,
        MouseMove,
        MouseButtonDown,
        MouseButtonUp,
        MouseWheel
    } type;
    int x, y;
    int key;
    int button;
    int wheelDelta;
};
```

---

## Migration Phases

### Phase 1: Viewer Protocol Extraction

**Goal:** Extract protocol logic from `ClientConnection` into separate class

**Tasks:**
1. Create `RfbProtocol` class with network I/O
2. Move encoding handlers to separate decoder classes
3. Create `Framebuffer` class for pixel storage
4. `ClientConnection` becomes thin wrapper

**Estimated effort:** 2-3 weeks

### Phase 2: Viewer Renderer Abstraction

**Goal:** Abstract rendering behind `IRenderer` interface

**Tasks:**
1. Create `IRenderer` interface
2. Implement `GdiRenderer` (existing code)
3. Implement `DirectXRenderer` (existing code)
4. Implement `ImGuiRenderer` (new)

**Estimated effort:** 1-2 weeks

### Phase 3: Viewer ImGui UI

**Goal:** Replace Win32 dialogs with ImGui

**Tasks:**
1. ImGui main window with framebuffer texture
2. Connection dialog (replace `SessionDialog`)
3. Authentication dialog (replace `AuthDialog`)
4. Settings windows
5. File transfer window
6. Text chat window

**Estimated effort:** 3-4 weeks

### Phase 4: Server Interface Layer

**Goal:** Create abstraction between `vncMenu` and `vncServer`

**Tasks:**
1. Create `IServerControl` interface
2. Implement adapter for `vncServer`
3. Refactor `vncMenu` to use interface
4. Add event queue for notifications

**Estimated effort:** 1-2 weeks

### Phase 5: Server ImGui UI

**Goal:** Replace Win32 tray/dialogs with ImGui

**Tasks:**
1. ImGui main window (replaces tray icon concept)
2. Settings window (replace 12-tab `PropertiesDialog`)
3. Client list window
4. About dialog
5. Status display

**Estimated effort:** 2-3 weeks

---

## Effort Summary

| Phase | Component | Weeks | Risk |
|-------|-----------|-------|------|
| 1 | Viewer Protocol Extraction | 2-3 | High |
| 2 | Viewer Renderer Abstraction | 1-2 | Medium |
| 3 | Viewer ImGui UI | 3-4 | Medium |
| 4 | Server Interface Layer | 1-2 | Low |
| 5 | Server ImGui UI | 2-3 | Low |
| **Total** | | **9-14** | |

---

## Recommendations

1. **Start with Server** - Lower risk, cleaner core separation
2. **Prototype First** - Build minimal ImGui viewer (connect + display) before full migration
3. **Keep Win32 Fallback** - Maintain ability to build with original GUI during transition
4. **Incremental Migration** - Replace one dialog at a time, not big bang

---

## Threading Considerations

### Current Model

- **Viewer:** Protocol runs in background thread (`omni_thread`), GUI in main thread
- **Server:** GUI runs in impersonation thread, server core in main thread

### ImGui Requirements

- ImGui is single-threaded (all calls from render thread)
- Need message queue between protocol thread and ImGui thread
- Framebuffer updates must be synchronized

### Proposed Threading

```
Viewer:
  [Protocol Thread]              [ImGui Thread]
       |                              |
       +-- Decode pixels              |
       +-- Write to framebuffer       |
       +-- Signal update --------->   +-- Lock framebuffer
                                      +-- Upload to GPU texture
                                      +-- Render ImGui
                                      +-- Present

Server:
  [Server Thread]                [ImGui Thread]
       |                              |
       +-- Handle clients             |
       +-- Queue events --------->    +-- Process event queue
                                      +-- Update UI state
                                      +-- Render ImGui
```

---

## Files to Modify/Create

### New Files (Abstractions)

```
src/
  common/
    IRenderer.h
    IFramebuffer.h
    IRfbClient.h
    IServerControl.h
    InputEvent.h
  viewer/
    RfbProtocol.cpp/.h      (extracted from ClientConnection)
    Framebuffer.cpp/.h
    ImGuiRenderer.cpp/.h
    ViewerApp.cpp/.h        (ImGui main loop)
  server/
    ServerControlAdapter.cpp/.h
    ServerApp.cpp/.h        (ImGui main loop)
```

### Files to Refactor

```
vncviewer/
  ClientConnection.cpp  -> Split into RfbProtocol + thin wrapper
  SessionDialog.cpp     -> Replace with ImGui
  AuthDialog.cpp        -> Replace with ImGui

winvnc/
  vncmenu.cpp           -> Replace with ImGui
  PropertiesDialog.cpp  -> Replace with ImGui
  vncserver.cpp         -> Add IServerControl implementation
```
