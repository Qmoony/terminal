# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Contains

This is the **Windows Terminal** and **Windows Console Host (conhost)** repository. It contains:
- The Windows Terminal application (`src/cascadia/`)
- The Windows Console Host (`src/host/`)
- Shared components (renderer, VT parser/adapter, buffer, types)
- The ConPTY (Windows Pseudo Console) subsystem (`src/winconpty/`)

This is a Windows-only C++/WinRT codebase built with MSBuild and Visual Studio. The primary solution file is `OpenConsole.slnx`.

## Building

Prerequisites: Visual Studio 2022 with C++ and UWP workloads, Windows SDK. Run submodule init first:

```shell
git submodule update --init --recursive
```

**PowerShell (preferred):**
```powershell
Import-Module .\tools\OpenConsole.psm1
Set-MsBuildDevEnvironment
Invoke-OpenConsoleBuild
```

**Cmd:**
```shell
.\tools\razzle.cmd
bcz
```

Build configurations: `Debug`, `Release`, `AuditMode` (enables extra CppCoreCheck static analysis).  
Platforms: `x64`, `x86`, `arm64`.

To build only the Terminal package from the command line (after running `razzle.cmd`):
```cmd
pushd src\cascadia\CascadiaPackage
bx
"C:\Program Files\Microsoft Visual Studio\2022\Preview\Common7\IDE\DeployAppRecipe.exe" bin\%ARCH%\%_LAST_BUILD_CONF%\CascadiaPackage.build.appxrecipe
popd
```

## Running Tests

**PowerShell:**
```powershell
Import-Module .\tools\OpenConsole.psm1
Invoke-OpenConsoleTests           # unit tests (default)
Invoke-OpenConsoleTests -?        # see all options
```

**Cmd scripts (after razzle.cmd):**
```shell
runut.cmd    # unit tests
runft.cmd    # feature tests
runuia.cmd   # UI Automation tests
```

Tests use **TAEF** (Test Authoring and Execution Framework). After setup, `%TAEF%` resolves to `te.exe`:

```shell
# Run all tests in a DLL
te.exe Console.Unit.Tests.dll

# Run a specific test class/method (supports wildcards)
te.exe Console.Unit.Tests.dll /name:*BufferTests*

# Debug a specific test (attach debugger to the PID it prints)
runut *Tests.dll /name:TextBufferTests::TestInsertCharacter /waitForDebugger
```

## Code Formatting

C++ uses clang-format (BasedOnStyle: Microsoft, with customizations in `.clang-format`):

```powershell
Invoke-CodeFormat                  # PowerShell
```
```shell
runformat.cmd                      # Cmd
```

XAML uses XamlStyler (config: `XamlStyler.json`). Run via:
```shell
.\tools\runxamlformat.cmd
```

Before formatting is available, download `clang-format.exe`:
```powershell
Import-Module .\tools\OpenConsole.psm1
Set-MsBuildDevEnvironment
Get-Format
```

## High-Level Architecture

### Layered Component Model (src/cascadia/)

The Terminal stack is composed of distinct layers with clear boundaries:

```
WindowsTerminal (EXE) — Win32 host, XAML Islands, window chrome
    └── TerminalApp (DLL) — tabs, panes, command palette, settings parsing, actions
            ├── TerminalControl (DLL) — UWP/XAML terminal widget, DX rendering, input
            │       └── TerminalCore (LIB) — Terminal state machine, text buffer, VT parsing
            └── TerminalSettingsModel (DLL) — JSON (de)serialization, settings model
```

- **`TerminalCore`**: The `Terminal` class owns the text buffer and VT state machine. It has no UI dependency and can be tested standalone.
- **`TerminalControl`**: `ControlCore` bridges `TerminalCore` to the WinRT/XAML layer. `ControlInteractivity` handles mouse/touch input. `HwndTerminal` provides a non-XAML Win32 terminal surface (used in WpfTerminalControl).
- **`TerminalApp`**: `TerminalPage` is the root XAML control (tabs, panes). `AppLogic` handles window-level concerns (titlebar, focus mode). `AppActionHandlers.cpp` contains one handler per action. `ShortcutActionDispatch` routes keybindings to typed events.
- **`TerminalSettingsModel`**: `CascadiaSettings` is the root settings object. `Profile` is the per-profile settings with cascade/inheritance via `INHERITABLE_SETTING`. `ActionMap` handles keybinding-to-action mapping.
- **`TerminalConnection`**: Connection backends — `ConptyConnection` (PTY to Win32 console processes), `AzureCloudShellConnection`, etc.

### Console Host (src/host/)

The legacy Windows Console Host (`conhost.exe` / `OpenConsole.exe`). Key classes:
- `CONSOLE_INFORMATION` (`consoleInformation.cpp`) — global console state
- `SCREEN_INFORMATION` (`screenInfo.cpp`) — buffer, cursor, selection
- `Settings` (`settings.cpp`) — user preferences

The host is built as a LIB, then packaged as both `OpenConsole.exe` (for development/testing without touching system32) and `conhostv2.dll` (for OS deployment).

### Renderer (src/renderer/)

- **`base/renderer.cpp`** — orchestrates rendering; breaks the text buffer into GDI-style primitives
- **`atlas/`** — `AtlasEngine`: the modern high-performance renderer. `BackendD3D` is the primary path (custom GPU glyph cache). `BackendD2D` is the fallback (Remote Desktop, older GPUs).
- **`gdi/`** — legacy GDI renderer (conhost classic)

### Shared Libraries

- **`src/terminal/parser`** — VT sequence state machine, decodes in-band escape sequences
- **`src/terminal/adapter`** — converts parsed VT verbs into console API calls
- **`src/buffer/out`** — `TextBuffer` and `ROW` — the core terminal text buffer
- **`src/types/`** — shared types: UIA providers, color utilities, viewport
- **`src/til/`** — Terminal Implementation Library: utility types (`til::rect`, `til::color`, `til::enumset`, string helpers, etc.)

### WinRT / C++/WinRT Patterns

All public API boundaries across DLLs use WinRT (`.idl` files define the interface). The main patterns:
- `.idl` defines the WinRT interface
- `.h` declares the implementation struct (inheriting from generated base)
- `.cpp` implements the methods
- `INHERITABLE_SETTING(type, name, default)` — implements cascading profile settings with Has/Clear support
- `WINRT_PROPERTY(type, name, default)` — simple getter/setter property

When adding settings, the flow is: `TerminalSettingsModel` (serialization) → `IControlSettings`/`ICoreSettings` (interface) → `TerminalSettings.cpp` (`_ApplyProfileSettings`) → `TerminalControl`/`TerminalCore`.

See `doc/cascadia/AddASetting.md` for the full step-by-step guide.

### Feature Flags

Controlled via `src/features.xml`. Each feature generates `Feature_XYZ::IsEnabled()` and `TIL_FEATURE_XYZ_ENABLED`. Features can be gated per-branding (`Dev`, `Preview`, `Release`, `WindowsInbox`) or per-branch. See `doc/feature_flags.md`.

## Coding Style

- Follow existing style in whichever file you're editing
- New code: Modern C++, follow [C++ Core Guidelines](https://github.com/isocpp/CppCoreGuidelines)
- Use **WIL** (`wil::unique_handle`, `RETURN_IF_*`, `LOG_IF_*`) for Win32/NT/COM API interactions — see `doc/WIL.md`
- Prefer `HRESULT` or exceptions over `NTSTATUS`. Functions always returning success should return `void`. Error-returning functions should be `noexcept` and `[[nodiscard]]`.
- In `TerminalApp`: understand C++/WinRT strong/weak references and coroutine safety before modifying
- Include ordering (enforced by clang-format): precomp header first, then local `"..."`, then system `<...>`

## Repository Layout Conventions

- `ut_*` subdirectories — unit tests for the adjacent code
- `ft_*` subdirectories — functional/feature tests
- `inc/` subdirectories — shared interface/header files
- `lib/` subdirectories — static library packaging of adjacent code
- `dll/` or `exe/` subdirectories — final binary packaging

## Branch Conventions

- Primary branch: `main`
- Feature branches: `dev/<alias>/<description>` (e.g., `dev/austdi/SomeCoolFeature`)  
  Branches prefixed with `dev/` automatically trigger CI (x86 + amd64 builds + unit/feature tests)
- `inbox` — special branch for syncing back to the Windows OS repo

## Debugging

- To debug `CascadiaPackage` in Visual Studio: right-click CascadiaPackage → Properties → Debug → set Application process to **Native Only**
- You cannot launch `WindowsTerminal.exe` directly; deploy via F5 from VS or via `DeployAppRecipe.exe`
- For debugging TAEF tests: use `/waitForDebugger` flag, then attach VS or WinDbg to the printed PID
- For conditional break early in conhost startup: set `HKCU\Console\DebugLaunch` (REG_DWORD) = 1 (Debug builds only)
