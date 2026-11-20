---
title: "Geometry and Scene-Aware Materials"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: notes
session: 5
topic_code: MAT-05
difficulty_tier: intermediate
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [04_procedural_textures_and_uv_maths, 05_runtime_materials_and_performance]
feeds_into: [lab_9_world_position_offset_and_parallax, lab_10_depth_effects_and_flipbooks, 07_composite_effects]
engine_version: "UE 5.7"
---

# Geometry and Scene-Aware Materials

---

## 1. Key Concepts

1. **World Position Offset moves vertices, not pixels.** A material can only bend a mesh where the mesh has vertices, so a four-vertex plane cannot ripple.
2. **Mask the movement.** Grass roots and flag poles must not move, so every wind or wave offset is multiplied by a mask that is 0 where the mesh is anchored.
3. **Add the world position to the phase.** Without it, every tree in the level sways in perfect step, which the eye reads as fake at once.
4. **Moving vertices does not move normals.** Lighting is computed from the original surface, so a waving flag needs its normal bent to match, or it lights as if flat.
5. **Offsets past the bounds get culled.** The engine decides visibility from the mesh's bounds before the material runs, so a large offset needs a larger Bounds Scale.
6. **Bump Offset fakes depth with one lookup.** It shifts UVs towards the viewer in proportion to height, which is cheap and convincing at shallow angles only.
7. **Parallax occlusion mapping ray-marches the height.** It looks deep at any angle, but its cost grows with the number of steps on every pixel, so use it where the camera gets close.
8. **Depth lets a surface know what is behind it.** Comparing the scene's depth with the pixel's own depth tells a translucent surface how close it is to the geometry behind, which softens or highlights intersections.
9. **Distance fields know what is nearby.** Distance to Nearest Surface reads the global distance field, so it sees objects in every direction, not only the one behind the pixel.
10. **A flipbook is a grid of frames and a clock.** Pick a frame from the time, turn it into a cell in the grid, and offset the UVs, so one texture plays an animation.

---

## 2. Framing Question

So far every material has changed the colour of a surface that stays where the mesh put it. A flag should ripple, grass should bend in the wind, a cobbled floor should look deep without millions of triangles, and a shield should glow where it cuts through the ground. Can a material change where a surface is, or what it knows about the world around it?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Explain what World Position Offset changes and what it costs | 5.1 |
| Make foliage sway with a masked, phase-shifted offset | 5.2 |
| Make a flag wave, and bend its normal to match | 5.3, 5.4 |
| Prevent culling of offset meshes with Bounds Scale | 5.5 |
| Fake depth with Bump Offset and with parallax occlusion mapping, and compare their cost | 5.6, 5.7 |
| Read scene depth and pixel depth, and soften intersections with Depth Fade | 5.8 |
| Highlight where a translucent surface meets geometry | 5.9 |
| Use Distance to Nearest Surface, and state its requirement | 5.10 |
| Play an animation from a flipbook texture | 5.11 |

---

## 4. Why This Matters

Animated geometry in games is mostly done by materials, not by skeletons. Every blade of grass, every banner, every swaying tree and every bobbing buoy in an open world moves through World Position Offset, because the GPU can move millions of vertices for almost nothing while the CPU would struggle with a few hundred animated actors.

The second half of this week gives materials awareness of the scene. Soft particles, shoreline foam, force fields, fog that thins near the camera, and fire that does not cut hard lines through the floor all depend on a material reading depth or distance. These effects are cheap compared with real geometry, and they are the difference between an effect that looks pasted on and one that sits in the world.

---

## 5. Core Ideas

Sections 5.1 to 5.7 are Session A, moving vertices and faking depth. Sections 5.8 to 5.11 are Session B, reading the scene and animating textures.

### 5.1 World Position Offset

Every material so far has run in the **pixel shader**, once per pixel. The **World Position Offset** input (WPO) runs in the **vertex shader**, once per vertex, before the triangles are drawn. Whatever you wire into it is added to each vertex's position in world space, in centimetres.

Two consequences follow:

- **Detail depends on the mesh.** A plane with four vertices can tilt, but it cannot ripple. A flag needs a mesh with many rows and columns of vertices. This week uses `SM_Plane_Dense` from `/Game/GD/Shared/Meshes`, a 200 x 100 unit plane with 64 x 32 quads.
- **It is cheap.** A mesh has far fewer vertices than the screen has pixels, so even a long WPO graph costs little compared with the same graph in the pixel shader.

The offset is in world space. To move a vertex along the mesh's own axes, such as out of a flag's surface, transform a local-space vector to world space with the `TransformVector` node, **Source** `Local Space` and **Destination** `World Space`.

### 5.2 Foliage Sway

Wind pushes the top of a plant further than its base. Three parts make a believable sway:

1. **Motion:** `Sine(Time * WindSpeed + phase)`, a smooth back-and-forth.
2. **Phase from position:** `phase = Dot(ObjectPosition, (0.01, 0.01, 0))` or the vertex's own `AbsoluteWorldPosition` scaled down. Each plant then starts its sway at a different moment, so the field ripples rather than marching in step.
3. **Mask:** multiply by how far up the mesh the vertex is. On a foliage card, `OneMinus` of the V coordinate is 0 at the bottom edge and 1 at the top, and squaring it keeps the base still and lets the tip bend most.

`offset = (WindDirection * Sine(...)) * Amplitude * mask`, wired into World Position Offset. Unreal ships the `SimpleGrassWind` material function, with inputs `WindIntensity`, `WindWeight` and `WindSpeed`. It does the same job with a rotation instead of a straight offset. Build the simple version first so you know what the function does.

### 5.3 A Waving Flag

A flag is a wave travelling along the cloth, growing from the pole outwards:

1. `x = U`, 0 at the pole and 1 at the free edge.
2. `wave = Sine(x * WaveCount * 6.283185 - Time * WaveSpeed)`, with **Period** 6.283185. Subtracting time makes the wave travel away from the pole.
3. `height = wave * Amplitude * x`. Multiplying by `x` pins the pole edge and lets the free edge flap most.
4. `TransformVector (0, 0, 1)` from local to world space, multiplied by `height`, into World Position Offset. On `SM_Plane_Dense`, local Z points out of the cloth.

### 5.4 Normals After WPO

Moving vertices does not change the normals the lighting uses. The flag above ripples, but it lights as if it were still flat: no light and dark bands across the folds. The fix is to bend the normal by the slope of the wave.

The slope is how fast `height` changes along the cloth. For the wave in 5.3, it is approximately:

`slope = Cosine(x * WaveCount * 6.283185 - Time * WaveSpeed) * WaveCount * 6.283185 * Amplitude * x / FlagLength`

where `FlagLength` is the cloth's length in centimetres, 200 for `SM_Plane_Dense`. A surface tilted by that slope has the tangent-space normal `Normalize(-slope, 0, 1)`. Wire that into `Normal`, and the folds catch the light. Make a `NormalStrength` parameter that scales `slope`, so you can exaggerate or soften the effect.

### 5.5 Bounds and Culling

Before any material runs, the engine checks each mesh's **bounds**, the box around it, against the camera to decide whether to draw it at all. The bounds come from the original vertices. If WPO pushes the mesh outside its box, the engine may cull it while it is still visible, and the mesh pops in and out at the edge of the screen.

`SM_Plane_Dense` is perfectly flat, so its box has no depth, and a flag waving 20 cm out of its plane can vanish when the camera looks along it. Select the actor and raise **Bounds Scale**, under **Rendering**, until the box covers the full movement. Larger bounds make culling slightly less efficient, so use the smallest value that works.

### 5.6 Bump Offset

A cobbled floor should look deeper than a flat plane. **Parallax** shifts the texture so that high parts appear closer to the viewer than low parts.

The `BumpOffset` node does this with one lookup. It takes a **Height** value, usually a height map sampled at the normal UVs, and a **HeightRatio** that scales the effect, and returns new UVs shifted towards the camera in proportion to height. Wire those UVs into every texture sample of the surface: base colour, normal and ORM.

It costs one extra texture sample and a few instructions. It is convincing when you look down at the surface, and it breaks down at grazing angles, where tall parts should hide the parts behind them but cannot.

### 5.7 Parallax Occlusion Mapping

**Parallax occlusion mapping** (POM) traces a short ray through the height map for every pixel. It steps along the view direction until the ray drops below the height, then uses that point's UVs. Tall parts now hide what lies behind them, and the depth holds up at low angles.

Use the engine's `ParallaxOcclusionMapping` material function. Its main inputs:

| Input | Meaning |
|:---|:---|
| `Heightmap Texture` | A `TextureObject` holding the height map |
| `UVs` | The tiled UVs you would otherwise use |
| `Height Ratio` | Depth of the effect. About 0.05 for cobbles |
| `Min Steps`, `Max Steps` | Ray steps at head-on and grazing angles. 8 and 32 are typical |
| `Heightmap Channel` | Which channel holds height, as a vector: (1, 0, 0, 0) for R |

Its `Parallax UVs` output replaces the UVs for every texture sample. Its real cost is the number of steps times a texture sample, on every pixel. **Stats** hides this: the compiler counts the loop body once, so POM shows only about 10% more instructions than Bump Offset. To see the true cost, fill the screen with each surface and compare the base pass time in `stat GPU`, as in Notes 05. Use POM on surfaces the camera gets close to, such as floors near the player, and Bump Offset everywhere else.

This week uses `T_Stone_Height` from `/Game/GD/Shared/Textures/Stone`, the cobblestone's ambient occlusion saved as height, as Notes 03 explained.

### 5.8 Scene Depth, Pixel Depth and Depth Fade

Two depths describe every translucent pixel:

| Node | Value |
|:---|:---|
| `PixelDepth` | Distance from the camera to the surface being shaded |
| `SceneDepth` | Distance from the camera to the opaque geometry behind it |

`SceneDepth - PixelDepth` is how far the opaque scene lies behind this translucent pixel. It is large in open air and falls to 0 where the surface touches the ground or cuts through an object. `SceneDepth` can only be read by translucent materials, because an opaque material is itself writing the depth.

Translucent sprites, such as smoke, show a hard line where they cut into the floor. The `DepthFade` node packages the fix: it returns 0 where the depth difference is 0 and rises to 1 over **FadeDistance** centimetres. Multiply it into **Opacity**, and the sprite fades out as it meets the ground. This is called a **soft particle**.

### 5.9 Intersection Highlights

Turn Depth Fade around and you get a highlight. `1 - DepthFade` is 1 exactly where the surface meets geometry and 0 a short distance away, a glowing line along every intersection.

A force field combines three signals:

1. **Intersection:** `1 - DepthFade` with **FadeDistance** about 20, multiplied by an `IntersectionColour`.
2. **Rim:** `Fresnel` from Notes 05, brightest at the silhouette.
3. **Body:** a faint constant, so the middle of the sphere is barely visible.

`Add` them into `Emissive Color`, and use their sum, clamped with `Saturate`, as `Opacity`. Use the **Translucent** blend mode, and **Unlit** shading so the field glows rather than catching the sun.

### 5.10 Distance to Nearest Surface

Depth only sees what is behind a pixel along the view ray. Foam around a rock in water should appear on every side of the rock, including the sides the camera cannot see past. The `DistanceToNearestSurface` node returns the distance in centimetres from the pixel to the nearest mesh in the scene, in any direction.

It reads the **global distance field**, a low-resolution volume describing every mesh's shape. This has one requirement: **Generate Mesh Distance Fields** must be on, in **Project Settings > Engine > Rendering**. It is on in this project. Because the field is low resolution, edges are soft and small objects may be missed, which suits foam, wetness lines and proximity glows, not sharp detail.

`foam = 1 - SmoothStep(0, FoamWidth, DistanceToNearestSurface)` makes a band hugging every object that touches the surface.

### 5.11 Flipbooks

A **flipbook**, or **sprite sheet**, packs the frames of an animation into one texture as a grid. `T_Fire_Flipbook` in `/Game/GD/Shared/Textures/Fire` holds 64 frames of a looping flame in 8 columns and 8 rows. Playing it is cell maths from Notes 04:

1. `frame = Floor(Time * FPS)`, then `Fmod` by 64 so it loops.
2. `column = Fmod(frame, 8)` and `row = Floor(frame / 8)`.
3. `UV' = (TextureCoordinate + (column, row)) / 8`.
4. Sample the texture with `UV'`.

The engine's `FlipBook` material function does the same with inputs `Texture`, `Animation Phase (0-1)`, `Number of Columns` and `Number of Rows`. Feed `Frac(Time * FPS / 64)` into the phase. Fire is light, so use the **Additive** blend mode and **Unlit** shading, and wire the colour into `Emissive Color`. The flipbook's alpha channel holds the flame's density, useful as `Opacity` for a translucent version.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes06/Demo/Levels/Lvl_Notes06`. The finished demo materials are in `/Game/GD/Notes06/Demo/Materials`. To reproduce the demo, create `/Game/Student/Notes06/MyWork` and save your assets there.

The shared assets are `SM_Plane_Dense` in `/Game/GD/Shared/Meshes`, `T_Fire_Flipbook` in `/Game/GD/Shared/Textures/Fire`, and the stone and leaf sets in `/Game/GD/Shared/Textures`.

### 6.1 Part A - Moving and Deepening Surfaces (Session A)

**Step 1 - Wind on leaf cards.** Create `MM_Foliage_WindSway_Masked`, starting from the leaf card graph in Notes 01 Step 7: **Masked**, **Two Sided Foliage**, `T_Leaf_Albedo` RGB to `Base Color` and A to `Opacity Mask`. Then build:

1. `mask = Power(OneMinus(ComponentMask (G) of TextureCoordinate), 2)`.
2. `phase = Dot(ObjectPosition, (0.01, 0.01, 0))`.
3. `sway = Sine(Time * WindSpeed + phase)`, **Period** 6.283185, with `WindSpeed` default 1.5.
4. `sway * mask * WindAmount * (1, 0.3, 0)` into World Position Offset, with `WindAmount` default 15.

Place six upright planes using it, scattered in a clump. Press **Simulate**. The tops sway, the bases stay still, and no two cards move in step.

**Step 2 - The flag.** Create `MM_Prop_WavingFlag_Opaque`. Build the wave from 5.3 with `WaveCount` 1.5, `WaveSpeed` 4 and `Amplitude` 12, and wire `TransformVector (0, 0, 1) * height` into World Position Offset. Wire a strong colour into `Base Color`. Place `SM_Plane_Dense`, rotate it upright beside a thin cylinder for a pole, and apply the material. Press **Simulate**. The flag ripples, but its shading stays flat.

**Step 3 - Bend the normal.** Add the slope from 5.4, with `NormalStrength` default 1 and `FlagLength` 200, and wire `Normalize(-slope, 0, 1)` into `Normal`. Simulate again. Light and dark bands now run along the folds.

**Step 4 - Bounds.** Orbit until you look along the flag's edge. It may vanish. Select it and set **Bounds Scale** to 2. It stays visible.

**Step 5 - Bump Offset and POM.** Create `MM_Env_BumpOffsetStone_Opaque` from the stone set, with a `TextureCoordinate` tiled 4 times. Add `BumpOffset` with **HeightRatio** 0.05 and `T_Stone_Height` sampled at the tiled UVs on **Height**, and wire its output into the UVs of all three stone samples. Then create `MM_Env_POMStone_Opaque`, with the `ParallaxOcclusionMapping` function in place of `BumpOffset`: a `TextureObject` of `T_Stone_Height` on `Heightmap Texture`, the tiled UVs on `UVs`, 0.05 on `Height Ratio`, 8 and 32 on `Min Steps` and `Max Steps`, and (1, 0, 0, 0) on `Heightmap Channel`. Wire `Parallax UVs` into all three stone samples.

Apply each to a floor tile side by side. Look straight down: they look alike. Lower the camera towards the floor: the Bump Offset tile flattens and swims, while the POM tile keeps deep gaps between the stones. Click **Stats** on both: POM reads only about 10% more instructions, because the loop is counted once. Fill the screen with each tile in Play and compare `stat GPU` to show the real difference.

### 6.2 Part B - Reading the Scene (Session B)

**Step 6 - Force field.** Create `MM_FX_ForceField_Trans`. Set **Blend Mode** `Translucent` and **Shading Model** `Unlit`. Build 5.9 with `IntersectionWidth` 20 on the `DepthFade` **FadeDistance**, a cyan `FieldColour` and `Fresnel` with **ExponentIn** 4. Apply it to a large sphere half sunk into the floor, with a cube passing through it. A bright line traces where the sphere meets the floor and the cube.

**Step 7 - Fire flipbook.** Create `MM_FX_FireFlipbook_Add` with **Blend Mode** `Additive` and **Shading Model** `Unlit`. Build the cell maths from 5.11 with `FPS` default 24, sample `T_Fire_Flipbook`, and multiply the colour by `FireStrength` 4 into `Emissive Color`. Apply it to an upright plane and press **Simulate**. The flame flickers through its 64 frames and loops.

**Step 8 - The same with the function.** Duplicate the fire material, replace the cell maths with the `FlipBook` function, and show that it plays the same animation from one node.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| WPO on a low-poly mesh | The surface tilts but never ripples | Use a mesh with enough vertices, such as `SM_Plane_Dense` |
| No mask on foliage sway | The whole card slides, base included | Multiply by a mask that is 0 at the base |
| No phase from position | Every plant sways in step | Add `Dot(ObjectPosition, ...)` to the sine's input |
| Offset in local units wired straight into WPO | The flag waves sideways after you rotate it | Transform the local direction to world space with `TransformVector` |
| Flag lit as flat | Folds have no light and dark bands | Bend the normal by the wave's slope |
| Mesh vanishes at screen edges | Popping when the camera moves | Raise **Bounds Scale** on the actor |
| Bump Offset at a grazing angle | Stones smear and swim | Use POM where the camera gets low |
| POM with high step counts everywhere | Large surfaces turn red in Shader Complexity | Keep **Max Steps** low, or use POM only near the player |
| SceneDepth in an opaque material | Compile error | Use **Translucent** for depth-reading effects |
| Distance to Nearest Surface returns nothing | No foam or glow appears | Turn on **Generate Mesh Distance Fields** |
| Flipbook shows several frames at once | A grid of flames on one plane | Divide the offset UVs by the number of columns and rows |

---

## 8. Exercises

1. A field contains 10,000 grass cards with 8 vertices each, and the screen is 1920 x 1080. Compare the number of times a WPO graph and a pixel graph of the same length run per frame.
2. Change the flag so it waves in two directions at once by adding a second, smaller wave along V. Extend the normal to include its slope.
3. Measure the instruction counts of Bump Offset and POM, then reduce POM's **Max Steps** to 8. At what angle does the cheaper POM start to look worse than Bump Offset?
4. Use `DepthFade` to make the fire flipbook a soft particle, so it fades instead of cutting a line where it meets the floor.
5. Make a buoy bob on water: offset the whole mesh up and down with a sine of time, and tilt it slightly with a second sine of a different speed.

---

## 9. Reflective Questions

1. WPO moves vertices on the GPU but leaves collision where the mesh was. Which gameplay situations would that break, and how do games hide it?
2. POM looks better than Bump Offset at every angle. Why would a shipped game still use Bump Offset on most surfaces?
3. Depth Fade only sees the geometry directly behind a pixel, while Distance to Nearest Surface sees in every direction. Describe an effect where that difference is visible.
4. A flipbook stores 64 frames in one texture. What do you lose by doing that instead of simulating the flame with noise as in Notes 04?

---

## 10. How This Builds

- **Lab 9** (`lab_9_world_position_offset_and_parallax`) applies Session A. You build a waving flag with corrected normals, then compare Bump Offset and POM on one surface.
- **Lab 10** (`lab_10_depth_effects_and_flipbooks`) applies Session B. You build a force field with intersection glow and a fire flipbook.
- **Week 6** (`07_composite_effects`) combines WPO waves, depth colour, panning normals, refraction and Fresnel into water, and uses a flipbook-style scanline for holograms.
- **Week 9** writes parallax occlusion mapping as an HLSL loop in a Custom node, where the cost of each ray step becomes visible in the code.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **Bounds** | The box around a mesh that the engine uses to decide whether it is on screen |
| **Bump Offset** | A one-sample parallax technique that shifts UVs towards the viewer by height |
| **Depth Fade** | A node that fades a translucent surface out where it approaches the geometry behind it |
| **Distance field** | A volume storing the distance to the nearest surface at each point |
| **Flipbook** | A texture holding the frames of an animation as a grid, played by offsetting UVs |
| **Parallax occlusion mapping** | A parallax technique that ray-marches a height map per pixel, so tall parts hide those behind |
| **Pixel depth** | The distance from the camera to the surface being shaded |
| **Scene depth** | The distance from the camera to the opaque geometry behind the current pixel |
| **Soft particle** | A translucent sprite that fades out where it meets geometry instead of cutting a hard line |
| **Vertex shader** | The stage that runs once per vertex, where World Position Offset is applied |
| **World Position Offset** | A material input that adds a world-space offset to each vertex |
