# Parallel For Loop

This page describes the **`Parallel For Loop`** node: inclusive index range, how **`Index`** is exposed to **Loop Body**, edge cases, and when to use it instead of [ForEach](06-ForEach-Loops.md). Scheduling (**chunk size**, **concurrency**, **game thread**) is shared with ForEach — see [Execution parameters](08-Execution-Parameters.md).

---

## 1. Purpose

**Parallel For Loop** iterates an integer **Index** over a **closed range**:

```text
FirstIndex, FirstIndex+1, …, LastIndex   (inclusive on both ends)
```

It does **not** require a **`TArray`**. You only supply **two integers** and the same **latent** branch model as ForEach: **Loop Body**, **Completed**, **Cancelled**.

**C++:** `ForLoopParallelSequence`  
**Blueprint display name:** **Parallel For Loop**  
**Category:** `Parallel | Primitives`

---

## 2. Node signature (conceptual)

**Core inputs**

| Pin | Type | Meaning |
|-----|------|---------|
| **First Index** | `int32` | Start of range (included). |
| **Last Index** | `int32` | End of range (included). |
| **Index** | `int32` (output per iteration) | Current logical index for this **Loop Body** invocation. |

**Exec / latent**

- Standard **latent** entry exec pin.
- **Branch** outputs: **Loop Body**, **Completed**, **Cancelled** (`EParallelForEachExec`).

**Advanced** (same as ForEach)

- **Force Parallel On Game Thread**
- **Log**
- **Chunk Size**
- **Max Concurrency**
- **Cancel Token**

---

## 3. How `Index` maps to the implementation

Internally the scheduler uses **`TotalCount`** consecutive “slots” **`0 … TotalCount−1`**.

- **`TotalCount = LastIndex - FirstIndex + 1`** when **`LastIndex >= FirstIndex`**.
- For each slot **`k`**, the **Blueprint `Index` pin** is set to **`FirstIndex + k`**.

So you always see the **same numbers** you would get from a classic **`for (i = First; i <= Last; ++i)`** loop.

There is **no separate “Array Element”** pin — only **Index**. Use **Index** to address your own arrays, buffers, or mathematical formulas.

---

## 4. Edge cases

### `LastIndex < FirstIndex`

**TotalCount** is treated as **0** (no iterations).

- **Loop Body** does **not** run.
- **Completed** still fires (same as empty ForEach).
- **Index** is irrelevant because the body never executes.

This is **not** an error; it is a convenient way to represent “invalid / empty range” if your game logic can produce inverted bounds.

### `FirstIndex == LastIndex`

Exactly **one** iteration: **`Index == FirstIndex`**.

### Large ranges

**TotalCount** can be large (millions). Memory use is dominated by **scheduling**, not by storing the range. Still mind **Blueprint work per iteration** and [thread safety](14-Thread-Safety.md).

### Negative indices

Allowed if they make sense for your math. The plugin does not clamp **First** / **Last** to non-negative values.

---

## 5. When to use Parallel For instead of ForEach

| Situation | Prefer |
|-----------|--------|
| You already have **`TArray<T>`** | [ForEach typed](06-ForEach-Loops.md) — you get **Element** + **Index**. |
| You only need **`i` from A to B`** | **Parallel For Loop** — no array copy/snapshot of elements. |
| Grid / 1D strip over coordinates | **Parallel For** with a formula, or nested loops in your design (each loop is a separate node invocation). |
| Processing external data addressed by index | **Parallel For** + **output buffer** / atomics keyed by **`Index`**. |

**Snapshot note:** ForEach **typed** snapshots the **array**. Parallel For **does not** copy an array—only the **range** is used. If you read external arrays by **Index** yourself, **you** are responsible for not mutating them unsafely on the game thread while workers read unless you use thread-safe structures or force **game thread** execution.

---

## 6. Chunks and `Index` order

Chunking splits the **internal** range **`0 … TotalCount−1`** into blocks. Within one chunk, indices are processed **in order**. Across **multiple workers**, **different chunks** run **concurrently**, so **Loop Body** may observe **Index** values in **non-monotonic** order over time—same warning as ForEach.

If you need **strict ascending Index order** on the Blueprint side, use a **single-thread** policy or **Force Parallel On Game Thread** and accept loss of parallel speedup.

---

## 7. Cancel and completion

Same as ForEach:

- **`Cancel Token`** → prefer **Cancelled** when stop is observed.
- Duplicate latent start on the same **callback target + UUID** → second **Parallel For** is **skipped** with a warning.

---

## 8. Quick Blueprint recipe

1. **Right-click in the Blueprint graph** -> search **`Get ParallelPrimitiveSubsystem`** (Game Instance Subsystems).
2. **Parallel For Loop**: **First Index** `0`, **Last Index** `N-1` for `N` steps.
3. **Loop Body**: use **Index** (e.g. **`Add`** to atomic, or **`Try Set Once`** on buffer).
4. **Completed**: next gameplay step.

For a concrete **atomic** pattern, use the same setup shown in [Sync primitives](11-Sync-Primitives.md): `Make Atomic Int32(0)` + `Add(...)` in `Loop Body`.

---

## 9. Full step-by-step example (human-friendly)

### Scenario

You have `5000` points represented by indices `0..4999`.  
For each index, you compute a score and want to:

- store the score at the same index in a result array,
- count how many scores are above a threshold,
- continue only when all work is done.

### What to prepare in Blueprint

Before running the loop, create variables:

- `Subsystem` (`ParallelPrimitiveSubsystem` reference)
- `HighScoreCount` (`ParallelAtomicInt32` object reference)
- `ScoreBuffer` (`ParallelOutputBufferFloat` object reference)
- `NumPoints` (`int32`, for example `5000`)

### Step-by-step wiring

1. **Get subsystem**  
   On `BeginPlay` (or your start event), right-click the graph and add `Get ParallelPrimitiveSubsystem`.
2. **Create sync objects once**  
   - `HighScoreCount = Make Atomic Int32(0)`  
   - `ScoreBuffer = Make Output Buffer Float(NumPoints, 0.0)`
3. **Run loop range**  
   Call `Parallel For Loop` with:
   - `First Index = 0`
   - `Last Index = NumPoints - 1`
   - keep defaults for first run (`Chunk Size`, `Max Concurrency`, `Log`)
4. **Loop Body logic**  
   For current `Index`:
   - compute `Score` (pure math / thread-safe logic),
   - `ScoreBuffer.Set(Index, Score)`,
   - if `Score > Threshold` -> `HighScoreCount.Add(1)`.
5. **Completed logic**  
   - `AllScores = ScoreBuffer.ToArrayCopy()`
   - `TotalHigh = HighScoreCount.Get()`
   - update UI / trigger next gameplay step.
6. **Cancelled logic (optional but recommended)**  
   If you use a cancel token, handle `Cancelled` separately:
   - show "operation aborted",
   - do partial cleanup.

### Why this pattern is safe and practical

- `Parallel For Loop` gives you stable numeric iteration without array snapshots.
- `OutputBuffer` is the safest way to keep one result per index.
- `AtomicInt32` avoids race conditions for shared counters.
- `Completed` is the right place to touch UI/actors and consume final data.

### First debug run (recommended)

If this is your first setup, set `Force Parallel On Game Thread = true` once.

- If logic works there but breaks in parallel mode, the issue is usually thread safety in `Loop Body`.
- Then move back to worker mode and keep only thread-safe work inside the body.

---

## 10. Related pages

- [Execution parameters](08-Execution-Parameters.md)
- [ForEach loops](06-ForEach-Loops.md)
- [Cancel Token](10-Cancel-Token.md)
- [Architecture](04-Architecture.md)
- [Blueprint workflow](05-Blueprint-Workflow.md)
