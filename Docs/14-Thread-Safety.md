# Thread safety & limitations

Parallel Primitive Operations can run **Blueprint** (and native) **loop bodies** on **worker threads** when **policy** and pins allow it. Unreal Engine assumes **most** gameplay and **`UObject`** APIs run on the **game thread**. This page explains **where** your code runs, what the **editor** checks, and how to stay safe—without hand-waving.

---

## 1. Where does my code run?

### Parallel ForEach / Parallel For

- **Policy = Parallel Workers** (and **Force Parallel On Game Thread** = **false**): workers run on **engine thread pool** tasks. Each iteration invokes the **Loop Body** branch in your Blueprint.
- Internally, loop body invocations are **serialized** so two workers cannot set up the same Blueprint callback at the same time—but your body's logic **still executes** on that worker's thread (unless policy forces **game thread**).

### Force Parallel On Game Thread / Game Thread policy

- **Loop Body** is queued to run on the **game thread**—same rules as any normal Blueprint execution.

### Worker Single Thread

- **One** pool thread runs chunks; **not** the main game thread, but **only one** worker—no **multi-core** parallel **Blueprint** bodies. Still **not** game thread: **treat like workers** for UObject safety unless you know every call you make is thread-safe.

### Execute Parallel Body

- **Continue**: always **game thread**, with an internal lock ensuring it doesn't overlap other invocations.
- **Body** with **Force On Game Thread**: **game thread**.
- **Body** without force: runs on a **pool thread** without the serialization lock — **especially sensitive** to unsafe Blueprint calls.

### Latent manager & utilities

- Completion and cancel checks happen in normal **game-thread** latent processing.

---

## 2. What Unreal expects on the game thread

Assume **unsafe off GT** unless documented otherwise:

- **Actors, Components, Pawns**, **World**, **Level**, **Physics**, **Spawning**, **Destroy**, most **UMG** / **Slate**, **timelines** tied to world, **many** **`BlueprintCallable`** functions on engine types.

**`BlueprintThreadSafe`** on a **`UFUNCTION`** is the engine's way to mark **"may be called from background Blueprint graphs."** The plugin's **atomics**, **buffer reads/writes** marked **`BlueprintThreadSafe`**, **Cancel** / **Is Cancelled**, etc., follow that contract.

---

## 3. What the editor actually validates

On **Blueprint compile**, **the editor compiler extension** finds calls to **`UParallelPrimitiveSubsystem`** functions:

- All typed **ForEach** variants, **For Loop Parallel Sequence**, **Execute Parallel Body**.

It locates the **Loop Body** / **Body** exec output pin and walks the exec subgraph from there.

**Rule:**

- Walk **downstream** along **exec** wires starting from that pin.
- **Skip** traversing into **Completed** / **Cancelled** (and related) outputs of the **same** parallel node so completion logic is not flagged as "body."
- For each **function call node** on that exec chain: if the function does **not** have **`BlueprintThreadSafe`** metadata → **unsafe**.
- Other node types (flow control, custom events, etc.) are **traversed** (their output exec pins are followed) but only **function call nodes** are classified this way.

**What is *not* covered by this walk**

- **Pure** / **no-exec** chains (math, getters) are **not** part of the **exec** traversal. Unsafe **pure** calls on **data** pins can still run from **worker** bodies—the validator does **not** prove purity safety.
- **Macros** / **collapsed graphs**—behavior depends on how they expand; treat complex macros as **review manually**.
- **Native** code you call from Blueprint—**your** responsibility.

### Auto-fix on compile

If unsafe nodes are found **and** **Force Parallel On Game Thread** (or **Force On Game Thread** for **Execute Parallel Body**) is **unlinked** and not already **true**, the compiler **sets the default** to **`true`** and logs a warning—**forcing game-thread** execution for that node so **Loop Body** runs on GT.

If the force pin is **linked** from another value, the compiler **cannot** auto-toggle; it **warns** only.

---

## 4. Allowed patterns in parallel bodies

Generally **safe** on **workers** (still profile and test):

- **`BlueprintThreadSafe`** plugin APIs: **atomics**, **output buffer** methods marked as such, **cancel** checks.
- **Plain math** on **locals** and **primitives** (float ops, vectors) if **no** hidden UObject access.
- **Reading** snapshot **Element** / **Index** provided by the node.

**Risky without review**

- **String** / **Text** operations that allocate or touch UObject internals.
- **Struct** methods that touch UObject.

---

## 5. Recommended "gather → apply" pattern

1. **Parallel phase:** only **thread-safe** math and writes to **atomics** / **output buffers** / plain buffers.
2. **Completed** (game thread): **`ToArrayCopy`**, spawn actors, update UI, play sounds, write to **game state**.

This keeps **workers** thin and **GT** doing **world** mutation.

---

## 6. Relationship to `BlueprintThreadSafe` in the plugin

Use the **marked** methods on **`UParallelAtomic*`**, **`UParallelOutputBuffer*`**, **`UParallelCancelToken`** from **Loop Body** when workers are enabled. Do **not** assume **other** Blueprint nodes are safe because "I'm using atomics nearby."

---

## 7. Checklist before enabling parallel workers

- [ ] **Loop Body** contains **no** actor spawn/destroy, **no** `SetActorLocation` (except via designs that run on GT), **no** widget edits, **no** random `Print String` if it touches unsafe paths (usually OK but I/O can still be questionable).
- [ ] **Project Settings** policy reviewed; **Force on Game Thread** used while prototyping.
- [ ] **Editor compile** warnings read; if **auto-forced GT**, understand you **lost worker parallelism** for that node.
- [ ] **Cancel** and **Completed** both handled if using tokens.
- [ ] **Order** of **Loop Body** not relied upon for correctness.

---

## 8. If you crash or assert

1. Set **Force Parallel On Game Thread** → reproduce. If crash disappears, suspect **non-thread-safe** body.
2. Strip the body down to **atomic + math** only → re-enable workers gradually.
3. Read **Output Log** for **compiler** warnings from the editor module.

---

## 9. Related pages

- [Editor tools](15-Editor-Tools.md) — compiler extension details, log category.
- [Sync primitives](11-Sync-Primitives.md) — what is **`BlueprintThreadSafe`**.
- [Architecture](04-Architecture.md) — locks, Loop Body invocation, worker dispatch.
- [Execute Parallel Body](12-Execute-Parallel-Body.md) — **Body** without GT lock.
- [Overview](01-Overview.md) — product-level limits.
