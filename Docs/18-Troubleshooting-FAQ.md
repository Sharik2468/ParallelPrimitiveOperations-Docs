# Troubleshooting & FAQ

Symptoms-first notes for **Parallel Primitive Operations**, plus short **FAQ** entries. Cross-links point to deeper Wiki pages.

---

## 1. Loop never finishes / `Completed` never fires

**Possible causes**

- **Latent chain broken** — another latent node or **missing** wire prevents the manager from resuming. Ensure **Loop Body** eventually **returns** into the latent node’s expected flow (same as stock UE latent loops).  
- **World / subsystem invalid** — **`GetWorld()`** null on subsystem so the latent action was never added; check **Output Log** for **`ParallelPrimitiveSubsystem`** errors.  
- **Stuck body** — infinite loop or **blocking** call inside **Loop Body** (especially on **game thread** policy) prevents the loop from ever finishing.  
- **Duplicate latent UUID** — second loop start ignored; first loop may still be “running” from the engine’s point of view.

**What to try**

- Enable **`Log`** on the node for iteration traces.  
- Use a **tiny** array/range to reproduce.  
- Read [Architecture](04-Architecture.md) for a deeper explanation of how loops complete.

---

## 2. Crashes or hangs with a parallel body

**Likely:** **non-thread-safe** Blueprint or native code running on **thread-pool** workers.

**Quick isolation**

1. Enable **Force Parallel On Game Thread** (or set **Project Settings** policy to **Game Thread**). If the crash **stops**, suspect **threading**.  
2. Strip **Loop Body** down to **atomics / math** only; add nodes back until it fails.  
3. Read compiler **warnings** from [Editor tools](15-Editor-Tools.md)—auto-forced GT may have been **overridden** by a **linked** **false** on the force pin.

**See** [Thread safety](14-Thread-Safety.md).

---

## 3. Cancel does not seem to work

- **Cooperative cancel** — there can be delay until the next **chunk** / **latent** check. See [Cancel Token](10-Cancel-Token.md).  
- **Wrong token instance** — **`Cancel`** must run on the **same** **`UParallelCancelToken`** wired to the loop.  
- **Already on `Completed`** — if the loop finished, **`Cancelled`** will not fire.  
- **GC** — token collected while loop runs → behavior undefined; keep a reference.

---

## 4. Wrong results from an output buffer

- **Index out of range** — **`Set`/`TrySetOnce`** no-op on invalid index.  
- **`TrySetOnce`** rejected — slot already written; second write returns **false**.  
- **Reading during writes** — **`ToArrayCopy`** mid-loop can race **logically** with your expectations; prefer **Completed** for a stable copy.  
- **Wrong buffer size** — **`Init`** smaller than iteration count.

**See** [Sync primitives](11-Sync-Primitives.md).

---

## 5. Editor validation flags “safe” nodes or misses unsafe ones

- **Exec-only analysis** — [Editor tools](15-Editor-Tools.md): only **function call nodes** reachable via the **exec** path from **Loop Body** are checked. **Pure** (non-exec) chains can still be unsafe.  
- **False negatives** — native code, dynamic calls, or unusual macros may **bypass** static checks.  
- **False positives** — rare; if a function **is** thread-safe but **lacks** **`BlueprintThreadSafe`** metadata, the validator treats it as unsafe (engine/plugin would need a metadata fix).

---

## 6. Performance worse than expected

- **Only one chunk** with high **Max Concurrency** — only **one** worker gets work; reduce **chunk size** or increase the iteration count. See [Execution parameters](08-Execution-Parameters.md).  
- **Force on Game Thread** or **auto-forced** after compile — **no** multi-core parallel bodies.  
- **`Log` enabled** — per-iteration logging.  
- **Huge Blueprint bodies** — Blueprint callbacks are serialized internally; very CPU-heavy Blueprint logic may **not scale** across cores the way native C++ does.

---

## 7. Plugin missing on an unsupported platform

The **`.uplugin`** allowlists **Win64**. Other platforms may **not build** the module. See [Installation](02-Installation.md).

---

## 8. FAQ

### Can I use UObject / actors inside `Loop Body`?

**Only safely** if the body **actually runs on the game thread** (**Force Parallel On Game Thread** or **Game Thread** policy). With **Parallel Workers**, assume **most** actor/component APIs are **unsafe**. Gather results in parallel, apply on **Completed**. — [Thread safety](14-Thread-Safety.md)

### Is iteration order deterministic?

**Not** when **multiple workers** process chunks. Use **per-index** buffers/atomics or **single-thread** policies for strict ordering. — [Overview](01-Overview.md), [ForEach](06-ForEach-Loops.md)

### Does the plugin handle multiplayer / replication?

**No special** networking layer. Loop execution, replicated attributes, and any RPC/gameplay state updates still follow normal UE networking rules. Prefer **server authority** for replicated state. — [Thread safety](14-Thread-Safety.md), [Architecture](04-Architecture.md)

### Why does my array not update during a typed ForEach?

**Snapshot** at loop start — mutations to the **source** array during the loop do not change **Element** for that run. — [ForEach](06-ForEach-Loops.md)

### Two parallel loops at once from the same event?

**Same latent UUID** → second start **ignored** with a warning. Chain **Completed** → second loop. — [Architecture](04-Architecture.md)

---

## 9. Reporting bugs

- Prefer your project’s **issue tracker**.  
- Include: **UE version**, **Win64** confirmation, **minimal** Blueprint or C++ repro, **Output Log** (especially any warnings from the plugin), and whether enabling **Force Parallel On Game Thread** changes behavior.

---

## 10. Related pages

- [README](../README.md) — full Wiki index  
- [Testing](17-Testing.md) — automation names and PIE requirement  
- [Blueprint workflow](05-Blueprint-Workflow.md)
