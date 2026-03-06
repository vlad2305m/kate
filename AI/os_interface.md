# Kate OS Interface Analysis

## Overview

Kate is a Qt Widgets-based advanced text editor and thin IDE for Linux, macOS, and Windows.
The application is organized into:

- **apps/lib/**: The shared `kateprivate` library (core logic for both Kate and KWrite)
- **apps/kate/**: The full Kate application (multi-window IDE)
- **apps/kwrite/**: The simpler KWrite single-window text editor

---

## OS-Specific Interfaces

### 1. DBus (Linux/BSD/Unix only)
- **Location**: `apps/lib/CMakeLists.txt`, `apps/lib/kateappadaptor.{cpp,h}`, `apps/lib/katerunninginstanceinfo.{cpp,h}`, `apps/kate/katewaiter.{cpp,h}`, `apps/kate/main.cpp`
- **Purpose**: IPC between multiple Kate instances, service registration (`org.kde.kate-<pid>`)
- **Guard**: `#ifdef WITH_DBUS` in code, `USE_DBUS` CMake option defaulting to OFF on Android
- **Status**: Already disabled on Android via `if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)` in CMakeLists

### 2. X11 / Wayland Window System
- **Location**: `apps/kate/main.cpp`, `apps/lib/katemainwindow.cpp`, `apps/lib/kateapp.cpp`
- **Purpose**: Window activation, startup notification (XDG Activation Token, KStartupInfo)
- **Guards**:
  - `#define HAVE_X11 __has_include(<KX11Extras>)` — automatically false when KWindowSystem X11 extras not installed
  - `#if __has_include(<KStartupInfo>)` — same
  - `#if UNIX AND NOT APPLE` — for `Qt::GuiPrivate` linking in `apps/kate/CMakeLists.txt`
- **KWindowSystem**: Used for `isPlatformWayland()`, `isPlatformX11()`, `activateWindow()`. Available on Android as a stub.

### 3. Terminal Detection (Unix)
- **Location**: `apps/lib/kateapp.cpp`
- **Purpose**: Detect if running inside a terminal (for stdin reading and forking behavior)
- **Guard**: `#ifdef HAVE_CTERMID` — automatically false on Android (no ctermid)
- **Status**: Handled by CMake `check_function_exists(ctermid HAVE_CTERMID)` — not found on Android

### 4. Daemon / Background Fork (Unix, not macOS)
- **Location**: `apps/lib/kateapp.cpp`
- **Purpose**: Fork into background when not blocking (desktop behavior)
- **Guard**: `#ifdef HAVE_DAEMON` — not available on Android
- **Status**: Handled by CMake `check_function_exists(daemon HAVE_DAEMON)` — not found on Android

### 5. malloc_trim (Linux glibc)
- **Location**: `apps/lib/CMakeLists.txt`, memory management
- **Purpose**: Release unused memory back to OS
- **Guard**: `#ifdef HAVE_MALLOC_TRIM` — checked via CMake
- **Status**: May or may not be available on Android depending on NDK libc

### 6. libintl (Gettext)
- **Location**: `apps/lib/CMakeLists.txt`, `apps/lib/kateapp.cpp`
- **Purpose**: Locale/translation setup for non-glibc systems
- **Guard**: `if (NOT WIN32 AND NOT HAIKU)` in CMakeLists — **needs `NOT ANDROID` added**
- **Status**: Not available on Android (no traditional gettext)

### 7. SingleApplication (Multi-instance management)
- **Location**: `apps/kate/CMakeLists.txt`, `apps/kate/main.cpp`, `3rdparty/SingleApplication/`
- **Purpose**: Ensure only one primary Kate instance, relay args to existing instance
- **Guard**: Already has `#if defined(Q_OS_ANDROID) || defined(Q_OS_IOS)` fallback in the library
- **Status**: Compiles on Android, gracefully falls back to normal QApplication behavior

### 8. KCrash
- **Location**: `apps/lib/CMakeLists.txt`, `apps/lib/kateapp.cpp`
- **Purpose**: Crash handler with stack traces
- **Guard**: None currently — **needs `NOT ANDROID` guard** in CMakeLists
- **Status**: KCrash does not support Android; needs to be made optional

### 9. KIO (File Operations)
- **Location**: `apps/lib/katefileactions.cpp`, `apps/lib/katemainwindow.{cpp,h}`
- **Purpose**: File copy/move/delete/launch operations, file manager integration
- **Status**: KIO does work on Android (Qt/KDE supports basic KIO on Android), but `KIO::OpenFileManagerWindowJob` and `KIO::ApplicationLauncherJob` may have limited function

### 10. macOS-specific
- **Location**: `apps/lib/kateapp.cpp`, `apps/kate/CMakeLists.txt`, `apps/kwrite/CMakeLists.txt`
- **Purpose**: macOS bundle, PATH fixing, sysctl
- **Guard**: `#if defined(Q_OS_MACOS)` / `#ifdef APPLE` — not applicable to Android

### 11. Windows-specific
- **Location**: `apps/lib/kateapp.cpp`, `apps/kate/CMakeLists.txt`
- **Purpose**: Console attachment, Windows icon
- **Guard**: `#ifdef Q_OS_WIN` / `WIN32` — not applicable to Android

### 12. Style Management (KStyleManager)
- **Location**: `apps/lib/kateapp.cpp`
- **Purpose**: Apply KDE Breeze style
- **Guard**: `#define HAVE_STYLE_MANAGER __has_include(<KStyleManager>)` — auto-detected

### 13. KUserFeedback
- **Location**: `apps/lib/CMakeLists.txt`, `apps/lib/kateapp.cpp`
- **Purpose**: Telemetry (optional)
- **Guard**: `#ifdef WITH_KUSERFEEDBACK` — optional dependency, off on Android

---

## Summary: Required Changes for Android

| Component | Status | Change Needed |
|-----------|--------|---------------|
| DBus | Already guarded `NOT ANDROID` | None |
| X11/Wayland | Auto-guarded via `__has_include` | Skip `Qt::GuiPrivate` linking on Android |
| ctermid/daemon | Auto-detected via CMake | None (auto false on Android) |
| libintl | `NOT WIN32 AND NOT HAIKU` guard | Add `NOT ANDROID` |
| KCrash | Required in CMakeLists | Make optional on Android |
| SingleApplication | Handles Android internally | None (falls back gracefully) |
| KIO | Works on Android | None |
| KWindowSystem | Works on Android | None |
| macOS/Windows specific | `#ifdef` guarded | None |
| ECMAddAndroidApk | Not included | Add to root CMakeLists |
