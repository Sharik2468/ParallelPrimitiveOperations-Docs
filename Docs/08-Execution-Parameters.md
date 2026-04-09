# Execution parameters

This page explains every **tuning pin** shared by typed **ForEach** and **Parallel For Loop**, plus how they interact with **Project Settings**. **`Execute Parallel Body`** only uses a **subset** (force game thread + log) — that is called out at the end.

---

## 1. Overview table

| Pin / setting | Applies to | Role |
|---------------|------------|------|
| **Force Parallel On Game Thread** | ForEach, Parallel For | **Overrides** project **execution policy** → **Game Thread** for this call. |
| **Log** | ForEach, Parallel For | Enables extra **Output Log** lines from loop execution (e.g. per-body thread ID when implemented). |
| **Chunk Size** | ForEach, Parallel For | Elements (or range slots) per **chunk**; drives **NumChunks** and granularity of work stealing. |
| **Max Concurrency** | ForEach, Parallel For | Upper bound on **parallel worker tasks** (for **Parallel Workers** policy only). |
| **Cancel Token** | ForEach, Parallel For | Optional **`UParallelCancelToken`** for cooperative cancel → **Cancelled** branch. |
| **Default Execution Policy** | Project Settings | Used when **Force** is **false**. |
| **Default Chunk Size** | Project Settings | Used when node **Chunk Size ≤ 0** (after resolution). |
| **Default Max Concurrency** | Project Settings | Used when node **Max Concurrency ≤ 0** (combined with auto worker count). |

---

## 2. `Force Parallel On Game Thread` (`bForceParallelOnGameThread`)

**Blueprint pin name:** e.g. **Force Parallel On Game Thread** (Advanced).

**Effect**

- When **true**, the loop **always** uses **Game Thread** policy for that call, **regardless** of **Project Settings → Default Execution Policy**.
- One task is dispatched to the **game thread** so every chunk runs there — same latent + chunk structure, but no **thread pool** workers.

**Why use it**

- **Debugging** Blueprint that is not safe on background threads.
- Temporary **safety** while prototyping.
- Forcing **deterministic** interaction with **UObject** / world APIs that require GT.

**Trade-off**

- You lose **CPU parallelism** across cores for the loop; you may still get **latent** “spread” across frames depending on engine load, but the heavy work is **not** offloaded to the pool.

**Note:** **`Execute Parallel Body`** has a **separate** pin called **`Force On Game Thread`** with the same idea but it is a different node — check the right node when looking at settings.

---

## 3. `Log`

**Effect**

- When **true**, the plugin emits a log line per body invocation (index, which thread ran it, thread ID).

**When to enable**

- Local **diagnostics**, **order** experiments, verifying which thread runs the body.

**When to disable**

- **Shipping** or large **N** — logging can **hurt performance** and flood the log.

---

## 4. `Chunk Size`

**Resolution logic**

1. If the **node pin** value is **`> 0`**, use it as the chunk size.
2. Otherwise use **`Default Chunk Size`** from **Project Settings** (default **64** if the setting is not configured).
3. The result is **clamped to at least 1** — you can never have a zero-sized chunk.

**Meaning**

- The loop’s **`TotalCount`** iterations are grouped into **`NumChunks = ceil(TotalCount / ChunkSize)`** chunks.
- Each **worker** repeatedly **claims** the next chunk index (atomically) and processes **all indices inside that chunk** **sequentially**.

**Intuition**

| Chunk size | Behavior |
|------------|----------|
| **Small** (e.g. 1–8) | Many chunks, **finer** load balance across workers, **more** overhead from claiming / scheduling. |
| **Medium** (e.g. 32–128) | Common default range; balances **cache locality** and **parallelism**. |
| **Large** (≥ **TotalCount**) | **One** chunk → only **one** worker does all work even if **Max Concurrency > 1** (other workers find no chunks). |

**Tuning tip:** If **Max Concurrency** is high but **`NumChunks` is 1**, increasing concurrency **does nothing**. Increase **`TotalCount`/ChunkSize ratio** (smaller chunks or more elements).

---

## 5. `Max Concurrency`

**Only matters** when the **resolved policy** is **`ParallelWorkers`**. For **`GameThread`** or **`WorkerSingleThread`**, effective concurrency is **1** regardless of this pin.

**Resolution logic** (when there is at least one iteration to run)

1. If the **node pin** is **`> 0`**, use it.
2. Otherwise use **`Default Max Concurrency`** from Project Settings.
3. If that is also **`0`**, auto-pick based on the platform's worker thread count (at least 1).
4. The final value is **clamped** between 1 and the number of chunks — extra workers beyond chunk count would sit idle.

**Reading that in English**

- **`0` on the node** means “do **not** use this pin; fall back to **settings**, and if settings are also **`0`**, use **platform auto** worker count.”
- **`0` in Default Max Concurrency** in settings means “use **auto**” when the node also passes **`0`**.
- You **never** get more concurrent workers than **chunks** — extra workers would sit idle.

**Empty loop (zero iterations)**

- No workers are started and **Completed** fires immediately — concurrency resolution is skipped.

---

## 6. Relationship to `EParallelExecutionPolicy` (project defaults)

Configured in **Project Settings → Plugins → Parallel Primitive Operations** (`UParallelPrimitiveOperationsSettings`).

| Policy | Effect on scheduling |
|--------|----------------------|
| **Parallel Workers** | Multiple tasks on the **engine thread pool** (up to `min(MaxConcurrency, NumChunks)`). |
| **Worker Single Thread** | **One** task on the **thread pool** (not game thread); effective concurrency forced to 1. |
| **Game Thread** | **One** task on the **game thread**. |

**Precedence**

- **`Force Parallel On Game Thread` == true** → always **Game Thread** policy for that invocation.
- Otherwise → **Default Execution Policy** from settings.

See [Developer Settings](09-Developer-Settings.md) for editing and config files.

---

## 7. `Cancel Token`

Optional **`UParallelCancelToken`**. When **`Cancel()`** sets the shared flag, the latent tick prefers **Cancelled** over **Completed**. Cooperative — not instant mid-instruction. Full detail: [Cancel Token](10-Cancel-Token.md).

---

## 8. Tuning recommendations (practical)

These are **rules of thumb**, not benchmarks.

| Scenario | Suggestions |
|----------|-------------|
| **Huge N**, cheap per-element body | Moderate **chunk size** (32–128), **Max Concurrency** default/auto or slightly below CPU cores; enable **Parallel Workers** in settings. |
| **Heavy** per-element Blueprint | Prefer **Force on Game Thread** or **Worker Single Thread** first until the body is **thread-safe**; then re-enable **Parallel Workers**. |
| **Small N** (< few hundred) | Parallelism may not pay off; **chunk** large or use **simple For Loop** if latency dominates. |
| **Must preserve GT-only UObject rules** | **Force Parallel On Game Thread** or **Game Thread** policy. |
| **Debugging order** | **Force on Game Thread** + **Log**; remember even GT scheduling can interleave with other game work — for **strict** order use **single worker** policies. |
| **Avoid log spam** | **Log** off for large **N**. |

---

## 9. `Execute Parallel Body` subset

**Pins:** **`Force On Game Thread`**, **`Log`** only (Advanced). **No** chunk, concurrency, or cancel on that node. Policy is implicit: **game thread** vs **`Async(ThreadPool)`** for the **single** **Body** invocation. See [Execute Parallel Body](12-Execute-Parallel-Body.md).

---

## 10. Related pages

- [Developer Settings](09-Developer-Settings.md)
- [Architecture](04-Architecture.md) — how workers are launched and chunks are processed.
- [ForEach loops](06-ForEach-Loops.md) · [Parallel For Loop](07-Parallel-For-Loop.md)
- [Cancel Token](10-Cancel-Token.md)
- [Overview](01-Overview.md) — glossary (**Max Concurrency**, **chunk**).
