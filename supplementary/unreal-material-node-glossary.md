---
title: "Material Node Glossary"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: supplementary
topic_code: MAT-S3
difficulty_tier: foundation
duration: "Reference, look up as needed"
prerequisites: [01_pbr_and_material_editor]
feeds_into: [09_hlsl_fundamentals]
engine_version: "UE 5.7"
---

# Material Node Glossary

---

## 1. How to Use This Glossary

This glossary lists every material node and engine material function used in the node-based notes and labs, Notes 01 to 08 and Labs 1 to 14. It is grouped by what each node is for, so you can scan a group when you know the job but not the node.

Each table has four columns:

- **Node** is the name as it appears in the Material Editor palette. Where the notes use a different name, it is given in brackets.
- **Description** says what the node computes, in one or two sentences.
- **Inputs / Outputs** lists the pin names exactly as the editor shows them. "None" means the node takes its value from its **Details** panel instead of a pin. An unnamed output is the node's single result.
- **Typical usages** lists up to six effects or techniques the node is used for in this module and in games generally.

Search the palette by the node name. Right-click in an empty part of the graph and start typing, and the list filters as you type.

---

## 2. Constants and Parameters

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `Constant` | A single fixed number set in **Details**. Hold **1** and click to place one. | In: none. Out: the value | Thresholds, scale factors, remapping, fixed roughness |
| `Constant2Vector` | Two fixed numbers. Hold **2** and click to place one. | In: none. Out: the vector, R, G | UV offsets, tiling amounts, hash seeds, rotation centres |
| `Constant3Vector` | Three fixed numbers, shown as a colour picker. Hold **3** and click to place one. | In: none. Out: the vector, R, G, B | Flat colours, tints, luminance weights, flat normals |
| `Constant4Vector` | Four fixed numbers. | In: none. Out: RGBA, R, G, B, A, RGB | Packed values, function defaults |
| `ScalarParameter` | A named number that material instances, Blueprints and dynamic instances can change without recompiling. Hold **S** and click to place one. | In: none. Out: the value | Effect strength, tiling control, animation speed, roughness control, runtime control, artist tuning |
| `VectorParameter` | A named colour or vector that instances can change. Hold **V** and click to place one. | In: none. Out: RGB, R, G, B, A, RGBA | Tints, colour themes, glow colours, wind direction, runtime control |
| `TextureSampleParameter2D` | A texture sample whose texture instances can swap. | In: UVs, Apply View MipBias. Out: RGB, R, G, B, A, RGBA | Master materials, texture swapping, surface variants |
| `StaticSwitchParameter` | Chooses between two branches when the shader compiles. Each setting compiles a separate shader permutation. | In: True, False. Out: the chosen branch | Feature toggles, master materials, optional maps, permutation control |
| `CollectionParameter` | Reads a value from a Material Parameter Collection, shared by every material that uses it. | In: none. Out: the value | Global weather, cinematic effects, scanner pulses, Blueprint control, world state |
| `QualitySwitch` | Chooses a branch by the engine's material quality level. | In: Default, Low, High, Medium, Epic. Out: the chosen branch | Scalability, performance tiers, low-end fallbacks |

---

## 3. Coordinates and Positions

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `TextureCoordinate` | The mesh's UVs, with optional tiling set in **Details**. Hold **U** and click to place one. | In: none. Out: UV (float2) | Texture mapping, tiling, UV maths, procedural patterns, flipbooks |
| `Absolute World Position` (`AbsoluteWorldPosition`) | The world-space position of the pixel. In a post-process material it is rebuilt from depth. | In: none. Out: XYZ, XY, Z | World-aligned textures, triplanar mapping, height masks, scanner pulses, scanlines, terrain blending |
| `Object Position` (`ObjectPosition`) | The world-space position of the object's pivot. | In: none. Out: the position | World position offset pivots, per-object variation, radial effects |
| `VertexNormalWS` | The world-space normal of the vertex, before any normal map. | In: none. Out: the normal | Slope masks, auto-materials, snow cover, moss growth |
| `PixelNormalWS` | The world-space normal of the pixel, including the normal map. | In: none. Out: the normal | Toon lighting, curvature edges, surface-angle masks |
| `ReflectionVectorWS` | The direction a perfect mirror would reflect the view. | In: CustomWorldNormal. Out: the direction | Reflection mapping, lat-long sampling, fake reflections |
| `VertexColor` | Colour painted on the mesh's vertices. | In: none. Out: the colour, R, G, B, A | Vertex painting, blend masks, wind weights, damage masks |
| `ScreenPosition` | The pixel's position on screen. | In: none. Out: ViewportUV, PixelPosition | Post-process, screen effects, neighbour sampling, letterbox, vignette |
| `ViewSize` | The size of the view in pixels. | In: none. Out: the size (float2) | Aspect ratio, pixelation, letterbox, resolution independence |
| `Transform` (`TransformVector`) | Converts a vector between spaces, such as local to world, chosen in **Details**. | In: Input. Out: the transformed vector | Local-to-world vectors, world position offset, tangent space, view direction |

---

## 4. Depth and Scene

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `PixelDepth` | The distance from the camera to the pixel being drawn. | In: none. Out: R | Depth differences, distance fades, LOD fades |
| `SceneDepth` | The distance from the camera to the opaque surface behind the pixel. Translucent materials only. | In: UVs. Out: R | Water depth, intersections, soft particles, fog |
| `DepthFade` | Fades opacity to 0 where the surface comes close to the geometry behind it. Translucent materials only. | In: Opacity, FadeDistance. Out: the faded opacity | Soft particles, intersection glow, water edges, force fields |
| `DistanceToNearestSurface` | The distance from a point to the nearest surface, read from mesh distance fields. | In: World Position. Out: the distance | Shoreline foam, proximity glow, contact effects |
| `SceneTexture` | Reads a screen buffer chosen by **Scene Texture Id**, such as `PostProcessInput0`, `SceneDepth`, `WorldNormal` or `CustomStencil`. | In: UVs. Out: Color, Size, InvSize | Post-process, outlines, toon shading, stencil masks, edge detection, screen filters |

---

## 5. Arithmetic

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `Add` | A plus B, per channel. Hold **A** and click to place one. | In: A, B. Out: the sum | Offsets, layering light, combining masks, UV scrolling |
| `Subtract` | A minus B, per channel. | In: A, B. Out: the difference | Centring UVs, depth differences, remapping, distance from a value |
| `Multiply` | A times B, per channel. Hold **M** and click to place one. | In: A, B. Out: the product | Tinting, tiling, strength control, masking, intensity |
| `Divide` | A divided by B, per channel. Hold **D** and click to place one. | In: A, B. Out: the quotient | Normalising ranges, world-aligned UVs, pixel sizes, aspect ratio |
| `Abs` | The absolute value, which turns negatives positive. | In: Input. Out: the result | Triangle waves, symmetry, mirroring, distance to a line, box distances |
| `OneMinus` | 1 minus the input. Hold **O** and click to place one. | In: Input. Out: the result | Inverting masks, glossiness to roughness, edge glows |
| `Saturate` | Clamps to the range 0 to 1, usually at no cost. | In: Input. Out: the result | Mask cleanup, safe ranges, opacity limits |
| `Clamp` | Limits the input between a minimum and a maximum. | In: Input, Min, Max. Out: the result | Range limits, safe ranges, contrast limits |
| `Min` | The smaller of A and B. | In: A, B. Out: the result | Combining masks, SDF union, nearest edges, limits |
| `Max` | The larger of A and B. | In: A, B. Out: the result | Combining masks, SDF intersection, guards against zero |
| `Power` | Base raised to Exp. | In: Base, Exp. Out: the result | Contrast, falloff shaping, thin lines, rim sharpening |
| `Floor` | Rounds down to a whole number. | In: Input. Out: the result | Cell IDs, pixelation, flipbook frames, posterisation, random per tile |
| `Frac` | The part after the decimal point, 0 to 1. | In: Input. Out: the result | Tiling, stripes, cell positions, time loops, repeating patterns |
| `Fmod` | The remainder of A divided by B. | In: A, B. Out: the remainder | Checkerboards, frame wrapping, cycles, alternating rows |

---

## 6. Trigonometry

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `Sine` | A sine wave. **Period** sets how far the input travels for one full wave, 1 by default. | In: Input. Out: the result, -1 to 1 | Waves, pulses, flicker, rotation, wind, hashes |
| `Cosine` | A cosine wave, the sine shifted by a quarter period. **Period** works as for `Sine`. | In: Input. Out: the result, -1 to 1 | Rotation, circular motion, waves |
| `Arctangent2` | The angle of the point (X, Y) around the origin, in radians. | In: Y, X. Out: the angle, -pi to pi | Polar coordinates, radial sweeps, lat-long mapping, angular gradients |
| `Arccosine` | The angle whose cosine is the input, in radians. | In: Input. Out: the angle | Lat-long mapping, angle between vectors |

---

## 7. Vectors and Channels

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `AppendVector` | Joins two values into one longer vector. | In: A, B. Out: the combined vector | Building UVs, building colours, packing values, flipbook offsets |
| `ComponentMask` | Keeps only the channels ticked in **Details**. | In: Input. Out: the selected channels | Channel extraction, ORM unpacking, height from Z, splitting UVs |
| `Dot` (`DotProduct`) | Multiplies matching channels and adds the results. | In: A, B. Out: a single value | Luminance, hashes, facing tests, slope tests, lighting |
| `Length` | The length of a vector. | In: Input. Out: a single value | Radial gradients, circles, distance fields, vignettes |
| `Distance` | The distance between two points. | In: A, B. Out: a single value | Scanner pulses, radial masks, proximity effects |
| `Normalize` | Scales a vector to length 1. | In: VectorInput. Out: the unit vector | Directions, combining normals, view vectors |
| `DeriveNormalZ` | Rebuilds the Z of a normal from its X and Y. | In: InXY. Out: the full normal | Two-channel normals, texture packing |
| `Desaturation` | Blends a colour towards grey by **Fraction**. | In: Input, Fraction. Out: the result | Greyscale, selective colour, muted variants |

---

## 8. Interpolation and Shaping

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `Lerp` (`LinearInterpolate`) | Blends from A to B by Alpha: A at 0, B at 1. Hold **L** and click to place one. | In: A, B, Alpha. Out: the blend | Layer blending, masks, colour ramps, texture splatting, dissolves, toggles |
| `Step` | 1 where X is at least Y, 0 elsewhere. | In: Y, X. Out: 0 or 1 | Hard masks, thresholds, stripes, depth tests |
| `SmoothStep` | Rises smoothly from 0 to 1 as Value moves from Min to Max. | In: Min, Max, Value. Out: the result | Soft masks, anti-aliasing, foam edges, slope blends, dissolve edges, falloffs |
| `DDX` | How fast a value changes from one pixel to the next horizontally. | In: Value. Out: the rate | Anti-aliasing, mip control, edge widths |
| `DDY` | How fast a value changes from one pixel to the next vertically. | In: Value. Out: the rate | Anti-aliasing, mip control, edge widths |

---

## 9. Textures and UV Animation

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `TextureSample` | Reads a texture at the given UVs. **Sampler Type** must match the texture's compression. Hold **T** and click to place one. | In: UVs, Tex, Apply View MipBias. Out: RGB, R, G, B, A, RGBA | PBR textures, masks, normal maps, flipbooks, noise lookups |
| `TextureObject` | A reference to a texture, not a sample of it. | In: none. Out: the texture | Material functions, POM height maps, texture bombing |
| `Panner` | Moves UVs over time at a constant speed. | In: Coordinate, Time, Speed. Out: the moved UVs | Flowing water, scrolling emission, clouds, conveyor belts, lava flows |
| `Rotator` | Rotates UVs around a centre over time. | In: Coordinate, Time. Out: the rotated UVs | Spinning textures, fans, radar sweeps, swirls |
| `BumpOffset` | Shifts UVs by a height map and the view angle, to fake depth. | In: Coordinate, Height, HeightRatioInput. Out: the offset UVs | Cheap parallax, depth illusion, cobblestones |
| `Noise` | Procedural noise from a position, with **Function**, **Scale** and **Levels** in **Details**. | In: World Position, FilterWidth. Out: the value | Procedural variation, dissolves, breakup, clouds, grime, decal edges |

---

## 10. Time, View and Lighting

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `Time` | The current time in seconds. It pauses with the game unless **Ignore Pause** is ticked. | In: none. Out: the time | Animation, pulses, flicker, panning, scrolling, flipbooks |
| `Fresnel` | How much the surface faces away from the camera: 0 facing you, 1 at the silhouette. | In: ExponentIn, BaseReflectFractionIn, Normal. Out: the result | Rim lighting, holograms, force fields, water reflectance, energy shields |

---

## 11. Functions

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `FunctionInput` | An input pin of a material function. Its type and preview value are set in **Details**. | In: Preview. Out: the value passed in | Material functions, reusable graphs |
| `FunctionOutput` | An output pin of a material function. | In: the result. Out: none | Material functions, reusable graphs |
| `MaterialFunctionCall` | Places a material function in a graph. Its pins are the function's inputs and outputs. | In and Out: those of the function | Reuse, shared logic, engine functions |

---

## 12. Engine Material Functions

These are material functions that ship with the engine. Search the palette by name, or find them under `/Engine/Functions`.

| Node | Description | Inputs / Outputs | Typical usages |
|:---|:---|:---|:---|
| `CustomRotator` | Rotates UVs around a chosen centre by an angle from 0 to 1, where 1 is a full turn. | In: UVs, Rotation Center, Rotation Angle (0-1). Out: Rotated Values | Texture bombing, random rotation, rotated stripes |
| `HeightLerp` | Blends A to B using a height map, so the change follows the height rather than a straight fade. | In: A, B, Transition Phase, Height Texture, Contrast. Out: Results, Alpha, Lerp Alpha No Contrast | Height blending, moss in cracks, puddles, layered terrain |
| `BlendAngleCorrectedNormals` | Combines two normal maps correctly, keeping the detail of both. | In: BaseNormal, AdditionalNormal. Out: Result | Detail normals, water ripples, layered surfaces |
| `FlipBook` | Picks one frame of a sprite-sheet texture from an animation phase. | In: Animation Phase (0-1), Number of Columns, Number of Rows, Texture, UVs, MipBias/Level, Clamp Anim (see tooltip), Use Mip Bias (T) Level (F). Out: Result, Alpha, UVs, UV Center | Fire, smoke, explosions, animated sprites |
| `ParallaxOcclusionMapping` | Marches the view ray through a height map to find the UV the eye would really see. | In: Heightmap Texture, Height Ratio, Min Steps, Max Steps, UVs, Heightmap Channel, Reference Plane, and options for world coordinates, texture size and shadows. Out: Parallax UVs, Offset Only, Shadow, Pixel Depth Offset, World Position, Tangent Light Vector, Material Complexity - Steps Debug | Parallax, cobblestones, brick walls, deep surface detail |
| `SimpleGrassWind` | A ready-made wind sway for world position offset. | In: AdditionalWPO, WindIntensity, WindSpeed, WindWeight. Out: Result | Foliage wind, grass sway |
| `WorldAlignedTexture` | Projects a texture along the world axes and blends the projections by the normal. | In: Export Float 4, TextureObject, TextureSize, World Space Normal, WorldPosition, ProjectionTransitionContrast. Out: XY Texture, XYZ Texture, Z Texture | Triplanar mapping, terrain, rocks, unwrapped meshes |
| `WorldAlignedNormal` | The normal-map version of `WorldAlignedTexture`. | In: Normal, TextureObject, TextureSize, WorldSpace, ProjectionTransitionContrast, WorldPosition, Use High Quality Normals. Out: XY Texture, XYZ Texture, XYZFlatTop, Z Texture | Triplanar normals, terrain, rocks |
| `NormalFromHeightmap` | Builds a normal from a height map by comparing neighbouring samples. | In: Height Map, Normal Map Intensity, Height Map UV Offset, Coordinates, Height Map Channel Selector. Out: Result | Normals from height, procedural bumps, brick and mortar relief |
| `LandscapeLayerBlend` | Blends named landscape layers using the weights painted in Landscape mode. Its layers are listed in **Details**, and each adds an input pin. | In: one per layer. Out: the blend | Landscape painting, terrain layers, paths |
