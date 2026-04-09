# Execute Parallel Body

**Execute Parallel Body** is a **latent** node on **`ParallelPrimitiveSubsystem`** that runs **one** optional **“body”** chunk of work either on a **thread-pool worker** or on the **game thread**, while still giving you a small **exec** story: **Continue** (immediate, game thread), **Body**, then **Completed**.

It is **not** a loop over an array or index range—use [ForEach](06-ForEach-Loops.md) or [Parallel For](07-Parallel-For-Loop.md) for that.

---

## 1. Why this node exists

Sometimes you want:

- A **single** background-style task without inventing a fake **array of size 1** to feed ForEach.
- The same **latent** “register with the world, finish later” pattern as the loops.
- An explicit **Continue** pin that always fires on the **game thread** before **Body** runs—handy for **setup** that must stay on GT while **Body** does CPU work elsewhere (when **Force On Game Thread** is **false**).

---

## 2. Blueprint usage

1. **Right-click in the Blueprint graph** -> search **`Get ParallelPrimitiveSubsystem`** (Game Instance Subsystems).
2. Call **Execute Parallel Body** from the subsystem reference (category **`Parallel | Primitives`**, display name **Execute Parallel Body**).
3. Wire a **normal latent exec** into the node’s enter pin (e.g. after **Begin Play** or another latent **Completed**).

### Advanced pins

| Pin | Default | Effect |
|-----|---------|--------|
| **Force On Game Thread** | `false` | If **true**, **Body** runs via **`scheduled as(GameThread, …)`** instead of **`dispatched to the thread pool, …)`**. |
| **Log** | `false` | If **true**, when **Body** runs, a **`LogTemp`** line records **game thread** vs **worker** and **thread id**. |

There is **no** **Chunk Size**, **Max Concurrency**, or **Cancel Token** on this node.

---

## 3. Execution branches (`EParallelBodyExec`)

The enum drives **`ExpandEnumAsExecs="Branch"`** like the loops, but the values differ:

| Branch | When / where |
|--------|----------------|
| **Continue** | Fired **synchronously** during node startup, on the **game thread**, **before** **Body** is scheduled. Implemented in **the game-thread invoke helper** with **a scoped lock** on an internal **a critical section** so **Continue** does not overlap other guarded invokes. |
| **Body** | Scheduled **once**: either on the **game thread** (if **Force On Game Thread**) or on the **thread pool** (`EAsyncExecution::ThreadPool`). |
| **Completed** | Fired when the latent manager sees **the body-finished flag** after **Body** has finished. **the branch** is set to **Completed** and **`FinishAndTriggerIf`** resumes your graph. |

**Display quirk:** **`Continue`** uses **`UMETA(DisplayName=" ")`** in C++, so the pin label in the editor may look **blank** or minimal—use **tooltips** / pin order to identify it (first exec out after the node starts is **Continue** in practice).

---

## 4. Order of operations (timeline)

1. Latent action is **constructed** → **dispatch** runs once.
2. If **callback target** is invalid → warning log, **body finishes** immediately → **Completed** path on next latent tick (no **Continue**/**Body** in a useful sense).
3. **Valid target:** **the game-thread helper fires **Continue**** — fires on the **game thread** (with a lock so it doesn't overlap).
4. Then either:
   - **Force On Game Thread** → **`Body`** runs on the **game thread**, or
   - otherwise → **`Body`** runs on a **thread pool worker**.
5. On the next latent tick (game thread), the plugin observes that the body finished, sets the branch to **Completed**, and finishes.

**Important:** **Body** on the **worker** path uses a worker-thread path **without** the lock used for **Continue** and game-thread **Body**. That matches the engine’s general rule: **most** Blueprint execution is only safe on the **game thread**. Treat **Body** like a **parallel loop body** when **Force On Game Thread** is **false**—see [Thread safety](14-Thread-Safety.md).

---

## 5. Comparison with parallel loops

| Topic | Loops (ForEach / For) | Execute Parallel Body |
|--------|------------------------|------------------------|
| Iterations | Many (**TotalCount**) | **One** **Body** |
| **Index** / **Element** | Yes / typed ForEach | No |
| **Cancel** | **Cancel Token** | No built-in cancel |
| **Chunks / concurrency** | Yes | N/A |
| **Continue** pin | No | Yes (GT setup) |

---

## 6. Compiler integration

**`ExecuteParallelBody`** is included in the editor’s **parallel function set** ([Editor tools](15-Editor-Tools.md)): the **Loop Body** / **Body** subgraph is validated the same way—**`BlueprintThreadSafe`** calls are allowed without auto-forcing GT; otherwise the compiler may **auto-set** **Force On Game Thread** to **true** when the pin is unlinked, or warn you to set it yourself.

---

## 7. Duplicate latent guard

Same pattern as loops: if an **Execute Parallel Body** is **already running** for the same latent slot (same callback target + UUID), the second call is **skipped** with a warning in the Output Log.

---

## 8. When to use it

- **One-shot** CPU work off the game thread (math, data prep) with a **GT preamble** in **Continue**.
- Teaching the **latent + branch** model without arrays.
- Pipelines where **Completed** chains into the next latent node.

**When not to use it**

- Need **cancel** → use a **loop** with **Cancel Token** or redesign.
- Need **per-index** work → **Parallel For** / **ForEach**.

---

## 9. Full step-by-step example (human-friendly)

### Scenario

You want to prepare heavy data once (for example, build a large lookup table) and then apply results on the game thread.

This is exactly where `Execute Parallel Body` is useful:

- `Continue` for quick game-thread setup,
- `Body` for one heavy compute block,
- `Completed` for final apply.

### What to prepare in Blueprint

Create variables:

- `Subsystem` (`ParallelPrimitiveSubsystem` reference)
- `PreparedData` (your container/struct variable for the result)
- `bBuildDone` (`bool`, optional status flag)

### Step-by-step wiring

1. **Get subsystem**  
   From your start event, resolve `Parallel Primitive Subsystem`.
2. **Call Execute Parallel Body**  
   Keep `Force On Game Thread = false` for real async behavior (set true only for debug).
3. **Continue branch (immediate, game thread)**  
   Do fast setup only:
   - reset UI "Loading...",
   - clear old data,
   - set `bBuildDone = false`.
4. **Body branch (single heavy block)**  
   Run CPU-heavy calculations:
   - build intermediate values,
   - fill `PreparedData` (only in a thread-safe way),
   - do not touch actors/widgets/world objects here unless forced to game thread.
5. **Completed branch**  
   Finalize on game thread:
   - set `bBuildDone = true`,
   - apply prepared data to gameplay/UI,
   - continue to next node in the pipeline.

### Simple mental model

Read the node as:

`Continue now` -> `do one heavy task` -> `finish and continue`.

If you need **many iterations** with index/element, this is the wrong node; use ForEach or Parallel For Loop.

### Common mistakes to avoid

- Putting heavy work in `Continue` instead of `Body`.
- Calling unsafe UObject/UI functions from `Body` while `Force On Game Thread = false`.
- Expecting cancellation support (this node has no cancel token).
- Using it as a loop substitute.

### First debug run (recommended)

Start with `Force On Game Thread = true` and verify behavior.

Then switch back to `false` and keep `Body` strictly thread-safe.

---

## 10. Related pages

- [Blueprint workflow](05-Blueprint-Workflow.md) — latent wiring, branch enums.
- [Execution parameters](08-Execution-Parameters.md) — **Force** / **Log** naming vs loop pins.
- [Thread safety](14-Thread-Safety.md) — **Body** on workers.
- [Architecture](04-Architecture.md) — **the latent action for Execute Parallel Body**.
