# Editor: compiler & validation

The **`ParallelPrimitiveOperationsEditor`** module runs **only in the editor**. It registers a **Blueprint compiler extension** that scans your graphs for **parallel loop** nodes and can **warn**, **message-log**, and **auto-toggle** **Force on Game Thread** when it thinks the **Loop Body** is unsafe for background threads.

This page describes **what** it does, **how** to read its output, and **what it does not** guarantee.

---

## 1. Module role

| Aspect | Detail |
|--------|--------|
| **Module** | `ParallelPrimitiveOperationsEditor` |
| **Loads** | **UncookedOnly** — editor and commandlet contexts, **not** packaged games. |
| **Depends on** | `ParallelPrimitiveOperations` (runtime), `BlueprintGraph`, `KismetCompiler`, `Kismet`, `UnrealEd`, etc. |

**Startup**

1. Creates a **singleton** compiler extension object and protects it from garbage collection.
2. Registers it with Unreal's Blueprint compilation manager so it runs for **every Blueprint** compile in the editor.

**Shutdown**

- UE 5.6 does not expose an unregister API for compiler extensions; the extension remains active for the editor session — normal for editor lifecycle.

Your **game** `.Build.cs` should **not** depend on **`ParallelPrimitiveOperationsEditor`** unless you are writing **editor-only** tools.

---

## 2. `UParallelLoopCompilerExtension`

**Hook:** runs once per Blueprint compile

It walks **all** graph pages on the **`UBlueprint`**:

- Ubergraph pages  
- Function graphs  
- Macro graphs  
- Delegate signature graphs  
- Intermediate generated graphs  

For each parallel loop function call node belonging to **`UParallelPrimitiveSubsystem`** found in those graphs, it runs:

1. **Safety validation** — walks the exec subgraph from **Loop Body** and flags unsafe function calls.
2. Pin reordering is **wired in code but disabled** in the current version and has no effect on compile.

---

## 3. Which functions are “parallel loop” nodes?

The extension checks that the function belongs to **`UParallelPrimitiveSubsystem`** and is in the recognized set:

**Included**

- **`ForEachLoopParallelSequence_*`** — Int, Bool, Byte, Float, Name, String, Text, Vector, Rotator, Transform  
- **`ForLoopParallelSequence`**  
- **`ExecuteParallelBody`**

**Not included**

- **`MakeAtomic*`**, **`MakeOutputBuffer*`**, **`MakeCancelToken`** — no loop-body validation on those.

---

## 4. Finding the “body” exec pin 
The extension looks for an **output exec** pin named (case variations supported):

- **`LoopBody`**, **`Body`**, or **`then`** (case-insensitive).

If none match but **other** output exec pins exist, it **logs a warning** listing available exec pin names and **uses the first** output exec pin—**fallback** that may be wrong if the node layout is unusual.

If there are **no** output exec pins, validation is **skipped** with a warning.

---

## 5. Safety validation (the safety validator)

Starting from the Loop Body output exec pin, the validator:

**Algorithm (simplified)**

1. Follow **exec links** outward from that pin.  
2. If the **owning node** is one of **our** parallel nodes and the pin is **Completed** / **Cancelled** (or related), **do not** traverse into those branches (so completion wiring is not treated as “body”).  
3. For each reached node, if it is **function call node**:  
   - If the called function lacks **`BlueprintThreadSafe`** metadata → marked **unsafe**.  
4. Continue traversing **all output exec pins** of visited nodes (entire **exec subtree** of the body).

**Caveats**

- **Pure** / **data-only** nodes are **not** analyzed by this pass—only **exec** chains.  
- Only function call nodes are flagged; other node types (flow control, casts, etc.) are traversed but not labeled unsafe by this rule.

---

## 6. What happens when unsafe calls are found?

When the validator runs on a loop node:

1. Walks the **exec subgraph** starting from the resolved **Loop Body** pin.  
2. If **no** unsafe nodes → **verbose** log: thread-safe.  
3. If **unsafe** nodes exist:  
   - Tries **the force-game-thread pin search** — (**Force Parallel On Game Thread** for loops, **Force On Game Thread** for Execute Parallel Body).  
   - If the pin is **unlinked** and default is **not** already **`true`** → automatically sets it to **`true`** and logs a warning.  
   - If the pin is **linked** → **warning**: unsafe body but cannot auto-enable.  
   - If already **true** → warning that it already forces GT.  
   - If **no** force pin found → warning that auto-enable is impossible.  
4. **A warning in Compiler Results** with a token pointing at the **call node**:  
   - Either *“forcing Force Parallel On Game Thread forced to true”* or *“set Force Parallel On Game Thread forced to true.”*

**Result for you**

- The Blueprint **may compile** but with **warnings**.  
- The **node default** may **silently change** on compile so the next run uses **game thread**—**parallel speedup is lost** until you fix the body or unlink and set the pin explicitly.

---

## 7. Reading messages in the editor

- **Compiler Results** / **Message Log** during compile — look for **`Parallel primitive`** warnings tied to the subsystem **call node**.  
- **Output Log** category **`LogParallelLoopCompilerExt`**:  
  - **Verbose:** body considered safe.  
  - **Warning:** missing pins, fallback exec pin, forced GT, linked force pin, etc.

Filter the log with **`Parallel loop:`** or **`ParallelLoopCompilerExt`** when debugging.

---

## 8. “Bypassing” validation

There is **no** project toggle to disable the extension without **code** or **disabling the editor module**.

**Practical options**

- **Fix** the body to use **`BlueprintThreadSafe`** calls only, or move work to **Completed** on the game thread.  
- **Manually** set **Force Parallel On Game Thread** / fix the pin link so the compiler does not fight you.  
- **Accept** auto-forced GT (safe but slower).

**Risks of ignoring warnings**

- **Crashes** or **data corruption** if you **force** workers (e.g. link **`false`** into the force pin) while the body calls non-thread-safe APIs.

---

## 9. Pin reordering (dormant)

The editor module contains code for deterministic pin reordering on parallel nodes, but it is **not active** in the current version. You may see references to it in source, but it has no effect at compile time.

---

## 10. Related pages

- [Thread safety](14-Thread-Safety.md) — rules the validator approximates.  
- [Blueprint workflow](05-Blueprint-Workflow.md) — node categories.  
- [Architecture](04-Architecture.md) — runtime behavior when GT is forced vs workers.
