# Collision Channel and Profile Setup

Target engine: **UE 5.8**. Configuring custom collision channels, object types and collision profiles, and the physics settings that live beside them in `Config/DefaultEngine.ini`.

---

## The channel table

`ECollisionChannel` (`Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h:1098`) is a fixed 64-slot enum. Eight slots are engine-visible, six are hidden engine trace channels (`ECC_EngineTraceChannel1..6`, one of which is `COLLISION_GIZMO`), and 50 are yours (`ECC_GameTraceChannel1..50`, max enforced at `CollisionProfileDetails.cpp:63`):

| Slot | Kind | Set by |
|---|---|---|
| `ECC_WorldStatic`, `ECC_WorldDynamic`, `ECC_Pawn`, `ECC_PhysicsBody`, `ECC_Vehicle`, `ECC_Destructible` | object channel | engine |
| `ECC_Visibility`, `ECC_Camera` | trace channel (`TraceQuery="1"`) | engine |
| `ECC_GameTraceChannel1` … `ECC_GameTraceChannel50` | either, decided per project | `+DefaultChannelResponses=` |

A slot is a **trace channel** when its ini entry sets `bTraceType=True`, otherwise it is an **object channel**. The two are never interchangeable:

- Object channel — every component has exactly one (`SetCollisionObjectType`). `LineTraceSingleByObjectType`, `SweepSingleByObjectType`, `OverlapMultiByObjectType` and `FCollisionObjectQueryParams::AddObjectTypesToQuery` match against it.
- Trace channel — never an object type. `LineTraceSingleByChannel` and friends pass it in, and each candidate component's stored response to that channel (`ECR_Ignore` / `ECR_Overlap` / `ECR_Block`) decides the outcome.

Blueprint-facing code uses the parallel enums `ETraceTypeQuery` (`EngineTypes.h:1274`) and `EObjectTypeQuery` (`:1202`). Convert with `UEngineTypes::ConvertToCollisionChannel(ETraceTypeQuery)` (`:4066`), `ConvertToCollisionChannel(EObjectTypeQuery)` (`:4069`), `ConvertToObjectType(ECollisionChannel)` (`:4072`) and `ConvertToTraceType(ECollisionChannel)` (`:4075`). `TraceTypeQuery1`/`2` are `ECC_Visibility`/`ECC_Camera`; custom trace channels follow in slot order (`Collision/CollisionProfile.cpp:373-449`), so `TraceTypeQuery3` is the lowest-slot custom trace channel, not necessarily `ECC_GameTraceChannel1` — always convert rather than assuming the index.

---

## DefaultEngine.ini

Everything goes under `[/Script/Engine.CollisionProfile]`, which is `UCollisionProfile` (`Engine/Classes/Engine/CollisionProfile.h:160`, a `UDeveloperSettings`). Its config arrays are `Profiles` (:168), `DefaultChannelResponses` (:172), `EditProfiles` (:176), `ProfileRedirects` (:180) and `CollisionChannelRedirects` (:184).

### Declaring channels

```ini
[/Script/Engine.CollisionProfile]
; trace channels
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel1,DefaultResponse=ECR_Block,bTraceType=True,bStaticObject=False,Name="Weapon")
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel2,DefaultResponse=ECR_Ignore,bTraceType=True,bStaticObject=False,Name="Interaction")
; object channels
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel3,DefaultResponse=ECR_Block,bTraceType=False,bStaticObject=False,Name="Interactable")
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel4,DefaultResponse=ECR_Ignore,bTraceType=False,bStaticObject=True,Name="HazardZone")
```

`FCustomChannelSetup` fields (`CollisionProfile.h:100-118`): `Channel`, `DefaultResponse`, `bTraceType`, `bStaticObject`, `Name`. `DefaultResponse` is what every component responds with until a profile or C++ call overrides it. `bStaticObject=True` puts the channel in the "all static objects" query group.

### Declaring profiles

`FCollisionResponseTemplate` fields (`CollisionProfile.h:44-90`): `Name`, `CollisionEnabled`, `ObjectTypeName`, `CustomResponses` (array of `FResponseChannel`), `bCanModify`, and editor-only `HelpMessage`.

```ini
+Profiles=(Name="MyProjectile",CollisionEnabled=QueryAndPhysics,ObjectTypeName="WorldDynamic",CustomResponses=((Channel="WorldStatic",Response=ECR_Block),(Channel="WorldDynamic",Response=ECR_Block),(Channel="Pawn",Response=ECR_Block),(Channel="PhysicsBody",Response=ECR_Block),(Channel="Weapon",Response=ECR_Ignore),(Channel="Interaction",Response=ECR_Ignore)),HelpMessage="Projectiles")

+Profiles=(Name="MyTriggerPawn",CollisionEnabled=QueryOnly,ObjectTypeName="WorldDynamic",CustomResponses=((Channel="Pawn",Response=ECR_Overlap),(Channel="WorldStatic",Response=ECR_Ignore),(Channel="WorldDynamic",Response=ECR_Ignore),(Channel="PhysicsBody",Response=ECR_Ignore)),HelpMessage="Pawn-only trigger volume")

+Profiles=(Name="MyInteractable",CollisionEnabled=QueryAndPhysics,ObjectTypeName="Interactable",CustomResponses=((Channel="WorldStatic",Response=ECR_Block),(Channel="Pawn",Response=ECR_Block),(Channel="Interaction",Response=ECR_Block),(Channel="Weapon",Response=ECR_Ignore),(Channel="Visibility",Response=ECR_Block)),HelpMessage="Props the player can use")
```

Channel names inside `CustomResponses` are the display names (`"Pawn"`, `"Weapon"`), not the enum identifiers. `ObjectTypeName` must name an object channel; pointing it at a trace channel produces a profile that nothing can query by object type.

Channels not listed in `CustomResponses` fall back to that channel's `DefaultResponse`. Adding a new channel later therefore changes every existing profile that does not mention it — set the new channel's `DefaultResponse=ECR_Ignore` if you want to opt profiles in one at a time.

### Patching engine profiles

`+Profiles=` with an existing name replaces the whole template. To change only a few responses on a shipped profile, use `+EditProfiles=` (`FCustomProfile`, `CollisionProfile.h:142-153` — only `Name` and `CustomResponses`):

```ini
+EditProfiles=(Name="Pawn",CustomResponses=((Channel="Weapon",Response=ECR_Block)))
+EditProfiles=(Name="Ragdoll",CustomResponses=((Channel="Interaction",Response=ECR_Ignore)))
```

Engine profiles ship with `bCanModify=False` in `Engine/Config/BaseEngine.ini`; `EditProfiles` is the supported way to touch them.

### Renaming

`FRedirector` (`Engine/EngineTypes.h:4144`) has only `OldName` and `NewName`. These entries keep already-saved assets pointing at a renamed profile or channel instead of silently falling back to the default:

```ini
+ProfileRedirects=(OldName="Interactable",NewName="MyInteractable")
+CollisionChannelRedirects=(OldName="Interaction",NewName="Use")
```

---

## Engine profiles shipped in 5.8

From `Engine/Config/BaseEngine.ini:3103-3121`:

| Profile | CollisionEnabled | ObjectType | Note |
|---|---|---|---|
| `NoCollision` | `NoCollision` | WorldStatic | ignores Visibility and Camera |
| `BlockAll` | `QueryAndPhysics` | WorldStatic | blocks everything |
| `OverlapAll` | `QueryOnly` | WorldStatic | overlaps everything |
| `BlockAllDynamic` | `QueryAndPhysics` | WorldDynamic | movable version of BlockAll |
| `OverlapAllDynamic` | `QueryOnly` | WorldDynamic | movable trigger-style |
| `IgnoreOnlyPawn` | `QueryOnly` | WorldDynamic | ignores Pawn and Vehicle |
| `OverlapOnlyPawn` | `QueryOnly` | WorldDynamic | overlaps Pawn/Vehicle, ignores Camera |
| `Pawn` | `QueryAndPhysics` | Pawn | ignores Visibility |
| `Spectator` | `QueryOnly` | Pawn | blocks WorldStatic only |
| `CharacterMesh` | `QueryOnly` | Pawn | for the mesh under a character capsule |
| `PhysicsActor` | `QueryAndPhysics` | PhysicsBody | simulating props |
| `Destructible` | `QueryAndPhysics` | Destructible | |
| `InvisibleWall` / `InvisibleWallDynamic` | `QueryAndPhysics` | WorldStatic / WorldDynamic | ignores Visibility |
| `Trigger` | `QueryOnly` | WorldDynamic | overlaps all, ignores Visibility |
| `Ragdoll` | `QueryAndPhysics` | PhysicsBody | ignores Pawn and Visibility |
| `Vehicle` | `QueryAndPhysics` | Vehicle | |
| `UI` | `QueryOnly` | WorldDynamic | blocks Visibility, overlaps the rest |
| `WaterBodyCollision` | `QueryOnly` | (none) | overlaps dynamic/pawn, ignores Visibility and Camera |

`UCollisionProfile` also exposes these as `FName` constants: `NoCollision_ProfileName`, `BlockAll_ProfileName`, `OverlapAll_ProfileName`, `PhysicsActor_ProfileName`, `BlockAllDynamic_ProfileName`, `Pawn_ProfileName`, `Vehicle_ProfileName`, `DefaultProjectile_ProfileName` (`CollisionProfile.h:189-196`), plus `CustomCollisionProfileName` (:260) — the name a component reports once its responses no longer match any profile.

---

## ECollisionEnabled

`Engine/EngineTypes.h:1805`.

| Value | Query (traces) | Physics (contacts/forces) | Probe (contact data only) | Typical use |
|---|---|---|---|---|
| `NoCollision` | no | no | no | pure visual actors |
| `QueryOnly` | yes | no | no | triggers, sensors |
| `PhysicsOnly` | no | yes | no | invisible physics blockers |
| `QueryAndPhysics` | yes | yes | no | most gameplay objects |
| `ProbeOnly` | no | no | yes | contact reports without solving |
| `QueryAndProbe` | yes | no | yes | trace-visible contact probe |

`CollisionEnabledHasPhysics`, `CollisionEnabledHasQuery` and `CollisionEnabledHasProbe` (`EngineTypes.h:1848-1866`) are the helpers the engine uses; the combination matrix at `:1885` decides the effective mode for a pair of components.

---

## Applying it in C++

```cpp
#include "Components/PrimitiveComponent.h"
#include "Engine/CollisionProfile.h"

// One call sets ObjectType, CollisionEnabled and every channel response
MyMesh->SetCollisionProfileName(TEXT("MyProjectile"));
MyMesh->SetCollisionProfileName(UCollisionProfile::NoCollision_ProfileName);
const FName Current = MyMesh->GetCollisionProfileName();   // PrimitiveComponent.h:2040

// Per-channel overrides go AFTER the profile — the profile resets the container
MyMesh->SetCollisionResponseToChannel(ECC_Camera, ECR_Ignore);

// Whole-actor switch, restored to the previous per-component values when re-enabled
MyActor->SetActorEnableCollision(false);   // GameFramework/Actor.h:1926

// Re-read the ini table after a runtime config change
UCollisionProfile::Get()->LoadProfileConfig(/*bForceInit=*/true);   // CollisionProfile.h:237

// Resolve a profile name to the channel + responses a query would use
ECollisionChannel Channel;
FCollisionResponseParams ResponseParams;
UCollisionProfile::GetChannelAndResponseParams(TEXT("MyProjectile"), Channel, ResponseParams); // :213
```

Custom channels are referenced by their enum slot, never by name, in C++:

```cpp
// ECC_GameTraceChannel1 == "Weapon", ECC_GameTraceChannel3 == "Interactable" in the ini above
FHitResult Hit;
FCollisionQueryParams Params(TEXT("MyWeaponTrace"), false, this);

GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_GameTraceChannel1, Params);

FCollisionObjectQueryParams ObjectParams;
ObjectParams.AddObjectTypesToQuery(ECC_GameTraceChannel3);
GetWorld()->LineTraceSingleByObjectType(Hit, Start, End, ObjectParams, Params);
```

Wrapping the slots in a project header (`static constexpr ECollisionChannel MyGame_Weapon = ECC_GameTraceChannel1;`) keeps call sites readable and makes a later ini reshuffle a one-line change.

---

## Shape components

`UShapeComponent` (`Components/ShapeComponent.h:24`) is the `UPrimitiveComponent` base for the three trigger primitives. `ShapeColor` (:44) and `bDrawOnlyIfSelected` (:48) control the editor wireframe; `UpdateBodySetup()` (:131) rebuilds the body after a size change.

```cpp
MyBox->SetBoxExtent(FVector(50.f, 50.f, 100.f));      // Components/BoxComponent.h:39 (half-extents)
MySphere->SetSphereRadius(120.f);                      // Components/SphereComponent.h:33
MyCapsule->SetCapsuleSize(34.f, 88.f);                 // Components/CapsuleComponent.h:48 (radius, half-height)
```

Each takes a trailing `bool bUpdateOverlaps = true`; pass `false` when resizing several shapes in one frame and update once afterwards.

---

## Physics settings in DefaultEngine.ini

`[/Script/Engine.PhysicsSettings]` maps to `UPhysicsSettings` (`PhysicsEngine/PhysicsSettings.h:261`), which derives from `UPhysicsSettingsCore` (`PhysicsCore/Public/PhysicsSettingsCore.h`).

```ini
[/Script/Engine.PhysicsSettings]
DefaultGravityZ=-980.000000
BounceThresholdVelocity=200.000000
FrictionCombineMode=Average
RestitutionCombineMode=Average
MaxAngularVelocity=3600.000000
MaxDepenetrationVelocity=0.000000
ContactOffsetMultiplier=0.020000
MinContactOffset=2.000000
MaxContactOffset=8.000000
bSimulateSkeletalMeshOnDedicatedServer=True
DefaultShapeComplexity=CTF_UseSimpleAndComplex
bSubstepping=False
MaxSubstepDeltaTime=0.016667
MaxSubsteps=6
bTickPhysicsAsync=False
AsyncFixedTimeStepSize=0.033333
```

`DefaultShapeComplexity` takes an `ECollisionTraceFlag` (`PhysicsCore/Public/BodySetupEnums.h:13-20`): `CTF_UseDefault`, `CTF_UseSimpleAndComplex`, `CTF_UseSimpleAsComplex`, `CTF_UseComplexAsSimple`. `FrictionCombineMode` / `RestitutionCombineMode` take an `EFrictionCombineMode::Type`.

Physical surfaces live in the same section:

```ini
+PhysicalSurfaces=(Type=SurfaceType1,Name="Metal")
+PhysicalSurfaces=(Type=SurfaceType2,Name="Flesh")
```

---

## When collision is not behaving, check in this order

1. **Collision enabled** — `GetCollisionEnabled()` is not `NoCollision` on either component, and the pair's combination is not degraded to `NoCollision` (a `QueryOnly` component is invisible to a physics-only interaction, and vice versa).
2. **Responses are symmetric** — a block needs `ECR_Block` on both sides; an overlap needs both sides non-ignore with at least one `ECR_Overlap`.
3. **Profile ordering** — `SetCollisionProfileName` wipes manual responses. If the component reports `CustomCollisionProfileName`, something has overridden the profile after it was applied.
4. **Overlap events flag** — `SetGenerateOverlapEvents(true)` on **both** components; the field is private and cannot be assigned directly.
5. **Hit events flag** — `SetNotifyRigidBodyCollision(true)` on the simulating component, plus `bSimulatePhysics` for a non-zero `NormalImpulse`.
6. **Channel family** — a trace channel cannot be an object type, so `*ByObjectType` never matches it; `*ByChannel` accepts any channel and reads each component's response to it. Check `bTraceType` on the ini entry.
7. **Trace vs profile mismatch** — `*ByProfile` uses the profile's own object channel and responses, which may differ from the channel you expected.
8. **Complexity** — a body set to `CTF_UseComplexAsSimple` uses its triangle mesh for every query and can only collide as a static shape; it cannot simulate as a dynamic body (`BodySetupEnums.h:18`).
9. **Level streaming** — overlaps are not regenerated during streaming unless `bGenerateOverlapEventsDuringLevelStreaming` is set (`GameFramework/Actor.h:551`).
