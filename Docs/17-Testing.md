# Testing

The plugin ships **automation tests** under **`WITH_DEV_AUTOMATION_TESTS`**. They validate **sync primitives** under **`ParallelFor`** and **runtime node coverage** (latent loops, policies, cancel, typed ForEach) using a small **`UParallelPrimitiveAutomationHarness`**.

---

## 1. Where the code lives

| File | Role |
|------|------|
| `Source/ParallelPrimitiveOperations/Private/Tests/ParallelPrimitiveOperationsAutomationTest.cpp` | **`IMPLEMENT_SIMPLE_AUTOMATION_TEST`** definitions and latent command implementation. |
| `Source/ParallelPrimitiveOperations/Private/Tests/ParallelPrimitiveAutomationHarness.h` | Harness **`UObject`** — refs for **`Branch`**, **`Index`**, typed elements, counters, thread IDs. |
| `Source/ParallelPrimitiveOperations/Private/Tests/ParallelPrimitiveAutomationHarness.cpp` | **`HandleForEachExec`**, **`HandleParallelBodyExec`**, state reset. |

Tests compile **only** when **`WITH_DEV_AUTOMATION_TESTS`** is **1** (typical **Editor** / **Development** configurations).

---

## 2. Test: `SyncPrimitives`

**Full name:** **`Project.Plugins.ParallelPrimitiveOperations.SyncPrimitives`**  
**Flags:** **`EditorContext | ClientContext | EngineFilter`**

**What it does (summary)**

- **`UParallelAtomicBool` / `Int32` / `Float`**: **`ParallelFor`** over **4096** iterations — **`Add`**, **`CompareExchange`**, **`Exchange`**, **`Min`/`Max`** (float), etc.  
- **`UParallelOutputBuffer*`** (Int32, Float, Vector, Transform): parallel **`TrySetOnce`**, **`Set`**, **`ToArrayCopy`**, contention on a **single** index for **`TrySetOnce`**.

**World:** **No** game instance required — pure **`NewObject`** + engine **`ParallelFor`**.

**How to run**

1. **Session Frontend** (or **Tools → Session Frontend** depending on UE layout) → **Automation** tab.  
2. Enable **`Project.Plugins.ParallelPrimitiveOperations`** filter or search **`SyncPrimitives`**.  
3. **Start tests**.

Or **command line** (project-specific; example pattern):

```text
UnrealEditor-Cmd.exe YourProject.uproject -ExecCmds="Automation RunTests Project.Plugins.ParallelPrimitiveOperations.SyncPrimitives; Quit" -unattended
```

---

## 3. Test: `RuntimeNodes`

**Full name:** **`Project.Plugins.ParallelPrimitiveOperations.RuntimeNodes`**  
**Flags:** **`EditorContext | ClientContext | EngineFilter`**

**What it does**

Uses **`ADD_LATENT_AUTOMATION_COMMAND(FPPOParallelNodeCoverageLatentCommand)`** to spread work across ticks:

1. **Resolve** a **playable** **`UWorld`** + **`UGameInstance`** (`TryGetPlayableWorldAndGameInstance`) — prefers **Game / PIE / GamePreview**.  
2. **Get** **`UParallelPrimitiveSubsystem`**, create **`UParallelPrimitiveAutomationHarness`**.  
3. **Snapshot** and temporarily **mutate** **`UParallelPrimitiveOperationsSettings`** (policy, chunk, concurrency), **restore** in **`RestoreSettings()`** at the end.  
4. **Steps include:**  
   - Factory smoke (**MakeCancelToken**, buffers, atomics).  
   - **`ExecuteParallelBody`** worker vs game thread (thread ID checks).  
   - **`ForLoopParallelSequence`** under **GameThread**, **WorkerSingleThread**, **ParallelWorkers** policies (body thread expectations, index coverage).  
   - **Cancel** path: large range + **`CancelToken`** + cancel after **N** bodies → expect **Cancelled**, not **Completed**.  
   - **Typed ForEach** for **Int, Bool, Byte, Float, Name, String, Text, Vector, Rotator, Transform** — **ParallelWorkers**, assert body counts and **no** game-thread execution for loop body.

**Timeout:** each wait step uses **30 seconds** before failing with a timeout error.

**Important:** If **no** suitable world/game instance exists, the test **warns** and **skips** remaining steps:

> *"RuntimeNodes test skipped: no playable world with GameInstance is available. Start PIE (or run in a game context) and rerun."*

**Recommendation:** **Start PIE** (or run automation in a context that provides **Game + GameInstance**) before running **`RuntimeNodes`**.

---

## 4. Harness purpose

**`UParallelPrimitiveAutomationHarness`** is **`UObject`** with:

- Blueprint-callable callback functions that receive results from loop and body completions.  
- **Atomics** for counts (body, completed, cancelled, done flags).  
- A **set** of seen indices and thread IDs (protected by a lock) for coverage checks.  
- **Cancel-after-N** support for cancel tests.

It is **not** a public gameplay API—**for tests and as a reference** for how to wire **`FLatentActionInfo`** from C++.

---

## 5. CI / local runs

- **CI:** Run **`UnrealEditor-Cmd`** with **`-RunAutomationTests`** / **`-ExecCmds`** as your build farm documents; ensure **PIE** or a **headless game** context if you need **`RuntimeNodes`** to do more than skip.  
- **Local:** Editor **Automation** window is usually fastest for iteration.  
- **Shipping builds** typically strip **`WITH_DEV_AUTOMATION_TESTS`** — these tests are **dev/QA** tools.

---

## 6. Related pages

- [Troubleshooting & FAQ](18-Troubleshooting-FAQ.md)  
- [Architecture](04-Architecture.md)  
- [Installation](02-Installation.md)
