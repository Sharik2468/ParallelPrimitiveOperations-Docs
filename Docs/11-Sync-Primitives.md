# Sync: Atomics & output buffers

When **Loop Body** runs from **multiple threads**, ordinary Blueprint variables and **`TArray`** writes can **race**. This plugin provides **atomics** (counters, flags, floating aggregates) and **output buffers** (one slot per index) designed to be used from **parallel** bodies where marked **`BlueprintThreadSafe`**.

Still read **[Thread safety](14-Thread-Safety.md)**—these types help with **data races**, not with **calling arbitrary UObject / actor APIs** from workers.

---

## 1. Why you need sync

| Problem | Without sync | With plugin types |
|---------|--------------|-------------------|
| **`int32` counter += 1** from many iterations | Lost updates, UB | **`UParallelAtomicInt32`** **`Add`** |
| **“First writer wins” per index** | Races, torn writes | **`TrySetOnce`** on output buffer |
| **Running min / max / sum of floats** in parallel | Wrong math | **`Atomic Float`** **`Min` / `Max` / `Add`** (with float caveats) |

---

## 2. Factories (on `ParallelPrimitiveSubsystem`)

All are **`BlueprintCallable`**, category under **`Parallel | Primitives | Sync`**.

| Factory | Creates |
|---------|---------|
| **`Make Atomic Bool`** | **`UParallelAtomicBool`** |
| **`Make Atomic Int32`** | **`UParallelAtomicInt32`** |
| **`Make Atomic Float`** | **`UParallelAtomicFloat`** |
| **`Make Output Buffer Int32`** | **`UParallelOutputBufferInt32`** |
| **`Make Output Buffer Float`** | **`UParallelOutputBufferFloat`** |
| **`Make Output Buffer Vector`** | **`UParallelOutputBufferVector`** |
| **`Make Output Buffer Transform`** | **`UParallelOutputBufferTransform`** |

**Parameters**

- **Atomics:** initial value at creation (subsystem also calls **`Initialize`** internally after `NewObject`).
- **Buffers:** **`Size`** and **default value** for every slot; **`Init`** is applied inside the factory.

**Lifetime:** Objects are **`NewObject`** with outer **`this`** (subsystem). Hold a **reference** in a Blueprint variable for the duration of the loop and any **Completed** logic.

---

## 3. Atomics — API summary

All atomic **`Get` / `Set` / `Exchange` / `CompareExchange`** nodes marked **`BlueprintThreadSafe`** are intended for use from **Loop Body** even when the body runs on **worker** threads.

### `UParallelAtomicBool`

- **`Get`**, **`Set`**, **`Exchange`**, **`CompareExchange`**.

### `UParallelAtomicInt32`

- **`Get`**, **`Set`**, **`Add(Delta)`** — returns the **value after the add** (previous value + Delta).
- **`Exchange`**, **`CompareExchange`**.

### `UParallelAtomicFloat`

Stored as **an internal atomic bit representation** of **IEEE float bits** (not a hardware “atomic float” opcode).

- **`Get`**, **`Set`**, **`Exchange`**, **`CompareExchange`**.
- **`Add(Delta)`** — **atomic loop**: read bits → compute **new float** (handles **NaN** per helper rules) → updates atomically until success. Returns the **new** value after a successful update **by this call’s logical op** (under contention, many threads interleave; each **`Add`** still contributes atomically).
- **`Min(Value)`**, **`Max(Value)`** — same atomic loop approach: keeps the lower / higher of the current value vs the argument.

**Float caveats**

- **Non-associative:** parallel **`Add`** order affects the **exact** float bits vs a serial sum—expected for floats.
- **Use `Atomic Int32`** for exact counts; use **`Atomic Float`** for **approximate** sums or **min/max** tracking when acceptable.

---

## 4. Output buffers — contract

Each buffer type wraps:

- **`TArray<T> Data`** — one slot per index
- **Written flags** — tracks which slots have been written (0 = unwritten, non-zero = written)
- **Internal read/write lock** — protects concurrent access

### `Init(Size, DefaultValue)`

- **`BlueprintCallable`**, **not** thread-safe — call **before** starting the parallel loop (on the game thread). It resizes the internal data and written-flags arrays. Size is clamped to 0 or more.

### Query (`BlueprintThreadSafe`)

- **`Num()`** — total number of slots (thread-safe).
- **`IsValidIndex(Index)`** — bounds check (thread-safe).
- **`WasWritten(Index)`** — **true** if the slot was written at least once (thread-safe).

### Writes (`BlueprintThreadSafe`)

- **`TrySetOnce(Index, Value)`** — writes **only if the slot has never been written**. If index invalid → **`false`**. If slot **already written** → **`false`**. Otherwise writes value, sets flag, returns **`true`**. Use for **idempotent** “one iteration owns this slot.”
- **`Set(Index, Value)`** — thread-safe. Overwrites value and sets written flag; **no** “first only” check. Invalid index → no-op.

### `ToArrayCopy()`

- **`BlueprintCallable`**, **not** marked thread-safe — but internally safe to call (uses a read lock).
- Returns a **copy** of the buffer data.
- **Best practice:** call from **`Completed`** (game thread) after workers are done, or when you are sure **no concurrent writers** are running. Avoid hammering **`ToArrayCopy`** every iteration from **many** threads—it serializes on the lock and allocates.

---

## 5. Recommended workflow (always the same shape)

Use this sequence for reliable graphs:

1. **Before starting the loop (game thread)**  
   Create all shared sync objects once:
   - `MakeAtomic...` for global aggregates (count/sum/min/max/flags).
   - `MakeOutputBuffer...` for per-index outputs.
   Store them in Blueprint variables.
2. **Inside Loop Body**  
   Only do thread-safe operations:
   - Read current `Index` / typed `Element`.
   - Update atomics (`Add`, `Min`, `Max`, `CompareExchange`, ...).
   - Write per-index result into buffer (`TrySetOnce` or `Set`).
3. **Inside Completed**  
   Move from “parallel data” to gameplay actions:
   - Read final atomic values with `Get`.
   - Convert buffers via `ToArrayCopy`.
   - Apply results to actors/UI/game state on the game thread.

If you keep this separation, most race-condition bugs disappear.

---

## 6. Concrete usage cases (step-by-step)

### Case A: Count valid elements (Atomic Int32)

**Goal:** count how many elements pass a condition.

**Setup (before loop):**
- `ValidCount = Make Atomic Int32(0)`

**Loop Body:**
- If `Element` is valid for your rule -> `ValidCount.Add(1)`

**Completed:**
- `FinalCount = ValidCount.Get()`
- Use `FinalCount` for UI/gameplay flow.

Use this instead of `int32` variable increments inside Loop Body.

### Case B: Parallel sum/min/max for telemetry (Atomic Float)

**Goal:** collect aggregate stats from many float samples.

**Setup (before loop):**
- `Sum = Make Atomic Float(0.0)`
- `MinValue = Make Atomic Float(FLT_MAX-like large value)`
- `MaxValue = Make Atomic Float(-FLT_MAX-like low value)`

**Loop Body:**
- `Sum.Add(Element)`
- `MinValue.Min(Element)`
- `MaxValue.Max(Element)`

**Completed:**
- Read all three values with `Get()`
- Optional: compare `Sum` to serial baseline with tolerance.

Use this for metrics and analysis. For deterministic exact totals, prefer integer counters when possible.

### Case C: One result per source index (Output Buffer + TrySetOnce)

**Goal:** write exactly one processed result per input index.

**Setup (before loop):**
- `ResultBuffer = Make Output Buffer Float(InputLength, 0.0)`

**Loop Body:**
- `Processed = Compute(Element)`
- `bStored = ResultBuffer.TrySetOnce(Index, Processed)`
- Optionally increment an atomic error counter if `bStored == false` unexpectedly.

**Completed:**
- `Results = ResultBuffer.ToArrayCopy()`
- Continue with `Results` on game thread.

Choose this when each iteration owns a unique index and double writes should be considered a bug.

### Case D: Last-writer-wins updates (Output Buffer + Set)

**Goal:** allow overwriting by design (not first-writer semantics).

**Setup (before loop):**
- `Buffer = Make Output Buffer Vector(Num, FVector::ZeroVector)`

**Loop Body:**
- `Buffer.Set(Index, NewValue)`

**Completed:**
- `OutVectors = Buffer.ToArrayCopy()`

Use this only when overwrites are intentional. If not intentional, prefer `TrySetOnce`.

### Case E: Cancellation-aware processing (Atomic + Buffer + Token)

**Goal:** stop expensive work safely when user aborts.

**Setup (before loop):**
- `Token = Make Cancel Token()`
- `ProcessedCount = Make Atomic Int32(0)`
- `Buffer = Make Output Buffer Transform(Num, Identity)`
- Pass `Token` to loop `Cancel Token` pin.

**Loop Body:**
- Optional early check: `if Token.IsCancelled() -> return`
- Write `Buffer.TrySetOnce(Index, Value)`
- `ProcessedCount.Add(1)`

**Cancelled branch:**
- Read `ProcessedCount.Get()` for partial-progress diagnostics.

**Completed branch:**
- Read `Buffer.ToArrayCopy()` and continue normally.

---

## 7. Anti-patterns

| Anti-pattern | Why |
|--------------|-----|
| **`ToArrayCopy`** inside **every** **Loop Body** | Contention on **one** lock + allocation storm. |
| **`Init`** while workers are writing | Data race / undefined state—**init first**, then start loop. |
| **`Set`** from **multiple** iterations on the **same** index without design | Last writer wins—may hide bugs; prefer **`TrySetOnce`** if exactly one writer per index is intended. |
| **Huge buffers** you never shrink | Memory cost = **`sizeof(T) * Num`**; size buffers to workload. |
| **Atomics as a license to touch actors** | Atomics fix **value** races, not **UObject** thread rules. |

---

## 8. Category reference (Blueprint menu)

- **`Parallel | Primitives | Sync | Atomic`** — methods on atomic objects.
- **`Parallel | Primitives | Sync | OutputBuffer`** — methods on buffer objects.

---

## 9. Related pages

- [Thread safety](14-Thread-Safety.md)
- [Execution parameters](08-Execution-Parameters.md)
- [ForEach loops](06-ForEach-Loops.md) · [Architecture](04-Architecture.md)
