# Unreal Engine Asset Naming Conventions

> **Prerequisites:** Basic familiarity with the Unreal Editor, the Content Browser, the Outliner, Actors, Blueprints, and C++ classes.

## 1. Introduction

A consistent naming convention makes an Unreal Engine project easier to understand, navigate, maintain, and share with other developers.

Unreal Engine does not enforce a naming convention for assets. Epic publishes a recommended convention, and development teams establish their own conventions and ensure that everyone follows them. The Unreal Header Tool does, however, require the correct type prefixes on C++ classes (Section 4.1).

This document proposes a naming convention suitable for undergraduate game development projects. It is based on Epic's recommendations and mirrors the companion Unity conventions document, so names read the same in both engines wherever possible.

The objectives are to:

- Make asset types easy to identify.
- Give related assets consistent names.
- Improve project organisation.
- Make assets easier to find and search for.
- Support collaboration between programmers, artists, and designers.

For additional guidance, see [Epic's recommended asset naming conventions](https://dev.epicgames.com/documentation/unreal-engine/recommended-asset-naming-conventions-in-unreal-engine-projects).

## 2. General Naming Rules

Follow these rules when naming assets in an Unreal Engine project.

1. **Use PascalCase** for asset names, with each word beginning with a capital letter.
2. **Avoid spaces** and special characters in asset and folder names.
3. **Use descriptive names** that communicate an asset's purpose.
4. **Use consistent prefixes** to identify asset types.
5. **Use consistent suffixes** to distinguish asset variants or provide additional information.
6. **Avoid meaningless names**, such as `NewBlueprint`, `Test2`, or `Asset15`.
7. **Avoid revision names**, such as `PlayerFinal`, `PlayerFinal2`, or `PlayerFinalReallyFinal`. Use version control to manage revisions.
8. **Keep names short.** Long asset names inside deep folders can exceed Windows path length limits when the project is packaged.

### Examples

| Avoid | Prefer |
|:--|:--|
| `NewMaterial` | `MI_Env_StoneWall_Opaque` |
| `model3` | `SM_StoneWall` |
| `texture final` | `T_StoneWall_Albedo` |
| `player final 2` | `BP_Player` |
| `NewMap` | `Lvl_MainMenu` |
| `PlayerScript.h` | `PlayerCharacter.h` |

## 3. Recommended Asset Naming Convention

Use the following general format, which follows Epic's recommendation:

`[Prefix]_[AssetName]_[Descriptor]_[Variant]`

Each element has a specific purpose.

| Element | Purpose | Example |
|:--|:--|:--|
| Prefix | Identifies the asset type. | `T` |
| Asset name | Identifies the asset or related asset group. | `StoneWall` |
| Descriptor | Provides additional information about the asset. | `Normal` |
| Variant | Distinguishes between related alternatives. | `01` |

For example:

`T_StoneWall_Normal`

This identifies a normal map texture associated with a stone wall.

Not every asset needs every element. Use the elements that make the name clearer.

Two asset groups use a different pattern:

- **C++ classes** follow Epic's C++ coding standard, with type prefixes but no underscores (Section 4.1).
- **Materials** are named by what they make and how they blend (Section 6).

**Important:** Apart from the C++ class prefixes, these prefixes are conventions, not requirements imposed by Unreal Engine. Consistency within a project is more important than adopting a particular set of abbreviations.

## 4. Asset Naming Reference

The following table provides a recommended convention for common Unreal Engine assets. Prefixes marked with an asterisk are not in Epic's published list and are a project choice.

| Asset type | Prefix | Example |
|:--|:--|:--|
| Level (map) | `Lvl_` * | `Lvl_MainMenu` |
| C++ class | `A`, `U`, `F`, `E`, `I` | `APlayerCharacter` |
| Blueprint | `BP_` | `BP_EnemyGoblin` |
| Blueprint Actor Component | `AC_` | `AC_Health` |
| Blueprint Interface | `BI_` | `BI_Interactable` |
| Widget Blueprint | `WBP_` | `WBP_HealthBar` |
| Animation Blueprint | `ABP_` | `ABP_Player` |
| Enum (Blueprint) | `E_` | `E_WeaponType` |
| Structure (Blueprint) | `F_` | `F_WeaponStats` |
| Data Asset | `DA_` * | `DA_Weapon_Pistol` |
| Data Table | `DT_` | `DT_Weapons` |
| Curve Table | `CT_` | `CT_DamageFalloff` |
| Static Mesh | `SM_` | `SM_StoneWall` |
| Skeletal Mesh | `SK_` | `SK_Player` |
| Skeleton | `SKEL_` | `SKEL_Player` |
| Physics Asset | `PHYS_` | `PHYS_Player` |
| Material | `MM_` * | `MM_Env_StoneWall_Opaque` |
| Material Instance | `MI_` | `MI_Env_MossyStone_Opaque` |
| Material Function | `MF_` * | `MF_UV_Triplanar` |
| Material Parameter Collection | `MPC_` * | `MPC_Weather` |
| Post Process Material | `PPM_` | `PPM_Outline` |
| Texture | `T_` | `T_StoneWall_Albedo` |
| Normal map | `T_` | `T_StoneWall_Normal` |
| Animation Sequence | `AS_` | `AS_Player_Walk` |
| Animation Montage | `AM_` | `AM_Player_Attack` |
| Blend Space | `BS_` | `BS_Player_Locomotion` |
| Niagara System | `FXS_` | `FXS_Smoke` |
| Niagara Emitter | `FXE_` | `FXE_SmokePuff` |
| Niagara Function | `FXF_` | `FXF_WindForce` |
| Sound effect (Sound Wave) | `SFX_` * | `SFX_DoorOpen` |
| Music (Sound Wave) | `MUS_` * | `MUS_MainTheme` |
| Sound Cue | `SC_` * | `SC_Footsteps` |
| MetaSound Source | `MSS_` * | `MSS_EngineLoop` |
| Physics Material | `PM_` | `PM_Ice` |
| Font | `FNT_` * | `FNT_Heading` |
| Level Sequence | `LS_` | `LS_IntroSequence` |
| Render Target | `RT_` * | `RT_SecurityCamera` |

These prefixes are a proposed project standard. Other naming conventions are equally valid if they are applied consistently.

### 4.1 C++ Classes

C++ classes follow Epic's C++ coding standard rather than the asset prefix convention. The Unreal Header Tool requires the correct prefix on reflected types, so these are not optional.

| Prefix | Used for | Example |
|:--|:--|:--|
| `A` | Classes derived from `AActor` | `APlayerCharacter`, `ADoorTrigger` |
| `U` | Classes derived from `UObject`, including components | `UHealthComponent` |
| `F` | Structs and most other non-UObject types | `FWeaponStats` |
| `E` | Enums | `EWeaponType` |
| `I` | Interfaces | `IInteractable` |
| `T` | Templates | `TArray` |

The filename omits the prefix. `APlayerCharacter` lives in `PlayerCharacter.h` and `PlayerCharacter.cpp`.

```cpp
UCLASS()
class APlayerCharacter : public ACharacter
{
    GENERATED_BODY()
};
```

Avoid names such as:

- `playercharacter.h`
- `player_character.h`
- `MyActor.h`
- `PlayerScript.h`

Name a class by its role. Avoid generic suffixes such as `Manager`, `Handler`, and `Helper` unless the class genuinely coordinates other objects. Prefer a subsystem over a hand-rolled singleton manager.

| Purpose | C++ pattern | Blueprint pattern | Example |
|:--|:--|:--|:--|
| Controllable entity | `A<Entity>Character` / `A<Entity>Pawn` | `BP_<Entity>` | `APlayerCharacter`, `BP_Player` |
| Single responsibility component | `U<Concern>Component` | `AC_<Concern>` | `UHealthComponent`, `AC_Health` |
| Trigger or interaction volume | `A<Purpose>Volume` / `A<Purpose>Trigger` | `BP_<Purpose>Trigger` | `ADamageVolume`, `BP_CheckpointTrigger` |
| Game rules | `A<Project>GameMode` | `BP_<Project>GameMode` | `AArenaGameMode` |
| Player input and camera | `A<Project>PlayerController` | `BP_<Project>PlayerController` | `AArenaPlayerController` |
| Scene-wide or game-wide service | `U<Domain>Subsystem` | - | `UAudioSubsystem`, `USaveSubsystem` |
| UI element | `U<Element>Widget` | `WBP_<Element>` | `UHealthBarWidget`, `WBP_HealthBar` |
| Interface | `I<Ability>` (`U<Ability>` stub) | `BI_<Ability>` | `IInteractable`, `BI_Interactable` |

When a Blueprint extends a C++ class, give the Blueprint the `BP_` prefix and the same base name, or a more specific one. For example, `BP_EnemyGoblin` extends `AEnemyCharacter`.

For additional guidance, see [Epic's C++ coding standard](https://dev.epicgames.com/documentation/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine).

### 4.2 Class Members

Epic's coding standard uses PascalCase for all identifiers, including variables. This differs from the camelCase fields used in Unity.

| Item | Convention | Example |
|:--|:--|:--|
| Member variable | PascalCase | `float MoveSpeed;` |
| Boolean | `b` prefix, PascalCase | `bIsGrounded`, `bHasKey`, `bCanJump` |
| Function | PascalCase, verb first | `ApplyDamage()`, `SpawnEnemy()` |
| Delegate type | `F<Event>Signature` / `F<Event>Delegate` | `FOnHealthChangedSignature` |
| Delegate member | `On<Event>` | `OnHealthChanged` |
| Constant | PascalCase | `static constexpr int32 MaxLives = 3;` |
| Object pointer member | `TObjectPtr<>` | `TObjectPtr<UHealthComponent> HealthComponent;` |

Expose properties to the editor through `UPROPERTY` specifiers rather than making them public. Always give exposed properties a `Category`.

```cpp
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Movement")
float MoveSpeed = 600.0f;
```

Blueprint variables follow the same rules. Booleans keep the `b` prefix in Blueprint, although the editor hides it in the Details panel.

### 4.3 Data Assets and Data Tables

Data Assets are the Unreal equivalent of Unity ScriptableObjects. A Data Asset has two names: the C++ or Blueprint class that defines it, and each asset created from it.

| Item | Pattern | Example |
|:--|:--|:--|
| Class (data definition) | `U<Thing>Data` | `UWeaponData`, `UPlayerData` |
| Class (configuration) | `U<Thing>Config` / `U<Thing>Settings` | `UDifficultyConfig` |
| Data Asset | `DA_<Thing>_<Variant>` | `DA_Weapon_Pistol`, `DA_Difficulty_Hard` |
| Data Table row struct | `F<Thing>Row` | `FWeaponRow` |
| Data Table | `DT_<Things>` | `DT_Weapons` |

Do not add `DataAsset` to the class name. The `DA_` asset prefix already identifies it.

Use a Data Asset when each entry is a distinct object with its own references, such as a weapon with its own mesh and sounds. Use a Data Table when entries are rows of the same shape that designers edit side by side, such as a damage or loot table.

```cpp
UCLASS(BlueprintType)
class UWeaponData : public UPrimaryDataAsset
{
    GENERATED_BODY()
};
```

## 5. Naming Related Assets

Related assets should share a common base name.

This makes it easier to identify which assets belong to a particular character, environment, or object.

For example, a player character might have the following assets:

```text
BP_Player
SK_Player
SKEL_Player
PHYS_Player
T_Player_Albedo
T_Player_Normal
ABP_Player
AS_Player_Idle
AS_Player_Walk
AS_Player_Run
BS_Player_Locomotion
AM_Player_Attack
```

The shared name `Player` identifies the related assets.

The prefixes and suffixes distinguish their individual purposes.

Materials follow their own pattern (Section 6). The player's skin is `MI_Char_PlayerSkin_Subsurface`, named for what it makes, and it is stored with the other player assets in `Characters/Player/Materials/`.

### 5.1 Texture Descriptors

Textures often require additional descriptors because a model may use several texture maps.

| Texture type | Example | Compression setting |
|:--|:--|:--|
| Albedo or base colour | `T_StoneWall_Albedo` | Default, sRGB on |
| Normal map | `T_StoneWall_Normal` | Normalmap |
| Ambient occlusion | `T_StoneWall_AO` | Masks, sRGB off |
| Metallic map | `T_StoneWall_Metallic` | Masks, sRGB off |
| Roughness map | `T_StoneWall_Roughness` | Masks, sRGB off |
| Packed occlusion, roughness, metallic | `T_StoneWall_ORM` | Masks, sRGB off |
| Emission map | `T_StoneWall_Emission` | Default, sRGB on |

Use descriptors that match the texture's actual purpose. The descriptor also tells anyone checking the import settings which compression setting the texture should have.

Textures keep the asset-based pattern because a texture set belongs to one asset, whereas a material can be shared by many.

## 6. Materials

Materials use a three-field pattern: **Domain**, **Name** and **Usage**.

- **Material:** `MM_<Domain>_<Name>_<Usage>`
- **Material Instance:** `MI_<Domain>_<Name>_<Usage>`

Use the `MM_` prefix for every material, whether it is a master built to be instanced or a material used once. One prefix means every material sorts together in the Content Browser and is found by the same search. Epic's published list uses `M_`, so expect `M_` in Fab packs and engine content.

Every material and instance gets its own name. Two assets never share a name in different folders, because a search would then return results you cannot tell apart without opening them.

| Unreal | Unity equivalent |
|:--|:--|
| `MM_` Material | `SG_` Shader Graph |
| `MI_` Material Instance | `M_` Material |
| `MI_` child instance | `MV_` Material Variant |
| `MF_` Material Function | `SGS_` Sub Graph |

### 6.1 Field Glossary

**Domain** identifies where the material is used:

`Env` (environment), `Prop`, `Char`, `Foliage`, `FX`, `Decal`, `UI`, `Terrain`

The Domain field is unrelated to the **Material Domain** setting in the Material Details panel. The two only coincide for `Decal` and `UI`.

**Name** identifies what the material makes, in PascalCase. Choose the most useful of three kinds:

| Kind | Use it when | Examples |
|:--|:--|:--|
| Surface | The material reproduces a real surface | `Wood`, `PolishedSteel`, `StainedWood`, `MossyStoneFloor` |
| Effect | The material produces a visual effect | `RefractiveGlass`, `PanningLightStrip`, `RadarSweep`, `Dissolve` |
| Technique | The material exists to demonstrate or provide a technique | `HazardStripes`, `ORMPacked`, `ValueNoiseFBM`, `PBRMaster` |

A master built to be instanced is named for its role, such as `PBRMaster`, and each instance is named for the surface it makes, never for its parent.

**Usage** identifies how the material blends or shades:

| Usage | Unreal setting |
|:--|:--|
| `Opaque` | Blend Mode: Opaque |
| `Masked` | Blend Mode: Masked |
| `Trans` | Blend Mode: Translucent |
| `Add` | Blend Mode: Additive |
| `Subsurface` | Shading Model: Subsurface or Subsurface Profile |
| `Decal` | Material Domain: Deferred Decal |

Other shading models, such as Clear Coat and Unlit, keep the Usage of their blend mode, usually `Opaque`.

Post-process materials are not covered by this pattern. Use the `PPM_` prefix with a descriptive name, such as `PPM_Outline`.

### 6.2 Examples

| Asset | Name |
|:--|:--|
| A wood prop material | `MM_Prop_Wood_Opaque` |
| A shared master with parameters | `MM_Prop_PBRMaster_Opaque` |
| An instance of that master for polished steel | `MI_Prop_PolishedSteel_Opaque` |
| A child instance of the polished steel with a heat tint | `MI_Prop_HeatedSteel_Opaque` |
| A masked foliage card | `MM_Foliage_LeafCard_Masked` |
| Translucent glass | `MM_Prop_RefractiveGlass_Trans` |
| A procedural brick wall | `MM_Env_BrickWall_Opaque` |
| An animated radar screen | `MM_FX_RadarSweep_Opaque` |
| Candle wax | `MM_Prop_CandleWax_Subsurface` |

Name the effect when it is what the material is for, as in `MM_FX_RadarSweep_Opaque`. Do not add parameter values to a name: a brighter radar is the same material with its `SweepSpeed` or colour changed in an instance, so it keeps the same Name field.

### 6.3 Instance Hierarchies

- Create a Material Instance for each surface or variant rather than duplicating the master. Instances share the compiled shader, so they are cheaper and update automatically when the master changes.
- Create an instance by right-clicking its parent and choosing **Create Material Instance**, which sets the parent for you, then rename it for the surface it makes.
- A child instance can parent another instance, for example `MI_Env_MossyStone_Opaque` with `MI_Env_Stone_Opaque` as its parent. Give the child its own Name field, so the chain reads from general to specific.
- Create a new master only when the shading logic genuinely differs.

### 6.4 Quick Check

- Can you tell **where** it is used (Domain)?
- Can you tell **what** it makes (Name)?
- Can you tell **how** it blends (Usage)?
- Is the name unique in the project?

If the answer to all four is yes, the name is good.

## 7. Material Parameters, Static Switches, and Functions

### 7.1 Parameters

Material parameter names are used by name from Blueprints and C++, for example in `SetScalarParameterValue`. Choose them deliberately and do not rename them once instances or code depend on them.

| Parameter type | Convention | Example |
|:--|:--|:--|
| Vector (colour) | PascalCase noun | `BaseColor`, `GlowColour` |
| Texture | PascalCase noun | `BaseColorMap`, `NormalMap` |
| Scalar | PascalCase noun | `Roughness`, `GlowAmount` |
| Static Switch | `Use<Feature>` | `UseNormalMap`, `UseTriplanar` |

- Use PascalCase with no spaces. The editor allows spaces in parameter names, but they are error-prone when the name is typed in code.
- Match the engine's pin names where the meaning matches, such as `BaseColor`, `Metallic`, `Roughness`, and `Normal`, keeping the US spelling for those names. UK spelling is fine elsewhere, but keep each name consistent across all Master Materials.
- Organise parameters with **Group** and **Sort Priority** so instances present a tidy Details panel. Use groups such as Surface, Glow, and Animation.

### 7.2 Static Switches and Runtime Toggles

A Static Switch Parameter compiles a separate shader permutation for each state, in the same way as a Unity shader keyword.

- Use a Static Switch for features chosen per instance in the editor, such as whether a normal map is used.
- Static Switches cannot be changed at runtime on a Material Instance Dynamic. For runtime toggles, use a Scalar parameter between 0 and 1, and blend with a `Lerp`.
- Every Static Switch doubles the possible permutations. Prefer a Scalar parameter when the cost of the extra instructions is small.

### 7.3 Material Functions

Name Material Functions as `MF_<Category>_<Operation>`.

```text
MF_UV_Triplanar
MF_UV_ScreenSpace
MF_Math_RemapRange
MF_Lighting_Fresnel
```

Use plain words for input and output names, such as `UV`, `Strength`, and `Result`.

Tick **Expose to Library** and set a **Library Category** so the function appears in the Palette. Store Material Functions in a shared folder so they are found and reused rather than duplicated.

## 8. Naming Variants

Use numbered suffixes when several assets represent variations of the same object.

For example:

```text
SM_Rock_01
SM_Rock_02
SM_Rock_03
```

This is appropriate when the models represent different variations of the same type of rock.

However, do not use numbers simply to distinguish unrelated objects.

For example:

```text
SM_Rock_01
SM_Tree_01
SM_Barrel_01
```

These names communicate the purpose of each model more effectively.

Use meaningful descriptors when the differences between variants are important.

For example:

```text
SM_Wall_Stone
SM_Wall_Brick
SM_Wall_Wood
```

For materials, the Name field carries this distinction (`MI_Env_Stone_Opaque`, `MI_Env_MossyStone_Opaque`). Variants of one surface are child instances with their own Name field (Section 6.3).

## 9. Asset Names and Actor Labels

Asset names in the Content Browser and Actor labels in the Outliner serve different purposes.

An asset name identifies a file in the Content Browser. An Actor label identifies an object placed in a level.

These names do not need to follow exactly the same convention.

When a Blueprint such as `BP_EnemyGoblin` is placed in a level, the editor labels the Actor `BP_EnemyGoblin` by default. Rename placed Actors to something descriptive, such as:

```text
EnemyGoblin_Courtyard
```

Group placed Actors into Outliner folders, such as Lighting, Gameplay, and Set Dressing, so that another developer can understand the level structure.

## 10. Folder Organisation

Naming conventions should complement a sensible folder structure.

Place all project content inside a single top-level project folder within `Content/`. This separates your assets from Fab and Marketplace packs, starter content, and plugins, and makes migration between projects simpler.

For small projects, the following structure provides a useful starting point:

```text
Content/
  ProjectName/
    Animations/
    Audio/
      Music/
      SoundEffects/
    Blueprints/
    Data/            (Data Assets and Data Tables)
    Maps/
    Materials/
      Masters/
      Functions/
    Meshes/
    Textures/
    UI/
    VFX/
```

For larger projects, consider grouping assets by gameplay system, character, or environment.

For example:

```text
Content/
  ProjectName/
    Characters/
      Player/
        Animations/
        Blueprints/
        Materials/
        Meshes/
        Textures/
    Environments/
      Dungeon/
        Materials/
        Meshes/
        Textures/
    Core/
      Blueprints/
      Data/
    Maps/
    Materials/
      Masters/
      Functions/
```

In this structure the folders carry the asset association that material names deliberately leave out. Master Materials and Material Functions sit in a shared folder because many assets use them.

Use the `Developers` folder (enabled in the Content Browser view options) for personal experiments, and keep it out of shipped content.

Choose a structure that suits the size and needs of the project.

Avoid creating unnecessarily deep folder hierarchies for small projects.

## 11. Common Pitfalls

| Pitfall | Fix |
|:--|:--|
| Moving or renaming assets leaves redirectors behind, which cause broken references when files are committed separately. | Right-click the folder and choose **Fix Up Redirectors** before committing. |
| Renaming a C++ class breaks every Blueprint derived from it. | Add a class redirect to the `[CoreRedirects]` section of `DefaultEngine.ini` before renaming. |
| Changes made to a Data Asset during Play In Editor persist after the session ends. | Treat Data Assets as read-only at runtime. Copy values into the Actor or component that needs to change them. |
| Two materials with the same name in different folders, such as two `MM_Prop_PBR_Opaque`. | Give each its own Name field, such as `MM_Prop_Paint_Opaque` and `MM_Prop_ClearCoatPaint_Opaque`. |
| Adding parameter values or extra fields to a name, such as `MM_Prop_Paint_Opaque_Red05`. | Keep values as parameters and set them in an instance. |
| A material's Usage field no longer matches its Blend Mode or Shading Model. | Rename the material whenever these settings change. |
| Confusing the Domain name field with the Material Domain setting. | The Domain field describes where the material is used. The setting describes what kind of material it is. |
| Creating a Material Instance Dynamic every frame, or one per Actor for a single shared value. | Create each dynamic instance once and cache it. Use a Material Parameter Collection for global values, and Custom Primitive Data for per-Actor values. |
| Trying to change a Static Switch at runtime. | Use a Scalar parameter and a `Lerp` instead. |
| Using `MP_` for Material Parameter Collections, which clashes with Epic's prefix for Media Players. | Use `MPC_`. |

## 12. Practical Checklist

Before committing assets to version control, check the following.

- [ ] Asset names follow the project's naming convention.
- [ ] Names are descriptive and meaningful.
- [ ] Spaces and special characters have been avoided in names.
- [ ] Prefixes and suffixes are used consistently.
- [ ] Related assets share a common base name.
- [ ] Variants are clearly distinguished.
- [ ] C++ classes carry the correct type prefix, and filenames match the class name without it.
- [ ] Classes and Blueprints are named by role, not with a `Script` or `My` prefix or suffix.
- [ ] Materials follow the Domain, Name, Usage pattern, and every material name is unique.
- [ ] Material parameter names are set deliberately and contain no spaces.
- [ ] Redirectors have been fixed up.
- [ ] Assets are stored in appropriate folders inside the project folder.
- [ ] Temporary files and unnecessary duplicates have been removed.

## 13. Summary

A good naming convention makes an Unreal Engine project easier to understand and maintain.

Remember the following principles:

1. Use consistent naming rules throughout the project.
2. Prefer descriptive names over generic or abbreviated names.
3. Use prefixes to make asset types easier to identify.
4. Give related assets a common base name.
5. Name classes and Blueprints by role, and materials by what they make and how they blend.
6. Use meaningful descriptors and variant suffixes.
7. Organise assets into folders that support the project's workflow.
8. Follow established project conventions when working with an existing codebase.

The purpose of a naming convention is not to make filenames complicated. It is to make the project easier for every team member to understand.

## References

- [Epic Games: Recommended Asset Naming Conventions](https://dev.epicgames.com/documentation/unreal-engine/recommended-asset-naming-conventions-in-unreal-engine-projects)
- [Epic Games: Epic C++ Coding Standard for Unreal Engine](https://dev.epicgames.com/documentation/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine)
- [Tom Looman: Unreal Engine Naming Convention Guide](https://tomlooman.com/unreal-engine-naming-convention-guide/)
