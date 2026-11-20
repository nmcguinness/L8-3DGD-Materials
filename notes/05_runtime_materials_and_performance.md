---
title: "Runtime Materials, Screen-Space and Performance"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: notes
session: 4
topic_code: MAT-04
difficulty_tier: intermediate
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [03_master_materials_and_layering, 04_procedural_textures_and_uv_maths]
feeds_into: [lab_7_runtime_driven_effects, lab_8_post_process_decals_and_profiling, 06_geometry_and_scene_aware_materials]
engine_version: "UE 5.7"
---

# Runtime Materials, Screen-Space and Performance

---

## 1. Key Concepts

1. **Create a dynamic instance once, then reuse it.** Every Create Dynamic Material Instance call makes a new object, so creating one per frame leaks memory and costs time.
2. **Per object needs a dynamic instance, the whole world needs a collection.** A Material Parameter Collection changes one value for every material that reads it, so global weather or time of day costs one call, not one per object.
3. **Animate with a Timeline, not with Tick.** A Timeline runs only while it plays and gives you a curve you can shape, while Tick runs every frame forever.
4. **Parameter names are the contract between Blueprint and material.** A misspelt name fails silently, so copy the name from the material, never type it from memory.
5. **An emissive edge needs a soft mask.** A dissolve reads as burning only when a thin band just ahead of the cut glows, and that band comes from the same noise as the cut.
6. **A decal changes the surface it lands on, not the mesh.** Decals add damage, dirt and markings without new geometry or new UVs, which is why level artists reach for them first.
7. **Post-process materials run on every pixel of the screen.** Their cost does not depend on how many objects are in view, so every node in them is paid at full resolution.
8. **Mark objects with stencil, then find them in post-process.** Custom Depth and Stencil let a full-screen material treat one object differently, without touching that object's own material.
9. **Measure before you optimise.** The shader complexity view and the material Stats show where the cost is, so you fix the expensive material rather than the one you suspect.
10. **Translucency is paid once per layer.** Every overlapping translucent surface shades the same pixel again, so ten stacked smoke planes cost ten times one.

---

## 2. Framing Question

Your material looks right in the editor. In the game, the player shoots a crate and it should burn away, the weather turns and every surface should get wet, an enemy behind a wall should be outlined, and the frame rate has just dropped below 60. None of that can be baked into the material in advance. How does a game change a material while it runs, and how do you know which material is slowing the frame down?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Explain the difference between a Material Instance Constant and a Material Instance Dynamic | 5.1 |
| Create a dynamic instance in Blueprint and set its parameters | 5.2 |
| Drive a parameter over time with a Timeline | 5.3 |
| Change many materials at once with a Material Parameter Collection | 5.4 |
| Build a noise dissolve with an emissive edge, a pulse and a Fresnel rim | 5.5 |
| Place deferred decals and say what they can and cannot change | 5.6 |
| Write a post-process material and read scene textures | 5.7 |
| Outline selected objects with Custom Depth and Stencil | 5.8 |
| Read material cost from the shader complexity view, Stats and samplers | 5.9 |
| Explain translucency overdraw and reduce it | 5.10 |
| Provide cheaper versions of a material with quality switches | 5.11 |

---

## 4. Why This Matters

Most of what makes a game feel alive is a material changing at runtime: a health pickup pulsing, a shield flickering when hit, a corpse dissolving, rain darkening a street. These are parameter changes driven by gameplay, not new materials. A games programmer is usually the person who wires them, so the Blueprint side matters as much as the graph.

The second half of this week is about the screen as a whole. Decals and post-process materials touch pixels rather than objects, and they are where most visual polish lives. They are also where frame time disappears, because a full-screen effect pays for every pixel every frame. Knowing how to measure cost turns optimisation from guesswork into a short list of fixes.

---

## 5. Core Ideas

Sections 5.1 to 5.5 are Session A, materials driven at runtime. Sections 5.6 to 5.11 are Session B, the screen and its cost.

### 5.1 Constant and Dynamic Instances

In Notes 03 you made **Material Instance Constant** assets (`MI_`) in the Content Browser. Their values are fixed when you save them. A game cannot change a constant instance while it runs, because one asset is shared by every object that uses it.

A **Material Instance Dynamic** (MID) is created at runtime, belongs to one object, and has parameters you can change every frame. It is never saved as an asset.

| | Material Instance Constant | Material Instance Dynamic |
|:---|:---|:---|
| Created | In the editor, as an asset | At runtime, from Blueprint or C++ |
| Shared | By every object that uses it | By the one object that created it |
| Change values while the game runs | No | Yes |
| Cost of changing a value | Not possible at runtime | A cheap parameter update, no shader compile |

Both share the parent's compiled shader. A dynamic instance is cheap to create, but not free, so create it once and keep a reference to it.

### 5.2 Creating a Dynamic Instance in Blueprint

The usual pattern runs once, on **Event BeginPlay**:

1. **Event BeginPlay -> Create Dynamic Material Instance.** Use the version whose **Target** is a Primitive Component, such as the actor's Static Mesh component. Set **Element Index** to the material slot, usually 0. Leave **Source Material** empty to use the material already on that slot.
2. Right-click the **Return Value** pin and choose **Promote to Variable**. Name the variable `DynamicMaterial`. This is the reference you reuse.
3. Wherever the value should change, drag `DynamicMaterial` into the graph and call **Set Scalar Parameter Value** or **Set Vector Parameter Value** on it, with the **Parameter Name** copied exactly from the material.

Create Dynamic Material Instance also replaces the material on that slot with the new instance, so the mesh shows the change straight away. Calling it again would make a second instance and throw the first away, which is why the reference is stored.

### 5.3 Timelines

A dissolve should run from 0 to 1 over about two seconds, then stop. A **Timeline** is a Blueprint node that plays a curve you draw:

1. In the Event Graph, right-click and choose **Add Timeline**. Name it `DissolveTimeline` and double-click it to open the Timeline editor.
2. Click **+ Track** and choose **Add Float Track**. Name the track `Alpha`.
3. Right-click the track to add two keys: time 0, value 0, and time 2, value 1. Set the Timeline **Length** to 2.
4. Back in the Event Graph, the node has execution inputs such as **Play**, **Play from Start** and **Reverse**, an **Update** output that fires every frame while it plays, and an `Alpha` output with the current value.
5. Wire **Update -> Set Scalar Parameter Value**, with `Alpha` into **Value** and `Dissolve` as the **Parameter Name**.

Trigger **Play from Start** from any event, such as a key press, an overlap or a hit. The Timeline only ticks while it plays, and you can reshape the curve, for example to ease in, without touching the graph.

### 5.4 Material Parameter Collections

A dynamic instance changes one object. Wet weather should change every surface at once. A **Material Parameter Collection** (MPC, prefix `MPC_`) is an asset holding named scalar and vector values that any material can read:

1. Create the asset: right-click in the Content Browser, choose **Materials > Material Parameter Collection** and name it `MPC_Weather`. Open it and add a scalar parameter named `Wetness` with a default of 0.
2. In each material that should react, add a **CollectionParameter** node. Set its **Collection** to `MPC_Weather` and **Parameter Name** to `Wetness`.
3. In Blueprint, call **Set Scalar Parameter Value** from the **Rendering > Material** category, the version with a **Collection** pin. Set **Collection** to `MPC_Weather`, **Parameter Name** to `Wetness` and **Value** to the new value.

One call changes every material that reads the collection, with no dynamic instances. Wetness itself is three changes driven by the same value: base colour darkens because water fills the surface's pores, roughness drops because a water film is smooth, and puddles can form where a height or noise mask is low.

A material can read at most two collections. Group related values, such as all weather values, into one collection.

### 5.5 Runtime-Friendly Material Effects

Three effects are built to be driven from outside the material.

**Noise dissolve.** Use the **Masked** blend mode. Compare a noise value with a `Dissolve` parameter running from 0 to 1:

1. `n` is a noise value from 0 to 1, from the `Noise` node or a texture.
2. `cut = n - Dissolve`. Wire `cut -> Opacity Mask`. Pixels whose noise is below `Dissolve` are cut away, so the surface eats inwards as `Dissolve` rises.
3. `edge = 1 - SmoothStep(0, EdgeWidth, cut)`. This is 1 just ahead of the cut and 0 further in, the same soft band you built in Notes 04.
4. `edge * EdgeColour * EdgeStrength -> Emissive Color`.

To make sure the surface is whole at 0 and gone at 1, remap `Dissolve` before the comparison: `Dissolve * 1.2 - 0.1` pushes the cut past both ends of the noise range, so no stray pixels are cut at 0 or left standing at 1.

**Emissive pulse.** `Sine(Time * PulseSpeed) * 0.5 + 0.5` runs smoothly between 0 and 1. Multiply it into the emissive colour for a breathing glow. Set `PulseSpeed` from a dynamic instance to make one pickup pulse faster than the others.

**Fresnel rim.** The `Fresnel` node outputs 0 where the surface faces the camera and 1 where it turns away, at the silhouette. Its **ExponentIn** controls how thin the rim is. `Fresnel * RimColour -> Emissive Color` gives shields, holograms and highlighted pickups their glowing edge. Week 6 uses the same node for water.

### 5.6 Decals

A **deferred decal** projects a material onto whatever surfaces lie inside a box. It changes the surface's base colour, normal or roughness in the G-buffer, before lighting, so it is lit like the surface it lands on.

To make one:

1. Create a material and set **Material Domain** to `Deferred Decal` and **Blend Mode** to `Translucent`.
2. Wire the inputs you want to change. An unwired input leaves the surface below unchanged. **Opacity** says how strongly the decal covers, and is usually a texture's alpha or a procedural mask.
3. Place a **Decal Actor**, set its **Decal Material**, and scale its box to cover the area. The decal projects along the box's X axis.

Decals suit damage, dirt, signs, footprints and markings, and anything the level designer places by hand. They cannot change the silhouette, and each decal costs shading for every pixel it covers, so many large overlapping decals add up.

### 5.7 Post-Process Materials

A **post-process material** runs once for every pixel on screen, after the scene has rendered. It reads the rendered image and the G-buffer, and writes a new colour.

1. Set **Material Domain** to `Post Process`. **Blendable Location** decides when it runs. `Scene Color After Tonemapping` works on the final display colours, which is the simplest place for an outline or a colour grade.
2. Read the scene with the **SceneTexture** node. Its **Scene Texture Id** selects what to read: `PostProcessInput0` is the rendered image, `SceneDepth` is the distance to each pixel, and `CustomDepth` and `CustomStencil` are described in 5.8.
3. Wire the result into `Emissive Color`, the only colour input a post-process material uses.
4. Apply it: place a **Post Process Volume**, tick **Infinite Extent (Unbound)** so it covers the whole level, and add the material under **Rendering Features > Post Process Materials**.

A post-process material costs the same with one object on screen or a thousand, because it runs per pixel. At 1920 x 1080 that is about two million evaluations every frame, so keep it short.

### 5.8 Custom Depth and Stencil

To outline only an enemy, the post-process material needs to know which pixels belong to that enemy. **Custom Depth** is a second depth buffer containing only the objects you choose. **Custom Stencil** stores a number from 0 to 255 for each of those objects' pixels.

1. In **Project Settings > Engine > Rendering > Postprocessing**, set **Custom Depth-Stencil Pass** to `Enabled with Stencil`. This turns on the extra buffer for the project.
2. Select the object. Under **Rendering**, tick **Render CustomDepth Pass** and set **CustomDepth Stencil Value**, for example 1 for enemies and 2 for pickups.
3. In the post-process material, `SceneTexture` with **Scene Texture Id** `CustomStencil` reads that number per pixel.

An outline is the edge of the stencil shape. Sample the stencil at the pixel and at its four neighbours, one pixel away up, down, left and right:

1. `ScreenPosition` gives this pixel's position, and the `SceneTexture` node's **InvSize** output gives the size of one pixel in UV units.
2. For each neighbour, add an offset of one pixel and read `CustomStencil` there.
3. `outline = Saturate(neighbour total) - Saturate(centre)`. A pixel is on the outline when a neighbour is marked but the pixel itself is not.
4. `Lerp(PostProcessInput0, OutlineColour, outline)` into `Emissive Color`.

Changing the stencil value from Blueprint, with **Set Custom Depth Stencil Value**, turns the outline on or off at runtime, for example while the player aims at an object.

### 5.9 Measuring Material Cost

Three tools tell you what a material costs. Use them in this order.

| Tool | Where | What it shows |
|:---|:---|:---|
| Shader complexity view | Viewport view mode menu, **Optimization Viewmodes > Shader Complexity** | Instructions per pixel across the whole screen, green for cheap, red and white for expensive. Shows which surfaces to investigate |
| Stats | **Stats** button in the Material Editor toolbar | The instruction count for the base pass and the number of texture samplers for one material |
| GPU profiler | Console command `ProfileGPU`, or `stat GPU` for a live summary | Milliseconds per rendering pass, so you see whether base pass, translucency or post-process is the problem |

Instruction count matters, but it is not the only cost:

- **Texture samplers.** A material can use at most 16 samplers. Each `Texture Sample` normally uses one. Setting **Sampler Source** to `Shared: Wrap` on a sample lets many textures share one sampler, which lifts the limit when you need it.
- **Dependent texture reads,** from Notes 02, stall the GPU while it waits for one texture before it can read the next.
- **Permutations,** from Notes 03. Each static switch value that an instance uses compiles another shader, which costs memory and load time rather than frame time.

### 5.10 Translucency and Overdraw

An opaque surface hides what is behind it, so the GPU can skip shading hidden pixels. A translucent surface cannot hide anything. Every translucent layer over a pixel is shaded and blended on top of the layers behind it. This repeated shading of the same pixel is **overdraw**.

Ten smoke planes stacked in front of the camera shade the same pixels ten times. The shader complexity view shows this as bright red and white patches where translucent surfaces overlap.

To reduce it:

- Use `Masked` instead of `Translucent` wherever a hard edge is acceptable, as the fence did in Lab 2.
- Use fewer, larger translucent layers rather than many small overlapping ones.
- Keep translucent materials short, because their instructions are paid once per layer.
- Trim particle sprites so their quads do not include large fully transparent corners, which still cost shading.

### 5.11 Quality Switches

Not every player has the same hardware. The **QualitySwitch** node has one input per material quality level, `Low`, `Medium`, `High` and `Epic`, plus `Default` for any level left unwired. The engine compiles each branch and uses the one that matches the player's scalability settings.

Use it to drop expensive work on low settings: skip the detail normal map, use one noise octave instead of four, or replace a translucent effect with a masked one. The extra branches cost compile time and shader memory, like static switches, so use it for real savings only.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes05/Demo/Levels/Lvl_Notes05`. The finished demo assets are in `/Game/GD/Notes05/Demo`, with materials in `Materials` and Blueprints in `Blueprints`. To reproduce the demo, create `/Game/Student/Notes05/MyWork` and save your assets there.

### 6.1 Part A - A Burning Crate and a Rainy Street (Session A)

**Step 1 - The dissolve material.** In `MyWork`, create `MM_Prop_Dissolve_Masked`. With nothing selected, set **Blend Mode** to `Masked`. Build:

1. A `Noise` node with **Levels** 4, **Output Min** 0 and **Output Max** 1, fed with `AppendVector(TextureCoordinate * 4, 0)`. Call it `n`.
2. A `ScalarParameter` `Dissolve`, default 0. Remap it: `Dissolve * 1.2 - 0.1`.
3. `cut = n - remapped Dissolve`, into `Opacity Mask`.
4. `edge = 1 - SmoothStep(0, EdgeWidth, cut)`, with a `ScalarParameter` `EdgeWidth`, default 0.05.
5. `edge * EdgeColour * EdgeStrength` into `Emissive Color`, with a `VectorParameter` `EdgeColour` (1, 0.35, 0.05) and a `ScalarParameter` `EdgeStrength`, default 20.
6. A wood or steel texture set from `/Game/GD/Shared/Textures` into `Base Color`, `Normal` and the ORM inputs, as in Notes 01.

Apply it to a cube. Drag `Dissolve` from 0 to 1 in the material preview: the cube burns away with a glowing edge.

**Step 2 - The Blueprint.** Create a Blueprint Class based on **Actor**, named `BP_DissolveProp`. Add a **Static Mesh** component, set its mesh to a cube and its material to `MM_Prop_Dissolve_Masked`. In the Event Graph:

1. **Event BeginPlay -> Create Dynamic Material Instance**, with the Static Mesh component as **Target** and **Element Index** 0. Promote the **Return Value** to a variable named `DynamicMaterial`.
2. Add a Timeline named `DissolveTimeline` with a float track `Alpha` from 0 at 0 seconds to 1 at 2 seconds, as in 5.3.
3. **DissolveTimeline.Update -> Set Scalar Parameter Value** on `DynamicMaterial`, with **Parameter Name** `Dissolve` and **Value** from `Alpha`.
4. Add a **Keyboard Event** for `F`, and wire **Pressed -> Play from Start** on the Timeline.

Place two `BP_DissolveProp` actors in the level. In the Details panel of one actor, under **Input**, set **Auto Receive Input** to `Player 0`. Press **Play**, then press F. Only that crate burns away, because only its dynamic instance changed.

**Step 3 - The weather collection.** Create `MPC_Weather` with a scalar parameter `Wetness`, default 0. Create `MM_Env_WetGround_Opaque` from the stone set in `/Game/GD/Shared/Textures/Stone`, and add a `CollectionParameter` reading `Wetness`:

1. `Base Color = StoneAlbedo * Lerp(1, 0.55, Wetness)`. Wet surfaces are darker.
2. `Roughness = Lerp(StoneORM.G, 0.08, Wetness)`. Water is smooth.

Apply it to the floor and to a wall. In the level, create `BP_WeatherControl`, a Blueprint Actor whose Event Graph runs **Event BeginPlay -> Play from Start** on a looping Timeline `Rain` with a track from 0 to 1 and back over 8 seconds, and wires **Update -> Set Scalar Parameter Value** with **Collection** `MPC_Weather`, **Parameter Name** `Wetness` and **Value** from the track. Place one in the level and press **Play**. Every surface using the material gets wet and dries together, from one call.

**Step 4 - A pulse and a rim.** Create `MM_Prop_EnergyPickup_Opaque`. Wire `Fresnel` with **ExponentIn** 3, multiplied by a cyan `RimColour`, and add `Sine(Time * PulseSpeed) * 0.5 + 0.5` multiplied by a `PulseStrength` of 5. `Add` both into `Emissive Color`. Apply it to a sphere and show the glowing silhouette breathing in and out.

### 6.2 Part B - Outlines, Decals and Profiling (Session B)

**Step 5 - Turn on the stencil.** In **Project Settings > Engine > Rendering > Postprocessing**, set **Custom Depth-Stencil Pass** to `Enabled with Stencil`. Select one `BP_DissolveProp` in the level and, under **Rendering**, tick **Render CustomDepth Pass** and set **CustomDepth Stencil Value** to 1.

**Step 6 - The outline.** Create `PPM_Outline`. Set **Material Domain** to `Post Process` and **Blendable Location** to `Scene Color After Tonemapping`. Build the four-neighbour test from 5.8, with `OutlineColour` (1, 0.6, 0) as a `VectorParameter`. Place a **Post Process Volume**, tick **Infinite Extent (Unbound)**, and add `PPM_Outline` to its post-process materials. The marked crate gets an orange outline, even where it is behind the other crate.

**Step 7 - A decal.** Create `MM_Decal_HazardMarking_Decal` with **Material Domain** `Deferred Decal` and **Blend Mode** `Translucent`. Reuse the hazard stripe graph from Notes 04 for `Base Color`, and wire a rectangle mask from a box distance field into `Opacity`. Place a **Decal Actor** on the floor and set its material. Rotate the decal and show that the stripes wrap over any bump or step in the floor.

**Step 8 - Find the expensive material.** Open `/Game/GD/Notes05/Lab08/Levels/Lvl_Lab08_Heavy`, the scene Lab 8 asks you to optimise. Switch the viewport to **Optimization Viewmodes > Shader Complexity**. The smoke planes are white where they overlap, and one prop is red on its own. Open that prop's material, click **Stats**, and read the instruction count and samplers. Type `stat GPU` in the console and press **Play** to compare the translucency and base pass times. Lab 8 fixes each problem in turn.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| Creating a dynamic instance every frame or every time a value changes | Memory climbs, values reset, frame time grows | Create it once on BeginPlay and store it in a variable |
| Setting the parameter on the source material instead of the dynamic instance | No change at runtime | Call Set Scalar Parameter Value on the stored `DynamicMaterial` |
| Parameter name typed differently from the material | Nothing happens and no error appears | Copy the name from the material's parameter node |
| Timeline Update not wired | The Timeline plays but the material never changes | Wire **Update** to the Set Scalar Parameter Value node |
| Dissolve never reaches fully gone or fully whole | A few pixels remain at 1, or holes show at 0 | Remap `Dissolve` slightly past both ends of the noise range |
| Collection parameter read from the wrong collection | The material ignores the weather | Check the **Collection** on every `CollectionParameter` node |
| Decal material not in the Deferred Decal domain | The decal shows nothing, or the material errors | Set **Material Domain** to `Deferred Decal` |
| Custom Depth-Stencil Pass left disabled | The outline never appears | Enable it with stencil in Project Settings |
| Post-process material in an unbound volume left with Infinite Extent off | The effect only shows inside the volume's box | Tick **Infinite Extent (Unbound)** |
| Optimising by guesswork | Hours spent on a cheap material | Find the red areas in Shader Complexity first, then read Stats |

---

## 8. Exercises

1. A level has 200 street lamps. Each should flicker at its own rate, and all should dim together at dawn. Which lamps get dynamic instances, which value goes in a collection, and how many Blueprint calls does dawn cost?
2. Rewrite the dissolve so it burns from the bottom of the object upwards. Hint: mix the noise with the object's local height before comparing it with `Dissolve`.
3. Extend `PPM_Outline` so stencil value 1 gives a red outline and stencil value 2 gives a green one.
4. A translucent explosion uses 40 overlapping sprites and drops the frame rate on low settings. List three changes that reduce its cost, and say which one you would try first.
5. Use `QualitySwitch` so the wet ground uses the full stone texture set on `High` and a flat colour on `Low`. Compare the Stats of both.

---

## 9. Reflective Questions

1. A dynamic instance and a collection can both change a value at runtime. When would using a collection for a per-object effect be a mistake?
2. Timelines and Tick can both animate a parameter. What does a Timeline give you that a Tick-based version does not?
3. A post-process outline and an inverted-hull outline mesh both outline an object. What does each cost, and which scales better with many outlined objects?
4. The shader complexity view showed overlapping smoke as the most expensive part of the screen. Why does the instruction count in Stats not reveal this?

---

## 10. How This Builds

- **Lab 7** (`lab_7_runtime_driven_effects`) applies Session A. You build one per-object effect driven by a dynamic instance and a Timeline, and one global effect driven by a collection.
- **Lab 8** (`lab_8_post_process_decals_and_profiling`) applies Session B. You build a stencil outline and a decal, then measure and fix the expensive scene from Step 8.
- **Week 5** (`06_geometry_and_scene_aware_materials`) moves vertices with World Position Offset and reads scene depth for soft intersections, building on the post-process scene textures from this week.
- **Week 6** combines Fresnel, panning normals, depth and translucency into water, and builds a toon shader as a post-process material.
- **Week 7** (`08_screen_space_post_processing`) builds full-screen filters, letterbox bars and selective colour on the post-process and stencil basics from Session B.
- **Week 9** rewrites this week's dissolve as HLSL in a Custom node and compares its cost with the node version.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **Custom Depth** | A second depth buffer that contains only the objects you mark, readable in post-process materials |
| **Custom Stencil** | A number from 0 to 255 stored for each marked object's pixels, used to tell objects apart in post-process |
| **Deferred decal** | A material projected from a box onto the surfaces inside it, changing their G-buffer values before lighting |
| **Dissolve** | An effect that cuts a surface away over time by comparing noise against a rising threshold |
| **Fresnel** | The increase in reflection and rim brightness where a surface turns away from the viewer |
| **Material Instance Dynamic** | A material instance created at runtime for one object, with parameters that can change every frame |
| **Material Parameter Collection** | An asset of named values that many materials read, changed with one Blueprint call |
| **Overdraw** | Shading the same pixel more than once, usually because translucent surfaces overlap |
| **Post-process material** | A material that runs once for every screen pixel after the scene renders |
| **Quality switch** | A node that compiles one branch per quality level and uses the branch matching the player's settings |
| **Shader complexity** | A viewport view that colours each pixel by the instructions spent shading it |
| **Timeline** | A Blueprint node that plays a curve over time and fires Update while it plays |
