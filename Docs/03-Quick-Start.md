# Quick start

This walkthrough gets you from **zero** to a **working parallel ForEach** in Blueprint. It stays minimal on purpose; tuning and safety rules live in other Wiki pages.

---

## 1. What you will build

You will:

1. In the Blueprint graph, add **`Get ParallelPrimitiveSubsystem`** and use its blue pin as the entry point.
2. Run **Parallel ForEach Loop Int** on an **array of integers**.
3. Do something simple in **Loop Body** (e.g. accumulate with an **atomic** or just print/log).
4. Continue the game flow from **Completed**.

**Assumptions**

- Plugin is **installed and enabled** (see [Installation](02-Installation.md)).
- You are in a **Win64** PIE or standalone session with a normal **Game Instance** (default in editor play).

---

## 2. Why start from the subsystem?

All parallel loop nodes are **callable functions** on **`UParallelPrimitiveSubsystem`**. Unlike a standalone “magic” node, you:

1. Resolve **`Parallel Primitive Subsystem`** once per call chain (or cache it in a variable).
2. Call **`Parallel ForEach Loop Int`** (or another overload) **on that subsystem reference**.

That pattern matches how **Game Instance Subsystems** work in UE: one subsystem instance per game instance, stable across the session, and a natural place for **latent** (long-running) operations.

---

## 3. Blueprint steps (minimal int array loop)

### Step 1 — Pick a gameplay Blueprint graph

Use any gameplay Blueprint graph, for example:

- **Level Blueprint**
- **Actor** placed in the level
- **Player Controller** or **Game Mode** Blueprint

You need an execution pin (e.g. **Begin Play**) to start the latent chain.

### Step 2 — Get the subsystem

1. From your start event, right-click the graph and add **`Get ParallelPrimitiveSubsystem`** (Game Instance Subsystems).
2. From the **blue subsystem pin**, pull and call loop/sync functions (`Parallel ForEach Loop ...`, `Parallel For Loop`, `Make Atomic ...`, etc.).

### Step 3 — Call the loop node

From the **subsystem output pin**, call:

- **`Parallel ForEach Loop Int`**  
  (display name may match; category **`Parallel | Primitives`**)

**Wire:**

- **`Target Array`** — your `TArray<int32>` (e.g. a variable **MyInts** with a few elements for testing).
- **Latent execution** — connect the **white exec** from your start event into this node’s **execute** input (latent nodes use the standard latent pin pattern).

### Step 4 — Loop Body vs Completed

The node exposes execution branches (enum-driven), typically:

| Branch | When it runs |
|--------|----------------|
| **Loop Body** | Each iteration (with **`Index`** and **`Element`** set for that step). |
| **Completed** | All iterations finished successfully (and you did not take the cancel path). |
| **Cancelled** | Cancel token requested stop (optional for this minimal tutorial). |

**Minimal wiring:**

1. From **Loop Body**, run your per-element logic (keep it **light and thread-safe** for real parallel use—see [Thread safety](14-Thread-Safety.md)).
2. **Do not** branch “off” the latent chain incorrectly: latent ForEach-style nodes expect the body to **return** into the same latent node’s flow (same pattern as the engine’s latent loops). Follow the pin labels in the editor.
3. From **Completed**, run whatever should happen **after** the whole array is processed (e.g. print “Done”, enable UI).

### Step 5 — Advanced pins (optional first pass)

Expand **Advanced** on the node to see:

- **`Force Parallel On Game Thread`** — forces work onto the **game thread** (slower but safest for debugging Blueprint that is not thread-safe).
- **`Log`** — extra logging from the subsystem.
- **`Chunk Size`**, **`Max Concurrency`**, **`Cancel Token`** — see [Execution parameters](08-Execution-Parameters.md) and [Cancel Token](10-Cancel-Token.md).

For a **first successful run**, you can leave defaults: with **parallel workers** and multiple threads, **order of Loop Body calls is not guaranteed** across indices (see below).

---

## 4. Slightly richer example (optional): sum with an atomic

If you want to verify **parallel-safe** aggregation:

1. Before the loop, on the same subsystem, call **`Make Atomic Int32`** with initial value **`0`**.
2. Store the returned object in a variable (e.g. **`Sum`**).
3. In **Loop Body**, call **`Add`** on **`Sum`** with **`Element`** (or `1` to count iterations).
4. On **Completed**, **`Get`** the atomic and print or store the total.

This gives you a fast sanity check of parallel-safe aggregation without extra test assets.

---

## 5. What to expect in PIE / Standalone

### Latent behavior

- The loop **does not** freeze the entire game in a single frame like a tight synchronous For Loop might. The engine **ticks** the latent action until work completes, then fires **Completed** (or **Cancelled**).

### Order of iterations

- With **default parallel policy** and **multiple workers**, **do not assume** **Loop Body** runs in **array order** (index 0, then 1, then 2…). Different indices may be processed **out of order** or interleaved.
- If you need **strict order**, use **Force Parallel On Game Thread** or change **Project Settings → Plugins → Parallel Primitive Operations** to a **single-threaded** policy, or design logic that only uses **per-index** writes (e.g. output buffers). Details: [Overview — limitations](01-Overview.md).

### Empty array

- If the array is **empty**, expect the loop to **complete** without executing the body (no iterations)—behavior aligns with “zero total count” in the implementation. Always handle **Completed** for cleanup.

### Multiplayer

- This quick start does **not** cover replication. Treat parallel loops like any other **server/client** logic: **authority**, **RPCs**, and **game state** still follow UE rules. See [Troubleshooting & FAQ](18-Troubleshooting-FAQ.md) when expanded.

---

## 6. Common first-time pitfalls

| Symptom | Likely cause |
|--------|----------------|
| No **Parallel** nodes in context menu | Plugin disabled, wrong Blueprint type, or module failed to build. |
| Subsystem is **None** | **World context** invalid, or called too early before Game Instance exists. |
| **Crash** in **Loop Body** | **Non-thread-safe** Blueprint (actors, UI, most UObject calls) while running on **worker threads**. Force **game thread** while learning, then read [Thread safety](14-Thread-Safety.md). |
| **Completed** never fires | Broken latent chain (another latent action swallowed execution), or logic error; check Output Log. |

---

## 7. Next steps

- **[Execution parameters](08-Execution-Parameters.md)** — chunk size, concurrency, logging, force game thread.
- **[Cancel Token](10-Cancel-Token.md)** — stop long loops cooperatively.
- **[Sync: Atomics & output buffers](11-Sync-Primitives.md)** — safe results from parallel bodies.
- **[Overview](01-Overview.md)** — mental model and glossary.

Once this flow works, try **Parallel For Loop** over a numeric range ([Parallel For Loop](07-Parallel-For-Loop.md)) or another **typed ForEach** (float, vector, etc.) using the **same subsystem** pattern.
