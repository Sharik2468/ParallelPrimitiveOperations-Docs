# Parallel Primitive Operations

Blueprint-first parallel and sequential iteration over primitive arrays for UE5, with cancellation, synchronization helpers, and editor validation.

This `README.md` is the main entry point to plugin docs in [`Docs/`](Docs/).

---

## Start here

If you are new to the plugin, read in this order:

1. [Overview & goals](Docs/01-Overview.md)
2. [Installation & requirements](Docs/02-Installation.md)
3. [Quick start](Docs/03-Quick-Start.md)
4. [Thread safety & limits](Docs/14-Thread-Safety.md)

After that, use the full index below as a reference.

---

## Documentation index

| # | Page | What it covers |
|---|------|----------------|
| 1 | [Overview & goals](Docs/01-Overview.md) | Why the plugin exists, what problems it solves, and boundaries |
| 2 | [Installation & requirements](Docs/02-Installation.md) | UE version, target platform, enabling plugin, module dependencies |
| 3 | [Quick start](Docs/03-Quick-Start.md) | First working Blueprint graph with subsystem + loop + completion |
| 4 | [Architecture](Docs/04-Architecture.md) | `UGameInstanceSubsystem`, latent lifecycle, operation model |
| 5 | [Blueprint workflow](Docs/05-Blueprint-Workflow.md) | Recommended graph flow and subsystem usage patterns |
| 6 | [ForEach: typed nodes](Docs/06-ForEach-Loops.md) | Typed primitive array loops and branch semantics |
| 7 | [Parallel For Loop](Docs/07-Parallel-For-Loop.md) | Index-based iteration and when to use it vs ForEach |
| 8 | [Execution parameters](Docs/08-Execution-Parameters.md) | `ChunkSize`, `MaxConcurrency`, force game thread, logging |
| 9 | [Project settings (Developer Settings)](Docs/09-Developer-Settings.md) | Global defaults and `EParallelExecutionPolicy` |
| 10 | [Cancellation: Cancel Token](Docs/10-Cancel-Token.md) | Create token, cancel active loops, cancellation checks |
| 11 | [Sync: Atomics & output buffers](Docs/11-Sync-Primitives.md) | Thread-safe accumulation and per-index output writes |
| 12 | [Execute Parallel Body](Docs/12-Execute-Parallel-Body.md) | One-shot parallel body execution pattern |
| 13 | [Utility nodes](Docs/13-Utility-Nodes.md) | Non-loop helper APIs and navigation to related docs |
| 14 | [Thread safety & limits](Docs/14-Thread-Safety.md) | Unsafe operations, UObject/game-thread constraints |
| 15 | [Editor: compiler & validation](Docs/15-Editor-Tools.md) | Blueprint compile-time loop-body checks and diagnostics |
| 16 | [C++ integration](Docs/16-Cpp-Integration.md) | Module usage from C++, headers, API entry points |
| 17 | [Testing](Docs/17-Testing.md) | Automated checks and test harness organization |
| 18 | [Troubleshooting & FAQ](Docs/18-Troubleshooting-FAQ.md) | Common issues, performance tuning, debugging tips |

Manual scenario: [Parallel Sync manual test](Docs/ParallelSync_ManualTest.md) (buffers, atomics, cancel flow in Blueprint).

---

## Read by goal

- I need a working setup quickly: [02 Installation](Docs/02-Installation.md) -> [03 Quick start](Docs/03-Quick-Start.md).
- I need stable gameplay behavior: [14 Thread safety](Docs/14-Thread-Safety.md) -> [08 Execution parameters](Docs/08-Execution-Parameters.md).
- I need cancellation and sync primitives: [10 Cancel Token](Docs/10-Cancel-Token.md) -> [11 Sync primitives](Docs/11-Sync-Primitives.md).
- I need C++ usage details: [04 Architecture](Docs/04-Architecture.md) -> [16 C++ integration](Docs/16-Cpp-Integration.md).
- I need to debug a bad graph or warning: [15 Editor tools](Docs/15-Editor-Tools.md) -> [18 Troubleshooting & FAQ](Docs/18-Troubleshooting-FAQ.md).

---

## Docs layout

```
ParallelPrimitiveOperations/
├── README.md   # Main docs hub and navigation
└── Docs/       # Topic pages
```

## Notes

- The plugin documentation intentionally separates "how to start" from "safety and architecture." Read both before production use.
- If you extend the API, update this file and add a dedicated page in `Docs/` to keep the index complete.
