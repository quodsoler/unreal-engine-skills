# Unreal Engine Skills for AI Agents

A collection of 31 AI agent skills for Unreal Engine C++ development. Built for game developers who want AI coding agents to help write correct, production-quality UE5 C++ code. Works with Claude Code, Cursor, Windsurf, and any agent that supports the [Agent Skills spec](https://agentskills.io).

**Target engine: Unreal Engine 5.8.** Every API in every skill was checked against the UE 5.8 headers and source, and every C++ example compiles against 5.8. Where an API changed, each skill carries a "Deprecated — do not use" table mapping the old form to the 5.8 form, so an agent stops emitting the old one.

**Contributions welcome!** Found an inaccuracy or want to improve a skill? [Open a PR](#contributing).

## What are Skills?

Skills are markdown files that give AI agents specialized knowledge and workflows for specific tasks. When you add these to your project, your agent can recognize when you're working on an Unreal Engine task and apply the right patterns, APIs, and best practices.

## How Skills Work Together

Skills reference each other and build on shared context. The `ue-project-context` skill is the foundation — it captures your project's modules, target platforms, and conventions so other skills can give relevant advice.

```
                      ue-project-context
        (writes .agents/ue-project-context.md; every skill reads it)
                               │
   ┌──────────┬────────────┬───┴──────┬───────────┬──────────┬──────────┐
   ▼          ▼            ▼          ▼           ▼          ▼          ▼
 Core C++   Gameplay    Render/VFX   World      AI/Logic   UI/Input   Tools
 cpp-found  framework   materials    world-lvl  ai-nav     ui-umg     editor-tools
 actor-comp abilities   niagara      procedural state-tree input-sys  testing
 bp-interop tags-msg    audio        physics    mass-      
 module-    char-move   sequencer    serialize   entity
  build     mover       cameras      data-assets
 async      animation
            game-feat
            net-repl
```

See each skill's **Related Skills** section for the full dependency map.

## Available Skills

<!-- SKILLS:START -->
| Skill | Description |
|-------|-------------|
| [ue-actor-component-architecture](skills/ue-actor-component-architecture/) | Actor and component lifecycle — constructor vs BeginPlay, PostInitializeComponents, EndPlay, SpawnActor, attachment, ticking |
| [ue-ai-navigation](skills/ue-ai-navigation/) | AI controllers, behavior tree nodes, blackboards, perception, NavMesh pathfinding, EQS, Smart Objects |
| [ue-animation-system](skills/ue-animation-system/) | AnimInstance, thread-safe update, FAnimInstanceProxy, montages, notifies, blend spaces, curves, root motion |
| [ue-async-threading](skills/ue-async-threading/) | UE::Tasks, Async/AsyncTask, ParallelFor, FRunnable, timers, tickers, thread-safety rules |
| [ue-audio-system](skills/ue-audio-system/) | UAudioComponent, MetaSounds, submixes, attenuation, concurrency, modulation, Quartz |
| [ue-blueprint-cpp-interop](skills/ue-blueprint-cpp-interop/) | Exposing C++ to Blueprint — UFUNCTION/UPROPERTY meta keys, latent actions, async action nodes, interfaces |
| [ue-character-movement](skills/ue-character-movement/) | CharacterMovementComponent — movement modes, floor detection, root motion, network prediction, movement bases |
| [ue-cpp-foundations](skills/ue-cpp-foundations/) | UCLASS, UPROPERTY, UFUNCTION, containers, delegates, strings, GC and TObjectPtr, subsystems |
| [ue-data-assets-tables](skills/ue-data-assets-tables/) | DataAsset, DataTable, CurveTable, soft references, Asset Manager, streamable loading, cook rules |
| [ue-editor-tools](skills/ue-editor-tools/) | Detail customizations, editor utility widgets, UToolMenus, editor modes, asset definitions, editor subsystems |
| [ue-game-features](skills/ue-game-features/) | Game Feature plugins, GameFeatureAction, GameFrameworkComponentManager, init states |
| [ue-gameplay-abilities](skills/ue-gameplay-abilities/) | GAS — abilities, gameplay effects and components, attribute sets, cues, ability tasks |
| [ue-gameplay-cameras](skills/ue-gameplay-cameras/) | Spring-arm and first-person rigs, view-target blends, camera modifiers, shakes, Gameplay Camera System |
| [ue-gameplay-framework](skills/ue-gameplay-framework/) | GameMode, GameState, PlayerController, PlayerState, Pawn, HUD, damage, travel, sessions |
| [ue-gameplay-tags-messaging](skills/ue-gameplay-tags-messaging/) | Native and ini gameplay tags, containers, queries, tag replication, async message system |
| [ue-input-system](skills/ue-input-system/) | Enhanced Input — input actions, mapping contexts, triggers, modifiers, runtime key rebinding |
| [ue-mass-entity](skills/ue-mass-entity/) | Mass Entity ECS — processors, queries, fragments, tags, observers, traits, relations |
| [ue-materials-rendering](skills/ue-materials-rendering/) | Dynamic material instances, parameter collections, render targets, post process, Substrate, Nanite, MegaLights |
| [ue-module-build-system](skills/ue-module-build-system/) | Build.cs, Target.cs, .uproject and .uplugin, module and plugin creation, build error fixes |
| [ue-mover](skills/ue-mover/) | Mover plugin — movement modes, layered moves, modifiers, sync state, rollback networking (Experimental) |
| [ue-networking-replication](skills/ue-networking-replication/) | Property replication, RPCs, relevancy and dormancy, subobject lists, push model, Iris |
| [ue-niagara-effects](skills/ue-niagara-effects/) | Spawning Niagara systems, User parameters, data interfaces, data channels, sim caches, pooling |
| [ue-physics-collision](skills/ue-physics-collision/) | Collision channels and profiles, traces and overlaps, hit events, rigid bodies, constraints, Chaos |
| [ue-procedural-generation](skills/ue-procedural-generation/) | PCG graphs and custom nodes, procedural and dynamic meshes, instancing, splines, noise |
| [ue-project-context](skills/ue-project-context/) | Creates `.agents/ue-project-context.md`, the project document every other skill reads first |
| [ue-sequencer-cinematics](skills/ue-sequencer-cinematics/) | Level Sequence playback, bindings, cine cameras, camera rigs, Movie Render Graph |
| [ue-serialization-savegames](skills/ue-serialization-savegames/) | USaveGame, slot management, actor snapshots, FArchive, versioning, config persistence |
| [ue-state-trees](skills/ue-state-trees/) | State Tree tasks, conditions, evaluators, considerations, schemas, transitions, Mass integration |
| [ue-testing-debugging](skills/ue-testing-debugging/) | Automation and CQTest tests, logging and verbosity, assertions, Unreal Insights profiling, debug drawing |
| [ue-ui-umg-slate](skills/ue-ui-umg-slate/) | UMG user widgets, Slate, Common UI activatable widgets, MVVM view models, widget animation |
| [ue-world-level-streaming](skills/ue-world-level-streaming/) | World Partition, data layers, level streaming, level instances, HLOD, travel |
<!-- SKILLS:END -->

## Installation

### Option 1: CLI Install (Recommended)

Use [npx skills](https://github.com/vercel-labs/skills) to install skills directly:

```bash
# Install all skills
npx skills add quodsoler/unreal-engine-skills

# Install specific skills
npx skills add quodsoler/unreal-engine-skills --skill ue-cpp-foundations ue-gameplay-abilities

# List available skills
npx skills add quodsoler/unreal-engine-skills --list
```

This automatically installs to your `.agents/skills/` directory (and symlinks into `.claude/skills/` for Claude Code compatibility).

### Option 2: Clone and Copy

Clone the entire repo and copy the skills folder:

```bash
git clone https://github.com/quodsoler/unreal-engine-skills.git
cp -r unreal-engine-skills/skills/* .agents/skills/
```

### Option 3: Git Submodule

Add as a submodule for easy updates:

```bash
git submodule add https://github.com/quodsoler/unreal-engine-skills.git .agents/unreal-engine-skills
```

Then reference skills from `.agents/unreal-engine-skills/skills/`.

### Option 4: Fork and Customize

1. Fork this repository
2. Customize skills for your specific project
3. Clone your fork into your projects

### Option 5: SkillKit (Multi-Agent)

Use [SkillKit](https://github.com/rohitg00/skillkit) to install skills across multiple AI agents (Claude Code, Cursor, Copilot, etc.):

```bash
# Install all skills
npx skillkit install quodsoler/unreal-engine-skills

# Install specific skills
npx skillkit install quodsoler/unreal-engine-skills --skill ue-cpp-foundations ue-gameplay-abilities

# List available skills
npx skillkit install quodsoler/unreal-engine-skills --list
```

## Usage

Once installed, just ask your agent to help with Unreal Engine tasks:

```
"Add a replicated health attribute with GAS"
→ Uses ue-gameplay-abilities skill

"Set up World Partition streaming for my open world"
→ Uses ue-world-level-streaming skill

"Create a Niagara system driven by C++ parameters"
→ Uses ue-niagara-effects skill

"Write an automation test for my inventory system"
→ Uses ue-testing-debugging skill
```

You can also invoke skills directly:

```
/ue-cpp-foundations
/ue-gameplay-abilities
/ue-networking-replication
```

## Skill Categories

### Core C++
- `ue-cpp-foundations` — UCLASS, UPROPERTY, UFUNCTION, containers, delegates, strings, GC and TObjectPtr, subsystems
- `ue-actor-component-architecture` — Actor and component lifecycle — constructor vs BeginPlay, PostInitializeComponents, EndPlay, SpawnActor, attachment, ticking
- `ue-blueprint-cpp-interop` — Exposing C++ to Blueprint — UFUNCTION/UPROPERTY meta keys, latent actions, async action nodes, interfaces
- `ue-module-build-system` — Build.cs, Target.cs, .uproject and .uplugin, module and plugin creation, build error fixes
- `ue-async-threading` — UE::Tasks, Async/AsyncTask, ParallelFor, FRunnable, timers, tickers, thread-safety rules
- `ue-project-context` — Creates `.agents/ue-project-context.md`, the project document every other skill reads first

### Gameplay Systems
- `ue-gameplay-framework` — GameMode, GameState, PlayerController, PlayerState, Pawn, HUD, damage, travel, sessions
- `ue-gameplay-abilities` — GAS — abilities, gameplay effects and components, attribute sets, cues, ability tasks
- `ue-gameplay-tags-messaging` — Native and ini gameplay tags, containers, queries, tag replication, async message system
- `ue-character-movement` — CharacterMovementComponent — movement modes, floor detection, root motion, network prediction, movement bases
- `ue-mover` — Mover plugin — movement modes, layered moves, modifiers, sync state, rollback networking (Experimental)
- `ue-animation-system` — AnimInstance, thread-safe update, FAnimInstanceProxy, montages, notifies, blend spaces, curves, root motion
- `ue-game-features` — Game Feature plugins, GameFeatureAction, GameFrameworkComponentManager, init states
- `ue-networking-replication` — Property replication, RPCs, relevancy and dormancy, subobject lists, push model, Iris

### Rendering, VFX and Audio
- `ue-materials-rendering` — Dynamic material instances, parameter collections, render targets, post process, Substrate, Nanite, MegaLights
- `ue-niagara-effects` — Spawning Niagara systems, User parameters, data interfaces, data channels, sim caches, pooling
- `ue-audio-system` — UAudioComponent, MetaSounds, submixes, attenuation, concurrency, modulation, Quartz
- `ue-sequencer-cinematics` — Level Sequence playback, bindings, cine cameras, camera rigs, Movie Render Graph
- `ue-gameplay-cameras` — Spring-arm and first-person rigs, view-target blends, camera modifiers, shakes, Gameplay Camera System

### World and Data
- `ue-world-level-streaming` — World Partition, data layers, level streaming, level instances, HLOD, travel
- `ue-procedural-generation` — PCG graphs and custom nodes, procedural and dynamic meshes, instancing, splines, noise
- `ue-physics-collision` — Collision channels and profiles, traces and overlaps, hit events, rigid bodies, constraints, Chaos
- `ue-serialization-savegames` — USaveGame, slot management, actor snapshots, FArchive, versioning, config persistence
- `ue-data-assets-tables` — DataAsset, DataTable, CurveTable, soft references, Asset Manager, streamable loading, cook rules

### AI and Logic
- `ue-ai-navigation` — AI controllers, behavior tree nodes, blackboards, perception, NavMesh pathfinding, EQS, Smart Objects
- `ue-state-trees` — State Tree tasks, conditions, evaluators, considerations, schemas, transitions, Mass integration
- `ue-mass-entity` — Mass Entity ECS — processors, queries, fragments, tags, observers, traits, relations

### UI and Input
- `ue-ui-umg-slate` — UMG user widgets, Slate, Common UI activatable widgets, MVVM view models, widget animation
- `ue-input-system` — Enhanced Input — input actions, mapping contexts, triggers, modifiers, runtime key rebinding

### Tools and Testing
- `ue-editor-tools` — Detail customizations, editor utility widgets, UToolMenus, editor modes, asset definitions, editor subsystems
- `ue-testing-debugging` — Automation and CQTest tests, logging and verbosity, assertions, Unreal Insights profiling, debug drawing

## API Accuracy

Every skill was rewritten against the Unreal Engine 5.8 source. The method:

1. **Mechanical check.** Every UE-style identifier, member call, macro, `#include` path and Build.cs module name in every skill was matched against a symbol index built from all public headers under `Engine/Source` and `Engine/Plugins`. Anything that did not exist was fixed or removed.
2. **Signature check.** Every virtual a skill tells you to override, and every function it tells you to call, was read in the header and copied verbatim — constness, parameter order and defaults included.
3. **Deprecation sweep.** `UE_DEPRECATED(5.5)` through `(5.8)` was grepped in each module. Old forms a coding agent is likely to emit are listed in each skill's "Deprecated — do not use" table with the replacement named in the engine's own message.
4. **Behaviour audit.** Every behaviour claim (what runs where, when a pointer is valid, what a flag defaults to) was re-read against the engine `.cpp`, not just the header. Several were backwards and are fixed.
5. **Compile test.** Every C++ example (about 945 blocks, `.h`/`.cpp` pairs and Build.cs rules) was built in a UE 5.8 project, Editor and Game targets, non-unity, with warnings as errors. That caught missing includes, non-exported functions, editor-only APIs used in game code and module dependencies the text never named.

Corrections include APIs that never existed (`TOverloaded`, `FStateTreeActorContext`, `UAISense_Sight::ReportSightEvent`), wrong class prefixes (`AGameplayCueNotify_Static` is a `UObject`), enum values that do not compile (`EGameplayModOp::Multiplicative`), moved modules (MassEntity is a runtime module in 5.8, not a plugin), and a required override missing from every State Tree node template.

If you find an API call that doesn't match the engine source, please [open an issue](https://github.com/quodsoler/unreal-engine-skills/issues).

## Contributing

Found a way to improve a skill? Have a new skill to suggest? PRs and issues welcome!

### Guidelines

- `name` must match directory name exactly (lowercase, hyphens only)
- `description` must be quoted, start with "Use when", carry concrete trigger phrases, and name at least one sibling skill
- `metadata` must declare `version` and the `engine` a skill targets
- `SKILL.md` must be at most 500 lines (move details to `references/`, and link every reference from the body)
- Every API must be verified against the Unreal Engine source at the stated version before submitting; cite `header:line` in the PR

## License

[MIT](LICENSE) — Use these however you want.
