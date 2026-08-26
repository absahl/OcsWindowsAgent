# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is the **OCS Inventory NG Windows Agent** — a native C++ (MFC/Win32) application that runs on Windows client machines, inventories hardware/software, and reports to an OCS Inventory server over HTTP(S). It ships as a Windows service plus a handful of companion executables/DLLs, packaged with an NSIS installer.

There is no cross-platform or scripting layer here: this repo is Visual Studio C++ only (VS2022, toolset v143), targeting Windows Vista+. Everything is built via `OCSInventory.sln` / MSBuild — there is no CMake, no package manager, no test runner beyond the manual `TestSysInfo` harness.

## Build

Building requires Windows + Visual Studio 2022 (Community or higher) with the C++ desktop workload, MFC/ATL support, and Strawberry Perl (for OpenSSL/cURL builds).

1. **Build third-party dependencies first** (zlib, OpenSSL, cURL, ZipArchive) — required before the solution will link:
   ```
   External_Deps\OCS_Make_Required_Libs.bat        (x86)
   External_Deps\OCS_Make_Required_Libs_x64.bat     (x64)
   ```
   Edit the path variables at the top of the `.bat` (`VC_PATH`, `PERL_PATH`, `OCSAGENT_PATH`) to match the local machine before running. Source archives (`zlib-*`, `openssl-*`, `curl-*`, `ZipArchive/`, `tinyxml/`) live under `External_Deps/` but are gitignored — they must be fetched/extracted separately. Output libs/DLLs are copied into `External_Deps/` and the solution's `Release/` output folder.
2. **Build the solution**: open `OCSInventory.sln` in Visual Studio and build (or `msbuild OCSInventory.sln /p:Configuration=Release /p:Platform=Win32`). Project build order matters — `SysInfo` and `OCSInventory Front` are shared libraries consumed by nearly every other project, so they must build before `Agent`, `Service`, `OcsSystray`, etc.
3. **Windows service message file**: `Service/NTServiceMsg.mc` must be compiled with `mc.exe` (part of the Windows SDK) into `NTServiceMsg.h`/`.bin` before `Service` builds — done automatically by `OCS_Make_Required_Libs*.bat`, or run `mc.exe NTServiceMsg.mc` manually from `Service/`.
4. **Installer**: `NSIS_agent_setup/OCS-NG_Windows_Agent_Setup_x86.nsi` and `_x64.nsi` build the distributable installer via NSIS, pulling compiled binaries from the `Release/` folders.
5. **Code signing**: `autosign_release.bat` runs `signtool` against `Release\*.exe` and `Release\*.dll` using a named cert (`FactorFX`) — only relevant for official release builds, not local dev.

There is no CI-runnable test suite. `TestSysInfo` is a standalone MFC dialog app for manually exercising `SysInfo` inventory collection during development — build and run it interactively to validate hardware/software detection changes rather than relying on automation.

## Architecture

The solution (`OCSInventory.sln`) is composed of these VS projects, each producing an `.exe` or `.dll`:

- **`Agent`** — the main entry point (`COCSInventoryApp` in `OCSInventory.cpp`/`.h`). Parses command-line args (see `OPTIONS.TXT`), orchestrates the prolog → inventory → response cycle, and drives the plugin system. Contains the built-in "capacities" (`Cap*.cpp/.h`): `CapRegistry`, `CapIpdiscover`, `CapDownload`, `CapExecute`, `CapKeyFinder`, each a self-contained inventory/action module deriving from `CapacityAbstract`.
- **`SysInfo`** — the hardware/software detection library. One class per inventory domain (`Cpu`, `Memory`, `DiskInfo`, `NetworkAdapter`, `Printer`, `Software`, `Bios`, `VMSystem`/`DetectVM`, etc.), most paired with a `*List` collection class and backed by WMI queries (`Wmi.cpp`) or the registry. This is the largest project and the one most inventory-feature work touches.
- **`OCSInventory Front`** — shared core library (despite the name, not a UI project) used by `Agent` and the communication providers. Owns the request/response object model (`PrologRequest`/`PrologResponse`/`InventoryRequest`/`InventoryResponse`), XML serialization (`XMLInteract`, `Markup`), config (`Config`, `ServerConfig`), the `CComProvider` DLL-loading abstraction, `Zip`/`flate` compression, and `Log`.
- **`ComHTTP`** — the default *communication provider*: an OCS-specific plugin DLL implementing `ConnexionAbstract` over HTTP(S) (via cURL/OpenSSL). Communication providers are loaded dynamically at runtime by `CComProvider::load()` in `OCSInventory Front`, so a different transport could be swapped in as its own DLL implementing the same exported factory functions (`NEW_SERVER_OBJECT`/`NEW_SERVER_CONFIG_OBJECT`, see `ComProvider.h`).
- **`Service`** — wraps `Agent` as a Windows NT service (`NTService.cpp`, `OcsService.cpp`) so it runs unattended; requires `NTServiceMsg.mc` compiled first.
- **`OcsSystray`** / **`OcsNotifyUser`** — user-facing companion apps: system tray icon + inventory viewer dialog, and TAG-prompt/download-progress notification dialogs respectively.
- **`OcsWmi`** — small WMI helper DLL used by `SysInfo`/`Agent` for WMI-based queries.
- **`Download`** — handles the agent's package-download-and-deploy feature (`Package`, `BlackList`).
- **`OCSPlugin_Example`** — a minimal reference implementation of the plugin API (`PluginApi.h`) showing how to hook `START`/`PROLOG_WRITE`/`PROLOG_RESP`/`INVENTORY`/`END`/`CLEAN`; use as the template when adding a new plugin.
- **`TestSysInfo`** — standalone MFC test harness for `SysInfo`, not part of the shipped product.

### Plugin system

Two distinct extension points exist and are easy to conflate:
1. **Capacities** (`Agent/Cap*.cpp`) — built into the Agent binary itself, compiled in.
2. **Plugins** — separate DLLs dropped into the agent's plugin directory, loaded at runtime by `CPlugins::Load()` (`Agent/Plugins.h/.cpp`) up to `MAX_PLUGINS` (64), each implementing the six hook exports declared in `PluginApi.h`. `OCSPlugin_Example` and `Pre_Installed_Plugins/Saas.ps1` (a PowerShell-based SaaS detection plugin) are examples of this mechanism.

### Data flow

Command-line/config parsing (`Config`, `ServerConfig`) → `CComProvider` loads the configured communication DLL (`ComHTTP.dll` by default) → agent sends a **prolog** request/response handshake with the server → capacities and plugins populate an **inventory** request (XML, optionally zlib-compressed via `Zip`/`flate`) → inventory is sent to the server or written locally (`/local`, `/xml` command-line switches) → server's inventory response may trigger further actions (package download via `Download`, registry/execute capacities, etc.).

### Versioning

Agent version lives in the `.rc`/resource files per project (see `CHANGELOG` for the human-readable history, currently `2.11.0.1`). Bump version resources and `CHANGELOG` together when cutting a release; this has historically been done as its own commit (see `refactor(agent): update version to X` / `refactor(changelog): update for X` commits).

## Conventions

- Encoding/text macros use MFC's `CString` and `_T()`/`LPCTSTR` throughout — this is a Unicode MFC build, not raw `std::string`/`char*`.
- Header guards follow the classic MFC/AppWizard style (`AFX_<CLASS>_H__<GUID>__INCLUDED_`) rather than `#pragma once` alone, though many files use both.
- Cross-DLL exports use project-specific macros (e.g. `OCSINVENTORYFRONT_API`, `OCS_PROVIDER_API`, `OCSINVENTORY_API_EXPORTED`) defined per project — check the relevant `stdafx.h`/header before adding new exported symbols.
- Command-line switches and `ocsinventory.ini` config keys are documented in `OPTIONS.TXT` — update it when adding/changing agent CLI options or config keys.
