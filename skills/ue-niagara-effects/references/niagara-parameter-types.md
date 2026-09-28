# Niagara Parameter Types — C++ to Niagara Type Mapping

Maps Niagara editor parameter types to their `UNiagaraComponent` setters and the native C++ types.
A parameter must live in the `User.` namespace in the Niagara editor to be settable from C++.

Sources (all under `Engine/Plugins/FX/Niagara/Source/Niagara/`): `Public/NiagaraComponent.h`,
`Public/NiagaraFunctionLibrary.h`, `Classes/NiagaraDataInterfaceArrayFunctionLibrary.h`,
`Public/NiagaraComponentPoolMethodEnum.h`.

---

## Scalar Types

| Niagara Editor Type | C++ Method | C++ Type |
|---|---|---|
| `float` | `SetVariableFloat(FName, float)` | `float` |
| `int32` | `SetVariableInt(FName, int32)` | `int32` |
| `bool` | `SetVariableBool(FName, bool)` | `bool` |

---

## Vector Types

| Niagara Editor Type | C++ Method | C++ Type | Notes |
|---|---|---|---|
| `Vector2D` | `SetVariableVec2(FName, FVector2D)` | `FVector2D` | |
| `Vector` | `SetVariableVec3(FName, FVector)` | `FVector` | |
| `Position` | `SetVariablePosition(FName, FVector)` | `FVector` | LWC-aware; use this, not `SetVariableVec3`, for world positions |
| `Vector4` | `SetVariableVec4(FName, const FVector4&)` | `FVector4` | |
| `Color` | `SetVariableLinearColor(FName, const FLinearColor&)` | `FLinearColor` | Do not use the Vec3/Vec4 setters for a Color parameter |
| `Quaternion` | `SetVariableQuat(FName, const FQuat&)` | `FQuat` | Stored internally as `FQuat4f` |
| `Matrix` | `SetVariableMatrix(FName, const FMatrix&)` | `FMatrix` | |

---

## Object / Reference Types

| Niagara Editor Type | C++ Method | Argument Type |
|---|---|---|
| `Object` | `SetVariableObject(FName, UObject*)` | `UObject*` |
| `Actor` | `SetVariableActor(FName, AActor*)` | `AActor*` |
| `Material` | `SetVariableMaterial(FName, UMaterialInterface*)` | `UMaterialInterface*` |
| `Static Mesh` | `SetVariableStaticMesh(FName, UStaticMesh*)` | `UStaticMesh*` |
| `Texture` | `SetVariableTexture(FName, UTexture*)` | `UTexture*` |
| `Texture Render Target` | `SetVariableTextureRenderTarget(FName, UTextureRenderTarget*)` | `UTextureRenderTarget*` |

---

## Getters

Every getter is `[[nodiscard]]`, takes the name plus a `bool& bIsValid` out parameter, and is `const`:
`GetVariableBool`, `GetVariableInt`, `GetVariableFloat`, `GetVariableVec2`, `GetVariableVec3`,
`GetVariableVec4`, `GetVariablePosition`, `GetVariableColor`, `GetVariableQuat`, `GetVariableMatrix`.

Note the asymmetry: the setter is `SetVariableLinearColor`, the getter is `GetVariableColor`.

```cpp
bool bIsValid = false;
const FLinearColor Tint = NiagaraComp->GetVariableColor(FName("User.TintColor"), bIsValid);
```

---

## Data Interface Binding

| Niagara DI Type | Binding Method | Argument |
|---|---|---|
| Skeletal Mesh | `UNiagaraFunctionLibrary::OverrideSystemUserVariableSkeletalMeshComponent` | `USkeletalMeshComponent*` |
| Static Mesh (component) | `UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMeshComponent` | `UStaticMeshComponent*` |
| Static Mesh (asset) | `UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh` | `UStaticMesh*` |
| Texture | `UNiagaraFunctionLibrary::SetTextureObject` | `UTexture*` |
| 2D Array Texture | `UNiagaraFunctionLibrary::SetTexture2DArrayObject` | `UTexture2DArray*` |
| Volume Texture | `UNiagaraFunctionLibrary::SetVolumeTextureObject` | `UVolumeTexture*` |
| Any DI (typed) | `UNiagaraFunctionLibrary::GetDataInterface<TDIType>(UNiagaraComponent*, FName)` | returns `TDIType*` |
| Any DI (untyped) | `UNiagaraFunctionLibrary::GetDataInterface(UClass*, UNiagaraComponent*, FName)` | returns `UNiagaraDataInterface*` |

The `Override*` and `Set*Object` helpers take the parameter name as `const FString&`; `GetDataInterface`
takes `FName`. Mixing the two up is the most common compile error in this area.

---

## Array DI Function Library

`UNiagaraDataInterfaceArrayFunctionLibrary` (`Classes/NiagaraDataInterfaceArrayFunctionLibrary.h`).
All entries take `(UNiagaraComponent* NiagaraSystem, FName OverrideName, ...)`.

| Element Type | Set Whole Array | Set Single Element | Get Whole Array | Get Single Element |
|---|---|---|---|---|
| `float` | `SetNiagaraArrayFloat` | `SetNiagaraArrayFloatValue` | `GetNiagaraArrayFloat` | `GetNiagaraArrayFloatValue` |
| `FVector2D` | `SetNiagaraArrayVector2D` | `SetNiagaraArrayVector2DValue` | `GetNiagaraArrayVector2D` | `GetNiagaraArrayVector2DValue` |
| `FVector` | `SetNiagaraArrayVector` | `SetNiagaraArrayVectorValue` | `GetNiagaraArrayVector` | `GetNiagaraArrayVectorValue` |
| `FVector` (Position) | `SetNiagaraArrayPosition` | `SetNiagaraArrayPositionValue` | `GetNiagaraArrayPosition` | `GetNiagaraArrayPositionValue` |
| `FVector4` | `SetNiagaraArrayVector4` | `SetNiagaraArrayVector4Value` | `GetNiagaraArrayVector4` | `GetNiagaraArrayVector4Value` |
| `FLinearColor` | `SetNiagaraArrayColor` | `SetNiagaraArrayColorValue` | `GetNiagaraArrayColor` | `GetNiagaraArrayColorValue` |
| `FQuat` | `SetNiagaraArrayQuat` | `SetNiagaraArrayQuatValue` | `GetNiagaraArrayQuat` | `GetNiagaraArrayQuatValue` |
| `FMatrix` | `SetNiagaraArrayMatrix` | `SetNiagaraArrayMatrixValue` | `GetNiagaraArrayMatrix` | `GetNiagaraArrayMatrixValue` |
| `int32` | `SetNiagaraArrayInt32` | `SetNiagaraArrayInt32Value` | `GetNiagaraArrayInt32` | `GetNiagaraArrayInt32Value` |
| `uint8` | `SetNiagaraArrayUInt8` | `SetNiagaraArrayUInt8Value` | `GetNiagaraArrayUInt8` | `GetNiagaraArrayUInt8Value` |
| `bool` | `SetNiagaraArrayBool` | `SetNiagaraArrayBoolValue` | `GetNiagaraArrayBool` | `GetNiagaraArrayBoolValue` |

Details that bite:

- The `UInt8` entries take and return `TArray<int32>` in the Blueprint-facing API, not `TArray<uint8>`.
- Single-element setters take `int Index` and a **non-defaulted** `bool bSizeToFit`; pass it explicitly.
  `bSizeToFit=true` grows the array to fit the index.
- All four Matrix entries take a trailing `bool bApplyLWCRebase = true`, which controls whether the
  large-world-coordinate tile offset is applied to the translation.

**Low-precision C++-only overloads.** Array DIs store `FVector3f` / `FVector2f` / `FQuat4f` /
`FMatrix44f` internally. When you already hold single-precision data, use the non-Blueprint
overloads to skip the conversion:

```cpp
// SetNiagaraArrayFloat(UNiagaraComponent*, FName, TConstArrayView<double>)
// SetNiagaraArrayInt32(UNiagaraComponent*, FName, TConstArrayView<int64>)
// SetNiagaraArrayVector2D(UNiagaraComponent*, FName, TConstArrayView<FVector2f>)
// SetNiagaraArrayVector(UNiagaraComponent*, FName, TConstArrayView<FVector3f>)
// SetNiagaraArrayVector4(UNiagaraComponent*, FName, TConstArrayView<FVector4f>)
// SetNiagaraArrayQuat(UNiagaraComponent*, FName, TConstArrayView<FQuat4f>)
// SetNiagaraArrayMatrix(UNiagaraComponent*, FName, TConstArrayView<FMatrix44f>)
// SetNiagaraArrayUInt8(UNiagaraComponent*, FName, TConstArrayView<uint8>)
```

---

## ENCPoolMethod Values

`Public/NiagaraComponentPoolMethodEnum.h`. Used as the `PoolingMethod` argument of
`SpawnSystemAtLocation` and `SpawnSystemAttached`.

| Value | Behavior |
|---|---|
| `None` | No pooling. The component is destroyed when the system finishes if `bAutoDestroy` is true. |
| `AutoRelease` | Allocated from the pool and returned automatically. Interaction after the spawning tick is unsafe, so do not cache the pointer. |
| `ManualRelease` | Allocated from the pool; you own it and must call `ReleaseToPool()`. Use for persistent effects whose parameters you keep updating. |
| `ManualRelease_OnComplete` | `UMETA(Hidden)` internal state: released manually but returned to the pool only on completion. |
| `FreeInPool` | `UMETA(Hidden)` internal state marking a component that is currently sitting in the pool. |

---

## Deprecated FString Setters

Every `const FString&` variant on `UNiagaraComponent` carries
`UE_DEPRECATED(5.3, "This method will be removed in a future release.  Please update to use the FName variant")`.
They are scheduled for removal, not merely slower. Do not emit them; use the `FName` setter in the
right-hand column.

| Do not emit | Use instead |
|---|---|
| `SetNiagaraVariableFloat(const FString&, float)` | `SetVariableFloat(FName, float)` |
| `SetNiagaraVariableInt(const FString&, int32)` | `SetVariableInt(FName, int32)` |
| `SetNiagaraVariableBool(const FString&, bool)` | `SetVariableBool(FName, bool)` |
| `SetNiagaraVariableVec2(const FString&, FVector2D)` | `SetVariableVec2(FName, FVector2D)` |
| `SetNiagaraVariableVec3(const FString&, FVector)` | `SetVariableVec3(FName, FVector)` |
| `SetNiagaraVariableVec4(const FString&, FVector4)` | `SetVariableVec4(FName, const FVector4&)` |
| `SetNiagaraVariableLinearColor(const FString&, FLinearColor)` | `SetVariableLinearColor(FName, const FLinearColor&)` |
| `SetNiagaraVariableQuat(const FString&, FQuat)` | `SetVariableQuat(FName, const FQuat&)` |
| `SetNiagaraVariableMatrix(const FString&, FMatrix)` | `SetVariableMatrix(FName, const FMatrix&)` |
| `SetNiagaraVariableObject(const FString&, UObject*)` | `SetVariableObject(FName, UObject*)` |
| `SetNiagaraVariableActor(const FString&, AActor*)` | `SetVariableActor(FName, AActor*)` |
| `SetNiagaraVariablePosition(const FString&, FVector)` | `SetVariablePosition(FName, FVector)` |

---

## Parameter Namespace Rules

Niagara parameters are `Namespace.VariableName`:

- `User.MyParam` — authored as User Exposed; the only namespace overridable from C++.
- `System.Age`, `System.DeltaTime`, `System.ExecutionState` — engine-owned, read-only.
- `Emitter.MyEmitterVar` — per-emitter simulation state; not reachable from C++.
- `Particle.Position`, `Particle.Velocity` — per-particle; not reachable from C++.
- `Module.MyModuleVar` — private to a module stack node; not reachable from C++.

Names are case-sensitive and must include the namespace prefix. `UNiagaraFunctionLibrary::GetAllUserParameters`
returns the authored `User.` parameters of a system as `FNiagaraUserParameterInfo` (`ParameterName`,
`ParameterType`, `TypeName`), which is the reliable way to confirm a name from code.
