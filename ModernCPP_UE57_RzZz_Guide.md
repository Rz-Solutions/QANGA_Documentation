# Modern C++ in UE 5.7 by RzZz, Implementation Guide

This document provides comprehensive guidance for implementing modern C++ features in Unreal Engine 5.7.3 projects, based on extensive testing and debugging made on the QPolice plugin and updated after the UE 5.3.2 → 5.7.3 migration.

## Engine C++ Standard Status (UE 5.7.3)

- **C++20 is the minimum and default** — C++14 and C++17 are `[Obsolete]` in UBT and will not compile
- **C++23 is NOT production-ready** — Epic marks it experimental; MSVC's `/std:c++23preview` breaks PCH/include chains across UE modules and third-party plugins, causing hundreds of undefined type errors
- Previously broken features (`std::format`, `std::ranges`) now work cross-platform thanks to Clang 20 (required for Linux)

### Why C++23 Doesn't Work Yet

UBT maps `CppStandardVersion.Cpp23` to MSVC's `/std:c++23preview` flag (or `/std:c++latest` on older MSVC). This flag changes how the compiler resolves includes and processes precompiled headers, breaking virtually every translation unit that relies on UE's transitive include chains. In our project this resulted in 170+ files with errors across both first-party and third-party plugins.

**Bottom line:** Stay on C++20 until Epic officially supports C++23. When they do, it'll likely come with a new `BuildSettingsVersion` that adjusts the include order accordingly.

### Build Configuration

C++ standard is set at the **Target level** — no per-module Build.cs changes needed:

```cs
// Qanga.Target.cs (same pattern for QangaEditor.Target.cs and QangaServer.Target.cs)
public class QangaTarget : TargetRules
{
    public QangaTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Game;
        DefaultBuildSettings = BuildSettingsVersion.V6;
        IncludeOrderVersion = EngineIncludeOrderVersion.Latest;
        CppStandard = CppStandardVersion.Cpp20;  // Default and minimum for UE 5.7
    }
}
```

Available `CppStandardVersion` values in UE 5.7:

| Value | Description |
|-------|-------------|
| `Cpp20` | C++20 (default, minimum) |
| `Cpp23` | C++23 (experimental — DO NOT USE, breaks PCH/includes) |
| `Latest` | Latest supported by compiler (same issue as Cpp23) |
| `Default` | Resolves to `Cpp20` |

## C++20 Features

### Concepts

Concepts provide compile-time interface validation with zero runtime overhead and full UE5 compatibility.

```cpp
// Interface validation with compile-time checking
template<typename T>
concept GameActor = requires(T* actor, const T* constActor) {
    { actor->GetActorType() } -> std::same_as<EActorType>;
    { actor->SetActorType(EActorType{}) } -> std::same_as<void>;
    { constActor->IsActive() } -> std::same_as<bool>;
    { actor->Initialize() } -> std::same_as<void>;
    { actor->Cleanup() } -> std::same_as<void>;
    { constActor->IsValid() } -> std::same_as<bool>;
};

// Usage with automatic interface validation
template<GameActor T>
void ConfigureActor(T* Actor) {
    if (Actor->IsValid()) {
        Actor->Initialize();
    }
}
```

Benefits:
- Compile-time interface validation
- Superior error messages compared to SFINAE
- Zero runtime performance impact
- Full compatibility with UE5 reflection system

### Coroutines

Coroutines enable asynchronous operations with clean syntax, but require careful lifetime management in UE5.

#### Core Implementation Pattern

```cpp
// Coroutine task implementation
namespace ProjectCoroutines {
    template<typename T = void>
    struct Task {
        struct promise_type {
            Task get_return_object() {
                return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
            }
            std::suspend_never initial_suspend() { return {}; }
            std::suspend_never final_suspend() noexcept { return {}; }
            void unhandled_exception() { std::terminate(); }

            // For void return types
            void return_void() requires std::is_void_v<T> {}

            // For non-void return types
            void return_value(const T& value) requires (!std::is_void_v<T>) {
                result = value;
            }

            T result{};
        };

        std::coroutine_handle<promise_type> coro;

        explicit Task(std::coroutine_handle<promise_type> h) : coro(h) {}
        ~Task() { if (coro) coro.destroy(); }

        // Move-only semantics
        Task(const Task&) = delete;
        Task& operator=(const Task&) = delete;
        Task(Task&& other) noexcept : coro(std::exchange(other.coro, {})) {}
        Task& operator=(Task&& other) noexcept {
            if (this != &other) {
                if (coro) coro.destroy();
                coro = std::exchange(other.coro, {});
            }
            return *this;
        }

        bool done() const { return coro && coro.done(); }
        T get_result() requires (!std::is_void_v<T>) { return coro.promise().result; }
    };

    // Timer awaitable for UE5 integration
    struct TimerAwaitable {
        float DelaySeconds;

        bool await_ready() { return DelaySeconds <= 0.0f; }
        void await_suspend(std::coroutine_handle<> handle) {
            // For simple delays during development, resume immediately
            // Production implementations should integrate with UE5 timer system
            handle.resume();
        }
        void await_resume() {}
    };

    TimerAwaitable DelayFor(float Seconds) { return {Seconds}; }
}
```

#### Usage Example

```cpp
// Coroutine implementation
ProjectCoroutines::Task<bool> UAsyncHelper::ProcessDataAsync() {
    UWorld* World = GetWorld();
    if (!World) {
        UE_LOG(LogTemp, Error, TEXT("Invalid world context"));
        co_return false;
    }

    // Load asset asynchronously
    UClass* AssetClass = co_await LoadClassAsync(AssetPath);
    if (!AssetClass) {
        UE_LOG(LogTemp, Error, TEXT("Failed to load asset class"));
        co_return false;
    }

    // Small delay for processing
    co_await DelayFor(0.1f);

    // Process the loaded data
    if (UGameSubsystem* Subsystem = UGameSubsystem::Get(World)) {
        Subsystem->ProcessUpdate();
        co_return true;
    }

    co_return false;
}
```

#### Critical Lifetime Management

The most common coroutine issue is premature Task destruction causing apparent hanging.

```cpp
// INCORRECT - Task destroyed immediately, killing coroutine
void SomeMethod() {
    auto UpdateTask = Helper->ProcessDataAsync(); // Destroyed at end of scope
    // Coroutine dies here, appears to hang
}

// CORRECT - Store task to prevent destruction
class UGameSubsystem : public UWorldSubsystem {
    // Store the task to keep coroutine alive
    TOptional<ProjectCoroutines::Task<bool>> ActiveProcessingTask;

public:
    void StartProcessing() {
        // Store task to prevent immediate destruction
        ActiveProcessingTask = Helper->ProcessDataAsync();
    }
};
```

### std::span

std::span provides memory-safe array views.

```cpp
class UComponentManager {
    void ProcessComponents(std::span<FComponentData> components) {
        // Range algorithms work cross-platform now (Clang 20)
        auto count = std::ranges::count_if(components, [](const auto& comp) {
            return comp.IsActive();
        });

        for (const auto& component : components) {
            if (component.IsActive()) {
                component.Update();
            }
        }
    }
};
```

### std::format

Previously broken on Linux (libc++ < 17). Now works cross-platform with Clang 20.

```cpp
#include <format>

// Works on both Windows and Linux
auto message = std::format("Entity {} has {} HP at ({:.1f}, {:.1f})",
                           EntityName, Health, Position.X, Position.Y);

// UE5 FString alternative — still preferred for TCHAR/wide string interop
FString Message = FString::Printf(TEXT("Entity %s has %d HP"), *EntityName, Health);
```

**Recommendation:** Use `std::format` for internal/debug std::string work. Use `FString::Printf` for anything that interfaces with UE5 APIs (logging, UI, networking).

### std::ranges

Previously broken on Linux. UE 5.7 itself uses `std::ranges` in engine code (e.g. `AndroidJavaEnv.h`).

```cpp
#include <ranges>
#include <algorithm>

// Filter + transform pipelines
auto activePositions = entities
    | std::views::filter([](const auto& e) { return e.IsAlive(); })
    | std::views::transform([](const auto& e) { return e.GetLocation(); });

// Range algorithms — cleaner than iterator pairs
std::ranges::sort(entities, {}, &FEntity::Priority);
auto it = std::ranges::find_if(entities, [](const auto& e) { return e.IsTarget(); });
auto count = std::ranges::count_if(entities, &FEntity::IsActive);
```

## C++23 Features (Reference Only — Not Usable Yet)

These features are documented for future reference. They require `CppStandard = CppStandardVersion.Cpp23` which currently breaks UE 5.7.3 builds (see "Why C++23 Doesn't Work Yet" above).

### std::expected — Error Handling Without Exceptions

Perfect for UE5 since exceptions are disabled. Replaces out-parameters and custom Result types.

```cpp
#include <expected>

std::expected<UClass*, FString> LoadAssetSafe(const FSoftClassPath& Path) {
    UClass* Loaded = Path.TryLoadClass<AActor>();
    if (!Loaded) {
        return std::unexpected(FString::Printf(TEXT("Failed to load: %s"), *Path.ToString()));
    }
    return Loaded;
}

// Usage — clean error propagation
auto Result = LoadAssetSafe(AssetPath);
if (Result) {
    SpawnActor(*Result);
} else {
    UE_LOG(LogGame, Error, TEXT("%s"), *Result.error());
}

// Monadic chaining
auto FinalResult = LoadAssetSafe(Path)
    .and_then([](UClass* Class) -> std::expected<AActor*, FString> {
        AActor* Actor = GetWorld()->SpawnActor(Class);
        if (!Actor) return std::unexpected(TEXT("Spawn failed"));
        return Actor;
    })
    .transform([](AActor* Actor) {
        Actor->Initialize();
        return Actor;
    });
```

### Deducing this

Eliminates const/non-const overload duplication with explicit object parameters.

```cpp
struct FEntityData {
    TArray<FVector> Positions;

    // Single implementation handles both const and non-const
    auto&& GetPositions(this auto&& self) {
        return std::forward_like<decltype(self)>(self.Positions);
    }
};

// CRTP without the template boilerplate
struct FBaseComponent {
    void Initialize(this auto&& self) {
        self.OnInitialize();  // Calls derived class method
    }
};
```

### std::to_underlying

Clean enum-to-integer conversion.

```cpp
#include <utility>

UENUM()
enum class EWeaponType : uint8 { Melee, Ranged, Magic };

int32 TypeIndex = std::to_underlying(EWeaponType::Ranged);  // 1
```

### std::unreachable

Optimizer hint for impossible code paths.

```cpp
#include <utility>

switch (ActorType) {
    case EActorType::Player: return HandlePlayer();
    case EActorType::NPC: return HandleNPC();
    case EActorType::Vehicle: return HandleVehicle();
    default: std::unreachable();  // UB if reached — optimizer can eliminate this branch
}
```

### Multidimensional operator[]

```cpp
struct FHeightMap {
    TArray<float> Data;
    int32 Width;

    float& operator[](int32 x, int32 y) {
        return Data[y * Width + x];
    }

    const float& operator[](int32 x, int32 y) const {
        return Data[y * Width + x];
    }
};

// Usage
FHeightMap Map;
float Height = Map[10, 25];
```

### if consteval

```cpp
constexpr float ComputeDistance(FVector A, FVector B) {
    if consteval {
        // Compile-time: use simple calculation
        float dx = A.X - B.X, dy = A.Y - B.Y, dz = A.Z - B.Z;
        return /* compile-time sqrt */;
    } else {
        // Runtime: use SIMD-optimized path
        return FVector::Dist(A, B);
    }
}
```

## Known Limitations

### Designated Initializers with USTRUCT

UE5's UHT (Unreal Header Tool) is incompatible with designated initializers in USTRUCT declarations. This is a UHT limitation, not a compiler one.

```cpp
// COMPILATION FAILURE — USTRUCT incompatible
USTRUCT(BlueprintType)
struct FGameConfig {
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxPlayers = 10;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float SessionTimeout = 5.0f;
};

// This will not compile with UPROPERTY
FGameConfig Config = {
    .MaxPlayers = 20,        // ERROR: USTRUCT doesn't support designated initializers
    .SessionTimeout = 3.0f
};

// WORKAROUND — Factory functions
static FGameConfig CreateCustomConfig() {
    FGameConfig Config;
    Config.MaxPlayers = 20;
    Config.SessionTimeout = 3.0f;
    return Config;
}
```

### Exception Handling

UE5 disables exceptions by default. Use UE5-compatible error handling — `std::expected` (C++23) is the cleanest alternative.

```cpp
// COMPILATION FAILURE — UE5 disables exceptions
ProjectCoroutines::Task<bool> BadExample() {
    try {
        co_await SomeOperation();
        co_return true;
    } catch (const std::exception& e) {  // ERROR: Exceptions disabled
        co_return false;
    }
}

// CORRECT — Macro-based early returns for coroutines
#define PROJECT_CORO_CHECK(condition, returnValue, message) \
    if (!(condition)) { \
        UE_LOG(LogTemp, Error, TEXT("Coroutine failed: %s"), message); \
        co_return returnValue; \
    }

ProjectCoroutines::Task<bool> GoodExample() {
    UWorld* World = GetWorld();
    PROJECT_CORO_CHECK(World, false, TEXT("Invalid world"));

    UClass* AssetClass = co_await LoadClassAsync();
    PROJECT_CORO_CHECK(AssetClass, false, TEXT("Failed to load asset"));

    co_return true;
}

// BEST (C++23) — Use std::expected for non-coroutine code
std::expected<bool, FString> BestExample() {
    UWorld* World = GetWorld();
    if (!World) return std::unexpected(TEXT("Invalid world"));
    return true;
}
```

### Template Argument Deduction

Explicit template arguments are required for coroutine return types.

```cpp
// COMPILATION ERROR — Missing template argument
ProjectCoroutines::Task ProcessAsync() {  // ERROR: Task<> needs argument
    co_return;
}

// CORRECT — Explicitly specify void for no return value
ProjectCoroutines::Task<void> ProcessAsync() {
    co_await DelayFor(0.1f);
    co_return;
}
```

## Cross-Platform Compatibility (UE 5.7.3)

**Toolchain:** MSVC 2022 (Windows) | Clang 20 (Linux, required)

| Feature | Windows MSVC | Linux Clang 20 | Status |
|---------|-------------|----------------|--------|
| Concepts | Full | Full | Safe |
| Coroutines | Full | Full | Safe |
| `std::span` | Full | Full | Safe |
| `std::format` | Full | Full | Fixed in 5.7 |
| `std::ranges` | Full | Full | Fixed in 5.7 |
| `std::expected` | Full | Full | C++23 |
| Deducing `this` | Full | Full | C++23 |
| `std::to_underlying` | Full | Full | C++23 |
| `std::unreachable` | Full | Full | C++23 |
| `std::print` | Partial | Full | C++23, MSVC may lag |
| Multidim `operator[]` | Full | Full | C++23 |

## Common Pitfalls and Solutions

### Coroutine Hanging

**Problem:** Coroutines appear to hang at co_await calls
**Root Cause:** Task<> objects destroyed before coroutine completes
**Solution:** Store Task objects in member variables

```cpp
// CAUSES HANGING
void StartAsyncOperation() {
    auto task = SomeCoroutine();  // Destroyed at end of scope
}

// PREVENTS HANGING
TOptional<ProjectCoroutines::Task<bool>> ActiveTask;

void StartAsyncOperation() {
    ActiveTask = SomeCoroutine();  // Task stays alive
}
```

### Timer Callback Crashes

**Problem:** EXCEPTION_ACCESS_VIOLATION in timer callbacks
**Root Cause:** Objects destroyed while timers still reference them
**Solution:** Use TWeakObjectPtr and validate before access

```cpp
// CRASH PRONE
GetWorld()->GetTimerManager().SetTimer(TimerHandle, [this]() {
    this->ProcessUpdate();  // 'this' might be destroyed
}, 1.0f, true);

// CRASH SAFE
TWeakObjectPtr<UGameSubsystem> WeakThis = this;
GetWorld()->GetTimerManager().SetTimer(TimerHandle, [WeakThis]() {
    if (WeakThis.IsValid()) {
        WeakThis->ProcessUpdate();
    }
}, 1.0f, true);
```

### ProcessEvent Safety

**Problem:** ProcessEvent crashes with invalid function pointers
**Root Cause:** Function not found or object destroyed
**Solution:** Always validate functions and prefer C++ APIs

```cpp
// CRASH PRONE
UFunction* Func = Component->FindFunction("SomeFunction");
Component->ProcessEvent(Func, &Params);  // Func might be nullptr

// SAFE
UFunction* Func = Component->FindFunction("SomeFunction");
if (Func && IsValid(Component)) {
    Component->ProcessEvent(Func, &Params);
}

// PREFERRED — Direct C++ call
if (auto* GameComponent = Component->GetGameComponent()) {
    GameComponent->ExecuteAction();
}
```

### UObject Integration

Proper UObject-based implementation for coroutine helpers.

```cpp
UCLASS()
class UAsyncOperationHelper : public UObject {
    TWeakObjectPtr<UWorld> CachedWorld;

    static UAsyncOperationHelper* Get(const UObject* WorldContext) {
        UWorld* World = WorldContext->GetWorld();
        if (!World) return nullptr;

        UAsyncOperationHelper* Helper = NewObject<UAsyncOperationHelper>();
        Helper->Initialize(World);
        return Helper;
    }
};
```

## Implementation Checklist

### Before Starting
- [ ] Verify `CppStandard = CppStandardVersion.Cpp20` in all Target.cs files (C++23 not yet usable)
- [ ] Test cross-platform compilation (Windows + Linux)
- [ ] Plan coroutine lifetime management strategy

### During Implementation
- [ ] Use concepts for interface validation
- [ ] Use `std::expected` for error handling (replaces out-params and custom Result types)
- [ ] Use `std::ranges` and `std::format` freely (cross-platform since UE 5.7)
- [ ] Store Task<> objects to prevent hanging
- [ ] Use TWeakObjectPtr for timer callbacks
- [ ] Validate UFunction pointers before ProcessEvent
- [ ] Prefer C++ APIs over Blueprint ProcessEvent

### After Implementation
- [ ] Test coroutine execution from start to finish
- [ ] Verify no hanging at co_await points
- [ ] Check for memory leaks in Task destruction
- [ ] Validate cross-platform builds

## Best Practices

1. **Error handling:** Use `std::expected` over out-params, macros, or bool returns
2. **Concepts first:** Easiest C++20 integration, best error messages
3. **Coroutines last:** Most complex, requires careful lifetime management
4. **Store tasks:** Always store coroutine Task objects in member variables
5. **Weak references:** Use TWeakObjectPtr for any timer or callback references
6. **Validate everything:** Check function pointers, object validity, world context
7. **`std::format` vs `FString::Printf`:** Use std::format for std::string, FString::Printf for UE5 APIs
8. **Ranges over algorithms:** Prefer `std::ranges::sort(v, {}, &T::Member)` over `std::sort` with comparators