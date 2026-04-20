# Electron startup performance analysis

## summary

Electron startup has several deterministic, code-visible blockers before `app.ready` and before first renderer paint: synchronous main-process bootstrapping in `lib/browser/init.ts`, synchronous `sendSync` preload handshakes in renderer init paths, and eager native/service initialization in `shell/browser/electron_browser_main_parts.cc`.

Top 3 highest-impact improvements:
1. Replace sync preload bootstrap IPC (`BROWSER_NONSANDBOX_LOAD` / `BROWSER_SANDBOX_LOAD`) with async preload transfer paths.
2. Stop eagerly loading heavy browser API modules in `lib/browser/init.ts` before app code runs.
3. Defer unconditional native service setup in `ElectronBrowserMainParts` (media capture, spellchecker/extensions factories, resource-heavy startup work) until first use.

---

## 1. low-risk improvements (easy to merge)

### A. Avoid synchronous Squirrel probe in main init
- **Affected files:** `lib/browser/init.ts` (`fs.existsSync` in Windows startup block)
- **Current behavior:** On Windows, startup always does `fs.existsSync(update.exe)` + path normalization before app code runs.
- **Issue (why it slows startup):** This is synchronous filesystem I/O on the main thread in the critical path.
- **Suggested improvement:** Gate this probe behind a one-time async check after `ready`, or cache result in process-level state and avoid repeated sync disk access in startup path.
- **Expected impact:** **Low**

### B. Defer default menu module load until event callback executes
- **Affected files:** `lib/browser/init.ts` (`require('@electron/internal/browser/default-menu')`)
- **Current behavior:** `default-menu` is required during bootstrap, then used for `app.once('will-finish-launching', ...)`.
- **Issue (why it slows startup):** Module parse/execute cost is paid unconditionally even for apps that quickly override menu behavior.
- **Suggested improvement:** Resolve the module inside the event callback (or via lazy getter) so it is loaded only when needed.
- **Expected impact:** **Low**

### C. Remove unconditional early creation of media capture dispatcher
- **Affected files:** `shell/browser/electron_browser_main_parts.cc` (`PreCreateThreads`, `MediaCaptureDevicesDispatcher::GetInstance()`)
- **Current behavior:** Media capture dispatcher is force-created before app UI startup regardless of API usage.
- **Issue (why it slows startup):** Unused service initialization consumes startup CPU and adds synchronous setup work on UI thread.
- **Suggested improvement:** Lazily initialize on first media-capture API access instead of unconditional startup construction.
- **Expected impact:** **Medium**

### D. Delay heavy IPC handlers not needed for first window
- **Affected files:** `lib/browser/init.ts`, `lib/browser/rpc-server.ts`, `lib/browser/guest-view-manager.ts`
- **Current behavior:** `rpc-server` and `guest-view-manager` are eagerly required during browser init.
- **Issue (why it slows startup):** Both modules register many handlers and load dependencies before first window, even when `<webview>` / related channels are unused.
- **Suggested improvement:** Register these modules lazily (e.g., on first relevant IPC channel or feature enablement).
- **Expected impact:** **Medium**

---

## 2. high-impact improvements (needs maintainer discussion)

### A. Replace synchronous renderer preload handshake with async bootstrap
- **Affected areas of the codebase:** `lib/renderer/init.ts`, `lib/sandboxed_renderer/init.ts`, `lib/preload_realm/init.ts`, `lib/renderer/ipc-renderer-internal-utils.ts`, `lib/browser/rpc-server.ts`
- **Current behavior:** Renderer startup calls `invokeSync(...)` (`sendSync`) to fetch preload metadata/scripts (`BROWSER_NONSANDBOX_LOAD` / `BROWSER_SANDBOX_LOAD`) before continuing.
- **Architectural limitation:** Main and renderer are synchronously coupled during renderer bootstrap; preload discovery/transfer blocks renderer progress.
- **Proposed change:** Introduce async preload bootstrap protocol (promise-based IPC + staged preload execution) and unblock renderer init from sync round-trips.
- **Tradeoffs / risks:** Requires ordering guarantees for preload execution and compatibility work for error propagation semantics.
- **Expected impact:** **High**

### B. Stop eager browser API preloading in main init
- **Affected areas of the codebase:** `lib/browser/init.ts` (forced requires for `protocol`, `service-worker-main`, `web-contents`, `web-frame-main`, `web-contents-view`)
- **Current behavior:** Multiple browser API modules are loaded before handing control to app code.
- **Architectural limitation:** Startup currently favors eager API readiness over minimal bootstrap.
- **Proposed change:** Move to lazy API activation with explicit “first access” initialization points while preserving API readiness guarantees.
- **Tradeoffs / risks:** Could change subtle timing of app code that expects side effects by `ready`; needs compatibility audit.
- **Expected impact:** **High**

### C. Make pre-ready native initialization feature-demanded
- **Affected areas of the codebase:** `shell/browser/electron_browser_main_parts.cc` (`PreCreateThreads`, `PreMainMessageLoopRun`), `shell/app/electron_main_delegate.cc`
- **Current behavior:** Startup eagerly performs locale/resource bundle setup, spellchecker factory creation, extension keyed-service factory setup, and other subsystem initialization.
- **Architectural limitation:** Broad “initialize-most-things-before-ready” model increases deterministic startup floor.
- **Proposed change:** Convert optional subsystems to on-demand initialization keyed to API usage and build flags.
- **Tradeoffs / risks:** Lifecycle complexity increases; race/order bugs are possible without strict initialization contracts.
- **Expected impact:** **Medium–High**

### D. Rework preload delivery for sandboxed renderer to avoid upfront full script reads
- **Affected areas of the codebase:** `lib/browser/rpc-server.ts` (`readPreloadScript`, `Promise.all`), `lib/sandboxed_renderer/preload.ts`
- **Current behavior:** Browser reads all sandboxed preload files (`fs.promises.readFile`) and sends contents before renderer continues.
- **Architectural limitation:** Preload content acquisition is all-at-once and front-loaded.
- **Proposed change:** Stream or chunk preload delivery, and execute incrementally so first-frame work can overlap with remaining preload I/O.
- **Tradeoffs / risks:** More complex preload execution model and debugging surface.
- **Expected impact:** **Medium–High**

---

## 3. startup timeline breakdown (important)

1. **Process start**
   - Entry points (`shell/app/electron_main_{linux,mac,win}.cc`) initialize command line and hand off to `ContentMain`.

2. **Electron native initialization**
   - `ElectronMainDelegate::BasicStartupComplete` / `PreSandboxStartup` / `PreBrowserMain` (`shell/app/electron_main_delegate.cc`) set switches, logging, feature list, mojo, resource behavior.
   - **Blocking steps:** resource/logging setup, switch mutation, early subsystem init.

3. **Main process JS environment + app bootstrap**
   - `ElectronBrowserMainParts::PostEarlyInitialization` (`shell/browser/electron_browser_main_parts.cc`) creates Node environment and runs `node_bindings_->LoadEnvironment(...)`.
   - `lib/browser/init.ts` executes, performs sync startup work (package discovery/load, Windows sync fs probe, eager module requires).
   - `node_bindings_->JoinAppCode()` blocks until `process.appCodeLoaded()` is called.
   - **Blocking steps:** synchronous JS module loading and sync filesystem/Module `_load` operations.

4. **Main process ready + window creation phase**
   - `PreMainMessageLoopRun` triggers launch notifications; `Browser::DidFinishLaunching` (`shell/browser/browser.cc`) creates user-data directory and resolves `whenReady`.
   - App code creates `BrowserWindow`.
   - **Blocking steps:** synchronous user-data directory creation before ready resolution.

5. **Renderer bootstrap to first meaningful paint**
   - Renderer init (`lib/renderer/init.ts` or sandbox variants) runs `invokeSync` preload IPC, then executes preload scripts.
   - Common renderer init (`lib/renderer/common-init.ts`) wires IPC/webFrame/webview setup before page app code fully settles.
   - **Blocking steps:** sync IPC round-trip + preload loading/execution in renderer critical path.

---

## 4. quick wins (top 5)

1. Remove sync Windows `update.exe` probe from `lib/browser/init.ts` startup path.
2. Lazily require `default-menu` instead of loading it unconditionally in bootstrap.
3. Lazily initialize `MediaCaptureDevicesDispatcher` instead of force-creating in `PreCreateThreads`.
4. Defer `guest-view-manager` / portions of `rpc-server` registration until feature use.
5. Split preload IPC so renderer startup is not blocked by synchronous `sendSync` handshakes.
