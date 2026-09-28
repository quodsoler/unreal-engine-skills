# Automation, CQTest and Functional Test Patterns

Target engine: **UE 5.8**. Companion to `SKILL.md`. Every API here is from `Runtime/Core/Public/Misc/AutomationTest.h`, `Developer/CQTest/Public` or `Developer/FunctionalTesting/Classes`.

---

## Where test code lives

Two layouts work:

1. **In the module under test** — put the `.cpp` in `Source/MyGame/Private/Tests/` and guard the whole file with `#if WITH_DEV_AUTOMATION_TESTS` (or `WITH_AUTOMATION_TESTS`, i.e. `WITH_DEV_AUTOMATION_TESTS || WITH_PERF_AUTOMATION_TESTS`, both 0 in Test and Shipping). The guard is needed: when `WITH_AUTOMATION_WORKER` is 0 (`!UE_BUILD_SHIPPING`, `Misc/Build.h:127`) `IMPLEMENT_*_AUTOMATION_TEST` still declares the class and compiles `RunTest`, it only skips the static registration (`Misc/AutomationTest.h:4296-4375`).
2. **In a dedicated module** — a separate `MyGameTests` module keeps test helpers out of shipping code.

```csharp
// Source/MyGameTests/MyGameTests.Build.cs
using UnrealBuildTool;

public class MyGameTests : ModuleRules
{
    public MyGameTests(ReadOnlyTargetRules Target) : base(Target)
    {
        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "Core",
            "CoreUObject",
            "Engine",
            "MyGame",            // the module under test
        });

        // CQTest fixtures and the functional-test actor live in developer modules.
        if (Target.bBuildDeveloperTools)
        {
            PrivateDependencyModuleNames.AddRange(new string[] { "CQTest", "FunctionalTesting" });
        }

        if (Target.bBuildEditor)
        {
            PrivateDependencyModuleNames.Add("UnrealEd");
        }
    }
}
```

```csharp
// MyGameEditor.Target.cs — tests run in the editor process
ExtraModuleNames.Add("MyGameTests");

// MyGame.Target.cs — keep them out of the shipped game
if (Configuration != UnrealTargetConfiguration.Shipping)
{
    ExtraModuleNames.Add("MyGameTests");
}
```

`bBuildDeveloperTools` defaults to true for Editor and Program targets and for any non-Test, non-Shipping configuration. `bForceDisableAutomationTests = true;` in a target rules file drops `WITH_DEV_AUTOMATION_TESTS` and `WITH_PERF_AUTOMATION_TESTS` to 0 regardless.

### Naming

The `PrettyName` argument is a dot-separated path that becomes the tree in the Session Frontend:

```
MyGame.Inventory.AddItem
MyGame.Inventory.Overflow
MyGame.Pathfinding.SimpleGrid
MyGame.Assets.PrimaryWeaponLoad
```

---

## Pattern: pure logic test, no world

```cpp
#include "Misc/AutomationTest.h"
#include "MyMathLibrary.h"

IMPLEMENT_SIMPLE_AUTOMATION_TEST(
    FMyMathLerpTest,
    "MyGame.Math.LerpClamped",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::SmokeFilter)

bool FMyMathLerpTest::RunTest(const FString& Parameters)
{
    TestEqual(TEXT("Lerp at 0"),    UMyMathLibrary::LerpClamped(0.f, 10.f, 0.f),  0.f,  0.001f);
    TestEqual(TEXT("Lerp at 1"),    UMyMathLibrary::LerpClamped(0.f, 10.f, 1.f),  10.f, 0.001f);
    TestEqual(TEXT("Lerp at 0.5"),  UMyMathLibrary::LerpClamped(0.f, 10.f, 0.5f), 5.f,  0.001f);
    TestEqual(TEXT("Alpha above 1"), UMyMathLibrary::LerpClamped(0.f, 10.f, 2.f),  10.f, 0.001f);
    TestEqual(TEXT("Alpha below 0"), UMyMathLibrary::LerpClamped(0.f, 10.f, -1.f), 0.f,  0.001f);
    return true;
}
```

`SmokeFilter` marks the test as fast enough to run on every check-in.

---

## Pattern: UObject test

`NewObject<>` needs an outer. `GetTransientPackage()` keeps the object out of any world and out of any package that could be saved.

```cpp
#include "Misc/AutomationTest.h"
#include "UObject/Package.h"
#include "MyInventoryComponent.h"

IMPLEMENT_SIMPLE_AUTOMATION_TEST(
    FMyInventoryAddRemoveTest,
    "MyGame.Inventory.AddRemove",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)

bool FMyInventoryAddRemoveTest::RunTest(const FString& Parameters)
{
    UMyInventoryComponent* Inv = NewObject<UMyInventoryComponent>(GetTransientPackage());
    UE_RETURN_ON_ERROR(Inv != nullptr, TEXT("Inventory created"));

    Inv->AddItem(FName("Sword"), 3);
    TestEqual(TEXT("Count after add"), Inv->GetItemCount(FName("Sword")), 3);

    Inv->RemoveItem(FName("Sword"), 1);
    TestEqual(TEXT("Count after remove"), Inv->GetItemCount(FName("Sword")), 2);

    Inv->RemoveItem(FName("Sword"), 99);
    TestEqual(TEXT("Count clamped at 0"), Inv->GetItemCount(FName("Sword")), 0);

    TWeakObjectPtr<UMyInventoryComponent> WeakInv(Inv);
    TestValid(TEXT("Weak ref still valid"), WeakInv);
    return true;
}
```

---

## Pattern: complex (parameterized) test over assets

`IMPLEMENT_COMPLEX_AUTOMATION_TEST` adds `GetTests`; each entry of `OutTestCommands` is handed back to `RunTest` as `Parameters`, and the matching `OutBeautifiedNames` entry is what the Session Frontend shows.

```cpp
#include "Misc/AutomationTest.h"
#include "AssetRegistry/AssetRegistryModule.h"
#include "MyItemData.h"

IMPLEMENT_COMPLEX_AUTOMATION_TEST(
    FMyDataAssetValidationTest,
    "MyGame.Assets.DataAssetValidation",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)

void FMyDataAssetValidationTest::GetTests(TArray<FString>& OutBeautifiedNames,
                                          TArray<FString>& OutTestCommands) const
{
    FAssetRegistryModule& AssetRegistryModule =
        FModuleManager::LoadModuleChecked<FAssetRegistryModule>(TEXT("AssetRegistry"));

    TArray<FAssetData> Assets;
    AssetRegistryModule.Get().GetAssetsByClass(UMyItemData::StaticClass()->GetClassPathName(), Assets);

    for (const FAssetData& Asset : Assets)
    {
        OutBeautifiedNames.Add(Asset.AssetName.ToString());
        OutTestCommands.Add(Asset.GetObjectPathString());
    }
}

bool FMyDataAssetValidationTest::RunTest(const FString& Parameters)
{
    UMyItemData* ItemData = LoadObject<UMyItemData>(nullptr, *Parameters);
    UE_RETURN_ON_ERROR(ItemData != nullptr,
        FString::Printf(TEXT("Failed to load %s"), *Parameters));

    TestFalse(TEXT("DisplayName is set"), ItemData->DisplayName.IsEmpty());
    TestTrue(TEXT("MaxStack > 0"), ItemData->MaxStack > 0);
    TestNotNull(TEXT("Icon resolves"), ItemData->Icon.LoadSynchronous());
    return true;
}
```

Add `"AssetRegistry"` to the module's `PrivateDependencyModuleNames`.

---

## Pattern: negative test with an expected log line

A test fails if the code under test logs an `Error` or `Warning` unless the message was declared expected first.

```cpp
IMPLEMENT_SIMPLE_AUTOMATION_TEST(
    FMyInventoryOverflowTest,
    "MyGame.Inventory.OverflowRejectsItem",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::NegativeFilter)

bool FMyInventoryOverflowTest::RunTest(const FString& Parameters)
{
    AddExpectedMessage(TEXT("Inventory is full"), ELogVerbosity::Warning,
                       EAutomationExpectedMessageFlags::Contains);

    UMyInventoryComponent* Inv = NewObject<UMyInventoryComponent>(GetTransientPackage());
    Inv->SetCapacity(1);
    Inv->AddItem(FName("Sword"), 1);

    TestFalse(TEXT("Second item rejected"), Inv->AddItem(FName("Shield"), 1));
    return true;
}
```

Signatures (all on `FAutomationTestBase`):

```cpp
void AddExpectedMessage(FString ExpectedPatternString, ELogVerbosity::Type ExpectedVerbosity,
    EAutomationExpectedMessageFlags::MatchType CompareType = EAutomationExpectedMessageFlags::Contains,
    int32 Occurrences = 1, bool IsRegex = true);
void AddExpectedMessagePlain(FString ExpectedString, ELogVerbosity::Type ExpectedVerbosity,
    EAutomationExpectedMessageFlags::MatchType CompareType = EAutomationExpectedMessageFlags::Contains,
    int32 Occurrences = 1);
void AddExpectedError(FString ExpectedPatternString,
    EAutomationExpectedErrorFlags::MatchType CompareType = EAutomationExpectedErrorFlags::Contains,
    int32 Occurrences = 1, bool IsRegex = true);
void AddExpectedErrorPlain(FString ExpectedString,
    EAutomationExpectedErrorFlags::MatchType CompareType = EAutomationExpectedErrorFlags::Contains,
    int32 Occurrences = 1);
```

`EAutomationExpectedErrorFlags` is a namespace alias of `EAutomationExpectedMessageFlags`, whose `MatchType` values are `Contains` and `Exact`.

---

## Pattern: latent command that waits

```cpp
#include "Misc/AutomationTest.h"

DEFINE_LATENT_AUTOMATION_COMMAND_ONE_PARAMETER(
    FMyWaitForFlagCommand, TSharedPtr<bool>, bDone);

bool FMyWaitForFlagCommand::Update()
{
    return bDone.IsValid() && *bDone;
}

DEFINE_LATENT_AUTOMATION_COMMAND_TWO_PARAMETER(
    FMyWaitForFlagOrTimeoutCommand, TSharedPtr<bool>, bDone, float, Timeout);

bool FMyWaitForFlagOrTimeoutCommand::Update()
{
    if (bDone.IsValid() && *bDone)
    {
        return true;
    }
    return GetCurrentRunTime() > Timeout;   // give up; assert the flag afterwards
}
```

`GetCurrentRunTime()` comes from `IAutomationLatentCommand` and returns seconds since the command started. `DEFINE_LATENT_AUTOMATION_COMMAND` (no parameters) through `DEFINE_LATENT_AUTOMATION_COMMAND_FIVE_PARAMETER` exist, plus `DEFINE_ENGINE_LATENT_AUTOMATION_COMMAND` / `DEFINE_EXPORTED_LATENT_AUTOMATION_COMMAND` variants that export the command class with an `_API` macro (`AutomationTest.h:3994`) — they do not take a world.

## Pattern: latent command that asserts

A latent command cannot call `TestEqual` unless it holds the test. Capture the `FAutomationTestBase*`.

```cpp
class FMyValidateAfterAsyncCommand : public IAutomationLatentCommand
{
public:
    FMyValidateAfterAsyncCommand(FAutomationTestBase* InTest,
                                 TSharedPtr<bool> InDone,
                                 TSharedPtr<int32> InResult)
        : Test(InTest), bDone(InDone), Result(InResult)
    {
    }

    virtual bool Update() override
    {
        if (!bDone.IsValid() || !*bDone)
        {
            return false;   // called again next frame
        }
        Test->TestEqual(TEXT("Async result"), *Result, 42);
        return true;
    }

private:
    FAutomationTestBase* Test;
    TSharedPtr<bool> bDone;
    TSharedPtr<int32> Result;
};

bool FMyAsyncCalcTest::RunTest(const FString& Parameters)
{
    TSharedPtr<bool> bDone = MakeShared<bool>(false);
    TSharedPtr<int32> Result = MakeShared<int32>(0);

    UMyCalcSubsystem::RunAsync([bDone, Result](int32 Value)
    {
        *Result = Value;
        *bDone = true;
    });

    ADD_LATENT_AUTOMATION_COMMAND(FMyValidateAfterAsyncCommand(this, bDone, Result));
    return true;
}
```

`ADD_LATENT_AUTOMATION_COMMAND(X)` expands to `FAutomationTestFramework::Get().EnqueueLatentCommand(MakeShareable(new X))`, so pass a temporary, never a pointer you own.

---

## Pattern: spec (BDD)

```cpp
#include "Misc/AutomationTest.h"
#include "UObject/Package.h"
#include "MyInventoryComponent.h"

BEGIN_DEFINE_SPEC(FMyInventorySpec, "MyGame.Inventory.Spec",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)
    UMyInventoryComponent* Inventory = nullptr;
END_DEFINE_SPEC(FMyInventorySpec)

void FMyInventorySpec::Define()
{
    BeforeEach([this]()
    {
        Inventory = NewObject<UMyInventoryComponent>(GetTransientPackage());
    });

    Describe(TEXT("AddItem"), [this]()
    {
        It(TEXT("stores a new item"), [this]()
        {
            Inventory->AddItem(FName("Sword"), 1);
            TestEqual(TEXT("Count"), Inventory->GetItemCount(FName("Sword")), 1);
        });

        It(TEXT("rejects past capacity"), [this]()
        {
            Inventory->SetCapacity(0);
            TestFalse(TEXT("Rejected"), Inventory->AddItem(FName("Sword"), 1));
        });

        LatentIt(TEXT("finishes an async load"), FTimespan::FromSeconds(5.0),
            [this](const FDoneDelegate& Done)
            {
                Inventory->LoadAsync(FSimpleDelegate::CreateLambda([Done]() { Done.Execute(); }));
            });
    });

    AfterEach([this]()
    {
        Inventory = nullptr;
    });
}
```

`FAutomationSpecBase` supplies `Describe`, `It`, `LatentIt`, `BeforeEach`, `LatentBeforeEach`, `AfterEach`, `LatentAfterEach`, and the `x`-prefixed disabled forms `xIt` / `xDescribe`. The overloads accept an optional `EAsyncExecution` and an `FTimespan` timeout. `DEFINE_SPEC(TClass, PrettyName, TFlags)` declares a spec with no member variables; `BEGIN_DEFINE_SPEC` / `END_DEFINE_SPEC` is the form that has them.

---

## CQTest

### Fixture with a world and actors

```cpp
#include "CQTest.h"
#include "Components/ActorTestSpawner.h"
#include "MyEnemy.h"

TEST_CLASS_WITH_FLAGS(FMyEnemyTests, "MyGame.Enemy",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)
{
    FActorTestSpawner Spawner;

    TEST_METHOD(Enemy_OnSpawn_StartsAtFullHealth)
    {
        AMyEnemy& Enemy = Spawner.SpawnActor<AMyEnemy>();
        ASSERT_THAT(IsNear(100.0f, Enemy.GetHealth(), 0.001f)); // AreEqual static_asserts on floats
    }

    TEST_METHOD(Enemy_WhenDamaged_LosesHealth)
    {
        AMyEnemy& Enemy = Spawner.SpawnActor<AMyEnemy>();
        Enemy.ApplyDamage(30.0f);
        ASSERT_THAT(IsNear(70.0f, Enemy.GetHealth(), 0.001f));
    }
};
```

`FSpawnHelper` (the base of `FActorTestSpawner` and `FMapTestSpawner`) offers `SpawnActor<T>(const FActorSpawnParameters&, UClass*)`, `SpawnActorAt<T>(Location, Rotation, …)`, `SpawnObject<T>()` and `UWorld& GetWorld()`. All of them return references, not pointers.

### Building a configured object

```cpp
#include "ObjectBuilder.h"

TEST_METHOD(Enemy_WithOverriddenSpeed_UsesIt)
{
    AMyEnemy& Enemy = TObjectBuilder<AMyEnemy>(Spawner)
        .SetParam(TEXT("MaxSpeed"), 900.0f)
        .SetParam(TEXT("bIsElite"), true)
        .Spawn();

    ASSERT_THAT(IsNear(900.0f, Enemy.GetMaxSpeed(), 0.001f));
}
```

`TObjectBuilder<T>` takes an `FSpawnHelper&` for actors (deferred construction) or a `UObject*` outer for plain `UObject`s. `SetParam(FName, Value)` writes through the reflection system, so the property must be a `UPROPERTY`. `Spawn(FTransform)` finalizes and returns `T&`; calling it twice is an error.

### Latent steps

```cpp
TEST_METHOD(Enemy_AfterDelay_Patrols)
{
    AMyEnemy& Enemy = Spawner.SpawnActor<AMyEnemy>();

    TestCommandBuilder
        .Do([&Enemy]() { Enemy.BeginPatrol(); })
        .WaitDelay(FTimespan::FromSeconds(1.0))
        .Until([&Enemy]() { return Enemy.IsMoving(); })
        .Then([this, &Enemy]() { ASSERT_THAT(IsTrue(Enemy.IsMoving())); });
}
```

`FTestCommandBuilder` also has `StartWhen(Query)`, `DoAsync`/`ThenAsync` (taking a `TAsyncResult<T>`), and every method has an overload whose first argument is a `const TCHAR*` description. Timeouts default to `CQTest::DefaultTimeout`.

### Map-based fixture

```cpp
#include "Components/MapTestSpawner.h"

TEST_CLASS(FMyLevelTests, "MyGame.Levels")
{
    TUniquePtr<FMapTestSpawner> Spawner;

    BEFORE_EACH()
    {
        Spawner = MakeUnique<FMapTestSpawner>(TEXT("/Game/Maps"), TEXT("Test_Arena"));
        Spawner->AddWaitUntilLoadedCommand(TestRunner); // TestRunner is the fixture's static TTestRunner* (CQTest.h:259)
    }

    TEST_METHOD(Arena_HasPlayerStart)
    {
        ASSERT_THAT(IsNotNull(Spawner->FindFirstPlayerPawn()));
    }
};
```

`FMapTestSpawner::CreateFromTempLevel(FTestCommandBuilder&)` builds a throwaway level instead of loading one from disk. `AddWaitUntilLoadedCommand` must be called outside a latent action, i.e. from `BEFORE_EACH()`.

---

## Functional tests

The header form is in `SKILL.md`. The matching `.cpp`:

```cpp
#include "MyFunctionalTest.h"

void AMyFunctionalTest::PrepareTest()
{
    Super::PrepareTest();
    TimeLimit = 10.0f;            // seconds; 0 means no limit
    PreparationTimeLimit = 5.0f;  // budget for IsReady() to return true
    TimesUpResult = EFunctionalTestResult::Failed;
}

bool AMyFunctionalTest::IsReady_Implementation()
{
    return Super::IsReady_Implementation() && IsValid(TargetActor);
}

void AMyFunctionalTest::StartTest()
{
    Super::StartTest();

    StartStep(TEXT("Validate spawn"));
    AssertTrue(IsValid(TargetActor), TEXT("Target actor resolved"), this);
    AssertEqual_Vector(TargetActor->GetActorLocation(), FVector::ZeroVector,
                       TEXT("Target starts at origin"), 1.0f, this);
    FinishStep();

    StartStep(TEXT("Apply damage"));
    AssertValue_Float(TargetActor->GetActorLocation().Z, EComparisonMethod::Greater_Than_Or_Equal_To,
                      0.0f, TEXT("Target above the floor"), this);
    FinishStep();

    FinishTest(EFunctionalTestResult::Succeeded, TEXT("All checks passed"));
}
```

Place the actor in a map such as `Content/Maps/Test_MyFeature.umap`. `AFunctionalTest` also exposes `bIsEnabled`, `Author`, `Description`, `TimesUpMessage`, `LogErrorHandling` / `LogWarningHandling` (an `EFunctionalTestLogHandling`), `WantsToRunAgain()` and the `TestFinishedObserver` delegate.

`UFunctionalTestingManager::RunAllFunctionalTests(UObject* WorldContextObject, bool bNewLog = true, bool bRunLooped = false, FString FailedTestsReproString = TEXT(""))` is a `UBlueprintFunctionLibrary` static meant for Blueprint: `UFunctionalTestingManager` is `MinimalAPI` and the function carries no `FUNCTIONALTESTING_API` (`FunctionalTestingManager.h:28,53`), so calling it from your own module's C++ fails to link.

---

## Running tests

### Session Frontend

**Window > Session Frontend > Automation**, connect to the local session, filter, tick and **Start Tests**.

### Command line

```
# Run everything under a name prefix and exit when the queue drains
UnrealEditor-Cmd.exe MyGame.uproject -ExecCmds="Automation RunTests MyGame.Inventory" -unattended -nopause -testexit="Automation Test Queue Empty" -log -abslog=TestOutput.log

# Run one filter bucket
UnrealEditor-Cmd.exe MyGame.uproject -ExecCmds="Automation RunFilter Smoke" -unattended -nopause -testexit="Automation Test Queue Empty"

# Enumerate, then run everything
UnrealEditor-Cmd.exe MyGame.uproject -ExecCmds="Automation List; Automation RunAll" -unattended -nopause
```

`Automation` subcommands: `List`, `RunTests <Filter>` (alias `RunTest`), `RunFilter <Name>`, `RunAll`, `SetFilter`, `SetPriority`, `SetMinimumPriority <Critical|High|Medium|Low|None>`, `StartRemoteSession <Guid>`, `Quit`, `SoftQuit`. Separate several with `;`.

`-testexit="<phrase>"` makes the process exit once that phrase appears in the log; `-unattended` and `-nopause` stop modal dialogs from blocking CI.

---

## Grouping failures and reporting numbers

```cpp
bool FMyPipelineTest::RunTest(const FString& Parameters)
{
    PushContext(TEXT("Phase 1: Setup"));
    TestTrue(TEXT("Subsystem present"), Subsystem != nullptr);
    PopContext();

    PushContext(TEXT("Phase 2: Execute"));
    TestEqual(TEXT("Processed count"), Subsystem->Process(), 10);
    PopContext();

    const double Start = FPlatformTime::Seconds();
    Subsystem->Process();
    const double DurationMs = (FPlatformTime::Seconds() - Start) * 1000.0;

    FAutomationTestFramework::Get().AddAnalyticsItemToCurrentTest(
        FString::Printf(TEXT("ProcessTimeMs=%.2f"), DurationMs));
    return true;
}
```

`PushContext` / `PopContext` exist both on `FAutomationTestBase` and on `ExecutionInfo`; errors recorded while a context is active carry the context string into the report.
