---
name: ue-project-context
description: "Use when creating, refreshing or repairing `.agents/ue-project-context.md`, the project document every other UE skill reads first. Also use when the user says 'project context', 'set up context', 'UE context', 'scan my project', 'onboard the agent to my project', 'update project context', 'configure project', 'what engine version is this project on', 'which plugins do we have enabled', or complains that UE advice keeps coming out generic. For Build.cs and Target.cs mechanics, see ue-module-build-system; for naming and API-macro conventions, see ue-cpp-foundations; for GAS specifics, see ue-gameplay-abilities."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Project Context

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill produces and maintains `.agents/ue-project-context.md` — the single file the other UE skills read to learn the project's engine version, module layout, enabled plugins, coding conventions, gameplay framework classes, networking model and build targets. It reads project files only (`.uproject`, `*.Build.cs`, `*.Target.cs`, `Config/Default*.ini`, `Plugins/*/*.uplugin`); it calls no engine API. This is the one skill in this set that is deliberately a conversation: every line it writes must come from the codebase or from the user, never from a guess.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing — this skill is what creates it.

The default path is **scan first, then interview**. Build a draft from the codebase before asking anything, then let the user correct it. Do not open with a menu of options.

| Request is about… | Go to |
|---|---|
| First-time setup, or the file is missing | [Step 1: Scan and draft](#step-1-scan-and-draft) |
| What to read and what to pull out of each file | [What to scan](#what-to-scan) |
| Naming a plugin the project has enabled | [Plugin checklist](#plugin-checklist) |
| Reviewing the draft with the user | [Step 2: Correct the draft](#step-2-correct-the-draft) |
| Saving or refreshing an existing context file | [Step 3: Save and confirm](#step-3-save-and-confirm) |
| No C++ source to scan (Blueprint-only, or a new empty project) | [Fallback questionnaire](#fallback-questionnaire) |
| The shape of the output file | [Document template](#document-template) |

## Step 1: Scan and draft

1. Check whether `.agents/ue-project-context.md` exists. If it does, read it and treat it as the previous draft rather than starting over.
2. Scan the files in [What to scan](#what-to-scan). Fill every field you can prove from a file.
3. Mark anything the files cannot answer as `[unknown]` in the draft. Never invent a team size, a coding rule or a networking model.
4. Present the whole draft in one message, then move to [Step 2](#step-2-correct-the-draft).

A partial scan is still worth presenting. Engine version, module list and plugin list come out of the files almost entirely, which removes most of the interview.

### What to scan

| File | Pull out |
|---|---|
| `*.uproject` | `EngineAssociation` (e.g. `"EngineAssociation": "5.8"`; a GUID means a registered source build; empty means a native project inside an engine tree), `Modules[]` (`Name`, `Type`, `LoadingPhase`), `Plugins[]` (`Name`, `Enabled`), `TargetPlatforms[]` |
| `Source/*/*.Build.cs` | Module class name, `PublicDependencyModuleNames`, `PrivateDependencyModuleNames`, `PublicIncludePathModuleNames`, third-party include/library paths, any `PublicDefinitions` |
| `Source/*.Target.cs` | `Type` (`TargetType.Game`, `Editor`, `Client`, `Server`, `Program`), `ExtraModuleNames`, `DefaultBuildSettings`, `IncludeOrderVersion`, platform conditionals |
| `Config/DefaultEngine.ini` | `[/Script/EngineSettings.GameMapsSettings]` — `GameDefaultMap`, `ServerDefaultMap`, `GlobalDefaultGameMode`, `GameInstanceClass` (the module is `EngineSettings`, not `Engine`); `[/Script/Engine.RendererSettings]` for the rendering feature set |
| `Config/DefaultGame.ini` | Project display name and version; `[/Script/Engine.AssetManagerSettings]` — `PrimaryAssetTypesToScan`, which reveals the data-driven asset layout |
| `Config/DefaultInput.ini` | `[/Script/Engine.InputSettings]` — `DefaultPlayerInputClass` / `DefaultInputComponentClass` (`/Script/EnhancedInput.EnhancedPlayerInput` / `EnhancedInputComponent` means Enhanced Input is live; `InputSettings.h:219,223`), leftover legacy `ActionMappings`/`AxisMappings`; `[/Script/EnhancedInput.EnhancedInputDeveloperSettings]` for Enhanced Input project settings |
| `Config/DefaultGameplayTags.ini` | `[/Script/GameplayTags.GameplayTagsSettings]` — `GameplayTagList`, `GameplayTagTableList`, `ImportTagsFromConfig` |
| `Plugins/*/*.uplugin` | In-house plugin names, their `Modules[]` and their `Plugins[]` dependencies |
| `Source/*/Public/**` (spot-check 3–5 headers) | Naming prefixes in practice, `TObjectPtr` versus raw pointers, API macro style (`MYGAME_API` on each declaration, or a per-header `#define UE_API MYGAME_API`), `DEFINE_LOG_CATEGORY` names, `check`/`ensure` usage |

Also grep `Config/Default*.ini` for every other `[/Script/...]` section the project has customised — those sections name the systems the team actually configures.

### Plugin checklist

Cross-check `Plugins[]` in the `.uproject` against this list and record maturity wherever it changes the advice other skills give. These are `.uplugin` names in 5.8.

| Area | Plugins to look for |
|---|---|
| Abilities & modularity | `GameplayAbilities`, `GameFeatures` (Beta), `ModularGameplay` (Beta), `SmartObjects`, `InstancedActors` (Experimental) |
| Input | `EnhancedInput` |
| UI | `CommonUI`, `ModelViewViewModel` (Beta) |
| AI & simulation | `StateTree`, `GameplayStateTree`, `MassGameplay` (Experimental), `MassAI` (Experimental) |
| Animation & movement | `Mover` (Experimental), `MotionWarping` (Beta), `PoseSearch`, `Chooser`, `AnimationBudgetAllocator` |
| Camera | `GameplayCameras` (Experimental), `EngineCameras` |
| Networking | `Iris` (Beta), `ReplicationGraph` (Beta) |
| Audio | `Metasound`, `AudioModulation` |
| FX & procedural | `Niagara`, `PCG`, `ProceduralMeshComponent`, `GeometryScripting` |
| Data & services | `DataRegistry` (Beta), `OnlineSubsystem`, `OnlineServices`, `SignificanceManager` |
| Streaming & messaging | `LevelStreamingPersistence` (Experimental), `AsyncMessageSystem` (Experimental) |

Record the `.uplugin` name, not the friendly display name, so other skills can match it. A project plugin under `Plugins/` that is absent from `.uproject` `Plugins[]` is still enabled unless its `.uplugin` says `"EnabledByDefault": false` (`FPlugin::IsEnabledByDefault`, `Projects/Private/PluginManager.cpp:421-435`); an engine plugin absent from `Plugins[]` is enabled only if its `.uplugin` has `"EnabledByDefault": true`. Record which rule applied.

## Step 2: Correct the draft

Present the draft, then put two questions to the user:

> What is wrong in this draft, and what is missing?

Work through their answers, then re-present only the sections that changed. Repeat until the user says it is accurate.

Fill the `[unknown]` fields the same way, a few at a time, highest value first:

1. **Conventions that cannot be read off the code** — assertion policy, header organisation, rules enforced in review.
2. **Gameplay framework class names** — GameMode, GameState, PlayerController, PlayerState, Pawn/Character, GameInstance. Request the class name, not whether one exists.
3. **Networking model** — listen or dedicated server, classic replication or `Iris`, push model on or off, `ReplicationGraph` in use.
4. **Save system, streaming model, AI stack** — only the parts the files did not already prove.
5. **Team and source control** — skip entirely for a solo developer.

`[unknown]` and "not yet established" are valid final answers. A field the team has not decided is more useful written down as undecided than filled with a plausible invention.

## Step 3: Save and confirm

- Write the file to `.agents/ue-project-context.md`, creating `.agents/` if needed.
- Stamp the engine version and the date at the top.
- Tell the user the other UE skills now read this file automatically, and that re-running this skill refreshes it as the project changes.
- On a refresh, keep the sections the user did not revisit and update the date.

The skills in this repo target UE 5.8. Record the project's actual engine version verbatim from `EngineAssociation` so that advice can be adjusted when the project sits on a different version, and note whether it is a launcher build or a source build.

## Fallback questionnaire

Use this only when there is no `Source/` tree to scan (Blueprint-only project, or an empty project being planned). Work through one group at a time and confirm each before moving on.

**Engine and project.** Project name and a one-sentence description. Engine version, launcher or source build. Project type (game, simulation, visualisation, tool, plugin) and genre. Target platforms.

**Modules.** Module names, which is the primary game module, and each module's host type (`Runtime`, `Editor`, `Developer`, `CookedOnly`, `UncookedOnly`, `Program`). Key dependencies per module.

**Plugins.** Walk the [Plugin checklist](#plugin-checklist) by area rather than reciting names. Then in-house plugins under `Plugins/`, Fab/marketplace plugins critical to gameplay, and any licence restriction worth recording.

**Conventions.** Epic's `F`/`U`/`A`/`E`/`I` prefixes or a house variant. `TObjectPtr` for `UPROPERTY` object references, or raw pointers. Module API macro style — `MYGAME_API` on each declaration, or a per-header `#define UE_API MYGAME_API`. Log category names. Assertion policy (`check`, `checkf`, `ensure`, `ensureMsgf`, `verify`) and where each is allowed. Public/Private header layout. Any rule enforced in review.

**Gameplay framework.** Class names for GameMode, GameState, PlayerController, PlayerState, Pawn/Character, GameInstance. Subsystem classes in use and what each owns.

**GAS.** Whether `GameplayAbilities` is enabled; if so, the AbilitySystemComponent subclass, AttributeSet class names, the ability base class, and where the tag hierarchy is defined.

**Networking.** Listen server, dedicated server, or single-player. `Iris` or classic replication. Push model on or off. `ReplicationGraph` in use. Expected player count and tick budget.

**Input.** `EnhancedInput` or legacy mappings. Where `UInputMappingContext` and `UInputAction` assets live, and which class adds the contexts.

**UI.** UMG only, `CommonUI`, or `CommonUI` plus `ModelViewViewModel`. Base widget classes and how the HUD/menu stack is managed.

**AI.** Behavior Trees, `StateTree`/`GameplayStateTree`, Mass, or a custom stack. Navigation setup and whether `SmartObjects` is in use.

**Streaming.** World Partition with Data Layers, or classic sub-levels. Level-instancing approach.

**Saves.** `USaveGame` subclass names, slot naming, versioning approach.

**Build.** Which target types ship. Custom preprocessor defines. Third-party libraries and how they are integrated. Platform-specific code paths. Engine fork or modifications.

**Team.** Size and roles. Source control (Perforce, Git, Plastic). Branching and asset-lock policy. Review bar. Documentation home.

## Document template

Write `.agents/ue-project-context.md` in this shape. Drop any section the project genuinely has nothing to say about.

```markdown
# UE Project Context

*Engine: UE 5.8 · Last updated: [date]*

## Engine & Project
**Engine version:** [5.8 — launcher build / source build at [path]]
**Project name:** [name] · **Type:** [game / sim / tool] · **Genre:** [genre]
**Description:** [one sentence]
**Target platforms:** [list]

## Modules
**Primary game module:** [ModuleName]

| Module | Host type | Public deps | Private deps | Notes |
|---|---|---|---|---|
| [MyGame] | Runtime | Core, CoreUObject, Engine, InputCore | [list] | [purpose] |
| [MyGameEditor] | Editor | [list] | [list] | [purpose] |

## Plugins
| Plugin | Maturity | Used for |
|---|---|---|
| [EnhancedInput] | [stable] | [input] |
| [Mover] | [Experimental] | [locomotion prototype] |

**In-house plugins:** [Name — purpose]
**Fab / marketplace:** [Name — purpose, licence notes]

## Coding conventions
**Prefixes:** [standard F/U/A/E/I, exceptions]
**Object references:** [TObjectPtr in UPROPERTY / raw pointers]
**API macro style:** [MYGAME_API per declaration / per-header #define UE_API MYGAME_API]
**Log categories:** `[LogMyGame]` — [scope]
**Assertions:** [check where / ensure where / verify where]
**Header layout:** [Public/Private per module / flat]
**Enforced rules:** [list]

## Gameplay framework
| Role | Class |
|---|---|
| GameMode | `[AMyGameMode]` |
| GameState | `[AMyGameState]` |
| PlayerController | `[AMyPlayerController]` |
| PlayerState | `[AMyPlayerState]` |
| Pawn / Character | `[AMyCharacter]` |
| GameInstance | `[UMyGameInstance]` |

**Subsystems:** `[UMyClass]` ([UGameInstanceSubsystem / UWorldSubsystem / ULocalPlayerSubsystem]) — [owns]

## GAS
**Enabled:** [yes / no]
**AbilitySystemComponent:** `[UMyAbilitySystemComponent]` — lives on [PlayerState / Pawn]
**AttributeSets:** `[UMyAttributeSet]` — [attributes]
**Ability base:** `[UMyGameplayAbility]`
**Tag source:** [Config/DefaultGameplayTags.ini / native tags in [file]]

## Networking
**Model:** [single-player / listen server / dedicated server]
**Replication stack:** [classic / Iris (Beta)]
**Push model:** [on / off] · **ReplicationGraph:** [yes / no]
**Budget:** [N players, tick rate]

## Input
**Stack:** [EnhancedInput / legacy]
**Mapping contexts:** [asset paths] · **Added by:** `[class]`

## UI
**Stack:** [UMG / CommonUI / CommonUI + ModelViewViewModel]
**Base widgets:** `[UMyUserWidget]` · **Layer / stack management:** [description]

## AI
**Stack:** [Behavior Trees / StateTree / GameplayStateTree / Mass / custom]
**Navigation:** [NavMesh setup] · **SmartObjects:** [yes / no]

## Streaming
**Model:** [World Partition + Data Layers / sub-levels]
**Notes:** [cell size, streaming sources, level instances]

## Saves
**SaveGame classes:** `[UMySaveGame]` · **Slots:** [naming] · **Versioning:** [approach]

## Build
**Targets:** [Game, Editor, Client, Server]
**Defines:** `[MYGAME_WITH_CHEATS]` — [purpose]
**Third-party:** [Library — binary / source]
**Platform notes:** [platform: constraint]
**Engine modifications:** [none / fork at [repo] — [what changed]]

## Team
**Size & roles:** [N engineers, N designers, N artists]
**Source control:** [Perforce / Git / Plastic] · **Branching:** [strategy]
**Review bar:** [description] · **Docs:** [where]
```

## Deprecated — do not use

Old forms an agent is likely to write into a context document, or into the code it generates from one.

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `GENERATED_UCLASS_BODY()` | `GENERATED_BODY()` | `#define GENERATED_UCLASS_BODY(...) GENERATED_BODY_LEGACY()` in `CoreUObject/Public/UObject/ObjectMacros.h:803` |
| `UPROPERTY() UObject* Ptr;` | `UPROPERTY() TObjectPtr<UObject> Ptr;` | `struct TObjectPtr` in `CoreUObject/Public/UObject/ObjectPtr.h:519` |
| `NetUpdateFrequency = 10.f;` | `SetNetUpdateFrequency(10.f)` / `GetNetUpdateFrequency()` | `UE_DEPRECATED(5.5)` in `Engine/Classes/GameFramework/Actor.h:903` |
| `MinNetUpdateFrequency = 2.f;` | `SetMinNetUpdateFrequency(2.f)` / `GetMinNetUpdateFrequency()` | `UE_DEPRECATED(5.5)` in `Engine/Classes/GameFramework/Actor.h:908` |
| `UDataLayerSubsystem` accessors | the `UDataLayerManager` equivalents | `UE_DEPRECATED(5.3)` in `Engine/Public/WorldPartition/DataLayer/DataLayerSubsystem.h:37` |
| `bEnableDynamicComponentInputBinding` in `DefaultInput.ini` | drop the key | `UE_DEPRECATED(5.7)` in `Engine/Classes/GameFramework/InputSettings.h:96` |
| `SpeechMappings` in `DefaultInput.ini` | drop the key | `UE_DEPRECATED(5.7)` in `Engine/Classes/GameFramework/InputSettings.h:211` |
| `[/Script/Engine.GameMapsSettings]` | `[/Script/EngineSettings.GameMapsSettings]` | `class UGameMapsSettings` in `Runtime/EngineSettings/Classes/GameMapsSettings.h:101` |
| `MetaSounds` as a plugin name | `Metasound` | `Engine/Plugins/Runtime/Metasound/Metasound.uplugin` |

## Common Mistakes

**Opening with a questionnaire instead of a draft:** the `.uproject`, the `*.Build.cs` files and `Config/Default*.ini` answer most of the document. Scan, draft, then find out what is wrong — do not make the user dictate what is already on disk.

**Guessing to fill a gap:** a fabricated team size or assertion policy is worse than a blank, because every other skill then acts on it. Write `[unknown]` and move on.

**Writing friendly plugin names:** record the `.uplugin` name (`ModelViewViewModel`, `Metasound`, `GameplayAbilities`), not the display name, so other skills can match it against `.uproject` `Plugins[]`.

**Misjudging plugin state:** a plugin is active if its `.uproject` `Plugins[]` entry has `"Enabled": true`, or if it is enabled by default and not disabled there — and a project plugin under `Plugins/` with no `EnabledByDefault` key counts as enabled by default.

**Putting `GameMapsSettings` under the `Engine` module:** the class lives in the `EngineSettings` module, so the section is `[/Script/EngineSettings.GameMapsSettings]`. The wrong section silently does nothing.

**Omitting maturity:** an Experimental or Beta plugin changes what other skills should recommend. `Mover` (Experimental), `Iris` (Beta) and `GameplayCameras` (Experimental) each need the label next to the name.

**Dropping module dependencies:** most UE link errors trace to a missing entry in `PublicDependencyModuleNames`/`PrivateDependencyModuleNames`. Capture both lists per module verbatim.

**Overwriting on a refresh:** on a re-run, merge into the existing file. Sections the user did not revisit keep their content.

## Related Skills

Skills that read `.agents/ue-project-context.md`, and what each takes from it:

- `ue-module-build-system` — module list, host types, Build.cs dependency lists, target types
- `ue-cpp-foundations` — naming prefixes, `TObjectPtr` policy, API macro style, assertion policy
- `ue-gameplay-framework` — GameMode, GameState, PlayerController, PlayerState, Pawn classes
- `ue-gameplay-abilities` — GAS setup, AbilitySystemComponent owner, AttributeSet names
- `ue-gameplay-tags-messaging` — tag source (`DefaultGameplayTags.ini` or native tags) and messaging stack
- `ue-networking-replication` — networking model, Iris versus classic, push model, ReplicationGraph
- `ue-input-system` — Enhanced Input status, mapping context locations, PlayerController class
- `ue-ui-umg-slate` — CommonUI / ModelViewViewModel status and base widget classes
- `ue-blueprint-cpp-interop` — API macro style and which systems are Blueprint-facing
- `ue-actor-component-architecture` — subsystem list and component conventions
- `ue-character-movement` — character class and movement stack
- `ue-mover` — whether the `Mover` plugin (Experimental) is enabled
- `ue-gameplay-cameras` — whether `GameplayCameras` (Experimental) is enabled
- `ue-ai-navigation` — AI stack, navigation setup, SmartObjects status
- `ue-state-trees` — StateTree / GameplayStateTree plugin status
- `ue-world-level-streaming` — World Partition versus sub-levels, Data Layer usage
- `ue-serialization-savegames` — SaveGame classes, slot naming, versioning
- `ue-testing-debugging` — log categories, assertion policy, module list
- `ue-editor-tools` — Editor module names and editor plugin list
- `ue-game-features` — GameFeatures / ModularGameplay plugin status
