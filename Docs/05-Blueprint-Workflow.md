# Blueprint workflow

This guide is about **day-to-day Blueprint usage**: where nodes live in the menu, how you reach them through **`ParallelPrimitiveSubsystem`**, what the pins mean, and habits that keep latent graphs reliable. It complements [Quick start](03-Quick-Start.md) (first success path) and [Architecture](04-Architecture.md) (what happens under the hood).

---

## 1. The golden rule: nodes live on the subsystem

Almost every plugin feature is exposed as a **`UFUNCTION` on `UParallelPrimitiveSubsystem`**. In Blueprint that means:

1. You obtain a reference to **`Parallel Primitive Subsystem`** (Game Instance subsystem).
2. You **drag off that blue pin** (or use the compact call node) and search for **Parallel ForEach**, **Make Atomic Int32**, **Execute Parallel Body**, etc.

There is **no** separate global “Parallel ForEach” node that works without the subsystem in the shipped API. If you cannot find a loop node, first confirm you are calling it **on the subsystem output**, not from an empty graph context.

**Exception-style note:** `UParallelCancelToken` exposes **`Cancel`** / **`IsCancelled`** on the **token object** itself (category **`Parallel | Primitives | Sync`**), not only on the subsystem.

---

## 2. Node categories (Palette & context menu)

Unreal builds the Blueprint menu from **`Category=`** metadata. For this plugin you will see a hierarchy like:

| Category path | What you find there |
|-------------|---------------------|
| **`Parallel → Primitives`** | All **latent** loops (**Parallel For Loop** and typed **Parallel ForEach Loop …**), **Execute Parallel Body**, **`Make Cancel Token`**. |
| **`Parallel → Primitives → Sync → Atomic`** | **`Make Atomic Bool`**, **`Make Atomic Int32`**, **`Make Atomic Float`**. |
| **`Parallel → Primitives → Sync → Output Buffer`** | **`Make Output Buffer Int32`**, **Float**, **Vector**, **Transform**. |
| **`Parallel → Primitives → Sync`** | **`Cancel`** / **`Is Cancelled`** when the target is a **Cancel Token** object (see token’s own class). |

**Tips for finding nodes**

- Open the **Palette** (left panel) and expand **Parallel**, or right-click the graph and type **`Parallel`** / **`ForEach Parallel`** / **`Parallel Primitive`**.
- If **context sensitive** filtering hides subsystem methods, drag from the **subsystem reference pin** first, or temporarily disable **Context Sensitive** in the action menu to locate the **subsystem class** under **Add Node → Game Instance Subsystem**.

Typed ForEach nodes share the same **Advanced** pins; only the array/element types change (see [ForEach loops](06-ForEach-Loops.md)).

---

## 3. Getting `ParallelPrimitiveSubsystem`

### Standard pattern: `Get ParallelPrimitiveSubsystem`

1. Right-click your Blueprint graph.
2. Search and add **`Get ParallelPrimitiveSubsystem`** (Game Instance Subsystems).

### Practical graph usage

In normal gameplay Blueprints you do not need to manually configure class selection or a separate context node:

- Right-click the graph and place **`Get ParallelPrimitiveSubsystem`**.
- Use the returned **blue pin** as the call source for plugin functions.

**Requirements in plain language**

- There must be a **running game world** with a **Game Instance** (normal **PIE**, standalone, packaged game). In **editor utility** worlds or some tool contexts, subsystems may not exist — expect **`None`** or warnings.

**Static helper (optional in custom Blueprint/C++)**

If you wrap **`UParallelPrimitiveSubsystem::GetSubsystemFromContext`** in a Blueprint function library, it mirrors the same resolution: world from context → game instance → subsystem. In regular Blueprint graphs, prefer the direct getter node **`Get ParallelPrimitiveSubsystem`** unless you already expose a project-specific helper.

### Caching the subsystem reference

It is fine to store the return value in a **variable** (e.g. on your Actor) after **Begin Play** and reuse it. The subsystem lifetime matches the **game instance**. Do not cache it in assets that outlive the instance without re-resolving.

---

## 4. Calling subsystem functions

### Callable vs “pure” helpers

- **Callable (white exec):** loop nodes, **Execute Parallel Body**, **`Make Cancel Token`**, **`Make Atomic *`**, **`Make Output Buffer *`** — they execute when their exec pin fires (and latent ones keep the latent chain alive).
- **Atomic / buffer “read” nodes** on those objects are often **`BlueprintPure`** / **`BlueprintThreadSafe`** where marked in C++ — no exec pin; use them inside **Loop Body** carefully (see [Thread safety](14-Thread-Safety.md)).

### Chaining from the subsystem pin

**Recommended style**

1. **`Get ParallelPrimitiveSubsystem`** -> store or wire the subsystem reference.
2. From that reference, call **`Parallel ForEach Loop Int`** (or another method).

You can also use a **single** node that combines “get subsystem + call” if you build a macro or function library in your project; the plugin itself ships **subsystem** methods only.

---

## 5. Latent nodes and execution flow

Loop nodes and **Execute Parallel Body** are **latent**: they span multiple frames and the Blueprint graph waits for them to finish before continuing. In the editor you see:

- A **latent exec input** (enter the node from a normal exec chain: **Begin Play**, **Completed** of another latent node, etc.).
- **Branch outputs** driven by enums (see below).

**Rules of thumb**

- **Do not** break the latent contract: after you enter a latent loop, the engine expects the latent manager to own completion. Avoid wiring the same latent **UUID** slot to **two** parallel loops at once (see [Architecture](04-Architecture.md) — duplicate start warning).
- **Completed** / **Cancelled** are the **terminal** branches for loops; put cleanup and “what happens next” there.
- **Loop Body** runs **many times** (or zero times if the array/range is empty); keep it as light as policy allows.

## 6. Execution branches (`ExpandEnumAsExecs`)

### `EParallelForEachExec` — ForEach & Parallel For

Used by all typed **Parallel ForEach Loop …** nodes and **Parallel For Loop**.

| Branch | Meaning |
|--------|---------|
| **Loop Body** | One iteration; **`Index`** (and **Array Element** on typed nodes) apply to this step. |
| **Completed** | All iterations finished without taking the cancel path. |
| **Cancelled** | Cancel token requested stop; use this for user abort, level unload, etc. |

The editor shows these as **separate exec pins** because of **`ExpandEnumAsExecs="Branch"`** in the **`UFUNCTION` meta**.

### `EParallelBodyExec` — Execute Parallel Body

| Branch | Meaning |
|--------|---------|
| **Continue** | Fires **immediately** on the **game thread** when the node starts (intended for “keep going on the main thread” setup). The enum’s display name is a **blank** string in C++ (`UMETA(DisplayName=" ")`), so the pin label may look empty or minimal in UI—watch the pin **tooltip** or order. |
| **Body** | The work meant to run on a **worker** (or forced game thread). |
| **Completed** | **Body** finished; latent action completes. |

Details: [Execute Parallel Body](12-Execute-Parallel-Body.md).

---

## 7. Pins and **Advanced** section

Most loop nodes collapse these under the **Advanced** section on the node (click the expander arrow to see them):

| Pin | Role |
|-----|------|
| **`Force Parallel On Game Thread`** | Forces **game-thread** execution policy for **that** loop (overrides project default policy). Use when debugging or when the body must be GT-only. |
| **`Log`** | Extra **Output Log** lines from the subsystem (iteration / thread info when implemented). Costs some performance if left on in shipping. |
| **`Chunk Size`** | Elements per chunk; **`0`** typically means “use project default” (see [Execution parameters](08-Execution-Parameters.md)). |
| **`Max Concurrency`** | Cap on parallel workers; **`0`** means resolve from defaults / auto (same doc). |
| **`Cancel Token`** | Optional **`Cancel Token`** object; if **`Cancel`** is called, prefer the **Cancelled** branch. |

**Execute Parallel Body** only has **Force On Game Thread** and **Log** in its Advanced section — there are no chunk, concurrency, or cancel pins on that node.

**Factory nodes** (**Make Atomic …**, **Make Output Buffer …**, **Make Cancel Token**) expose their parameters on the main pin list (size, default value, initial value).

---

## 8. Wildcard ForEach vs typed arrays

- **Parallel ForEach Loop Int** (and Bool, Float, …) expect a **concrete** **`TArray<T>`** and expose **Array Element** + **Array Index** (display names on Int).

Use typed nodes for clear signatures and explicit element/index pins.

---

## 9. Object lifetime and variables

- **Subsystem** lives with the **game instance**.
- **`Make Atomic Int32`** (etc.) creates a **`UObject`** owned by the subsystem outer—hold it in a **variable** if you need it across nodes.
- **Cancel Token** should be the **same instance** passed into the loop **and** into whatever calls **Cancel** later.
- **Do not** assume unnamed temporaries survive arbitrary latent delays unless they are stored or the graph keeps references—standard UE Blueprint GC rules apply.

---

## 10. Project Settings touchpoint

Defaults for policy/chunk/concurrency come from **Project Settings → Plugins → Parallel Primitive Operations** (`UParallelPrimitiveOperationsSettings`). Blueprint nodes can **override** behavior per call (especially **Force Parallel On Game Thread** and non-zero chunk/concurrency). See [Developer Settings](09-Developer-Settings.md).

---

## 11. Editor validation (what you will see)

When you compile a Blueprint, the **ParallelPrimitiveOperationsEditor** module may **inspect** the **Loop Body** subgraph and **reorder pins**. If you see compiler messages about unsafe nodes, treat them as **guidance**—see [Editor tools](15-Editor-Tools.md). The graph can still be wrong if you bypass recommendations.

---

## 12. Common wiring mistakes

| Mistake | What goes wrong |
|---------|------------------|
| Searching for **Parallel ForEach** without a **subsystem** pin | Node does not appear or wrong overload; always pull off **Parallel Primitive Subsystem**. |
| **World context** invalid / **None** | Subsystem is **`None`**; loops never start or log errors. |
| Starting a **second** parallel loop from the **same latent slot** before the first finishes | Second call **ignored**; warning in log ([Architecture](04-Architecture.md)). |
| Heavy **actor / UI** work in **Loop Body** with **parallel workers** | Crashes or subtle bugs; use **Force on Game Thread** while learning, then [Thread safety](14-Thread-Safety.md). |
| Expecting **strict index order** in **Loop Body** with **many workers** | Order is **not** guaranteed; design for **per-index** writes or single-thread policy. |
| Forgetting **Completed** / **Cancelled** | Latent chain never continues; pending work or UI never updates. |

---

## 13. Related pages

- [Quick start](03-Quick-Start.md) — minimal Int ForEach walkthrough.
- [ForEach loops](06-ForEach-Loops.md) — typed nodes, branches, element pins.
- [Parallel For Loop](07-Parallel-For-Loop.md) — index range loops.
- [Execution parameters](08-Execution-Parameters.md) — chunk, concurrency, logging.
- [Cancel Token](10-Cancel-Token.md) — wiring cancel from UI or timers.
- [Sync primitives](11-Sync-Primitives.md) — atomics and buffers in the loop body.
- [Thread safety](14-Thread-Safety.md) — what belongs in **Loop Body**.
- [Editor tools](15-Editor-Tools.md) — compiler extension and validation.
