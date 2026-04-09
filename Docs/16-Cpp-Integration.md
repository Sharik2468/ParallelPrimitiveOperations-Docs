# C++ integration

This page is for **game or plugin modules** that call **`UParallelPrimitiveSubsystem`**, sync objects, or settings from **native code**. Blueprint-only projects can skip it.

---

## 1. Module dependency

Add the **runtime** module to your **`*.Build.cs`**:

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    // ...
    "ParallelPrimitiveOperations",
});
```

Use **`PublicDependencyModuleNames`** if your **public** headers expose plugin types (e.g. a function taking **`UParallelCancelToken*`**). If all includes stay in **`.cpp`** files and your API does not mention plugin types, **`PrivateDependencyModuleNames`** is enough.

**Do not** add **`ParallelPrimitiveOperationsEditor`** to **game** runtime modules—it is **UncookedOnly** and pulls editor stacks.

The plugin runtime **`ParallelPrimitiveOperations.Build.cs`** already links **Core**, **CoreUObject**, **Engine**, **DeveloperSettings** (and private **Projects**). You need those in your module only if **your** code uses them directly.

---

## 2. Headers (common)

| Need | Include |
|------|---------|
| Subsystem API | `#include "Parallel/ParallelPrimitiveSubsystem.h"` |
| Branch enums | `#include "Parallel/ParallelPrimitiveTypes.h"` |
| Project defaults | `#include "Parallel/ParallelPrimitiveOperationsSettings.h"` |
| Cancel token | `#include "Parallel/ParallelCancelToken.h"` |
| Atomics | `#include "Parallel/Sync/ParallelAtomicInt32.h"` (etc.) |
| Output buffers | `#include "Parallel/Sync/ParallelOutputBufferFloat.h"` (etc.) |

Paths are under the plugin’s **Public** folder; Unreal’s include rules resolve them once the module dependency is set.

---

## 3. Resolving the subsystem

### From `UWorld`

```cpp
if (UWorld* World = /* ... */)
{
    if (UGameInstance* GI = World->GetGameInstance())
    {
        if (UParallelPrimitiveSubsystem* PPO = GI->GetSubsystem<UParallelPrimitiveSubsystem>())
        {
            // use PPO
        }
    }
}
```

### From any `UObject` with world context

```cpp
UParallelPrimitiveSubsystem* PPO = UParallelPrimitiveSubsystem::GetSubsystemFromContext(SomeObject);
```

**`GetSubsystemFromContext`** logs **`KismetExecutionMessage`** on **null** context or missing game instance and returns **nullptr**.

---

## 4. Calling loop APIs from C++

Parallel loop functions are **`UFUNCTION`s** designed for Blueprint latent wiring. From C++ you still pass **`FLatentActionInfo`** if you want the **same** latent completion model:

```cpp
FLatentActionInfo LatentInfo;
LatentInfo.CallbackTarget = this;              // UObject that implements the callback UFunction
LatentInfo.ExecutionFunction = UMyClass::StaticClass()->FindFunctionByName(GET_FUNCTION_NAME_CHECKED(UMyClass, OnParallelLoopLatent));
LatentInfo.Linkage = 0;                        // output exec index
LatentInfo.UUID = FMath::Rand();               // must be unique per concurrent latent action on this target

EParallelForEachExec Branch = EParallelForEachExec::Completed;
int32 Index = 0;

Subsystem->ForLoopParallelSequence(
    LatentInfo,
    Branch,
    Index,
    0, 255,
    /*bForceParallelOnGameThread*/ false,
    /*bLog*/ false,
    /*ChunkSize*/ 16,
    /*MaxConcurrency*/ 4,
    /*CancelToken*/ nullptr);
```

Your callback function must be a **`UFUNCTION`** on the `CallbackTarget` object — it will be called by name when the latent action fires. **Easiest path:** drive loops from **Blueprint**, or follow the automation harness pattern in `ParallelPrimitiveOperationsAutomationTest.cpp` which shows how to set up latent command + `UFUNCTION` callbacks correctly.

Use typed variants like **`ForEachLoopParallelSequence_Int`**, **`ForEachLoopParallelSequence_Float`**, etc., for native ForEach integration.

---

## 5. Factories and sync objects

Prefer subsystem factories so **outer** and **world** are consistent:

```cpp
UParallelAtomicInt32* Counter = Subsystem->MakeAtomicInt32(0);
UParallelOutputBufferFloat* Buf = Subsystem->MakeOutputBufferFloat(1024, 0.f);
UParallelCancelToken* Token = Subsystem->MakeCancelToken();
```

Alternatively **`NewObject<UParallelAtomicInt32>(Outer)`** + **`Initialize`** if you manage lifetime yourself.

---

## 6. Reading / mutating default settings

```cpp
const UParallelPrimitiveOperationsSettings* Defaults = GetDefault<UParallelPrimitiveOperationsSettings>();
// read DefaultExecutionPolicy, DefaultChunkSize, DefaultMaxConcurrency

UParallelPrimitiveOperationsSettings* Mutable = GetMutableDefault<UParallelPrimitiveOperationsSettings>();
// editor / config tools only — know what you are doing with saved config
```

Settings are stored in **`DefaultGame.ini`** via the standard UE developer settings mechanism.

---

## 7. Extending the plugin

- **Prefer** subclassing **`UGameInstanceSubsystem`** in a **separate** project module **or** contributing upstream—avoid editing **engine** source.  
- If you add new **parallel** functions on **`UParallelPrimitiveSubsystem`** and want **editor validation** to cover them, register them in the compiler extension's function set (see `ParallelLoopCompilerExtension.cpp`).  
- Keep **BlueprintThreadSafe** metadata accurate on any new **callable** API intended for **worker** contexts.

---

## 8. Related pages

- [Installation](02-Installation.md) — enabling the plugin, Win64.  
- [Architecture](04-Architecture.md) — latent actions, shared state.  
- [Thread safety](14-Thread-Safety.md) — native callbacks on workers.  
- [Testing](17-Testing.md) — automation harness patterns.
