# ForEach: typed nodes

This page covers typed **parallel / sequential ForEach** nodes: available overloads, what the **exec branches** mean, and how **Index** / **Element** behave. For scheduling knobs (chunk size, concurrency, game thread), see [Execution parameters](08-Execution-Parameters.md). For subsystem wiring, see [Blueprint workflow](05-Blueprint-Workflow.md).

---

## 1. Where these nodes live

All ForEach entry points are **`BlueprintCallable`** methods on **`UParallelPrimitiveSubsystem`**. In Blueprint:

1. Right-click the graph and add **`Get ParallelPrimitiveSubsystem`**.
2. From the subsystem reference, pick the node you need.

There is no separate “free floating” ForEach node in the plugin API.

---

## 2. Typed overloads (full list)

Each overload shares the same **Advanced** scheduling pins; only the **array** and **element** types change.

| Display name (Blueprint) | Element type | Typical use |
|--------------------------|--------------|-------------|
| **Parallel ForEach Loop Int** | `int32` | Counters, IDs, packed indices |
| **Parallel ForEach Loop Bool** | `bool` | Bitmasks, flags |
| **Parallel ForEach Loop Byte** | `uint8` / byte | Small enums, raw bytes |
| **Parallel ForEach Loop Float** | `float` | Samples, scalars |
| **Parallel ForEach Loop Name** | `FName` | Stable names, tags |
| **Parallel ForEach Loop String** | `FString` | Text processing (mind allocations) |
| **Parallel ForEach Loop Text** | `FText` | Localized text (usually heavier) |
| **Parallel ForEach Loop Vector** | `FVector` | Positions, directions |
| **Parallel ForEach Loop Rotator** | `FRotator` | Orientations |
| **Parallel ForEach Loop Transform** | `FTransform` | Full transforms |

On **Parallel ForEach Loop Int**, the pins are labeled **Array Element** and **Array Index** in metadata; other types use the standard parameter names in the UI.

---

## 3. Branch semantics (`EParallelForEachExec`)

| Branch | When it runs | Typical use |
|--------|----------------|--------------|
| **Loop Body** | Once per iteration (or **never** if `Num == 0`). **`Index`** is valid; on typed nodes **Element** is the snapshot value for that index. | Per-element math, writing into [output buffers](11-Sync-Primitives.md), atomics, etc. |
| **Completed** | All iterations finished **and** cancel was not taken on the cancel path. | Enable UI, start next pipeline step, log “done”. |
| **Cancelled** | **Cancel Token** requested cancellation and the latent manager observed it. | User abort, unload, timeout handling. |

**Ordering:** With **multiple workers** and **Parallel Workers** policy, **Loop Body** may run in **non-array order** across indices. Do not rely on “previous index finished first” for correctness. See [Overview](01-Overview.md).

**Empty array:** **`TotalCount == 0`** → no worker iterations; **Completed** still fires (no **Loop Body**). See [Architecture](04-Architecture.md).

---

## 4. Element, Index, and snapshot semantics (typed nodes)

### Snapshot at start

When you start a typed ForEach, the engine **copies** the **`TArray<T>`** you passed in into an internal **snapshot** (see [Architecture](04-Architecture.md)).

**Why**

- Avoids data races if something on the **game thread** resizes or mutates the **original** array while background workers read it.

**What you should assume**

- **Element** in **Loop Body** reflects the array **as it was when the loop started**, for that **Index**.
- Changes to your **original** Blueprint array variable **during** the loop **do not** retroactively change values seen in an already-running loop.

### Reference vs value in Blueprint

Pins behave like normal Blueprint **outputs** for the current iteration: you read **Element** and **Index** for this step. Treat **Element** as **read-only** for algorithm correctness unless you intentionally use it only as local scratch (it does not write back to the source array).

## 5. Comparison with **Parallel For Loop**

| Topic | ForEach | [Parallel For Loop](07-Parallel-For-Loop.md) |
|--------|---------|-----------------------------------------------|
| Input | **Array** (`TArray<T>`) | **First Index**, **Last Index** (inclusive range) |
| **Element** pin | Yes (typed ForEach) | No — only **Index** |
| Internal indexing | `0 … Num−1` mapped to elements | Logical **`Index = First + k`** |
| Empty work | `Num == 0` | **`Last < First`** → zero iterations |

Use **ForEach** when data already lives in an array. Use **Parallel For** when you only need a numeric range (simulate grid coordinates, worker IDs, etc.).

---

## 6. Example scenarios (Blueprint-friendly)

### Large float buffer processing

- Array of **floats** (samples, heights). **Parallel ForEach Loop Float**, body writes into **`Make Output Buffer Float`** via **`Try Set Once`** or **`Set`** by **Array Index**. **Completed** → **`To Array Copy`** on game thread for further use.

### Integer ID batch

- **Parallel ForEach Loop Int** over IDs; body only updates **Atomic Int** statistics or a **buffer** keyed by index.

### Vectors without UObject

- **Parallel ForEach Loop Vector**; body normalizes or transforms into another **buffer** or **TArray** you preallocated—avoid touching **actors** unless **forced to game thread**.

## 7. Pitfalls

| Pitfall | Consequence |
|---------|-------------|
| Mutating the **source** array mid-loop expecting new **Element** values | **Element** comes from **snapshot** — stale view. |
| Using **Loop Body** for **actor** spawn/destroy with **parallel workers** | **Unsafe** — use **Force Parallel On Game Thread** or redesign ([Thread safety](14-Thread-Safety.md)). |
| Assuming **Loop Body** order == array order under **Parallel Workers** | Wrong results for order-dependent logic. |
| Starting a **second** ForEach from the same **latent slot** while the first runs | Second call **ignored** ([Architecture](04-Architecture.md)). |

---

## 8. Related pages

- [Parallel For Loop](07-Parallel-For-Loop.md) — range-based loop.
- [Execution parameters](08-Execution-Parameters.md) — chunk, concurrency, force GT, log.
- [Cancel Token](10-Cancel-Token.md) — **Cancelled** branch.
- [Sync primitives](11-Sync-Primitives.md) — buffers and atomics in the body.
- [Blueprint workflow](05-Blueprint-Workflow.md) — categories and latent wiring.
- [Editor tools](15-Editor-Tools.md) — validation of **Loop Body** graphs.
