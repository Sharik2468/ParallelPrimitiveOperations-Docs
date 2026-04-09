# Installation & requirements

This page explains what you need to use **Parallel Primitive Operations**, how to add it to a project, and how to hook it up from **C++** if you are not only using Blueprints.

---

## 1. Unreal Engine version

The plugin manifest declares:

```json
"EngineVersion": "5.6.0"
```

**Practical meaning**

- The plugin is developed and tested against the **5.6** engine line. Use **Unreal Engine 5.6** (any compatible 5.6.x patch is usually fine; if the editor warns about version mismatch, rebuild the plugin or align engine and plugin versions).
- Older engines (5.5 and below) or 5.7+ **may** compile after changes, but that is **not** supported by this manifest—you would be on your own.

If you clone or vendor the plugin into another project, keep the engine version in mind before reporting issues.

---

## 2. Platforms

Both runtime and editor modules list **`PlatformAllowList": [ "Win64" ]`** in `ParallelPrimitiveOperations.uplugin`.

**What that means for you**

- **Windows 64-bit** editor and **Win64** game targets are the **supported** configuration out of the box.
- For **other platforms** (Linux, Mac, consoles, mobile): the modules may **not be built** for those targets unless you change the allowlist and verify the code paths (thread pool, atomics, etc.). Do **not** assume the packaged build will include the plugin on a platform that is not allowlisted—plan a platform audit before shipping.

**Cooking / packaging**

- If you ship only **Win64**, you are aligned with the default plugin settings.
- If you add a platform without updating the plugin, you may get **missing module** or **stripped plugin** behavior. Treat non-Win64 as **unsupported until you explicitly enable and test it**.

---

## 3. Adding the plugin to a project

### Option A — Plugin inside your project (typical)

1. Copy the whole **`ParallelPrimitiveOperations`** folder under your project’s **`Plugins/`** directory, e.g.  
   `YourProject/Plugins/ParallelPrimitiveOperations/`.
2. **Restart the editor** or use **File → Refresh Visual Studio Project** if you use C++.
3. Enable the plugin:
   - **Edit → Plugins**, search for **“Parallel Primitive Operations”**, check **Enabled**, restart if prompted.  
   Or edit your **`.uproject`** JSON and set `"Enabled": true` for this plugin (same effect as the Plugins window).

### Option B — Engine-wide plugin (advanced)

You can install under the engine’s `Plugins/` folder; only do this if you understand shared engine installs and version control. Project-local `Plugins/` is usually easier for teams.

### Blueprint-only vs C++ project

- **Blueprint-only project:** Enabling the plugin is enough. The editor will compile the plugin **the first time** it is needed (or use prebuilt binaries if you ship them).
- **C++ project:** After enabling, **generate project files** and **build** the solution so `ParallelPrimitiveOperations` and `ParallelPrimitiveOperationsEditor` compile with your game.

### Dependencies on other plugins

The `.uplugin` file does **not** list other plugins as required. The runtime module depends only on standard engine modules (see below). No extra marketplace plugin is required for core loop nodes.

---

## 4. Plugin modules

| Module | Type | Role |
|--------|------|------|
| **`ParallelPrimitiveOperations`** | **Runtime** | `UParallelPrimitiveSubsystem`, latent loops, atomics, output buffers, cancel tokens, settings class. Ships with your game. |
| **`ParallelPrimitiveOperationsEditor`** | **UncookedOnly** | Blueprint compiler extension, loop-body safety validation, editor-only hooks. **Not** packaged into cooked builds; only exists in the editor. |

You do not reference the **Editor** module from game runtime code—only from editor-only code if you extend tooling.

---

## 5. Dependencies for third-party C++ code

If your game module calls subsystem APIs or includes plugin headers, add the runtime module to **`YourGameModule.Build.cs`**:

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    // ... your existing modules ...
    "ParallelPrimitiveOperations",
});
```

Use **`PublicDependencyModuleNames`** if your **public** headers expose plugin types (e.g. a function taking `UParallelCancelToken*`). If usage is **only** in `.cpp` files and never leaks into your public API, **`PrivateDependencyModuleNames`** is also valid.

**Headers you will include** (examples—not an exhaustive API list):

- `#include "Parallel/ParallelPrimitiveSubsystem.h"`
- `#include "Parallel/ParallelPrimitiveOperationsSettings.h"`
- Sync helpers under `Parallel/Sync/…`

The plugin’s own runtime **`ParallelPrimitiveOperations.Build.cs`** pulls in:

- **Public:** `Core`, `CoreUObject`, `Engine`, `DeveloperSettings`
- **Private:** `Engine`, `Projects`

You do **not** need to duplicate those unless your code uses those modules directly.

**Editor extensions** (custom tools that touch `UParallelLoopCompilerExtension`, etc.) would additionally depend on **`ParallelPrimitiveOperationsEditor`** and UnrealEd stack—see [C++ integration](16-Cpp-Integration.md) when that page is expanded.

---

## 6. Verify after install

Quick checklist:

1. **Editor loads** without plugin load errors (Output Log).
2. **Plugins** window shows **Parallel Primitive Operations** enabled.
3. Open any **Blueprint** (e.g. Level Blueprint or Actor), right-click the graph:
   - Search for **`Get ParallelPrimitiveSubsystem`** and add it.
   - Drag from its blue pin and search **`Parallel`** — you should see nodes under category **`Parallel | Primitives`** (and sync subcategories).
4. **Project Settings → Plugins → Parallel Primitive Operations** — default execution policy and chunk defaults should appear (settings class `UParallelPrimitiveOperationsSettings`).
5. **PIE:** A graph that calls **`Get ParallelPrimitiveSubsystem`** should return a valid subsystem object during play.

If nodes are missing, confirm the plugin is enabled, the project targets **Win64**, and the module compiled successfully.

---

## 7. Content and packaging notes

- The plugin ships **no Blueprint assets or content** — only compiled code. There is nothing to migrate or cook beyond the plugin's compiled binaries.
- Whether the plugin loads is controlled by the **`Enabled`** flag in your **`.uproject`** (or the **Plugins** window in the editor). That is the only toggle you need to care about.

---

## 8. Related pages

- **[Quick start](03-Quick-Start.md)** — first Blueprint graph step-by-step.
- **[C++ integration](16-Cpp-Integration.md)** — deeper C++ usage (outline for now).
- **[Overview](01-Overview.md)** — what the plugin does and its limits.
