# Project settings (Developer Settings)

The plugin’s **default** loop behavior (when you leave **Advanced** pins at “use defaults”) comes from **`UParallelPrimitiveOperationsSettings`**, a standard Unreal **`UDeveloperSettings`** asset. This page explains **where** to edit it, **what** each field does, and **how** it combines with per-node pins. For the full resolution algorithm, see [Execution parameters](08-Execution-Parameters.md).

---

## 1. Where to find it in the editor

**Path:** **Edit → Project Settings → Plugins → Parallel Primitive Operations**

The section and display name are both **Parallel Primitive Operations**.

---

## 2. Class overview (`UParallelPrimitiveOperationsSettings`)

| Property | Meaning |
|----------|---------|
| **Config storage** | Values are stored in **game** config (e.g. `DefaultGame.ini` / per-platform **`Game.ini`** under your project’s **Config** and **Saved** hierarchy). |
| **Default values** | The plugin ships sensible defaults; you only need to change them if you want a different baseline. |
| **Blueprint access** | Settings are readable from Blueprint, but they are intended to be configured in **Project Settings** — not changed during gameplay. |

You do **not** need to create a separate asset — Unreal automatically manages these settings as a project-global singleton.

---

## 3. Default Execution Policy

Dropdown in **Project Settings → Execution** (category on the class).

| Value | Display name | What it does |
|-------|----------------|---------------|
| **Parallel Workers** | **Parallel Workers** | Loop work is scheduled on the **engine thread pool**. Chunks run in parallel across CPU cores. This is the **default**. |
| **Worker Single Thread** | **Worker Single Thread** | **One** background task processes chunks **sequentially**. The game thread is free, but there is no multi-core parallelism. |
| **Game Thread** | **Game Thread** | All chunk processing runs on the **game thread**. Safest for Blueprint that is not thread-safe, but provides no worker-thread speedup. |

**Override from nodes**

- If **Force Parallel On Game Thread** is **checked** on a **ForEach** / **Parallel For** node, that call **always** uses **Game Thread** policy for scheduling, **ignoring** this default.

---

## 4. `Default Chunk Size`

- **Type:** `int32`, **UI clamp ≥ 0** (`ClampMin` / `UIMin`).
- **Default in code:** **64**.
- **Meaning:** When the Blueprint node’s **Chunk Size** pin is **`≤ 0`**, the runtime uses **`Default Chunk Size`** from settings (then enforces a minimum effective chunk of **1** in code—the actual chunk size is always at least **1**).

**Intuition:** Larger chunks → fewer chunk objects, less scheduling overhead, less load balancing. Smaller chunks → more parallelism opportunities when **`NumChunks`** is large. See [Execution parameters](08-Execution-Parameters.md) §4.

---

## 5. `Default Max Concurrency`

- **Type:** `int32`, **UI clamp ≥ 0**.
- **Default in code:** **0**.
- **Meaning:** **`0` means “not set here—use automatic worker count.”** When the **node’s** **Max Concurrency** is also **`≤ 0`**, the engine uses **`FPlatformMisc::NumberOfWorkerThreadsToSpawn()`** (at least **1**), then **clamps** to the number of chunks.

If you set **`Default Max Concurrency` > 0**, that value becomes the **cap** whenever the node does **not** pass a positive **Max Concurrency**.

**Relationship**

- **Parallel Workers** only: concurrency > 1 matters.
- **Worker Single Thread** / **Game Thread** (including **Force on GT**): effective concurrency is **1** regardless of this number.

---

## 6. Config files and version control

- Defaults often live in **`Config/DefaultGame.ini`** (or merged from plugins) under a section for this settings class.
- Per-user or per-platform overrides may appear under **`Saved/Config/.../Game.ini`**.
- **Teams:** commit **`DefaultGame.ini`** (or documented overrides) so everyone shares the same parallel defaults; avoid relying only on local **Saved** edits.

**Restart:** Changing **Developer Settings** usually applies **on next editor run** or when the config is reloaded. If behavior seems stale, restart the editor after editing **DefaultGame.ini** manually.

---

## 7. How settings interact with Blueprint nodes

| Node pin | If you leave it “0” or unchecked | Effect |
|----------|-------------------------------------|--------|
| **Chunk Size** `≤ 0` | Uses **`Default Chunk Size`** (then `max(1, …)`). |
| **Max Concurrency** `≤ 0` | Uses **`Default Max Concurrency`**; if that is **`0`**, uses **auto** thread count. |
| **Force Parallel On Game Thread** **false** | Uses **`Default Execution Policy`**. |
| **Force Parallel On Game Thread** **true** | **Game Thread** policy for that invocation only. |

Per-node values **do not** change **Project Settings**; they only **override** resolution for that single loop call.

---

## 8. Blueprint-only projects

You **do not** need C++ to benefit: open **Project Settings** and adjust the three execution fields. The subsystem reads **`GetDefault<UParallelPrimitiveOperationsSettings>()`** at runtime when resolving loops.

---

## 9. Related pages

- [Execution parameters](08-Execution-Parameters.md) — how chunk size and concurrency are resolved at runtime.
- [Architecture](04-Architecture.md) — where settings are read in the loop pipeline.
- [Overview](01-Overview.md) — high-level policy glossary.
