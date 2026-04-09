# Overview & goals

This page is the entry point for **Parallel Primitive Operations**: why the plugin exists, what you get in practice, and where the boundaries are. The rest of the Wiki goes into installation, Blueprint nodes, and safety rules.

---

## 1. Short description

**Parallel Primitive Operations** is a plugin for **Unreal Engine 5.6** that adds **latent loops** in Blueprint (also usable from C++) over arrays of **primitive types** and over a **range of integer indices**. Unlike a normal Blueprint **For Each**, which runs the loop body synchronously on the game thread, this plugin can **spread work across the engine thread pool**, **batch elements into chunks**, and still feel like a familiar **latent node** with **Loop Body**, **Completed**, and **Cancelled** branches.

In plain terms: if you have a large array of numbers, vectors, strings, or other **simple types**, and each element needs **heavy but thread-safe** work (math, data prep), the plugin helps avoid blocking the frame as much as a classic graph loop would. Everything is driven through **`ParallelPrimitiveSubsystem`** at **Game Instance** level — get the subsystem once and call the node like any other latent UE call.

**Who it is for**

- **Technical designers and gameplay Blueprint authors** — primary audience: nodes under `Parallel|Primitives`, settings in Project Settings.
- **C++ programmers** — the same primitives and subsystem are available from code; you can keep loop logic in C++ and use Blueprint as orchestration.
- **Teams that want editor guidance** — a dedicated **editor module** inspects the graph around parallel loops and warns about potentially unsafe nodes in the loop body.

---

## 2. Key features

- **Primitive array iteration** — dedicated typed Blueprint nodes for `int`, `bool`, `byte`, `float`, `Name`, `String`, `Text`, `Vector`, `Rotator`, `Transform`.
- **Parallel For over a range** — loop from `FirstIndex` through `LastIndex` with the same scheduling parameters as ForEach.
- **Flexible execution policy** (via **Project Settings → Parallel Primitive Operations** and **Force on Game Thread** on the node):
  - **Parallel Workers** — multiple tasks on the engine **thread pool** (`ThreadPool`); elements are processed in **chunks**; concurrent workers are capped by **Max Concurrency** (or an auto estimate from the platform worker count).
  - **Worker Single Thread** — one background pool thread: no parallelism across workers, but the game thread is not running the loop body.
  - **Game Thread** — chunk work is scheduled on the **game thread** (handy for debugging or when project policy requires the main thread only).
- **Chunks (Chunk Size)** — the array is split into blocks; inside one chunk indices run **in order**, while multiple workers may share **different chunks**. Chunk size and concurrency can be set on the node and defaulted in settings.
- **UE latent model** — the loop does not stall a single tick forever; the engine ticks the latent action until all iterations finish (or cancel fires), then the matching branch runs.
- **Cancellation** — a **Cancel Token** lets you request stop from outside; the graph can take **Cancelled** instead of **Completed** when cancel is observed.
- **Result synchronization** — **atomic** counters/flags (`Atomic Bool/Int/Float`) and **per-index output buffers** (`Output Buffer` for int, float, vector, transform) to collect results from parallel iterations safely.
- **Extra nodes** — e.g. **Execute Parallel Body** (a one-shot body with a similar branch model) plus synchronization/object-factory helpers.
- **Editor tooling** — on Blueprint compile, a dedicated editor module **validates the exec subgraph** connected to **Loop Body** and warns about (or auto-fixes) unsafe node patterns.

Remember: **parallel does not mean “any Blueprint is allowed.”** The loop body must still obey thread-safety rules — see [Thread safety](14-Thread-Safety.md) and [Editor tools](15-Editor-Tools.md).

---

## 3. Differences from built-in UE loops (For Each, For Loop)

| Aspect | Standard Blueprint For Each / For Loop | Parallel Primitive Operations |
|--------|----------------------------------------|--------------------------------|
| **When the body runs** | Usually **same thread and stretch of time** while the node runs — classic loops are a synchronous pass; large arrays can hitch the frame. | The body is **scheduled** as a series of callbacks; heavy work can run on the **thread pool** (or one worker / game thread per policy). The frame is **not required** to run every iteration back-to-back like a simple loop. |
| **Latency model** | The stock **Latent For Each** is a **different** model (e.g. one element per tick); do not confuse it with this plugin. | Explicit **Loop Body** / **Completed** / **Cancelled** output pins on a latent subsystem node. |
| **Element order** | Normal For Each order is **predictable** (array order). | With **multiple workers**, **Loop Body** call order across indices is **not guaranteed** and may differ from array order. Order-dependent logic in the body is **unsuitable** for parallel mode. |
| **Data types** | For Each accepts **many** array types, including UObject and structs. | **Primitives and a fixed set of types** (see subsystem nodes). Actor arrays or arbitrary `USTRUCT` are **not** the target. |
| **Entry point** | Local node on the graph. | Call through **`UParallelPrimitiveSubsystem`** (Blueprint getter: **`Get ParallelPrimitiveSubsystem`**). |
| **Safety** | Game thread only — simpler, slower for heavy math. | Speedups only if the body **does not touch the world / UObject** unsafely; the editor helps catch common mistakes. |

Bottom line: the plugin **complements** built-in loops. Use it when **primitive scale** and **moving CPU work** off the hot path matter, under strict rules.

---

## 4. Limitations & assumptions

### Platform & engine

- The plugin manifest lists **Win64** and **Unreal Engine 5.6**. Other platforms may **omit the module** from the build — verify `.uplugin` and packaging before console/mobile; do not assume it works out of the box where the platform is not allowlisted.

### Data types

- Focus is on **primitives** and the types exposed in the API. A full **parallel For Each over actors** is **not** the goal (UObject, world, replication).

### Order & determinism

- With **multiple workers**, **Loop Body** invocation order is **non-deterministic** vs index. For strict order, use a policy with **one** executor (**Game Thread** or **Worker Single Thread**) or design so order does not matter (per-index atomics/buffers).

### Game thread & UObject

- Most Blueprint calls touching **actors, components, widgets** expect the **game thread**. The plugin **does not** change that: parallel bodies must stay **safe**. Details: [Thread safety](14-Thread-Safety.md).

### Cancellation timing

- **Cancel Token** is **cooperative**: work stops when the scheduler and iterations **reach a check**. It is not a hard mid-node interrupt for arbitrary Blueprint.

### Plugin content

- **No content** (`CanContainContent: false`) — code and editor logic only.

---

## 5. Glossary

- **Game Instance Subsystem** — subsystem bound to the **game instance**; here **`ParallelPrimitiveSubsystem`**. One per instance; good anchor for latent ops with correct **WorldContext**.

- **Latent action** — UE mechanism: the node **does not finish immediately**; the engine **revisits** it until done. Basis for **Completed** / **Cancelled**.

- **Loop Body** — branch fired per iteration (sets **Index** and, on typed nodes, **element**). The plugin ensures that at most one **Loop Body** invocation runs at a time in Blueprint — two workers never corrupt each other's output variables.

- **Completed** — all scheduled iterations **finished successfully** (and cancel did not take the cancel path).

- **Cancelled** — cancel via **Cancel Token** (or related state); graph can branch here instead of **Completed**.

- **Chunk** — contiguous index range of fixed **Chunk Size**. A worker atomically takes the **next** chunk index and processes indices inside it in order.

- **Max Concurrency** — upper bound on **concurrent workers** draining chunks (**Parallel Workers**). **0** in settings or on the node means “use defaults”: if no explicit cap, the engine automatically picks a sensible number based on the CPU, clamped to the number of chunks.

- **Parallel Workers** — policy: multiple tasks on the **thread pool** (runs on the engine thread pool).

- **Worker Single Thread** — one pool worker: **no** parallel chunk workers, and **not** running the body on the game thread unless switched otherwise.

- **Game Thread (policy / forced)** — loop worker work is **run on the game thread** — useful for debugging and code that **must** run only on UE’s main thread.

- **Primitive operations** — in the plugin name: not rendering “primitives,” but **simple value types** and **low-level** safe background work.

- **Output Buffer / Atomic** — plugin sync objects for **per-index writes** or **atomic arithmetic** without iteration races (see [Sync primitives](11-Sync-Primitives.md)).

---

## 6. Related Wiki pages

- [Installation & requirements](02-Installation.md) — enable the plugin and build deps.
- [Quick start](03-Quick-Start.md) — first working graph.
- [Architecture](04-Architecture.md) — subsystem and latent lifecycle beyond this overview.
- [Thread safety & limits](14-Thread-Safety.md) — read before heavy parallel use.

**Rule of thumb:** this plugin is a **yes** when you have lots of **numbers/vectors/strings** and **pure** heavy processing; **no** when the goal is iterating **actors** or strict **multi-threaded order** without changing execution policy.
