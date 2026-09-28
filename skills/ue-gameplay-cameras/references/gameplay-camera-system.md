# Gameplay Camera System Reference (UE 5.8)

Plugin: `Engine/Plugins/Cameras/GameplayCameras`. `GameplayCameras.uplugin` declares `"EnabledByDefault": true` and `"IsExperimentalVersion": true` (version name `0.1`), so it is on by default but the API is still Experimental. Modules: `GameplayCameras` (Runtime, `PreDefault`), `GameplayCamerasUncookedOnly`, `GameplayCamerasEditor`. It depends on the `EnhancedInput`, `StateTree` and `TemplateSequence` plugins.

Build.cs for your own module: add `"GameplayCameras"`. The plugin module itself publicly depends on `CinematicCamera`, `Core`, `CoreUObject`, `DeveloperSettings`, `Engine`, `EnhancedInput`, `GameplayTags`, `HeadMountedDisplay`, `MovieScene`, `MovieSceneTracks`, `RewindDebuggerRuntimeInterface`, `StateTreeModule`, `TemplateSequence` and `TraceLog`.

## Object model

| Object | Header | Role |
|---|---|---|
| `UCameraAsset` | `Core/CameraAsset.h` | top-level asset: one `UCameraDirector` plus enter/exit `UCameraRigTransition` lists and an interface parameter set |
| `UCameraRigAsset` | `Core/CameraRigAsset.h` | a camera behaviour: `RootNode` (`UCameraNode`), `GameplayTags`, `EnterTransitions`, `ExitTransitions`, `InitialOrientation` |
| `UCameraShakeAsset` | `Core/CameraShakeAsset.h` | a shake built from `UShakeCameraNode` trees |
| `UCameraDirector` | `Core/CameraDirector.h` | decides which rigs are active each frame |
| `UCameraNode` | `Core/CameraNode.h` | one step of the rig evaluation |
| `UCameraVariableAsset` | `Core/CameraVariableAssets.h` | a named value shared through the variable table |
| `UCameraRigProxyAsset` | `Core/CameraRigProxyAsset.h` | indirection so a director can name a rig that is resolved per-context |
| `FCameraEvaluationContext` | `Core/CameraEvaluationContext.h` | who/what the camera is evaluating for |
| `FCameraSystemEvaluator` | `Core/CameraSystemEvaluator.h` | the runtime that owns contexts, services and the blend stack |

`UCameraAsset::BuildCamera()` and `UCameraRigAsset::BuildCameraRig()` compile the authored graph into allocation info; `UGameplayCamerasSettings::bAutoBuildInPIE` does this automatically when entering PIE.

## Actors and components

| Class | Header | Notes |
|---|---|---|
| `UGameplayCameraComponentBase` | `GameFramework/GameplayCameraComponentBase.h` | abstract base, implements `IGameplayCameraSystemHost`; holds the activation, rig, shake and action API |
| `UGameplayCameraComponent` | `GameFramework/GameplayCameraComponent.h` | runs a `UCameraAsset` via `FCameraAssetReference CameraReference` |
| `UGameplayCameraRigComponent` | `GameFramework/GameplayCameraRigComponent.h` | runs a single `UCameraRigAsset` via `FCameraRigAssetReference` |
| `AGameplayCameraActorBase` | `GameFramework/GameplayCameraActorBase.h` | overrides `CalcCamera`; base of the two actors below |
| `AGameplayCameraActor` | `GameFramework/GameplayCameraActor.h` | `GetCameraComponent()`, `GetOutputCameraComponent()` |
| `AGameplayCameraRigActor` | `GameFramework/GameplayCameraRigActor.h` | `GetCameraRigComponent()` |
| `UGameplayCameraSystemComponent` | `GameFramework/GameplayCameraSystemComponent.h` | hosts a camera system on any actor |
| `AGameplayCameraSystemActor` | `GameFramework/GameplayCameraSystemActor.h` | actor wrapper; `GetCameraSystemComponent()` |
| `AGameplayCamerasPlayerCameraManager` | `GameFramework/GameplayCamerasPlayerCameraManager.h` | `APlayerCameraManager` that runs the system |
| `UGameplayControlRotationComponent` | `GameFramework/GameplayControlRotationComponent.h` | writes the evaluated orientation back to the control rotation |

### UGameplayCameraComponentBase API

```cpp
void ActivateCameraForPlayerIndex(int32 PlayerIndex, bool bSetAsViewTarget = true);
void ActivateCameraForPlayerController(APlayerController* PlayerController, bool bSetAsViewTarget = true);
UCineCameraComponent* GetOutputCameraComponent() const;
FBlueprintCameraEvaluationDataRef GetInitialResult() const;
FBlueprintCameraEvaluationDataRef GetConditionalResult(ECameraEvaluationDataCondition Condition) const;
FRotator GetEvaluatedCameraRotation() const;

FCameraRigInstanceID ActivatePersistentBaseCameraRig(UCameraRigAsset* CameraRig);
FCameraRigInstanceID ActivatePersistentGlobalCameraRig(UCameraRigAsset* CameraRig);
FCameraRigInstanceID ActivatePersistentVisualCameraRig(UCameraRigAsset* CameraRig);
void DeactivateCameraRig(FCameraRigInstanceID InstanceID, bool bImmediately = false);

FCameraRigInstanceID StartGlobalCameraModifierRig(const UCameraRigAsset* CameraRig, int32 OrderKey = 0);
FCameraRigInstanceID StartVisualCameraModifierRig(const UCameraRigAsset* CameraRig, int32 OrderKey = 0);
void StopCameraModifierRig(FCameraRigInstanceID InstanceID, bool bImmediately = false);

FCameraShakeInstanceID StartCameraShakeAsset(const UCameraShakeAsset* CameraShake, float ShakeScale = 1.f, ECameraShakePlaySpace PlaySpace = ECameraShakePlaySpace::CameraLocal, FRotator UserPlaySpaceRotation = FRotator::ZeroRotator);
bool IsCameraShakeAssetPlaying(FCameraShakeInstanceID InInstanceID) const;
bool StopCameraShakeAsset(FCameraShakeInstanceID InInstanceID, bool bImmediately = false);

FCameraActionInstanceID StartAction(const UCameraAction* CameraAction);
bool IsActionRunning(FCameraActionInstanceID InInstanceID);
bool StopAction(FCameraActionInstanceID InInstanceID);
bool StopAllActionsOfClass(TSubclassOf<UCameraAction> InActionClass);
```

Deactivation uses the inherited `UActorComponent::Deactivate()`. Properties: `DefaultPlayer` (`TEnumAsByte<EAutoReceiveInput::Type>`), `bRunStandaloneCameraSystem`, `bSetControlRotationWhenViewTarget`, `bPlaybackMode`. The two hooks a subclass implements are `virtual UCameraAsset* OnCreateEvaluationContext()` and `virtual void OnUpdateEvaluationContext(bool bForceApplyParameterOverrides)`.

`ECameraRigLayer` is `None`, `Base`, `Main`, `Global`, `Visual`. Only the main layer is driven by the director; base/global/visual are driven by the persistent and modifier-rig calls above.

### UGameplayCameraSystemComponent

```cpp
void GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView);
void ActivateCameraSystemForPlayerIndex(int32 PlayerIndex);
void ActivateCameraSystemForPlayerController(APlayerController* PlayerController);
bool IsCameraSystemActiveForPlayController(APlayerController* PlayerController) const;
void DeactivateCameraSystem(AActor* NextViewTarget = nullptr);
```

Plus `AutoActivateForPlayer` and `bSetPlayerControllerRotation`.

### AGameplayCamerasPlayerCameraManager

Setting a view target on this manager pushes an evaluation context for the target actor: a `UGameplayCameraComponentBase` supplies its own context, a plain `UCameraComponent` is wrapped, and any other actor is wrapped around its `CalcCamera` output. There is only ever one active view target — `PendingViewTarget` is unused, because blending happens on the rig blend stack instead.

```cpp
void StealPlayerController(APlayerController* PlayerController);   // not UE_API-exported (MinimalAPI class):
void ReleasePlayerController();                                     // Blueprint-only; unresolved external from game C++
void ActivateGameplayCamera(UGameplayCameraComponentBase* GameplayCamera, EGameplayCameraComponentActivationMode ActivationMode = EGameplayCameraComponentActivationMode::Push);
void DeactivateGameplayCamera(UGameplayCameraComponentBase* GameplayCamera, bool bDeactivateAllCameraRigs = false);
```

`EGameplayCameraComponentActivationMode` is `Push` or `InsertOrPush`. `EGameplayCamerasViewRotationMode` (`None`, `PreviewUpdate`) controls how the manager derives the player's view rotation; the project default is `UGameplayCamerasSettings::DefaultViewRotationMode`.

### UGameplayControlRotationComponent

```cpp
void ActivateControlRotationManagementForPlayerIndex(int32 PlayerIndex);
void ActivateControlRotationManagementForPlayerController(APlayerController* PlayerController);
void DeactivateControlRotationManagement();
```

None of the three is exported (`MinimalAPI` class, no `UE_API`, `GameFramework/GameplayControlRotationComponent.h:40,47,53`), so game C++ fails to link against them — call them from Blueprint, or set `AutoActivateForPlayer` instead.

Properties: `TArray<TObjectPtr<UInputAction>> AxisActions`, `AxisActionAngularSpeedThreshold` (degrees/second above which player input thaws a frozen control rotation), `AxisActionMagnitudeThreshold`, `AutoActivateForPlayer`. It installs an `FPlayerControlRotationEvaluationService` into the camera system.

## Camera directors

| Class | Configure with |
|---|---|
| `USingleCameraDirector` | `TObjectPtr<UCameraRigAsset> CameraRig` — always runs one rig |
| `UBlueprintCameraDirector` | `TSubclassOf<UBlueprintCameraDirectorEvaluator> CameraDirectorEvaluatorClass` |
| `UStateTreeCameraDirector` | `FStateTreeReference StateTreeReference` built with `UCameraDirectorStateTreeSchema` |
| `UPriorityQueueCameraDirector` | child contexts added with priorities by `FPriorityQueueCameraDirectorEvaluator::AddChildEvaluationContext` |

`UBlueprintCameraDirectorEvaluator` is a Blueprintable `UObject`. Implement the events `ActivateCameraDirector(UObject* EvaluationContextOwner, const FBlueprintCameraDirectorActivateParams& Params)`, `DeactivateCameraDirector(UObject* EvaluationContextOwner, const FBlueprintCameraDirectorDeactivateParams& Params)` and `RunCameraDirector(float DeltaTime, UObject* EvaluationContextOwner, const FBlueprintCameraDirectorEvaluationParams& Params)`, then call:

```cpp
void ActivateCameraRig(UCameraRigAsset* CameraRig, bool bForceNewInstance = false);
void ActivateCameraRigViaProxy(UCameraRigProxyAsset* CameraRigProxy, bool bForceNewInstance = false);
UCameraRigAsset* ResolveCameraRigProxy(const UCameraRigProxyAsset* CameraRigProxy) const;
AActor* FindEvaluationContextOwnerActor(TSubclassOf<AActor> ActorClass) const;
FBlueprintCameraEvaluationDataRef GetInitialContextResult() const;
FName AddChildEvaluationContext(UObject* ChildEvaluationContextOwner);
bool RemoveChildEvaluationContext(UObject* ChildEvaluationContextOwner, FName ChildSlotName);
bool RunChildCameraDirector(float DeltaTime, FName ChildSlotName);
```

The State Tree route uses `UCameraDirectorStateTreeSchema` with tasks `FGameplayCamerasActivateCameraRigTask` ("Activate Camera Rig") and `FGameplayCamerasActivateCameraRigViaProxyTask` ("Activate Camera via Proxy"); custom tasks derive from `FGameplayCamerasStateTreeTask` and conditions from `FGameplayCamerasStateTreeCondition`.

### Native camera director

```cpp
// MyCameraDirector.h
#pragma once

#include "Core/CameraDirector.h"
#include "MyCameraDirector.generated.h"

class UCameraRigAsset;

UCLASS(EditInlineNew)
class MYGAME_API UMyCameraDirector : public UCameraDirector
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Common")
    TObjectPtr<UCameraRigAsset> CombatRig;

protected:
    virtual FCameraDirectorEvaluatorPtr OnBuildEvaluator(FCameraDirectorEvaluatorBuilder& Builder) const override;
};

// MyCameraDirector.cpp
#include "MyCameraDirector.h"

#include "Core/CameraDirectorEvaluator.h"

namespace UE::Cameras
{

class FMyCameraDirectorEvaluator : public FCameraDirectorEvaluator
{
    UE_DECLARE_CAMERA_DIRECTOR_EVALUATOR(MYGAME_API, FMyCameraDirectorEvaluator)

protected:
    virtual void OnRun(const FCameraDirectorEvaluationParams& Params, FCameraDirectorEvaluationResult& OutResult) override
    {
        const UMyCameraDirector* MyDirector = GetCameraDirectorAs<UMyCameraDirector>();
        if (MyDirector->CombatRig)
        {
            OutResult.Add(GetEvaluationContext(), MyDirector->CombatRig);
        }
    }
};

UE_DEFINE_CAMERA_DIRECTOR_EVALUATOR(FMyCameraDirectorEvaluator)

}  // namespace UE::Cameras

FCameraDirectorEvaluatorPtr UMyCameraDirector::OnBuildEvaluator(FCameraDirectorEvaluatorBuilder& Builder) const
{
    using namespace UE::Cameras;
    return Builder.BuildEvaluator<FMyCameraDirectorEvaluator>();
}
```

Other `UCameraDirector` hooks: `virtual void OnBuildCameraDirector(UE::Cameras::FCameraBuildContext& BuildContext)` (report authoring errors into `BuildContext.BuildLog`), `virtual void OnGatherRigUsageInfo(FCameraDirectorRigUsageInfo& UsageInfo) const`, `virtual void OnExtendAssetRegistryTags(FAssetRegistryTagsContext Context) const`, `virtual void OnFactoryCreateAsset(const FCameraDirectorFactoryCreateParams& InParams)`.

Other `FCameraDirectorEvaluator` hooks: `OnInitialize`, `OnActivate`, `OnDeactivate`, `OnAddChildEvaluationContext`, `OnRemoveChildEvaluationContext`, `OnAddReferencedObjects`. `FCameraDirectorEvaluationResult::Add` has overloads for a rig, a rig proxy, and a rig on a specific `ECameraRigLayer` with an `OrderKey`; `Remove` deactivates one.

## Camera nodes that ship in 5.8

| Folder | Nodes |
|---|---|
| `Nodes/Attach` | `UAttachToActorCameraNode`, `UAttachToActorGroupCameraNode`, `UAttachToPlayerPawnCameraNode` |
| `Nodes/Common` | `UArrayCameraNode`, `UAutoFocusCameraNode`, `UBodyParametersCameraNode`, `UBoomArmCameraNode`, `UCameraRigCameraNode`, `UClippingPlanesCameraNode`, `UDampenPositionCameraNode`, `UDampenRotationCameraNode`, `UFieldOfViewCameraNode`, `UFilmbackCameraNode`, `UFirstPersonCameraNode`, `ULensParametersCameraNode`, `UOffsetCameraNode`, `UOrthographicCameraNode`, `UPostProcessCameraNode`, `USetLocationCameraNode`, `USetRotationCameraNode`, `USplineFieldOfViewCameraNode`, `USplineOffsetCameraNode`, `USplineOrbitCameraNode`, `UTargetRayCastCameraNode` |
| `Nodes/Blends` | `USimpleBlendCameraNode`, `USimpleFixedTimeBlendCameraNode`, `ULinearBlendCameraNode`, `UEasingBlendCameraNode`, `USmoothBlendCameraNode`, `UPopBlendCameraNode`, `UOrbitBlendCameraNode`, `ULocationRotationBlendCameraNode` |
| `Nodes/Collision` | `UCollisionPushCameraNode`, `UOcclusionMaterialCameraNode` |
| `Nodes/Framing` | `UBaseFramingCameraNode`, `UDollyFramingCameraNode`, `UPanningFramingCameraNode` |
| `Nodes/Input` | `UInput1DCameraNode`, `UInput2DCameraNode`, `UCameraRigInput1DSlot`, `UCameraRigInput2DSlot`, `UInputAxisBinding2DCameraNode`, `URawInputAxisBinding2DCameraNode`, `UInputAccumulator2DCameraNode`, `UAutoRotateInput2DCameraNode`, `UDrivenControlRotationCameraNode` |
| `Nodes/Shakes` | `UCameraShakeCameraNode`, `UCompositeShakeCameraNode`, `UEnvelopeShakeCameraNode`, `UPerlinNoiseLocationShakeCameraNode`, `UPerlinNoiseRotationShakeCameraNode` |
| `Nodes/Utility` | `UBlueprintCameraNode` |

`UInputAxisBinding2DCameraNode` reads `TArray<TObjectPtr<UInputAction>> AxisActions` from Enhanced Input, with `bIsAccumulated` controlling whether values integrate over time. `UBoomArmCameraNode` is the plugin's spring-arm equivalent: `FVector3dCameraParameter BoomOffset`, `FDoubleCameraParameter MaxForwardInterpolationFactor`, `MaxBackwardInterpolationFactor`, `bAdditiveRotation`.

`ECameraNodeSpace` (`Nodes/CameraNodeTypes.h`) selects the frame offsets are applied in: `CameraPose`, `ActiveContext`, `OwningContext`, `Pivot`, `Pawn`, `World`.

## Writing a camera node

A node is a `UCameraNode` (data, editable in the rig graph) paired with an `FCameraNodeEvaluator` (runtime, allocated in the evaluator storage). This complete node offsets the camera by a shoulder offset that designers can bind to a camera variable:

```cpp
// MyShoulderSwapCameraNode.h
#pragma once

#include "Core/CameraNode.h"
#include "Core/CameraParameters.h"
#include "MyShoulderSwapCameraNode.generated.h"

UCLASS(meta = (CameraNodeCategories = "Common"))
class MYGAME_API UMyShoulderSwapCameraNode : public UCameraNode
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Common")
    FVector3dCameraParameter ShoulderOffset;

protected:
    virtual FCameraNodeEvaluatorPtr OnBuildEvaluator(FCameraNodeEvaluatorBuilder& Builder) const override;
};

// MyShoulderSwapCameraNode.cpp - also include Core/CameraNodeEvaluator.h and Core/CameraParameterReader.h
#include "MyShoulderSwapCameraNode.h"

namespace UE::Cameras
{

class FMyShoulderSwapCameraNodeEvaluator : public FCameraNodeEvaluator
{
    UE_DECLARE_CAMERA_NODE_EVALUATOR(MYGAME_API, FMyShoulderSwapCameraNodeEvaluator)

protected:
    virtual void OnInitialize(const FCameraNodeEvaluatorInitializeParams& Params, FCameraNodeEvaluationResult& OutResult) override
    {
        SetNodeEvaluatorFlags(ECameraNodeEvaluatorFlags::None);
        const UMyShoulderSwapCameraNode* ShoulderNode = GetCameraNodeAs<UMyShoulderSwapCameraNode>();
        ShoulderOffsetReader.Initialize(ShoulderNode->ShoulderOffset);
    }

    virtual void OnRun(const FCameraNodeEvaluationParams& Params, FCameraNodeEvaluationResult& OutResult) override
    {
        const FVector3d Offset = ShoulderOffsetReader.Get(OutResult.VariableTable);
        FTransform3d Transform = OutResult.CameraPose.GetTransform();
        Transform.AddToTranslation(Transform.TransformVectorNoScale(Offset));
        OutResult.CameraPose.SetTransform(Transform);
    }

private:
    TCameraParameterReader<FVector3d> ShoulderOffsetReader;
};

UE_DEFINE_CAMERA_NODE_EVALUATOR(FMyShoulderSwapCameraNodeEvaluator)

}  // namespace UE::Cameras

FCameraNodeEvaluatorPtr UMyShoulderSwapCameraNode::OnBuildEvaluator(FCameraNodeEvaluatorBuilder& Builder) const
{
    using namespace UE::Cameras;
    return Builder.BuildEvaluator<FMyShoulderSwapCameraNodeEvaluator>();
}
```

The declaration macro is required — it registers the evaluator's RTTI type ID so evaluator casts (`CastThis`/`CastThisChecked`) work. `GetCameraNodeAs` is a plain `UObject` `Cast` of the node (`Core/CameraNodeEvaluator.h:336-339`).

`UCameraNode` protected hooks:

```cpp
virtual FCameraNodeChildrenView OnGetChildren();
virtual void OnPreBuild(FCameraBuildContext& BuildContext);
virtual void OnBuild(FCameraObjectBuildContext& BuildContext);
virtual FCameraNodeEvaluatorPtr OnBuildEvaluator(FCameraNodeEvaluatorBuilder& Builder) const;
```

A node that returns children must call `AddNodeFlags(ECameraNodeFlags::CustomGetChildren)`. `bIsEnabled` is an inherited `UPROPERTY` that the system honours.

`FCameraNodeEvaluator` protected hooks:

```cpp
virtual void OnBuild(const FCameraNodeEvaluatorBuildParams& Params);
virtual void OnInitialize(const FCameraNodeEvaluatorInitializeParams& Params, FCameraNodeEvaluationResult& OutResult);
virtual void OnTeardown(const FCameraNodeEvaluatorTeardownParams& Params);
virtual void OnAddReferencedObjects(FReferenceCollector& Collector);
virtual FCameraNodeEvaluatorChildrenView OnGetChildren();
virtual void OnUpdateParameters(const FCameraBlendedParameterUpdateParams& Params, FCameraBlendedParameterUpdateResult& OutResult);
virtual void OnRun(const FCameraNodeEvaluationParams& Params, FCameraNodeEvaluationResult& OutResult);
virtual void OnExecuteOperation(const FCameraOperationParams& Params, FCameraOperation& Operation);
virtual void OnSerialize(const FCameraNodeEvaluatorSerializeParams& Params, FArchive& Ar);
virtual void OnBuildDebugBlocks(const FCameraDebugBlockBuildParams& Params, FCameraDebugBlockBuilder& Builder);  // inside #if UE_GAMEPLAY_CAMERAS_DEBUG
```

`ECameraNodeEvaluatorFlags`: `None`, `NeedsParameterUpdate`, `NeedsSerialize`, `SupportsOperations`, and `Default` (all three, and the initial value of every evaluator). `OnUpdateParameters` needs `NeedsParameterUpdate`, `OnExecuteOperation` needs `SupportsOperations`, `OnSerialize` needs `NeedsSerialize`. Set them with `SetNodeEvaluatorFlags(ECameraNodeEvaluatorFlags)` or `AddNodeEvaluatorFlags(ECameraNodeEvaluatorFlags InFlags)` from the constructor or `OnInitialize`. `TCameraNodeEvaluator<CameraNodeType>` is a convenience base that types `GetCameraNode()`.

`FCameraNodeEvaluationParams` carries `Evaluator`, `EvaluationContext`, `DeltaTime`, `EvaluationType`, `bIsFirstFrame`, `bIsActiveCameraRig` and `bool IsStatelessEvaluation() const`. `FCameraNodeEvaluationResult` carries `CameraPose` (`FCameraPose`), `VariableTable`, `ContextDataTable`, `CameraRigJoints`, `PostProcessSettings`, `bIsCameraCut`, `bIsValid`, plus `void GetViewInfo(FMinimalViewInfo& OutViewInfo) const`, `OverrideAll` and `LerpAll`.

`FCameraPose` fields are private; use the generated accessors — `GetLocation`/`SetLocation`, `GetRotation`/`SetRotation`, `GetFieldOfView`/`SetFieldOfView`, `GetFocalLength`, `GetTargetDistance`, `GetAperture`, `GetFocusDistance`, and the composite `FTransform3d GetTransform() const` / `void SetTransform(FTransform3d Transform, bool bForceSet = false)`. Setters record a changed flag (`FCameraPoseFlags`) that the blend stack relies on, so never write the fields directly.

## Camera parameters and variables

Node properties that designers should be able to drive use camera parameter structs, one per type: `FBooleanCameraParameter`, `FInteger32CameraParameter`, `FFloatCameraParameter`, `FDoubleCameraParameter`, `FVector2fCameraParameter`, `FVector2dCameraParameter`, `FVector3fCameraParameter`, `FVector3dCameraParameter`, `FVector4fCameraParameter`, `FVector4dCameraParameter`, `FRotator3fCameraParameter`, `FRotator3dCameraParameter`, `FTransform3fCameraParameter`, `FTransform3dCameraParameter`. Each holds `Value`, `VariableID` and an optional `Variable` pointer to the matching `UCameraVariableAsset` subclass (`UBooleanCameraVariable`, `UFloatCameraVariable`, `UDoubleCameraVariable`, `UVector3dCameraVariable`, …).

Read a parameter at runtime through `TCameraParameterReader<T>`: call `Initialize(Node->Property)` in `OnInitialize`, then `Get(OutResult.VariableTable)` in `OnRun`. That single call resolves the constant value or the bound variable's current value.

`UCameraVariableAsset` exposes `FCameraVariableID GetVariableID() const`, `FCameraVariableDefinition GetVariableDefinition() const`, `virtual ECameraVariableType GetVariableType() const`, `FString GetDisplayName() const`. `UCameraVariableCollection` groups them; `FBuiltInCameraVariables` holds the engine's own. `FCameraVariableSetter` and `FCameraParameterSetterService` animate a variable toward a value over time.

## Evaluation contexts

`FCameraEvaluationContext` (`Core/CameraEvaluationContext.h`) is a `TSharedFromThis` class, not a `UObject`.

```cpp
FCameraEvaluationContext(const FCameraEvaluationContextInitializeParams& Params);
void Initialize(const FCameraEvaluationContextInitializeParams& Params);
UObject* GetOwner() const;
UWorld* GetWorld() const;
APlayerController* GetPlayerController() const;
const UCameraAsset* GetCameraAsset() const;
FCameraNodeEvaluationResult& GetInitialResult();
FCameraDirectorEvaluator* GetDirectorEvaluator() const;
TSharedPtr<FCameraEvaluationContext> GetParentContext();
TArrayView<const TSharedPtr<FCameraEvaluationContext>> GetChildrenContexts() const;
FIntPoint GetViewportSize() const;
FCameraNodeEvaluationResult& GetOrAddConditionalResult(ECameraEvaluationDataCondition Condition);
```

The class is not exported as a whole: `GetWorld()` and `GetViewportSize()` have no `GAMEPLAYCAMERAS_API` (`Core/CameraEvaluationContext.h:91,121`), so calling them from game C++ is a link error; the inline accessors are fine.

`FCameraEvaluationContextInitializeParams` has `Owner`, `CameraAsset` and `PlayerController`. Subclasses declare themselves with `UE_DECLARE_CAMERA_EVALUATION_CONTEXT(ApiDeclSpec, ClassName)` in the class body and `UE_DEFINE_CAMERA_EVALUATION_CONTEXT(ClassName)` at file scope, and may override `virtual void OnActivate(const FCameraEvaluationContextActivateParams& Params)`, `virtual void OnDeactivate(const FCameraEvaluationContextDeactivateParams& Params)` and `virtual void OnAddReferencedObjects(FReferenceCollector& Collector)`. `FGameplayCameraComponentEvaluationContext` and `FActorCameraEvaluationContext` are the shipped ones.

The initial result of a context is what rigs start from each frame — writing to `GetInitialResult().CameraPose` from gameplay code is how you feed an external pivot into the camera.

## Evaluation services

`FCameraEvaluationService` is a per-system helper registered with `FCameraSystemEvaluator::RegisterEvaluationService(TSharedRef<FCameraEvaluationService>)`. Hooks: `OnInitialize`, `OnPreUpdate`, `OnPostCameraDirectorUpdate`, `OnPostUpdate`, `OnTeardown`, `OnRootCameraNodeEvent`, `OnAddReferencedObjects`, `OnBuildDebugBlocks`. Declare with `UE_DECLARE_CAMERA_EVALUATION_SERVICE` / `UE_DEFINE_CAMERA_EVALUATION_SERVICE`.

Shipped services (`Services/`): `FCameraActionService`, `FCameraModifierService`, `FCameraParameterSetterService`, `FCameraShakeService`, `FOrientationInitializationService`, `FPlayerControlRotationEvaluationService` (`Services/PlayerControlRotationService.h`).

`UCameraAction` plus `FCameraActionEvaluator` are the scripted-moment layer (`Actions/AimAtActorCameraAction.h`, `AimAtCameraAction.h`, `BaseAimAtCameraAction.h` ship with the plugin); start them with `StartAction` on the component.

## Transitions

`UCameraRigTransition` holds `Conditions` (instanced `UCameraRigTransitionCondition` objects) and a `UBlendCameraNode` to run. Shipped conditions: `UIsCameraRigTransitionCondition` (`Transitions/DefaultTransitionConditions.h`) and `UGameplayTagTransitionCondition` (`Transitions/GameplayTagTransitionConditions.h`, matched against `UCameraRigAsset::GameplayTags`). A custom condition overrides `virtual bool OnTransitionMatches(const FCameraRigTransitionConditionMatchParams& Params) const`.

`ECameraRigInitialOrientation` on the rig, and the per-transition override, decide how a newly activated rig picks its starting orientation.

## Camera system evaluator

```cpp
void Initialize(const FCameraSystemEvaluatorCreateParams& Params);
void PushEvaluationContext(TSharedRef<FCameraEvaluationContext> EvaluationContext);
void RemoveEvaluationContext(TSharedRef<FCameraEvaluationContext> EvaluationContext);
void PopEvaluationContext();
void RegisterEvaluationService(TSharedRef<FCameraEvaluationService> EvaluationService);
void UnregisterEvaluationService(TSharedRef<FCameraEvaluationService> EvaluationService);
TSharedPtr<FCameraEvaluationService> FindEvaluationService(const FCameraObjectTypeID& TypeID) const;
void Update(const FCameraSystemEvaluationParams& Params);
void ViewRotationPreviewUpdate(const FCameraSystemEvaluationParams& Params, FCameraSystemViewRotationEvaluationResult& OutResult);
void GetEvaluatedCameraView(FMinimalViewInfo& DesiredView);
void ExecuteOperation(FCameraOperation& Operation);
```

Reach it through `IGameplayCameraSystemHost::GetCameraSystemEvaluator()`, implemented by `UGameplayCameraComponentBase`, `UGameplayCameraSystemComponent` and `AGameplayCamerasPlayerCameraManager` (the actors expose it through their components).

## Debugging

Console variables (all `GameplayCameras.*`):

| CVar | Effect |
|---|---|
| `GameplayCameras.Debug.Enable` | master switch for the on-screen debug drawing |
| `GameplayCameras.Debug.Categories` | comma list from `nodetree`, `directortree`, `blendstacks`, `services`, `posestats`, `viewfinder` (see `FCameraDebugCategories`) |
| `GameplayCameras.Debug.SystemID` | pick which camera system to draw when several run |
| `GameplayCameras.Debug.Trace` | emit camera system traces for Insights / Rewind Debugger |
| `GameplayCameras.Debug.NodeTree.Filter` | substring filter for the node tree view |
| `GameplayCameras.Debug.BlendStacks.Filter` | substring filter for the blend stack view |
| `GameplayCameras.Debug.BlendStack.ShowUnchanged` / `ShowVariableIDs` / `ShowDataIDs` | verbosity of the blend stack view |
| `GameplayCameras.Debug.PoseStats.ShowUnchanged` / `ShowVariableIDs` / `ShowDataIDs` | verbosity of the pose stats view |
| `GameplayCameras.Debug.ContextInitialResult.ShowCoordinateSystem` | draw the context's initial result axes |
| `GameplayCameras.Debug.Damping.ShowLocalSpace` | visualise damping nodes |
| `GameplayCameras.Framing.ShowEffectiveDeadZone` | visualise framing node dead zones |
| `GameplayCameras.Debug.ColorScheme` | debug palette |
| `GameplayCameras.TargetRayCastLength` | length used by `UTargetRayCastCameraNode` |
| `GameplayCameras.CriticalDamper.StabilizationThreshold` | damping stability tuning |

Project settings live on `UGameplayCamerasSettings` (`Config=GameplayCameras`, display name "Gameplay Cameras"): `bAutoBuildInPIE`, `DefaultViewRotationMode`, `CombinedCameraRigNumThreshold`, `MaxUsageSearchDistance`, `DefaultIKAimingAngleTolerance`, `DefaultIKAimingDistanceTolerance`, `DefaultIKAimingMaxIterations`, `DefaultIKAimingMinDistance`.

`ShowDebug Camera` still works, because the gameplay camera manager overrides `DisplayDebug` on top of the classic output.
