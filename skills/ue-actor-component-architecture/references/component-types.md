# Component Types Reference (UE 5.8)

Built-in component classes with inheritance, header paths, verbatim member signatures and creation snippets. Paths are relative to `Engine/Source/Runtime/Engine/Classes/` unless a module or plugin is named. Every class below exists in the 5.8 headers.

## Inheritance hierarchy

```
UObject
  └── UActorComponent                          Components/ActorComponent.h
        ├── UMovementComponent                 GameFramework/MovementComponent.h        → ue-character-movement
        │     ├── UProjectileMovementComponent GameFramework/ProjectileMovementComponent.h
        │     ├── URotatingMovementComponent   GameFramework/RotatingMovementComponent.h
        │     ├── UInterpToMovementComponent   Components/InterpToMovementComponent.h
        │     └── UNavMovementComponent → UPawnMovementComponent → UFloatingPawnMovement, UCharacterMovementComponent
        ├── UTimelineComponent                 Components/TimelineComponent.h
        └── USceneComponent                    Components/SceneComponent.h
              ├── UChildActorComponent         Components/ChildActorComponent.h
              ├── UAudioComponent              Components/AudioComponent.h               → ue-audio-system
              ├── UDecalComponent              Components/DecalComponent.h
              ├── UPostProcessComponent        Components/PostProcessComponent.h
              ├── USceneCaptureComponent → USceneCaptureComponent2D   Components/SceneCaptureComponent2D.h
              ├── UPhysicsConstraintComponent  PhysicsEngine/PhysicsConstraintComponent.h → ue-physics-collision
              ├── USpringArmComponent          GameFramework/SpringArmComponent.h        → ue-gameplay-cameras
              ├── UCameraComponent             Camera/CameraComponent.h                  → ue-gameplay-cameras
              ├── ULightComponentBase          Components/LightComponentBase.h
              │     ├── USkyLightComponent     Components/SkyLightComponent.h
              │     └── ULightComponent → UDirectionalLightComponent
              │           └── ULocalLightComponent → URectLightComponent; UPointLightComponent → USpotLightComponent
              └── UPrimitiveComponent          Components/PrimitiveComponent.h
                    ├── UShapeComponent → UCapsuleComponent, UBoxComponent, USphereComponent
                    ├── UArrowComponent, UBillboardComponent, UTextRenderComponent, USplineComponent
                    ├── UFXSystemComponent → UNiagaraComponent   (Niagara plugin)         → ue-niagara-effects
                    └── UMeshComponent         Components/MeshComponent.h
                          ├── UStaticMeshComponent → UInstancedStaticMeshComponent → UHierarchicalInstancedStaticMeshComponent; USplineMeshComponent
                          ├── USkinnedMeshComponent → USkeletalMeshComponent, UPoseableMeshComponent
                          └── UWidgetComponent   (UMG module)                            → ue-ui-umg-slate
```

`UProceduralMeshComponent` lives in the `ProceduralMeshComponent` plugin (`Plugins/Runtime/ProceduralMeshComponent`); `UDynamicMeshComponent` in the `GeometryFramework` module. Both are owned by `ue-procedural-generation`.

## Layer 1: UActorComponent

**Header**: `Components/ActorComponent.h`. Base of every component; no transform.

Public configuration (all `uint8 x:1` bit fields unless noted):

| Member | Line | Purpose |
|---|---|---|
| `struct FActorComponentTickFunction PrimaryComponentTick` | `:177` | Set `bCanEverTick`, `bStartWithTickEnabled`, `TickGroup`, `TickInterval` in the constructor |
| `TArray<FName> ComponentTags` | `:181` | Grouping; query with `ComponentHasTag(FName Tag) const` (`:552`) |
| `bAutoActivate` | `:317` | Component activates itself during initialization |
| `bWantsInitializeComponent` | `:340` | Enables `InitializeComponent()` / `UninitializeComponent()` |
| `bTickInEditor` | `:294` | Tick while editing |
| `bIsEditorOnly` | `:344` | Stripped from non-editor builds |
| `EComponentCreationMethod CreationMethod` | `:431` | `Native`, `SimpleConstructionScript`, `UserConstructionScript`, `Instance` |

`bReplicates` (`:267`) and `bIsActive` (`:322`) are private: use `SetIsReplicated(bool ShouldReplicate)` (`:634`), `SetIsReplicatedByDefault(const bool bNewReplicates)` (`:1471`), `GetIsReplicated() const` (`:637`), `IsActive() const` (`:607`).

Virtuals to override (verbatim):

```cpp
virtual void OnComponentCreated();                                            // :1331
virtual void OnRegister();                                                    // :830
virtual void OnUnregister();                                                  // :835
virtual void InitializeComponent();                                           // :919  requires bWantsInitializeComponent
virtual void UninitializeComponent();                                         // :955
virtual void BeginPlay();                                                     // :936
virtual void EndPlay(const EEndPlayReason::Type EndPlayReason);               // :949
virtual void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction); // :976
virtual void Activate(bool bReset = false);                                   // :580
virtual void Deactivate();                                                    // :586
virtual bool ShouldActivate() const;                                          // :807  return false to block activation
virtual void OnComponentDestroyed(bool bDestroyingHierarchy);                 // :1338
virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override; // :1246
```

Use for: health, stamina, inventory, cooldowns, status effects, per-actor save data, AI blackboard bridges. Creation:

```cpp
// Constructor
HealthComp = CreateDefaultSubobject<UMyHealthComponent>(TEXT("Health"));

// Runtime (inside an AActor member function)
UMyHealthComponent* Runtime = NewObject<UMyHealthComponent>(this, TEXT("RuntimeHealth"));
Runtime->RegisterComponent();
AddInstanceComponent(Runtime);
```

## Layer 2: USceneComponent

**Header**: `Components/SceneComponent.h`. Adds a relative transform and the attachment tree.

Private data with public accessors (do not write the members directly):

| Private member (line) | Read | Write |
|---|---|---|
| `FVector RelativeLocation` (`:139`) | `FVector GetRelativeLocation() const` (`:1435`) | `void SetRelativeLocation(FVector NewLocation, bool bSweep=false, FHitResult* OutSweepHitResult=nullptr, ETeleportType Teleport = ETeleportType::None)` (`:434`) |
| `FRotator RelativeRotation` (`:143`) | `FRotator GetRelativeRotation() const` (`:1478`) | `void SetRelativeRotation(FRotator NewRotation, bool bSweep=false, FHitResult* OutSweepHitResult=nullptr, ETeleportType Teleport = ETeleportType::None)` (`:447`) |
| `FVector RelativeScale3D` (`:150`) | `FVector GetRelativeScale3D() const` (`:1521`) | `void SetRelativeScale3D(FVector NewScale3D)` (`:473`) |
| `TObjectPtr<USceneComponent> AttachParent` (`:109`) | `USceneComponent* GetAttachParent() const` (`:701`) | attachment API below |
| `FName AttachSocketName` (`:113`) | `FName GetAttachSocketName() const` (`:705`) | attachment API below |
| `TArray<TObjectPtr<USceneComponent>> AttachChildren` (`:119`) | `const TArray<TObjectPtr<USceneComponent>>& GetAttachChildren() const` (`:697`) | attachment API below |
| `bAbsoluteLocation/Rotation/Scale` (`:171-179`) | `IsUsingAbsoluteLocation() const` (`:1574`) | `SetUsingAbsoluteLocation(const bool)` (`:1580`), `SetUsingAbsoluteRotation` (`:1605`), `SetUsingAbsoluteScale` (`:1629`) |
| `bVisible` (`:183`) | `virtual bool IsVisible() const` (`:853`), `GetVisibleFlag() const` (`:1649`) | `void SetVisibility(bool bNewVisibility, bool bPropagateToChildren=false)` (`:917`) |

Public: `TEnumAsByte<EComponentMobility::Type> Mobility` (`:303`) with `virtual void SetMobility(EComponentMobility::Type NewMobility)` (`:1296`) and `GetMobility() const` (`:1666`); `bHiddenInGame` (`:226`) with `void SetHiddenInGame(bool NewHidden, bool bPropagateToChildren=false)` (`:933`).

World-space transform: `GetComponentLocation()` (`:1066`), `GetComponentRotation()` (`:1072`), `GetComponentQuat()` (`:1078`), `GetComponentScale()` (`:1084`), `GetRelativeTransform()` (`:465`), `GetForwardVector()`/`GetUpVector()`/`GetRightVector()` (`:678-686`); `SetWorldLocation(FVector NewLocation, bool bSweep=false, FHitResult* OutSweepHitResult=nullptr, ETeleportType Teleport = ETeleportType::None)` (`:561`), `SetWorldRotation(FRotator ...)` (`:575`), `SetWorldTransform(const FTransform& NewTransform, ...)` (`:598`), `SetRelativeTransform(const FTransform& NewTransform, ...)` (`:461`), `AddWorldOffset(FVector DeltaLocation, ...)` (`:613`), `AddLocalOffset` (`:517`), `AddLocalRotation` (`:530`). Collision-aware move: `bool MoveComponent(const FVector& Delta, const FRotator& NewRotation, bool bSweep, FHitResult* Hit=NULL, EMoveComponentFlags MoveFlags = MOVECOMP_NoFlags, ETeleportType Teleport = ETeleportType::None)` (`:1042`). Sockets: `virtual FTransform GetSocketTransform(FName InSocketName, ERelativeTransformSpace TransformSpace = RTS_World) const` (`:801`), `GetSocketLocation` (`:809`), `GetSocketRotation` (`:817`), `DoesSocketExist` (`:832`).

Attachment (rules in `Engine/EngineTypes.h`):

```cpp
// Constructor / unregistered component                                       SceneComponent.h:734
Child->SetupAttachment(Parent);
Child->SetupAttachment(Parent, TEXT("SocketName"));

// Registered component at runtime                                              SceneComponent.h:752
Child->AttachToComponent(Parent, FAttachmentTransformRules::KeepRelativeTransform);
Child->AttachToComponent(Parent, FAttachmentTransformRules::KeepWorldTransform);
Child->AttachToComponent(Parent, FAttachmentTransformRules::SnapToTargetNotIncludingScale, TEXT("SocketName"));
Child->AttachToComponent(Parent, FAttachmentTransformRules(EAttachmentRule::SnapToTarget, EAttachmentRule::KeepWorld, EAttachmentRule::KeepRelative, /*bInWeldSimulatedBodies=*/true));

// Detach                                                                       SceneComponent.h:786
Child->DetachFromComponent(FDetachmentTransformRules::KeepWorldTransform);
Child->DetachFromComponent(FDetachmentTransformRules::KeepRelativeTransform);
```

| `FAttachmentTransformRules` preset (`EngineTypes.h:78-81`) | Location | Rotation | Scale | Typical use |
|---|---|---|---|---|
| `KeepRelativeTransform` | Relative | Relative | Relative | Keep the current local offset |
| `KeepWorldTransform` | World | World | World | Object already positioned in the world |
| `SnapToTargetIncludingScale` | Target | Target | Target | Hard snap to socket, adopt parent scale |
| `SnapToTargetNotIncludingScale` | Target | Target | Own | Hard snap to socket, keep own scale |

`EAttachmentRule` (`EngineTypes.h:62-71`): `KeepRelative`, `KeepWorld`, `SnapToTarget`. `EDetachmentRule` (`:112-118`): `KeepRelative`, `KeepWorld`. Tree queries: `GetChildrenComponents(bool bIncludeAllDescendants, TArray<USceneComponent*>& Children) const` (`:725`), `GetParentComponents(TArray<USceneComponent*>& Parents) const` (`:709`), `IsAttachedTo(const USceneComponent* TestComp) const` (`:1311`), `GetAttachmentRootActor() const` (`:1302`). Virtual hooks: `virtual void OnAttachmentChanged()` (`:1063`), `virtual void OnChildAttached(USceneComponent* ChildComponent)` (`:1343`), `virtual void OnChildDetached(USceneComponent* ChildComponent)` (`:1349`).

Use a bare `USceneComponent` as a pivot or group node:

```cpp
GroupPivot = CreateDefaultSubobject<USceneComponent>(TEXT("GroupPivot"));
GroupPivot->SetupAttachment(GetRootComponent());
MeshA->SetupAttachment(GroupPivot);
MeshB->SetupAttachment(GroupPivot);   // rotate GroupPivot to rotate both meshes
```

## Layer 3: UPrimitiveComponent

**Header**: `Components/PrimitiveComponent.h`. Adds render proxy, collision geometry, physics body (`FBodyInstance BodyInstance`, `:1445`) and overlap/hit events. Channels, profiles, traces and physics forces are owned by `ue-physics-collision`; the entry points are:

```cpp
virtual void SetCollisionEnabled(ECollisionEnabled::Type NewType);                                   // :2026  NoCollision, QueryOnly, PhysicsOnly, QueryAndPhysics, ProbeOnly, QueryAndProbe
virtual void SetCollisionProfileName(FName InCollisionProfileName, bool bUpdateOverlaps=true);       // :2036
virtual void SetCollisionObjectType(ECollisionChannel Channel);                                      // :2047
virtual void SetCollisionResponseToChannel(ECollisionChannel Channel, ECollisionResponse NewResponse); // :2944
virtual void SetCollisionResponseToAllChannels(ECollisionResponse NewResponse);                      // :2952
void SetGenerateOverlapEvents(bool bInGenerateOverlapEvents);                                        // :418
virtual void SetSimulatePhysics(bool bSimulate);                                                     // :1661
virtual void AddImpulse(FVector Impulse, FName BoneName = NAME_None, bool bVelChange = false);       // :1692
virtual void AddForce(FVector Force, FName BoneName = NAME_None, bool bAccelChange = false);         // :1759
```

Events (dynamic multicast, `:1457-1475`): `OnComponentHit` (`FComponentHitSignature`: `UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit`), `OnComponentBeginOverlap` (`FComponentBeginOverlapSignature`: `UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult`), `OnComponentEndOverlap` (`FComponentEndOverlapSignature`: `UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex`). Bind with `AddDynamic` to a `UFUNCTION()` whose parameters match exactly.

Rendering toggles: `SetCastShadow(bool NewCastShadow)` (`:1965`), `SetOwnerNoSee(bool)` (`:1953`), `SetOnlyOwnerSee(bool)` (`:1957`), `SetRenderCustomDepth(bool)` (`:2092`), `SetCustomDepthStencilValue(int32)` (`:2096`). Lightmap type is `GetLightmapType()`/`SetLightmapType()` (`:359`, the public field is deprecated).

## Mesh components

### UStaticMeshComponent (`Components/StaticMeshComponent.h`)

Rigid mesh. `virtual bool SetStaticMesh(UStaticMesh* NewMesh)` (`:450`); materials via `UMeshComponent::SetMaterial(int32 ElementIndex, UMaterialInterface* Material)` (`MeshComponent.h:121`).

```cpp
// Constructor
MeshComp = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
SetRootComponent(MeshComp);
MeshComp->SetMobility(EComponentMobility::Movable);

static ConstructorHelpers::FObjectFinder<UStaticMesh> MeshAsset(TEXT("/Game/Meshes/MyCrate")); // UObject/ConstructorHelpers.h:77
if (MeshAsset.Succeeded())
{
    MeshComp->SetStaticMesh(MeshAsset.Object);
}

// Runtime
MeshComp->SetStaticMesh(NewMesh);
MeshComp->SetMaterial(0, MaterialInstance);
```

### USkeletalMeshComponent (`Components/SkeletalMeshComponent.h`)

Skinned mesh with a skeleton; animation setup is owned by `ue-animation-system`.

| Member | Line |
|---|---|
| `void SetSkeletalMeshAsset(USkeletalMesh* NewMesh)` | `:369` |
| `virtual void SetAnimInstanceClass(class UClass* NewClass)` | `:1096` |
| `class UAnimInstance* GetAnimInstance() const` | `:1111` |
| `virtual FTransform GetSocketTransform(FName InSocketName, ERelativeTransformSpace TransformSpace = RTS_World) const override` | `SkinnedMeshComponent.h:1360` |
| `float UAnimInstance::Montage_Play(UAnimMontage* MontageToPlay, float InPlayRate = 1.f, ...)` | `Animation/AnimInstance.h:626` |

Direct bone posing is on `UPoseableMeshComponent` (`Components/PoseableMeshComponent.h`): `void SetBoneLocationByName(FName BoneName, FVector InLocation, EBoneSpaces::Type BoneSpace)` (`:37`).

```cpp
CharMesh = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("CharMesh"));
CharMesh->SetupAttachment(GetRootComponent());

// Runtime
if (UAnimInstance* Anim = CharMesh->GetAnimInstance())
{
    Anim->Montage_Play(AttackMontage);
}
const FTransform MuzzleTransform = CharMesh->GetSocketTransform(TEXT("Muzzle"));
```

### UInstancedStaticMeshComponent (`Components/InstancedStaticMeshComponent.h`)

Many instances of one mesh in one draw call. `UHierarchicalInstancedStaticMeshComponent` adds LOD/culling per cluster.

| Member | Line |
|---|---|
| `virtual int32 AddInstance(const FTransform& InstanceTransform, bool bWorldSpace = false)` | `:271` |
| `virtual TArray<int32> AddInstances(const TArray<FTransform>& InstanceTransforms, bool bShouldReturnIndices, bool bWorldSpace = false, bool bUpdateNavigation = true)` | `:275` |
| `virtual bool UpdateInstanceTransform(int32 InstanceIndex, const FTransform& NewInstanceTransform, bool bWorldSpace=false, bool bMarkRenderStateDirty=false, bool bTeleport=false)` | `:375` |
| `virtual bool BatchUpdateInstancesTransforms(int32 StartInstanceIndex, const TArray<FTransform>& NewInstancesTransforms, bool bWorldSpace=false, bool bMarkRenderStateDirty=false, bool bTeleport=false)` | `:388` |
| `virtual bool RemoveInstance(int32 InstanceIndex)` | `:417` |
| `virtual void ClearInstances()` | `:431` |

```cpp
void AMyForest::BuildInstances(const TArray<FTransform>& Transforms)
{
    TreeISM->ClearInstances();
    TreeISM->AddInstances(Transforms, /*bShouldReturnIndices=*/false, /*bWorldSpace=*/true);
}
```

## Shape components (`Components/ShapeComponent.h` base)

Invisible collision volumes for triggers and character collision.

| Class | Header | Size API |
|---|---|---|
| `UCapsuleComponent` | `Components/CapsuleComponent.h` | `InitCapsuleSize(float InRadius, float InHalfHeight)` (`:182`, constructor), `SetCapsuleSize(float InRadius, float InHalfHeight, bool bUpdateOverlaps=true)` (`:48`) |
| `UBoxComponent` | `Components/BoxComponent.h` | `InitBoxExtent(const FVector& InBoxExtent)` (`:64`), `SetBoxExtent(FVector InBoxExtent, bool bUpdateOverlaps=true)` (`:39`) |
| `USphereComponent` | `Components/SphereComponent.h` | `InitSphereRadius(float InSphereRadius)` (`:65`), `SetSphereRadius(float InSphereRadius, bool bUpdateOverlaps=true)` (`:33`) |

The bound handler is declared in the class body as a `UFUNCTION()` with the `FComponentBeginOverlapSignature` parameters: `void OnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);`

```cpp
// Trigger volume, set up in the AMyTriggerVolume constructor
TriggerBox = CreateDefaultSubobject<UBoxComponent>(TEXT("TriggerZone"));
TriggerBox->SetupAttachment(GetRootComponent());
TriggerBox->InitBoxExtent(FVector(200.f, 200.f, 100.f));       // half extents
TriggerBox->SetCollisionProfileName(TEXT("Trigger"));
TriggerBox->OnComponentBeginOverlap.AddDynamic(this, &AMyTriggerVolume::OnOverlapBegin);
```

`ACharacter` already owns a `UCapsuleComponent` root (`GetCapsuleComponent()`); see `ue-gameplay-framework`.

## Special components

### UChildActorComponent (`Components/ChildActorComponent.h`)

Embeds another actor at the component transform. `void SetChildActorClass(TSubclassOf<AActor> InClass)` (`:95`), `AActor* GetChildActor() const` (`:216`), `virtual void CreateChildActor(TFunction<void(AActor*)> CustomizerFunc = nullptr)` (`:211`), `void DestroyChildActor()` (`:227`). Lifecycle timing in [actor-lifecycle.md](actor-lifecycle.md#child-actors).

```cpp
// Constructor
ChildActorComp = CreateDefaultSubobject<UChildActorComponent>(TEXT("ChildActor"));
ChildActorComp->SetupAttachment(GetRootComponent());
ChildActorComp->SetChildActorClass(AMyTurret::StaticClass());

// BeginPlay or later
AMyTurret* Turret = Cast<AMyTurret>(ChildActorComp->GetChildActor());
```

### UArrowComponent (`Components/ArrowComponent.h`)

Direction gizmo for spawn points and facing. `FColor ArrowColor` (`:25`), `float ArrowSize` (`:29`), `float ArrowLength` (`:33`); hide in game with `SetHiddenInGame(true)`.

### UWidgetComponent (`Runtime/UMG/Public/Components/WidgetComponent.h`, module `UMG`)

Renders a `UUserWidget` in world or screen space. `void SetWidgetClass(TSubclassOf<UUserWidget> InWidgetClass)` (`:338`), `virtual UUserWidget* GetWidget() const` (`:207`), `void SetDrawSize(FVector2D Size)` (`:255`), `void SetDrawAtDesiredSize(bool bInDrawAtDesiredSize)` (`:321`), `void SetWidgetSpace(EWidgetSpace NewSpace)` (`:344`; `EWidgetSpace::World`, `EWidgetSpace::Screen`). Widget authoring is owned by `ue-ui-umg-slate`.

```cpp
// Build.cs: PublicDependencyModuleNames.Add("UMG");
HealthBarWidget = CreateDefaultSubobject<UWidgetComponent>(TEXT("HealthBar"));
HealthBarWidget->SetupAttachment(GetRootComponent());
HealthBarWidget->SetWidgetClass(UMyHealthBarWidget::StaticClass());
HealthBarWidget->SetDrawSize(FVector2D(200.f, 50.f));
HealthBarWidget->SetWidgetSpace(EWidgetSpace::Screen);
```

### UTimelineComponent (`Components/TimelineComponent.h`)

Curve-driven value playback without ticking your own interpolation; derives directly from `UActorComponent`.

## Audio and effects

### UAudioComponent (`Components/AudioComponent.h`) — owned by `ue-audio-system`

`void SetSound(USoundBase* NewSound)` (`:489`), `virtual void Play(float StartTime = 0.0f)` (`:517`), `virtual void Stop()` (`:583`), `virtual void FadeIn(float FadeInDuration, float FadeVolumeLevel = 1.0f, float StartTime = 0.0f, const EAudioFaderCurve FadeCurve = EAudioFaderCurve::Linear)` (`:500`), `virtual void FadeOut(float FadeOutDuration, float FadeVolumeLevel, const EAudioFaderCurve FadeCurve = EAudioFaderCurve::Linear)` (`:511`), `void SetVolumeMultiplier(float NewVolumeMultiplier)` (`:626`), `void SetPitchMultiplier(float NewPitchMultiplier)` (`:630`). Set `bAutoActivate = false` in the constructor for sounds that should not play on spawn.

### UNiagaraComponent (`Plugins/FX/Niagara/Source/Niagara/Public/NiagaraComponent.h`, module `Niagara`) — owned by `ue-niagara-effects`

`void SetAsset(UNiagaraSystem* InAsset, bool bResetExistingOverrideParameters = true)` (`:295`), `void SetVariableFloat(FName InVariableName, float InValue)` (`:533`), `void SetVariableLinearColor(FName InVariableName, const FLinearColor& InValue)` (`:470`); activation through the `UActorComponent` API (`Activate`, `Deactivate`).

## Physics

### UPhysicsConstraintComponent (`PhysicsEngine/PhysicsConstraintComponent.h`) — owned by `ue-physics-collision`

Joint between two bodies or a body and the world. `void SetConstrainedComponents(UPrimitiveComponent* Component1, FName BoneName1, UPrimitiveComponent* Component2, FName BoneName2)` (`:134`), `void SetAngularSwing1Limit(EAngularConstraintMotion MotionType, float Swing1LimitAngle)` (`:291`), `SetAngularSwing2Limit` (`:298`), `SetAngularTwistLimit(EAngularConstraintMotion ConstraintType, float TwistLimitAngle)` (`:305`); motion values `ACM_Free`, `ACM_Limited`, `ACM_Locked`.

## Cameras — owned by `ue-gameplay-cameras`

| Class | Header | Role |
|---|---|---|
| `USpringArmComponent` | `GameFramework/SpringArmComponent.h` | Collision-aware boom; children attach to `USpringArmComponent::SocketName` (`:158`) |
| `UCameraComponent` | `Camera/CameraComponent.h` | Supplies the view when the owner is the view target |

Setup, `bUsePawnControlRotation`, lag, FOV and the Gameplay Cameras plugin are documented in `ue-gameplay-cameras`.

## Component decision guide

| Need | Component | Owner skill for details |
|---|---|---|
| Logic only, no position | `UActorComponent` | this skill |
| Position anchor, pivot, grouping | `USceneComponent` | this skill |
| Rigid mesh | `UStaticMeshComponent` | this skill |
| Animated character or creature | `USkeletalMeshComponent` | `ue-animation-system` |
| Thousands of copies of one mesh | `UInstancedStaticMeshComponent` / `UHierarchicalInstancedStaticMeshComponent` | this skill, `ue-procedural-generation` |
| Mesh generated at runtime | `UProceduralMeshComponent`, `UDynamicMeshComponent` | `ue-procedural-generation` |
| Character collision root | `UCapsuleComponent` | `ue-gameplay-framework` |
| Trigger zone | `UBoxComponent`, `USphereComponent` | `ue-physics-collision` |
| Third-person camera rig | `USpringArmComponent` + `UCameraComponent` | `ue-gameplay-cameras` |
| World-space or screen-space UI on an actor | `UWidgetComponent` | `ue-ui-umg-slate` |
| Persistent particle effect | `UNiagaraComponent` | `ue-niagara-effects` |
| Spatial audio | `UAudioComponent` | `ue-audio-system` |
| Nested actor | `UChildActorComponent` | this skill |
| Direction marker | `UArrowComponent` | this skill |
| Physics joint | `UPhysicsConstraintComponent` | `ue-physics-collision` |
| Projectile, rotation, spline-following movement | `UProjectileMovementComponent`, `URotatingMovementComponent`, `UInterpToMovementComponent` | `ue-character-movement` |
| Walking character movement | `UCharacterMovementComponent`; Mover plugin `UMoverComponent` (Experimental in 5.8) | `ue-character-movement` |
| Abilities and attributes | `UAbilitySystemComponent` | `ue-gameplay-abilities` |
| Perception and navigation | `UAIPerceptionComponent`, `UNavModifierComponent` | `ue-ai-navigation` |
| State-machine logic on an actor | `UStateTreeComponent` | `ue-state-trees` |
| Input binding on a pawn | `UEnhancedInputComponent` | `ue-input-system` |
| Components injected by plugins or Game Features | `UPawnComponent`, `UControllerComponent`, `UPlayerStateComponent`, `UGameStateComponent` (ModularGameplay, Beta in 5.8) | `ue-game-features` |

## Finding components at runtime (`GameFramework/Actor.h`)

```cpp
// First match by class (templated cast)                                       Actor.h:3826
UMyHealthComponent* Health = Actor->FindComponentByClass<UMyHealthComponent>();

// All components of a class                                                    Actor.h:4024
TArray<UStaticMeshComponent*> Meshes;
Actor->GetComponents<UStaticMeshComponent>(Meshes);

// Stack-allocated for small counts                                             Actor.h:228
TInlineComponentArray<UPrimitiveComponent*> Primitives(Actor);

// By tag (ComponentTags)                                                       Actor.h:3835, :3815
UActorComponent* Tagged = Actor->FindComponentByTag<UActorComponent>(TEXT("Weapon"));
TArray<UActorComponent*> AllTagged = Actor->GetComponentsByTag(UActorComponent::StaticClass(), TEXT("Weapon"));

// By interface: template argument is the I-class                               Actor.h:3852, :3818
IMyInteractable* Interactable = Actor->FindComponentByInterface<IMyInteractable>();
UActorComponent* Raw = Actor->FindComponentByInterface(UMyInteractable::StaticClass());

// Default subobject by name (constructor-created components)                   UObject/Object.h:208
UObject* Named = Actor->GetDefaultSubobjectByName(TEXT("Mesh"));
```

`K2_GetComponentsByClass` (`Actor.h:3807`) is the Blueprint node; the header says to use `GetComponents()` in C++.

## Mobility

`EComponentMobility::Type` (`Engine/EngineTypes.h`): `Static` (baked lighting, never moves), `Stationary` (baked shadows, lights may change color/intensity), `Movable` (fully dynamic). Set in the constructor with `SetMobility(EComponentMobility::Type NewMobility)` (`SceneComponent.h:1296`); a `Static` component cannot move at runtime. `Movable` components cast dynamic shadows, so keep anything that never moves `Static`.

## Actor pooling

`SpawnActor`/`Destroy` churn for projectiles or casings costs GC time. Park actors instead: hide, disable collision and tick; reuse by reversing. `AActor` has no `IsActive()`; use `IsHidden()` (`Actor.h:4523`) as the parked flag. The pool array must be a `UPROPERTY` or the GC collects the parked actors.

```cpp
// MyProjectilePool.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyProjectilePool.generated.h"

class AMyProjectile;

UCLASS()
class MYGAME_API AMyProjectilePool : public AActor
{
    GENERATED_BODY()
public:
    AMyProjectile* Acquire(const FTransform& SpawnTransform);
    void Release(AMyProjectile* Projectile);
protected:
    UPROPERTY(EditDefaultsOnly, Category="Pool")
    TSubclassOf<AMyProjectile> ProjectileClass;

    UPROPERTY(Transient)                          // UPROPERTY keeps pooled actors reachable for the GC
    TArray<TObjectPtr<AMyProjectile>> Pool;
};
```

```cpp
// MyProjectilePool.cpp
#include "MyProjectilePool.h"
#include "MyProjectile.h"
#include "Engine/World.h"

AMyProjectile* AMyProjectilePool::Acquire(const FTransform& SpawnTransform)
{
    for (AMyProjectile* Projectile : Pool)
    {
        if (Projectile && Projectile->IsHidden())    // parked
        {
            Projectile->SetActorTransform(SpawnTransform);
            Projectile->SetActorHiddenInGame(false);
            Projectile->SetActorEnableCollision(true);
            Projectile->SetActorTickEnabled(true);
            return Projectile;
        }
    }

    FActorSpawnParameters Params;
    Params.Owner = this;
    Params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    AMyProjectile* NewProjectile = GetWorld()->SpawnActor<AMyProjectile>(ProjectileClass, SpawnTransform, Params);
    if (NewProjectile) { Pool.Add(NewProjectile); }
    return NewProjectile;
}

void AMyProjectilePool::Release(AMyProjectile* Projectile)   // park: reverse of Acquire
{
    Projectile->SetActorTickEnabled(false);
    Projectile->SetActorEnableCollision(false);
    Projectile->SetActorHiddenInGame(true);
}
```
