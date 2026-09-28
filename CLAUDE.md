# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Project KORA** (`KORAProject`) — a third-person souls-like action RPG built on **Unreal Engine 5.6**, derived from Epic's **Lyra Starter Game**. The Lyra Experience / GameFeature / Modular Gameplay / GAS scaffolding is retained; the shooter gameplay was replaced with stamina-based melee/ranged combat, parry/poise, and RPG item/equipment systems. `README.md` is an extensive (Korean) design + architecture doc — consult it for system deep-dives.

## Build & Run

There is no CLI build script. Standard UE workflow:

- **Generate project files**: right-click `KORAProject.uproject` → *Generate Visual Studio project files* (or run the engine's `GenerateProjectFiles`). Produces `KORAProject.sln`.
- **Build**: open `KORAProject.sln` in **Visual Studio 2022**, select `Development Editor | Win64`, build. Or via UBT:
  `<UE_5.6>/Engine/Build/BatchFiles/Build.bat KORAProjectEditor Win64 Development -Project="<abs>/KORAProject.uproject"`
- **Run**: F5 from VS, or launch `KORAProject.uproject` with UnrealEditor. `EditorStartupMap` / `GameDefaultMap` = `/Game/Levels/Barion/Maps/Tutorial_Map`.
- **RHI requirement**: DX12 / SM6 is required (Nanite). Do not switch the project to DX11/Vulkan.

### Tests

No automation test suite. `Source/KORAProject/Test/` contains manual in-editor test actors (`KRAerialComboTestActor`, `KRStarDashTestActor`) placed in a level to exercise specific abilities. Verify changes by PIE in `Tutorial_Map`.

## Modules & Plugins

Two first-party C++ modules (`Source/`):

| Module | Type / phase | Purpose |
|---|---|---|
| `KORAProject` | Runtime / Default | All gameplay code. |
| `KORACustomEditors` | Editor / PreDefault | In-editor tooling: **CSV→DataTable converter** (`UCSVToDataTableToolSubsystem::ConvertAllCSVsToDataTables`), a Slate "Construct Monster" window, directory watchers. |

In-repo plugins under `Plugins/`: Lyra-copied (`CommonGame`, `CommonUser`, `GameFeatures/DefaultFeature`, `GameplayMessageRouter`, `ModularGameplayActors`), plus `KRUtilManager` (editor asset-action utilities, used by `KORACustomEditors`), `WorldScannerPlugin` (module name `SphereReveal`), and `KawaiiPhysics`.

## Content & Git

- **Git LFS** tracks `*.uasset *.umap *.png *.jpg *.jpeg *.tga *.exr *.mp4`.
- Large art trees are **gitignored** and not in the repo: `Content/Levels/`, `Content/Megascans/`, `Content/MSPresets/`, `Content/VFX_SFX/`, `Content/MetaHumans/`, `Content/External/`, external actor/object folders. A fresh clone will have missing level/VFX/MetaHuman references.
- `Content/Data/DataTables/*` is **gitignored** — DataTables are generated locally from CSV source via the `KORACustomEditors` CSV→DataTable tool. Don't hand-author or commit `.uasset` DataTables there.
- Commit messages are Korean, prefixed `[feature]` / `[fix]` / `[refactor]`. Work branches are `feature/*`; integration branch is `DEV`, then `main`.

## Architecture

### Bootstrapping (Lyra-derived)

`AKRGameMode` resolves an **`UKRExperienceDefinition`** (via `UKRExperienceManagerComponent`), which lists GameFeatures to activate, `ActionSets`, and a **`UKRPawnData`**. `UKRPawnData` binds a pawn class + `UKRAbilitySet[]` + `UKRAbilityTagRelationshipMapping` + input config + default camera mode + default inventory/equip item tags. `UKRAbilitySet` grants abilities, effects, and attribute sets as a unit. `KRFrontendStateComponent` / `KRUserFacingExperience` drive the main-menu → game flow.

### GameInstance subsystems (`Source/KORAProject/SubSystem/`)

~13 `UGameInstanceSubsystem`s are the backbone for cross-cutting state: `KRInventorySubsystem`, `KRQuestSubsystem`, `KRShopSubsystem`, `KRDataTablesSubsystem`, `KREffectSubsystem`, `KRSoundSubsystem`, `KRUIRouterSubsystem`, `KRUIInputSubsystem`, `KRMapTravelSubsystem`, `KRLoadingSubsystem`, `KRCitizenStreamingSubsystem`, `KRDataAssetRegistry`. Prefer routing new global state/services through a subsystem rather than the GameMode/GameState.

### Characters & components

`AKRBaseCharacter` (implements `IAbilitySystemInterface`) + `UKRPawnExtensionComponent` (coordinates ASC/PawnData init) + `UKRCombatComponent` + `UKREquipmentManagerComponent`.
- `AKRHeroCharacter` adds `UKRStaminaComponent`, `UKRCoreDriveComponent` (ultimate gauge), `UKRGuardRegainComponent` (grey/regain HP), `UMotionWarpingComponent`.
- `AKREnemyPawn` adds `UStateTreeAIComponent` + `UKREnemyAttributeSet` (poise). Citizens/crowd NPCs use **Mass Entity** (`Characters/Citizen/` fragments/processors/traits, `DefaultMass.ini`).

**GAS init is lazy** to avoid a PlayerState/Pawn `BeginPlay` race: resolve the ASC at first use and cache it, never assume it's valid in `BeginPlay`.

### GAS

`UKRGameplayAbility` is the ability base: `EKRAbilityActivationPolicy` (`OnInputTriggered` / `WhileInputActive` / `OnSpawn`) plus `GetKR*FromActorInfo()` helpers. Abilities live in `GAS/Abilities/HeroAbilities/` and `GAS/Abilities/EnemyAbility/`; also `AttributeSets/`, `ExecCalc/` (damage), `GameplayCues/`, `AbilitySet/`. Input tag → ability activation is wired through `UKRAbilitySystemComponent` + the tag-relationship mapping.

### Gameplay tags

Native tags are declared per-domain in `Source/KORAProject/GameplayTag/` (`KRAbilityTag`, `KRCombatTag`, `KRInputTag`, `KRStateTag`, `KRUITag`, `KRSoundTag`, `KREnemyTag`, …) using `UE_DECLARE_GAMEPLAY_TAG_EXTERN` with `KRTAG_<DOMAIN>_<NAME>` macro names. Data-defined tags are in `Config/DefaultGameplayTags.ini`. Add new native tags to the matching domain file rather than a new one.

### AI & Quests — both StateTree

- **AI**: `Source/KORAProject/AI/StateTree/` with `Task/`, `Evaluator/`, `Condition/`, `PropertyFunctions/`. `KRAIEnemyController` runs the StateTree; integrates AI Perception + EQS.
- **Quests**: `Source/KORAProject/Quest/` also StateTree-driven, with modular `Condition/` checkers (kill / collect / talk / reach-location / …). Driven by `KRQuestSubsystem`.

### UI (CommonUI)

`Source/KORAProject/UI/` on **CommonUI** with a layer-stack `PrimaryLayout`. `KRUIRouterSubsystem` pushes/pops screens; `KRUIInputSubsystem` handles per-device / per-widget input rebinding and gameplay↔menu input-mode switching. Widgets consume UI-only structs (`FKRItemUIData`) via adapter libraries rather than reading gameplay objects directly.

### Cross-system communication

Decoupled events go through **`GameplayMessageRouter`** (`UGameplayMessageSubsystem` broadcast/listen) instead of direct references between subsystems/components.

## Conventions

- **Prefix everything `KR`**: `AKR*` actors, `UKR*` objects/components, `FKR*` structs, `EKR*` enums, `KRTAG_*` tags. New files follow the existing folder-by-domain layout under `Source/KORAProject/`.
- BlueprintCallable helper categories use the `KR|...` namespace (e.g. `Category = "KR|Ability"`).
- Explicit-or-shared PCHs (`PCHUsageMode.UseExplicitOrSharedPCHs`); include what you use.
