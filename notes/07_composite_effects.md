---
title: "Composite Effects"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: notes
session: 6
topic_code: MAT-06
difficulty_tier: advanced
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [05_runtime_materials_and_performance, 06_geometry_and_scene_aware_materials]
feeds_into: [lab_11_water_and_hologram, lab_12_toon_shading_and_landscape, 08_screen_space_post_processing]
engine_version: "UE 5.7"
---

# Composite Effects

---

## 1. Key Concepts

1. **A finished effect is several small effects added together.** Water is depth colour plus moving normals plus refraction plus Fresnel plus foam, and each part is simple on its own.
2. **Build one layer at a time and look at each.** Wire each signal straight into Emissive Color before you combine it, so you know what every part contributes.
3. **Drive every layer from a parameter.** A composite effect has many numbers, and an art director will want to change all of them without opening the graph.
4. **Two normal maps panning in different directions never repeat visibly.** One panning texture reads as a sliding image, and two at different speeds and scales read as moving water.
5. **Depth is the water's colour.** Shallow water shows the floor, deep water absorbs light, so the depth difference from Notes 06 drives colour, opacity and foam.
6. **Additive light needs no opacity.** A hologram only adds brightness, so it never darkens what is behind it and never needs sorting tricks.
7. **Toon shading quantises light, not colour.** Divide out the surface colour, step the lighting into bands, then multiply the colour back, or the bands smear across textures.
8. **Outlines come from discontinuities.** A sudden jump in depth between neighbouring pixels marks a silhouette, which is the same neighbour test as the stencil outline in Notes 05.
9. **Let the terrain paint itself.** An auto-material chooses grass or rock from the slope of the surface, so a level designer can sculpt without painting every layer by hand.
10. **World-aligned UVs make terrain tile evenly.** Terrain is stretched over hills, so sampling textures by world position keeps their scale the same everywhere.

---

## 2. Framing Question

Every effect in the last five weeks did one thing: a dissolve, a wave, an outline, a depth glow. A game's water is not one thing. It reflects the sky, shows the floor in the shallows, goes dark in the deep, moves constantly, foams where it touches the shore, and bends what is under it. How do you plan an effect that is ten techniques at once, and keep it cheap enough to cover half the screen?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Break a target effect into layers and build them in order | 5.1 |
| Colour and fade water by depth | 5.2 |
| Animate water with two panning normal maps in world space | 5.3 |
| Combine refraction, Fresnel and foam into finished water | 5.4, 5.5 |
| Build a hologram from scanlines, rim, flicker and additive blending | 5.6 |
| Quantise lighting into toon bands in a post-process material | 5.7 |
| Draw outlines from depth discontinuities | 5.8 |
| Blend landscape layers by slope, using world-aligned UVs | 5.9, 5.10 |

---

## 4. Why This Matters

Players remember the effects that combine everything: the water in a harbour, the shield over a base, a stylised world with ink lines, a mountain where snow and rock sit where they should. None of these introduces a new node. They are the techniques from Weeks 1 to 5 put together with care. This week is about that care: planning the layers, building them in an order you can check, and keeping the total cost in mind.

The second half covers two styles a games graduate is expected to know. Toon shading is how many stylised games are made, and it is built as a full-screen post-process material. Landscape auto-materials are how open worlds are textured, because no artist can hand-paint every slope.

---

## 5. Core Ideas

Sections 5.1 to 5.6 are Session A, water and holograms. Sections 5.7 to 5.10 are Session B, toon shading and landscapes.

### 5.1 Planning a Composite Effect

Start from a reference, a photo or a frame from another game, and list what you see as separate layers. For water:

| Layer | What you see | Technique | From |
|:---|:---|:---|:---|
| Depth colour | Green-blue shallows, dark blue depths | `SceneDepth - PixelDepth` into a `Lerp` | Notes 06 5.8 |
| Ripples | A moving, never-repeating surface | Two panning normal maps | Notes 01 Panner |
| Refraction | The floor bends under the surface | Index of refraction | Notes 01 glass |
| Reflection | More sky at a low angle | `Fresnel` | Notes 05 5.5 |
| Foam | White where water meets the shore | Depth difference near 0, broken by noise | Notes 06 5.9 |

Then build the layers in that order, and check each on its own by wiring it into `Emissive Color` before combining. When something looks wrong in the finished effect, you can switch back to any single layer.

### 5.2 Depth Colour

Water absorbs light, so the further light travels through it, the darker and bluer it gets. The depth from Notes 06 tells you how much water lies under each pixel:

1. `depth = SceneDepth - PixelDepth`. 0 at the shore, large over deep water.
2. `deep = Saturate(depth / DepthRange)`, with `DepthRange` about 300 cm. 0 in the shallows, 1 at full depth.
3. `colour = Lerp(ShallowColour, DeepColour, deep)`.
4. `opacity = Lerp(0.2, 0.95, deep)`. Shallow water is nearly clear, deep water nearly opaque.

The material must be **Translucent** to read `SceneDepth`. Set **Lighting Mode** to `Surface ForwardShading` under **Translucency**, as for the glass in Notes 01, so the surface gets sharp highlights.

### 5.3 Panning Normals in World Space

A water plane is often huge and made of many tiles, so its UVs are not a good place to sample a normal map. Sample by world position instead:

`uv = AbsoluteWorldPosition.xy / WaveScale`, with `WaveScale` about 400 cm per repeat.

Then pan two copies of `T_Water_Normal` from `/Game/GD/Shared/Textures/Water` in different directions at different speeds and scales:

1. `Panner` with **Speed** (0.02, 0.01) on `uv`.
2. `Panner` with **Speed** (-0.015, 0.02) on `uv * 0.7`.
3. Sample the normal map with each and combine them. Adding the two normals and normalising the result is a good approximation, and the `BlendAngleCorrectedNormals` function does it exactly.

Because the two patterns move at different speeds and scales, they never line up in the same way twice, so the eye cannot find the repeat. A `NormalStrength` parameter that lerps between a flat normal (0, 0, 1) and the combined normal lets you calm or roughen the water.

### 5.4 Refraction and Fresnel

**Refraction.** Set **Refraction Method** to `Index Of Refraction` and wire 1.33, water's index, into `Refraction`. The floor bends under the moving normals.

**Fresnel.** Water reflects much more at grazing angles: look straight down at a lake and you see the bottom, look across it and you see the sky. `Fresnel` with **ExponentIn** 5 rises towards the horizon. Leave its **Normal** input empty, because by default it uses the pixel normal, which already includes the water normal, so the reflection ripples with the surface. Add it to the opacity, so distant water turns more reflective and less transparent. Keep `Roughness` low, about 0.05, so the reflection of the sky stays sharp.

### 5.5 Foam

Foam forms where water meets the shore, which is where the depth falls to 0:

1. `edge = 1 - SmoothStep(0, FoamWidth, depth)`, with `FoamWidth` about 25 cm.
2. Break it up with noise: sample `T_Noise_Macro` from `/Game/GD/Shared/Textures/Noise` at `AbsoluteWorldPosition.xy / 150`, panned slowly, and keep foam only where the edge value is above the noise: `foam = SmoothStep(noise, noise + 0.1, edge)`. Close to the shore the edge value beats almost any noise, so the foam is solid. Further out only the darkest noise passes, so the foam breaks into patches.
3. `Lerp(colour, FoamColour, foam)`, and raise the opacity to 1 where there is foam.

The finished water costs several texture samples, a depth read and a refraction pass on every water pixel. On a large lake that is most of the screen, so it is worth checking it in the shader complexity view from Notes 05.

### 5.6 Holograms

A hologram is light with no surface: it only adds brightness. Use **Blend Mode** `Additive` and **Shading Model** `Unlit`. Additive blending never darkens what is behind it, needs no opacity, and is cheap.

Four layers make it read as a projection:

1. **Rim:** `Fresnel` with **ExponentIn** about 2, so the silhouette glows most.
2. **Scanlines:** horizontal lines that scroll upwards. `lines = Power(TriangleWave(AbsoluteWorldPosition.z * LineDensity - Time * ScrollSpeed), 8)`, using the triangle wave from Notes 04. World Z keeps the lines level across the whole figure.
3. **Body:** a faint constant, about 0.08, so the inside of the figure is visible.
4. **Flicker:** `0.85 + 0.15 * Sine(Time * FlickerSpeed)`, with a fast `FlickerSpeed` such as 23, multiplied over everything.

`(rim * 2 + lines * 0.6 + 0.08) * flicker * HoloColour * Strength -> Emissive Color`. Apply it to a character mesh, such as `SKM_Quinn_Simple` in `/Game/Characters/Mannequins/Meshes`, for the classic effect.

### 5.7 Toon Shading as Post-Process

Toon, or cel, shading replaces smooth lighting with a few flat bands. Doing it in a post-process material applies the look to the whole scene at once, with no changes to any object's material.

The difficulty is that the rendered image mixes surface colour and lighting together. Dividing by the surface colour separates them:

1. Set **Material Domain** `Post Process` and **Blendable Location** `Scene Color After Tonemapping`.
2. `lit = PostProcessInput0`, the rendered image, and `albedo = SceneTexture BaseColor`, the surface colour from the G-buffer.
3. `light = Luminance(lit) / Max(Luminance(albedo), 0.02)`. Luminance is `Dot(colour, (0.3, 0.59, 0.11))`. This is how brightly each pixel is lit, without its texture.
4. `band = Floor(light * Bands + 0.5) / Bands`, with `Bands` about 3. The lighting now jumps between a few levels.
5. `toon = albedo * band * Brightness`, with `Brightness` about 0.6. The rendered image has already been tonemapped and is brighter than the raw base colour, so the bands need scaling down.
6. Keep the sky unchanged: where `SceneDepth` is very large, use `lit` instead of `toon`.

Because the bands are taken from the lighting, not the colour, a textured surface keeps its texture inside each band.

### 5.8 Depth Outlines

Ink outlines sit at silhouettes, where the depth jumps between neighbouring pixels. The neighbour test from Notes 05 5.8 works on depth too:

1. Read `SceneDepth` at the pixel and at its four neighbours, one pixel away, using `ScreenPosition` and `InvSize`.
2. `jump = Sum(Abs(neighbour - centre)) / centre`. Dividing by the centre depth makes the test work at any distance, because far objects have larger depth values.
3. `outline = SmoothStep(Threshold, Threshold * 2, jump)`, with `Threshold` about 0.05.
4. `Lerp(toon, OutlineColour, outline)` into `Emissive Color`.

The outline appears wherever one object passes in front of another and along every silhouette, with no setup on the objects themselves.

### 5.9 Landscape Layers and Auto-Materials

A **Landscape** in Unreal is a large terrain mesh sculpted in **Landscape** mode. Its material usually blends several surfaces, called **layers**, such as grass, rock and snow. There are two ways to decide where each layer goes:

- **Painted layers.** The `LandscapeLayerBlend` node lists layers by name, and the artist paints a weight for each in Landscape mode. Precise, but slow over a large world.
- **Auto-material.** The material decides from the surface itself: slope, height or both. No painting is needed, and the layers follow the sculpt automatically.

Most games combine them: an auto-material sets the base, and painting adds paths and patches by hand.

### 5.10 Slope Blending with World-Aligned UVs

Grass grows on flat ground. Rock shows on cliffs. The world-space normal says how flat each point is:

1. `slope = ComponentMask (B) of VertexNormalWS`. 1 on flat ground, 0 on a vertical cliff.
2. `grass = SmoothStep(SlopeStart, SlopeEnd, slope)`, with `SlopeStart` 0.7 and `SlopeEnd` 0.85. 0 on steep ground, 1 on flat.
3. `Lerp(rock, grass, grass)` for base colour, normal and ORM separately, as in the splat blend from Notes 03.

Sample both layers with **world-aligned UVs**: `uv = AbsoluteWorldPosition.xy / TileSize`, with `TileSize` about 400 cm. Terrain UVs stretch over steep hills, so world-aligned sampling keeps the texture the same size everywhere. On very steep cliffs, top-down world UVs smear too. Triplanar mapping from the Lab 4 and Lab 9 Extensions fixes that.

This week uses `T_Grass_*` and `T_Rock_*` from `/Game/GD/Shared/Textures/Grass` and `Rock`, and `SM_Terrain_Hills` from `/Game/GD/Shared/Meshes`, a terrain mesh with flat ground and steep hills. The same material works unchanged on a real Landscape, because it reads only the world position and the normal.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes07/Demo/Levels/Lvl_Notes07`. It holds three areas: a terrain flooded with water, the hologram on a pedestal, and a second terrain with props inside the toon volume. The finished demo materials are in `/Game/GD/Notes07/Demo/Materials`. To reproduce the demo, create `/Game/Student/Notes07/MyWork` and save your assets there.

### 6.1 Part A - Water, Layer by Layer (Session A)

**Step 1 - Depth colour.** Create `MM_Env_Water_Trans`. Set **Blend Mode** `Translucent`, **Lighting Mode** `Surface ForwardShading` and **Refraction Method** `Index Of Refraction`. Build 5.2 with `DepthRange` 300, `ShallowColour` (0.1, 0.5, 0.5) and `DeepColour` (0.01, 0.05, 0.12). Wire the depth colour into `Base Color` and the depth opacity into `Opacity`. Apply it to a large plane floating over the terrain, so hills rise through it. The water is pale over the shallows and dark over the hollows.

**Step 2 - Ripples.** Add the two world-space panning normals from 5.3, with `WaveScale` 400 and `NormalStrength` 0.6, into `Normal`. The surface now moves, and the repeat is impossible to spot.

**Step 3 - Refraction and Fresnel.** Wire 1.33 into `Refraction`, 0.05 into `Roughness`, and add `Fresnel * 0.3` to the opacity. The hills bend under the surface, and the far water reflects the sky.

**Step 4 - Foam.** Add the foam from 5.5 with `FoamWidth` 25. A broken white line forms wherever a hill meets the water.

**Step 5 - Hologram.** Create `MM_FX_Hologram_Add` with **Blend Mode** `Additive` and **Shading Model** `Unlit`, and build 5.6. Place `SKM_Quinn_Simple` on a small cylinder and apply the material. Show each layer on its own by disconnecting the others, then show them combined.

### 6.2 Part B - Toon World and Auto-Terrain (Session B)

**Step 6 - Slope blend.** Create `MM_Terrain_SlopeBlend_Opaque` from 5.10. Apply it to `SM_Terrain_Hills`. Grass covers the flat ground and rock shows on the steep hills. Drag `SlopeStart` and `SlopeEnd` closer together to sharpen the boundary.

**Step 7 - Toon bands.** Create `PPM_Toon` from 5.7 with `Bands` 3. In the demo level it sits in `PPV_Toon`, a volume around the second terrain without **Infinite Extent (Unbound)**, so the toon look switches on as the camera flies into it. Shadowed sides turn to one flat dark band, and textures stay readable inside each band.

**Step 8 - Ink outlines.** Add the depth outline from 5.8 to `PPM_Toon`. Every hill, prop and the horizon gain an ink line. Set the volume's **Blend Weight** to 0 and back to 1 to compare the toon and normal looks.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| Water left Opaque | Compile error on `SceneDepth` | Set **Blend Mode** `Translucent` |
| Default translucency lighting | Soft, smeared highlights on water | Set **Lighting Mode** `Surface ForwardShading` |
| One panning normal | The surface reads as a sliding image | Pan two copies in different directions at different speeds and scales |
| Water normals from mesh UVs on tiled planes | Visible seams between water tiles | Sample by `AbsoluteWorldPosition` |
| Foam without noise | A clean, artificial white band | Break the edge with panned noise |
| Hologram left Translucent and lit | The figure looks grey and sorts badly | Use `Additive` and `Unlit` |
| Scanlines from UVs | Lines run at angles across the character | Use world Z |
| Toon bands taken from the final colour | Bands smear across textures | Divide by base colour before banding |
| Outline threshold in raw depth | Outlines near the camera only, or everywhere far away | Divide the depth jump by the centre depth |
| Terrain sampled with mesh UVs | Textures stretched on hillsides | Use world-aligned UVs |
| Slope from the pixel normal with a strong normal map | Rock speckles across flat ground | Use `VertexNormalWS` for the slope |

---

## 8. Exercises

1. List the layers you would need for lava flowing between rocks, and say which week's technique provides each.
2. Add a third terrain layer, snow, above a height threshold read from `AbsoluteWorldPosition.z`, and make it prefer flat ground.
3. Give the water World Position Offset waves from Notes 06, and correct its normal to match.
4. Change `PPM_Toon` so the darkest band is tinted blue rather than grey, a common stylistic choice.
5. Measure the water's cost in the shader complexity view and with `stat GPU` while it fills the screen. Which layer would you remove first on low settings?

---

## 9. Reflective Questions

1. You built the water layer by layer and checked each one. How would you debug a finished composite effect that you did not build yourself?
2. A hologram is additive, while water is translucent. What would go wrong if you swapped them?
3. Toon shading as a post-process applies to everything. Which objects would you want to exclude from it, and how could Custom Stencil help?
4. An auto-material follows the sculpt with no painting. What does a level designer lose by relying on it alone?

---

## 10. How This Builds

- **Lab 11** (`lab_11_water_and_hologram`) applies Session A. You build the hologram, and the water is in the Extension.
- **Lab 12** (`lab_12_toon_shading_and_landscape`) applies Session B. You build the toon post-process with outlines and a two-layer slope auto-material.
- **Week 7** (`08_screen_space_post_processing`) treats post-process materials as a topic of their own: screen filters, letterbox bars, world position from depth, crease outlines from normals and selective colour with Custom Stencil.
- **Week 8** (`09_hlsl_fundamentals`) moves into HLSL, starting by reading the code these graphs compile to.
- **Week 9** uses HLSL loops for multi-tap sampling, which extends the four-neighbour outline in 5.8 to any number of samples.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **Additive blending** | A blend mode that adds the material's colour to what is behind it, never darkening it |
| **Auto-material** | A landscape material that chooses its layers from slope, height or other surface properties instead of painting |
| **Cel shading** | Another name for toon shading |
| **Composite effect** | An effect built by combining several simpler techniques in layers |
| **Index of refraction** | How strongly a material bends light passing through it. 1.33 for water |
| **Landscape layer** | One surface, such as grass or rock, blended into a landscape material |
| **Scanlines** | Thin repeating horizontal lines, used to make a surface read as a projection or a screen |
| **Toon shading** | A style that replaces smooth lighting with a few flat bands, often with ink outlines |
| **World-aligned UVs** | Texture coordinates taken from world position, so a texture keeps its size across stretched geometry |
