---
title: "Master Materials, Material Functions and Layering"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-19"
doc-type: notes
session: 2
topic_code: MAT-02
difficulty_tier: foundation
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [01_pbr_and_material_editor]
feeds_into: [lab_3_master_material_instances, lab_4_material_functions_and_blending, 04_procedural_textures_and_uv_maths]
engine_version: "UE 5.7"
---

# Master Materials, Material Functions and Layering

---

## 1. Key Concepts

1. **Build one master, make many instances.** An instance changes values without compiling a new shader, so a hundred surfaces cost one compile instead of a hundred.
2. **Parameterise what varies, hard-code what does not.** Every parameter is a setting someone can get wrong, so expose only the values that actually differ between surfaces.
3. **Every static switch doubles the shaders.** Each switch compiles both branches as separate shaders, so ten switches can mean over a thousand variants.
4. **Treat parameter names as an API.** Instances store overrides by name, so renaming a parameter silently discards every override that used the old name.
5. **Group parameters so others can find them.** An instance with forty ungrouped parameters is unusable, and unusable masters get copied instead of reused.
6. **Write graph logic once, in a Material Function.** A fix to the function reaches every material that calls it, so a bug never has to be fixed twice.
7. **`Floor` gives the cell, `Frac` gives the position.** Almost every repeating pattern starts by splitting UVs into these two parts.
8. **Break repetition at more than one scale.** Large-scale variation hides the grid from a distance, and small-scale detail hides it close up, and neither alone is enough.
9. **Clamp the sprite and keep the derivatives.** A bombed sprite set to wrap bleeds into its neighbours, and cut UVs without derivatives draw lines at every cell edge.
10. **Let height decide the blend edge.** A plain Lerp blends two surfaces like a fade, but real moss fills cracks first, and only a height map knows where the cracks are.

---

## 2. Framing Question

Last week you built a wood material, a metal and a polished metal, and their graphs were identical except for three textures and one constant. A real game has hundreds of surfaces. If an art director asks for every surface in the game to be 10% rougher, how many graphs do you want to open?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Explain why instances are cheaper to create and edit than separate materials | 5.1, 5.2 |
| Build a master material with scalar, vector and texture parameters | 5.3 |
| Use a static switch, and calculate the shader permutations it adds | 5.4 |
| Group, order and name parameters so an instance is easy to use | 5.5 |
| Package reusable graph logic as a Material Function with inputs and outputs | 5.6 |
| Split UVs into cells with `Floor` and `Frac`, and give each cell a random value | 5.7 |
| Break visible texture repetition with macro variation and texture bombing | 5.8 |
| Blend two surfaces from a splat map, and sharpen the edge with a height map | 5.9 |

---

## 4. Why This Matters

Studios do not build materials one at a time. A technical artist builds a small set of master materials, and environment artists make every surface in the game as instances of them. If you can build a clean master, you can support a whole art team. If you cannot, every artist copies your graph, every copy drifts, and a single lighting change means editing hundreds of materials by hand.

The second half of this week deals with the most visible flaw in game environments: a tiling texture repeating across a large floor. Players spot the grid within seconds. The techniques here, cell maths, macro variation, texture bombing and splatting, are how shipped games hide it, and they all start from the same two nodes.

---

## 5. Core Ideas

Sections 5.1 to 5.5 are Session A. Sections 5.6 to 5.9 are Session B.

### 5.1 The Cost of One Material Per Surface

Every material compiles into its own set of shaders. Unreal compiles a separate shader for each kind of mesh the material is used on and each rendering feature it supports, so one material can mean dozens of compiled shaders. Ten near-identical materials therefore cost ten times the compile time, ten times the shader memory and ten graphs to maintain.

The three Week 1 lab materials show the problem. Their graphs were identical. Only the textures and one roughness constant differed. Those values are data, and data should not need a new shader.

### 5.2 Master Materials and Instances

A **master material** is an ordinary material built to be shared. Values that differ between surfaces are exposed as **parameters**. A **Material Instance** (asset type `Material Instance Constant`, prefix `MI_`) points at a parent material and overrides some or all of its parameters. It has no graph of its own.

To create one, right-click the master in the Content Browser and choose **Create Material Instance**. Open the instance and you see a list of the parent's parameters. Tick the box beside a parameter to override it, then change its value. The viewport updates immediately, because changing a parameter value does not compile anything.

| | Separate materials | One master, many instances |
|:---|:---|:---|
| New surface | Copy the graph and edit it | Create an instance and set values |
| Change a value | Edit the graph, click Apply, wait for the compile | Change the value, no compile |
| Fix a bug in the graph | Fix it in every copy | Fix it once in the master |
| Shaders compiled | One set per material | One set for the master |

An instance can also be the parent of another instance. This lets you build a chain such as `MM_Prop_PBRMaster_Opaque -> MI_Prop_Steel_Opaque -> MI_Prop_RustySteel_Opaque`, where the rusty version inherits everything from the steel and overrides only what it adds.

### 5.3 Parameters

A parameter is a node with a name. Its value in the master is the default, and instances can override it.

| Node | Shortcut | Holds | Typical use |
|:---|:---|:---|:---|
| `ScalarParameter` | `S` | One float | Roughness scale, emissive strength, tiling |
| `VectorParameter` | `V` | Four floats (RGBA) | Tint colours, two tiling values packed together |
| `TextureSampleParameter2D` | - | A texture, sampled like `Texture Sample` | Base colour, normal and ORM textures |

You can turn an existing node into a parameter: right-click a `Constant`, `Constant3Vector` or `Texture Sample` and choose **Convert to Parameter**, then name it. This is how you refactor a finished graph into a master.

Choose defaults that give a sensible result with no overrides, because the master's default look is what an artist sees first. A tint should default to white (1, 1, 1), so it changes nothing until someone sets it. A roughness scale should default to 1 for the same reason.

### 5.4 Static Switches and Permutations

Sometimes surfaces differ in structure, not just in values. Some need an ORM texture and some only need two numbers. You could sample the texture anyway and blend it away with a `Lerp`, but the GPU would still pay for the texture sample on every pixel.

A **static switch** removes the unused branch at compile time. The `StaticSwitchParameter` node has `True` and `False` inputs and a checkbox parameter. Each instance sets the checkbox, and the compiler builds only the chosen branch, so the unused texture sample does not exist in that shader.

The price is **permutations**. Unreal compiles a separate shader for every combination of switch values that an instance actually uses. In the worst case, `n` switches mean `2^n` shaders:

| Switches | Possible permutations |
|:---|:---|
| 1 | 2 |
| 3 | 8 |
| 5 | 32 |
| 10 | 1,024 |

Use a switch to remove real cost, such as a texture sample, a whole feature or an expensive branch. Use a `ScalarParameter` into a `Lerp` when the difference is only a value, because that costs one cheap blend and no extra shaders.

Two `StaticSwitchParameter` nodes with the same name act as one parameter. Use this when one decision changes several parts of the graph, such as switching both `Roughness` and `Metallic` from texture to constant.

### 5.5 Grouping, Ordering and Naming

Select any parameter node and look at Details. Two fields control how it appears in an instance:

- **Group** puts the parameter under a heading. Parameters with no group appear under **None**.
- **Sort Priority** orders parameters within a group, lowest first.

Use the same groups across every master in a project, so an artist always looks for tint under the same heading. Name parameters for what they control, in the same form everywhere:

| Good | Bad | Why |
|:---|:---|:---|
| `BaseColorTint` | `Param`, `Color2` | Says what it changes |
| `RoughnessScale` | `Rough` | Says whether it replaces or scales |
| `UseORMTexture` | `Switch1` | Reads as a yes/no question, which is what a switch is |

Name the assets themselves with the [naming conventions](../supplementary/unreal-naming-conventions.md): `MM_<Domain>_<Name>_<Usage>` for a material and `MI_<Domain>_<Name>_<Usage>` for an instance. The middle field says what the asset makes, for example `MM_Prop_PBRMaster_Opaque` for the master and `MI_Prop_PolishedSteel_Opaque` for one of its instances, so every asset has its own name.

Settle parameter names before you make instances. Instances store each override against the parameter's name, so if you rename `RoughnessScale` to `RoughnessMultiplier` in the master, every instance's override for it is lost without warning.

### 5.6 Material Functions

A **Material Function** (asset prefix `MF_`) is a reusable piece of graph. Where a master material shares a whole surface, a function shares one technique, such as a height blend or a tiling fix, and any number of materials can call it.

To build one:

1. Right-click in the Content Browser and choose **Materials > Material Function**. Name it `MF_<Category>_<Operation>`, for example `MF_UV_TextureBomb`, so functions sort by what kind of job they do.
2. Inside, add a `FunctionInput` node for each value the caller passes in. Set its **Input Name** and **Input Type**, for example `Function Input Vector 2` for UVs or `Function Input Scalar` for a number.
3. Build the graph from the inputs.
4. End it with one `FunctionOutput` node per result, each with an **Output Name**.
5. With nothing selected, tick **Expose to Library** so the function appears in the Palette and the right-click search.

To use it, drag the function asset into a material graph. It appears as a `MaterialFunctionCall` node with one pin per input and output. Save the function and every material that calls it recompiles with the change.

Functions and masters work together. The master decides what the surface is. Functions provide the techniques it uses. The rest of this section builds three techniques worth packaging this way.

### 5.7 UV Maths: Cells and Random Values

Multiply UVs by a number `n` and they run from 0 to `n` across the mesh instead of 0 to 1. Two nodes split that into a grid:

| Node | Output | Meaning |
|:---|:---|:---|
| `Frac` | The part after the decimal point, 0 to 1 | Where you are inside the current cell |
| `Floor` | The whole-number part | Which cell you are in, its **cell ID** |

So `Frac(UV * 4)` gives sixteen copies of a 0-to-1 square, and `Floor(UV * 4)` labels each copy (0,0), (1,0) and so on. Simple patterns follow directly:

1. **Stripes.** `TextureCoordinate -> ComponentMask (R)`, `-> Multiply` by the stripe count, `-> Frac`, `Frac -> Step.X`, with **Const Y** on `Step` set to 0.5. `Step` outputs 1 where `X` is at least `Y` and 0 elsewhere, so every cell is half black and half white.
2. **Checker.** `Floor(UV * n)`, split with `ComponentMask` into R and G, `Add` them, `Add -> Fmod.A`, `Constant` (2) `-> Fmod.B`. `Fmod` has no built-in value for B, so it needs the constant wired in. Neighbouring cells alternate between 0 and 1.

**A random value per cell.** To make each cell different, you need a number that looks random but is always the same for the same cell, so the pattern does not flicker. A **hash** turns the cell ID into such a number. The graph below is a widely used one:

1. `Floor(UV * n) -> Dot.A`
2. `Constant2Vector` (12.9898, 78.233) `-> Dot.B`
3. `Dot -> Sine`, with **Period** set to 6.283185
4. `Sine -> Multiply.A`, `Constant` (43758.5453) `-> Multiply.B`
5. `Multiply -> Frac`

The output is a value between 0 and 1 that changes from cell to cell. Change the `Constant2Vector` (for example to 39.346, 11.135) to get a second, independent random value for the same cell. The `Sine` node's **Period** is set to 2 x pi so it behaves like the mathematical sine the hash was designed for.

### 5.8 Repetition Breaking

A tiling texture repeats exactly, and eyes find exact repeats quickly. You break the repeat at two scales.

**Macro variation** works at large scale. Sample a soft, greyscale noise texture with UVs scaled far down, for example `UV * 0.05`, so one copy of the noise covers twenty copies of the floor. Use it to vary brightness:

1. `TextureCoordinate -> Multiply` by 0.05 `-> TextureSample (T_Noise_Macro).UVs`
2. `Constant` (0.7) `-> Lerp.A`, `Constant` (1.0) `-> Lerp.B`, `TextureSample.R -> Lerp.Alpha`
3. `Lerp -> Multiply.B`, with the floor's base colour on `Multiply.A`

Neighbouring tiles now differ slightly in brightness, so the grid stops lining up from a distance. Close up it does nothing, which is why you also need the second technique.

**Texture bombing** works at small scale. It scatters copies of a small sprite, such as a fallen leaf or a pebble, one per cell, each with its own random offset and rotation. Built from the cell maths in 5.7:

1. Scale the UVs: `UV * n`. Split into `Frac` (position in cell) and `Floor` (cell ID).
2. Hash the cell ID twice to get two random values, `r1` and `r2`.
3. Offset: subtract 0.5 from each random value, multiply by a jitter amount such as 0.3, and `Append` the two into a 2D offset. Add it to the cell position.
4. Rotate the result around the cell centre with the `CustomRotator` function: wire it to **UVs**, a `Constant2Vector` (0.5, 0.5) to **Rotation Center**, and `r1` to **Rotation Angle (0-1)**. The angle runs from 0 to 1, where 1 is a full turn, so a random value from the hash gives any rotation.
5. Scale around the centre so the sprite is smaller than the cell: subtract 0.5, multiply by 1.5, add 0.5.
6. Sample the sprite with these UVs, and use its alpha to `Lerp` it over the base surface.

Two settings stop the result from looking broken. Set the sprite texture's **X-axis Tiling Method** and **Y-axis Tiling Method** to `Clamp`, so a sprite pushed past the cell edge stops instead of wrapping and showing part of another copy. Then fix the thin lines that appear along every cell edge. They appear because `Frac` jumps from 1 back to 0 at the edge, the GPU thinks the texture is being viewed from very far away, and picks its smallest mip level. On the sprite's `Texture Sample`, set **Mip Value Mode** to `Derivative (explicit derivative to compute mip level)`. Two new inputs appear, `DDX(UVs)` and `DDY(UVs)`. Wire `UV * n` through a `DDX` node into `DDX(UVs)`, and through a `DDY` node into `DDY(UVs)`. The GPU then measures distance from the smooth UVs instead of the cut ones.

This version puts one sprite in each cell and cuts off any part that leaves its cell. Covering the cut by also sampling the neighbouring cells is the classic full version of the technique. It costs four samples instead of one, and is in the lab's Extension.

### 5.9 Texture Splatting and Height Blending

**Texture splatting** blends several surfaces using a **splat map**, a texture whose channels say how much of each surface to show. With RGBA you can drive up to four layers on top of a base. Terrain, roads and ruins are built this way.

A splat map is data, so set it to `Masks (no sRGB)`. For a two-layer blend of stone and moss:

1. `TextureSample (T_Splat_Mask).R` is the moss weight.
2. `Lerp` each attribute separately with that weight as **Alpha**: stone base colour on `A`, moss base colour on `B`. Repeat for normal and for ORM.
3. Wire the three `Lerp` outputs to `Base Color`, `Normal`, and the ORM channels.

A plain `Lerp` gives a soft, even fade, which looks like one photo dissolving into another. Real moss grows in the cracks first. A **height blend** fixes this by letting the stone's height map shift the edge. With `Weight` from the splat map and `Height` from the stone's height map inverted with `OneMinus` (so cracks score high):

1. `Weight -> Multiply.A`, `Constant` (2) `-> Multiply.B`
2. `Multiply -> Add.A`, `Height -> Add.B`
3. `Add -> Subtract.A`, `Constant` (1.5) `-> Subtract.B`
4. `Subtract -> Multiply.A`, a `ScalarParameter` named `BlendContrast`, default 8, `-> Multiply.B`
5. `Multiply -> Add.A`, `Constant` (0.5) `-> Add.B`
6. `Add -> Saturate`, and use this as the alpha for all three `Lerp`s

This computes `saturate((2 * Weight + Height - 1.5) * BlendContrast + 0.5)`. Where the weight is 0 the result is 0 for every height (all stone). Where it is 1 the result is 1 for every height (all moss). At a weight of 0.5 the result depends only on height, so cracks turn to moss and tops stay stone. As the weight rises, moss spreads up from the cracks. Raising `BlendContrast` makes the edge harder. Keep it at 1 or above, because below 1 the weight-0 and weight-1 ends no longer reach pure stone and pure moss.

Unreal ships a `HeightLerp` material function, found by searching the Palette, that does a similar job with inputs `A`, `B`, `Transition Phase`, `Height Texture` and `Contrast`. Build the graph yourself first, so you know what the function is doing when you use it.

The splat weight can come from somewhere other than a texture. **Vertex colour** stores weights on the mesh's vertices, so an artist paints them in the editor's Mesh Paint mode. The blend graph is identical, with the `VertexColor` node replacing the splat map. Vertex painting is in the lab's Extension, because it needs a mesh with enough vertices and time to paint.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes03/Demo/Levels/Lvl_Notes03`. Part A uses the five labelled spheres and Part B the large stone floor behind them. The finished demo assets are in `/Game/GD/Notes03/Demo/Materials`. To reproduce the demo, create `/Game/Student/Notes03/MyWork` with a `Functions` subfolder, and save each asset there under the name its step gives.

The textures are in `/Game/GD/Shared/Textures`, one folder per set, and every demo and lab uses the same copies. They come from free **Fab** listings, apart from a few generated for this module; the README lists each source. The shared textures already have the right compression settings. Before you change a setting on one, copy it into your own folder with **Copy Here**, because every material that samples the shared texture would change too. Part A uses the wood and steel sets from Lab 1.

### 6.1 Part A - One Master for Week 1 (Session A)

You rebuild the three Lab 1 materials and the Lab 2 paint as one master with four instances.

**Step 1 - Create the master.** In `MyWork`, create `MM_Prop_PBRMaster_Opaque` and open it. Add these parameters, with the group shown:

| Node | Name | Default | Group |
|:---|:---|:---|:---|
| `TextureSampleParameter2D` | `BaseColorTexture` | `/Engine/EngineResources/WhiteSquareTexture` | Base Colour |
| `VectorParameter` | `BaseColorTint` | (1, 1, 1) | Base Colour |
| `TextureSampleParameter2D` | `NormalTexture` | `/Engine/EngineMaterials/DefaultNormal` | Surface |
| `TextureSampleParameter2D` | `ORMTexture` | `T_Wood_ORM` | Surface |
| `ScalarParameter` | `RoughnessScale` | 1 | Surface |
| `ScalarParameter` | `Roughness` | 0.5 | Surface |
| `ScalarParameter` | `Metallic` | 0 | Surface |
| `StaticSwitchParameter` | `UseORMTexture` | True | Switches |

**Step 2 - Wire the graph.**

1. `BaseColorTexture.RGB -> Multiply.A`, `BaseColorTint -> Multiply.B`, `Multiply -> Base Color`
2. `NormalTexture.RGB -> Normal`
3. `ORMTexture.G -> Multiply.A`, `RoughnessScale -> Multiply.B`, `Multiply -> UseORMTexture.True`
4. `Roughness -> UseORMTexture.False`, `UseORMTexture -> Roughness`
5. Add a second `StaticSwitchParameter` and name it `UseORMTexture` too. Wire `ORMTexture.B -> True`, `Metallic -> False`, and its output `-> Metallic`
6. Add a third `StaticSwitchParameter`, also named `UseORMTexture`. Wire `ORMTexture.R -> True`, `Constant` (1) `-> False`, and its output `-> Ambient Occlusion`. Without this switch, a surface with the texture turned off would still pick up the AO of whatever texture `ORMTexture` holds.

Apply and save. Point out that the three switch nodes share one checkbox, because they share a name.

**Step 3 - Make the instances.** Make four instances. For each, right-click its parent in the Content Browser and choose **Create Material Instance**, which sets the parent for you, then rename the new asset:

| Instance | Parent | Overrides |
|:---|:---|:---|
| `MI_Prop_Wood_Opaque` | The master | `BaseColorTexture` = `T_Wood_Albedo`, `NormalTexture` = `T_Wood_Normal`, `ORMTexture` = `T_Wood_ORM` |
| `MI_Prop_Steel_Opaque` | The master | `BaseColorTexture` = `T_Steel_Albedo`, `NormalTexture` = `T_Steel_Normal`, `ORMTexture` = `T_Steel_ORM` |
| `MI_Prop_PolishedSteel_Opaque` | `MI_Prop_Steel_Opaque` | `RoughnessScale` = 0.3 |
| `MI_Prop_Paint_Opaque` | The master | `UseORMTexture` = off, `BaseColorTint` = (0.5, 0.02, 0.02), `Roughness` = 0.5, `Metallic` = 0 |

Apply each to a sphere. Drag `RoughnessScale` on the polished instance and show that the sphere updates with no compile. When you untick `UseORMTexture` on the paint instance, show the brief shader compile. No other instance uses that switch value yet, so the editor compiles a new permutation.

**Step 4 - Show what a master buys.** Open the master and change `RoughnessScale`'s default to 1.1. Every instance that does not override it gets 10% rougher at once. This answers the framing question.

### 6.2 Part B - A Floor That Does Not Repeat (Session B)

You use these sets from `/Game/GD/Shared/Textures`: the stone floor (`Stone/T_Stone_Albedo`, `T_Stone_Normal`, `T_Stone_ORM`, `T_Stone_Height`), the moss (`Moss/T_Moss_Albedo`, `T_Moss_Normal`, `T_Moss_ORM`), the noise texture `Noise/T_Noise_Macro` and the splat map `Splat/T_Splat_Mask`. The bombed sprite is the leaf from Week 1, `Leaf/T_Leaf_Albedo`. `T_Stone_Height` is the stone's ambient occlusion saved as its own texture. The scan ships no height map, and AO is dark in the cracks and bright on the stone tops, which is the shape a height blend needs.

**Step 5 - Show the problem.** In `MyWork`, create `MM_Env_MossyStoneFloor_Opaque` with the stone set wired as in Week 1, and a `TextureCoordinate` with **U Tiling** and **V Tiling** of 8 into all three samples. Apply it to a large plane and fly the camera up. The grid of repeats is obvious.

**Step 6 - Macro variation as a function.** In `MyWork/Functions`, create `MF_Color_MacroVariation`. Add inputs `Color` (Vector 3), `UVs` (Vector 2), `Scale` (Scalar, preview 0.05) and `Strength` (Scalar, preview 0.3). Build:

1. `UVs -> Multiply.A`, `Scale -> Multiply.B`, `Multiply -> TextureSample (T_Noise_Macro).UVs`
2. `Strength -> OneMinus`, `OneMinus -> Lerp.A`, `Constant` (1) `-> Lerp.B`, `TextureSample.R -> Lerp.Alpha`
3. `Lerp -> Multiply.A`, `Color -> Multiply.B`, `Multiply -> FunctionOutput` named `Result`

Tick **Expose to Library** and save. In `MM_Env_MossyStoneFloor_Opaque`, route the stone base colour through `MF_Color_MacroVariation` before `Base Color`, with the un-tiled `TextureCoordinate` on `UVs`. Fly up again. The grid is far harder to see.

**Step 7 - Texture bombing.** Check that `T_Leaf_Albedo` has **X-axis Tiling Method** and **Y-axis Tiling Method** set to `Clamp`. The shared copy already does. In `MyWork/Functions`, create `MF_UV_TextureBomb` following 5.8, with inputs `UVs` (Vector 2), `Cells` (Scalar, preview 4), `Jitter` (Scalar, preview 0.3), and a `Function Input Texture 2D` named `Sprite`. Sample `Sprite` by wiring it to the `Texture Sample`'s **Tex** input. Output the sprite's `RGB` as `Color` and its `A` as `Alpha`.

In `MM_Env_MossyStoneFloor_Opaque`, add a `TextureObject` set to `T_Leaf_Albedo` and wire it to `Sprite`. Then `Lerp` the bomb's `Color` over the macro-varied stone, with the bomb's `Alpha` as **Alpha**. Leaves appear scattered and rotated across the floor.

Look closely at the cell edges and find the thin lines. Set the sprite sample's **Mip Value Mode** to `Derivative (explicit derivative to compute mip level)` and wire `DDX` and `DDY` of the scaled UVs into `DDX(UVs)` and `DDY(UVs)`, as in 5.8. The lines disappear.

**Step 8 - Moss by splat map.** Add the moss set and `T_Splat_Mask` (already set to `Masks (no sRGB)`). Build the height blend from 5.9, using `T_Stone_Height` through `OneMinus` as `Height` and `T_Splat_Mask.R` as `Weight`. Use its result as the alpha for three `Lerp`s: stone and moss base colour, normal, and ORM. Put the leaf bombing after the base colour `Lerp`, so leaves lie on top of the moss.

Show the edge with `BlendContrast` at 1, then at 8. At 1 it is a soft fade. At 8 the moss sits in the cracks between stones.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| Parameter renamed after instances exist | Instances lose their overrides and show the master's default | Settle names first. If you must rename, re-enter the overrides |
| Instance override box not ticked | Changing the value does nothing | Tick the checkbox beside the parameter |
| Two parameters with the same name but different types | Compile error naming a duplicate parameter | Give every parameter of a different type its own name |
| Too many static switches | Long compiles and large shader caches | Use a `ScalarParameter` and `Lerp` when only a value differs |
| Function input not connected in the caller | Compile error on the `MaterialFunctionCall` node | Connect every input, or tick **Use Preview Value As Default** on the input |
| Function not visible in the Palette | Cannot find it by search | Tick **Expose to Library** in the function, then save |
| Sprite set to Wrap | Bits of the next sprite appear at cell edges | Set both tiling methods on the sprite texture to `Clamp` |
| Lines along every cell edge | Thin grid of dark or blurry lines | Set **Mip Value Mode** to `Derivative` and feed `DDX`/`DDY` of the uncut UVs |
| Splat map left as sRGB | Layers blend in at the wrong strength, too little in the mid-tones | Set the splat map to `Masks (no sRGB)` |
| Height not inverted | Moss grows on the tops of stones instead of in the cracks | Pass the height map through `OneMinus` |

---

## 8. Exercises

1. A master has four static switches, and its instances use only three combinations of them. How many permutations could it have, and how many are compiled? Explain the difference.
2. Refactor one of your own Week 1 or Week 2 materials into a master with no more than six parameters. List each parameter, its group and its default, and justify each default.
3. Build a stripe pattern and a checker pattern from `Frac`, `Floor`, `Step` and `Fmod` only. Make the stripe count a `ScalarParameter`.
4. Extend the hash from 5.7 so each cell of a checker gets a random grey between 0.3 and 0.7.
5. Build a three-layer splat (for example, stone, moss and mud) from the R and G channels of one splat map. Describe the order you blend the layers in, and why it matters.

---

## 9. Reflective Questions

1. When would you deliberately make a new master material instead of adding a switch to an existing one?
2. A function is fixed once and every caller gets the fix. What risk does that same property create on a team?
3. Macro variation and texture bombing both hide repetition. Which would a player notice missing first, and why?
4. The height blend uses the stone's height to decide where moss grows. What other data could drive where a second surface appears, and what would it look like?

---

## 10. How This Builds

- **Notes 02** (`02_texture_packing_and_spherical_coordinates`) is this week's extra reading. It measures what the packed ORM texture in your master saves, and explains the dependent lookups that texture bombing and splatting add.
- **Lab 3** (`lab_3_master_material_instances`) applies Session A. You produce five or more surfaces from one master using instances only.
- **Lab 4** (`lab_4_material_functions_and_blending`) applies Session B. You package macro variation as a function and build a height-blended splat material. Texture bombing, world-aligned and triplanar mapping, and vertex colour blending are in its Extension.
- **Week 3** (`04_procedural_textures_and_uv_maths`) takes the cell maths from 5.7 further: rotated stripes with clean edges, tiles and bricks, polar coordinates, distance shapes and noise built from the same hash, all without a texture.
- **Week 4** drives the parameters you created here at runtime from Blueprint, per instance with Dynamic Material Instances and globally with Material Parameter Collections.
- **Week 5** adds parallax to the same stone surface, first with `Bump Offset` and then with parallax occlusion mapping, and covers world-aligned and triplanar mapping in its Extension.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **Cell ID** | The whole-number part of scaled UVs, from `Floor`, which labels each repeat of a pattern |
| **Hash** | A calculation that turns an input such as a cell ID into a value that looks random but is always the same for the same input |
| **Height blend** | A blend whose edge is shifted by a height map, so one surface appears in low or high areas first |
| **Macro variation** | Large-scale variation, usually from a low-frequency noise texture, that hides repetition from a distance |
| **Master material** | A material built to be shared, exposing parameters for instances to override |
| **Material Function** | A reusable graph asset with inputs and outputs, called from materials |
| **Material Instance** | An asset that points at a parent material and overrides its parameters, with no graph of its own |
| **Parameter** | A named node whose value instances can override |
| **Permutation** | One compiled version of a material's shader for one combination of static switch values |
| **Splat map** | A texture whose channels store blend weights for several surface layers |
| **Static switch** | A parameter that removes one branch of the graph at compile time |
| **Texture bombing** | Scattering copies of a small texture, one per cell, with random offset and rotation |
| **Vertex colour** | Colour stored on a mesh's vertices, often used as blend weights painted by an artist |
