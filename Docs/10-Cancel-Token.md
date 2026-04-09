# Cancellation: Cancel Token

Parallel loops are **long-running latent** operations. **Cancel tokens** let you ask them to **stop cooperatively** from anywhere that can call **`Cancel`** on the same token object, then handle the **`Cancelled`** exec branch instead of **`Completed`**.

---

## 1. Creating a token

### Blueprint

1. Get **`Parallel Primitive Subsystem`**.
2. Call **`Make Cancel Token`** (category **`Parallel | Primitives`**).

**Return type:** **`UParallelCancelToken`** (Blueprint: **Cancel Token** or similar).

### Lifetime

- The token is a **`UObject`** created with an **outer** tied to the subsystem (same pattern as **`Make Atomic Int32`**). Store it in a **variable** if you need to call **`Cancel`** later (e.g. from UI, timer, or another actor).
- Keep a **strong reference** until the loop has finished or cancelled, or the token may be **garbage-collected** and cancel behavior becomes undefined for loops still holding a stale pointer.

**Lazy internal state**

- The underlying cancellation flag is **allocated on first use** — either when you call **`Cancel()`** or when the loop starts and registers the token. Until then, **`IsCancelled()`** returns **false**.

---

## 2. Passing the token into a loop

On typed **ForEach** and **Parallel For Loop**, expand **Advanced** and wire the **`Cancel Token`** pin.

- **`None` / not connected:** cancel is not observed from a token; completion goes to **`Completed`** when work finishes.
- **Valid reference:** the loop internally copies the shared cancel flag so workers and the latent tick can check it safely from any thread without accessing the UObject directly.

---

## 3. API on `UParallelCancelToken`

| Function | Meta | Behavior |
|----------|------|----------|
| **`Cancel`** | `BlueprintCallable`, **`BlueprintThreadSafe`** | Sets the cancel flag. Safe to call from the **game thread** or **worker** threads. |
| **`IsCancelled`** | `BlueprintPure`, **`BlueprintThreadSafe`** | **`true`** if the shared flag exists **and** loads **`true`**. |

**Category:** **`Parallel | Primitives | Sync`** (on the **token** object, not only on the subsystem).

You can poll **`IsCancelled`** inside **Loop Body** if you want **early exit** from a **long** per-iteration routine (still cooperative—see §6).

---

## 4. How the loop observes cancel

Each game-thread tick, the loop checks whether the cancel flag is set:

- If cancel is signaled → the latent action fires the **`Cancelled`** branch (exactly once).
- **`Completed`** is used when all iterations finish **without** taking the cancel path.

Cancellation is processed **between iterations and chunks** — not as an instant hard-kill of whatever Blueprint code is currently running.

---

## 5. `Cancelled` vs `Completed`

| Branch | When |
|--------|------|
| **Completed** | Normal finish: **`TotalCount`** iterations accounted for, cancel flag not driving terminal. |
| **Cancelled** | Cancel flag was set; terminal state was signaled as **cancelled**. |

Wire **both** in production graphs: **Completed** → success path; **Cancelled** → cleanup, revert UI, log “aborted”, etc.

---

## 6. Why and when to use Cancel Token

Use cancel when the player or game flow can invalidate the current job before it naturally finishes.

Typical reasons:

- Player closes a menu / clicks **Stop**.
- You start a new job and old results are no longer needed.
- Level transition begins and old world state is about to disappear.
- Job runs too long and should be aborted by timeout.

If none of these are possible, cancel token is optional.

---

## 7. Full step-by-step scenarios

### Scenario A: Player presses "Stop" on a long calculation

**Goal:** user can abort an active loop from UI.

1. **Start path (e.g. BeginPlay or button Start):**
   - Get `ParallelPrimitiveSubsystem`.
   - `ActiveToken = Make Cancel Token`.
   - Start `Parallel For Loop` or typed `ForEach`.
   - Wire `ActiveToken` into loop's **Cancel Token** pin.
2. **Loop Body:**
   - Do normal thread-safe work (atomics/buffers/math).
   - Optional early guard: `if ActiveToken.IsCancelled() -> return`.
3. **UI Stop button (`OnClicked`):**
   - If `ActiveToken` is valid -> call `ActiveToken.Cancel()`.
4. **Cancelled branch:**
   - Show status "Cancelled by user".
   - Clear loading indicators.
   - Keep or discard partial data by your game design.
5. **Completed branch:**
   - Normal success flow (apply final data, continue pipeline).

Why this is useful: player keeps control, and stale heavy work does not waste CPU.

### Scenario B: Timeout guard (auto-cancel)

**Goal:** prevent a job from running forever.

1. Create token and start loop exactly as in Scenario A.
2. Right after loop start, set **Timer by Event** for `N` seconds.
3. Timer callback:
   - If loop still active and token valid -> `Cancel()`.
4. In `Cancelled`, show "Timed out" or fallback behavior.

Practical tip: store `bLoopActive` bool; set it `true` on start and `false` in both `Completed` and `Cancelled`, so timer knows whether cancel is still needed.

### Scenario C: Level/owner teardown safety

**Goal:** avoid loop callbacks after owner context is being destroyed.

1. Store token when loop starts.
2. In `EndPlay` / teardown path:
   - If token valid and loop might still run -> `Cancel()`.
3. In `Cancelled`, run lightweight cleanup only (no risky object access).

Why this is useful: prevents "old job finishes in wrong lifecycle moment" issues.

### Scenario D: Shared cancel for a batch of loops

**Goal:** stop multiple related loops with one action.

1. Create one token `BatchToken`.
2. Pass the same token into several loops started for one workflow.
3. One `BatchToken.Cancel()` call cancels all loops that use it.

Use this when loops represent one logical job. Use separate tokens when loops are independent.

---

## 8. Limitations (read this)

| Topic | Detail |
|-------|--------|
| **Cooperative** | **`Cancel`** does **not** interrupt the CPU **inside** a random Blueprint opcode. Workers stop when they **check** cancel / stop flags between iterations or chunks. |
| **Latency** | There can be a **short delay** between **`Cancel()`** and **`Cancelled`** firing, depending on frame rate and how many iterations remain. |
| **Per-token** | One token instance = one shared flag. Two loops with **different** tokens are independent. Two loops with the **same** token will both see cancel when you call **`Cancel`** once. |
| **Not pause** | Cancel means **stop** the loop’s terminal path toward **Cancelled**, not “pause and resume.” |
| **Duplicate latent** | Starting a **second** parallel loop with the **same latent UUID** while the first runs is already **unsupported** ([Architecture](04-Architecture.md)); cancel does not fix that pattern. |

---

## 9. Related pages

- [ForEach loops](06-ForEach-Loops.md) · [Parallel For Loop](07-Parallel-For-Loop.md)
- [Execution parameters](08-Execution-Parameters.md) — **Cancel Token** pin placement
- [Architecture](04-Architecture.md) — how cancellation is checked and propagated at runtime
- [Blueprint workflow](05-Blueprint-Workflow.md) — categories and latent wiring
