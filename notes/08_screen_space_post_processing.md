---
title: "Screen-Space Post-Processing"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: notes
session: 7
topic_code: MAT-07
difficulty_tier: advanced
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [05_runtime_materials_and_performance, 07_composite_effects]
feeds_into: [lab_13_screen_filters, lab_14_scanner_pulse_and_selective_colour, 09_hlsl_fundamentals]
engine_version: "UE 5.7"
---

# Screen-Space Post-Processing

---

## 1. Key Concepts

1. **A post-process material runs once for every pixel on the screen.** Its cost does not depend on the scene. It depends on the screen resolution, so a 4K screen pays four times what a 1080p screen pays.
2. **Choose the blendable location on purpose.** Before tonemapping you work in linear light, and after tonemapping you work on the colours the player sees.
3. **Screen UVs run from 0 to 1 whatever the screen's shape.** Anything that must stay round or square needs the aspect ratio from `ViewSize`.
4. **To move the image, move where you sample it.** Distortion, pixelation and chromatic aberration all change the UVs fed into `PostProcessInput0`, never the colour.
5. **Every extra sample of the scene costs memory bandwidth.** Chromatic aberration reads the screen three times, and an edge detector reads it five or more times.
6. **Stack filters in one material where you can.** Each separate post-process material is another full-screen pass.
7. **Depth turns a screen pixel back into a place in the world.** In a post-process material, `AbsoluteWorldPosition` is rebuilt from depth, so effects can follow the ground.
8. **Normals find edges that depth misses.** A crease inside one object has no depth jump but a large change of normal.
9. **A stencil marks objects, and depth decides whether you can see them.** The stencil is drawn even behind walls, so test it against the scene depth.
10. **Leave the sky and the UI alone.** Exclude very distant pixels, and remember that the HUD is drawn after post-processing and is never affected by it.

---

## 2. Framing Question

In Notes 05 you outlined an object through walls, and in Notes 07 you turned a whole world into a cartoon. Both were post-process materials, but both treated the screen as a by-product. This week the screen is the subject. A camera lens bends colour at its edges, an old television curves and stripes its picture, a film is framed by black bars, and a game's scanner sweeps a ring of light across the landscape. How do you make the finished image itself the material, and what does it cost to touch every pixel every frame?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Choose a blendable location for a post-process effect | 5.1 |
| Work in screen UVs and correct for the aspect ratio | 5.2 |
| Distort and pixelate the image by changing where it is sampled | 5.3 |
| Build chromatic aberration, a vignette and film grain | 5.4 |
| Add letterbox bars that match a target aspect ratio on any screen | 5.5 |
| Recover world position from depth and sweep a scanner pulse across a level | 5.6 |
| Detect creases and silhouettes from normals and depth | 5.7 |
| Keep one object in colour with Custom Stencil and a depth test | 5.8 |
| Estimate and reduce the cost of full-screen effects | 5.9 |

---

## 4. Why This Matters

Post-processing is where a game gets its look. The same level reads as gritty, dreamy, retro or cinematic depending on a few full-screen materials, and most shipped games stack several of them. They are also where gameplay information lives: a scanner that reveals the world, a damage pulse at the screen edge, an object of interest left in colour while everything else fades to grey.

Because a post-process material touches every pixel, it is also where a careless graph costs the most. A technical artist is expected to know what each filter costs and how to combine filters into as few passes as possible.

---

## 5. Core Ideas

Sections 5.1 to 5.5 are Session A, filters in screen space. Sections 5.6 to 5.9 are Session B, effects that read the scene.

### 5.1 Where a Post-Process Material Runs

A post-process material has **Material Domain** `Post Process`. Its **Blendable Location** says where in the frame it runs:

| Blendable Location | Input image | Use it for |
|:---|:---|:---|
| `Scene Color After Tonemapping` | Final display colours, 0 to 1 | Filters that imitate a screen, lens or film, such as CRT lines, vignette, grain and letterbox |
| `Scene Color Before Bloom` | Linear HDR light, before bloom, exposure and tonemapping | Effects that should glow and bloom like lights in the scene, such as a scanner ring |
| `Scene Color Before DOF` | Linear HDR light, before depth of field. Translucent objects are not in the image yet, because by default they are added after depth of field | Effects that must not apply to glass, water or particles |
| `Replacing the Tonemapper` | Linear HDR light | Writing your own tonemapper, which this module does not need |

The list has a few more entries, such as `Scene Color After DOF` and `SSR Input`, for specialised uses. The two you will use most are `Scene Color After Tonemapping` for screen filters and `Scene Color Before Bloom` for effects that should glow.

The input image is the `SceneTexture` node with **Scene Texture Id** `PostProcessInput0`. When several post-process materials share a location, each one's `PostProcessInput0` is the previous one's output. **Blendable Priority**, in the material's **Post Process Material** settings, sets the order within a location. Higher values run later.

### 5.2 Screen UVs and Aspect Ratio

`ScreenPosition` gives **ViewportUV**: (0, 0) at the top-left of the view, (1, 1) at the bottom-right. Feeding UVs into the **UVs** input of a `SceneTexture` node reads the image at that point instead of at the current pixel.

Two more values are needed in almost every filter:

- `InvSize` on a `SceneTexture` node is the size of one pixel in UV units, as used for the neighbour reads in Notes 05 and 07.
- `ViewSize` gives the view's size in pixels. `aspect = ViewSize.x / ViewSize.y`, about 1.78 on a 16:9 screen.

UVs are stretched to fit the screen, so a circle drawn as `Length(uv - 0.5) < 0.3` comes out as a wide oval. Multiply the horizontal part by `aspect` first, `centred = (uv - 0.5) * (aspect, 1)`, and circles stay round.

### 5.3 Moving the Sample Point

You cannot move pixels. You can only choose, for each pixel, where in the image to read from. Every distortion is a change of UVs before the `PostProcessInput0` read:

- **Heat haze or underwater wobble:** `uv + (Sine(uv.y * 40 + Time * 3) * 0.002, 0)`. The image shimmers sideways in thin bands.
- **Barrel distortion,** the curve of an old screen: `c = uv - 0.5`, then `uv' = 0.5 + c * (1 + Curve * Dot(c, c))`. Pixels near the corners read from further out, so the picture bulges. Where `uv'` falls outside 0 to 1, draw black.
- **Pixelation:** divide the screen into blocks and read every pixel of a block from its centre. `cells = ViewSize / PixelSize`, then `uv' = (Floor(uv * cells) + 0.5) / cells`. With `PixelSize` 6, each block is 6 by 6 screen pixels on any resolution. A `PixelSize` of 1 gives back the original image.

### 5.4 Lens and Film Filters

**Chromatic aberration.** A cheap lens bends red and blue light by slightly different amounts, so colour fringes appear towards the edges of the frame. Sample the image three times:

1. `dir = uv - 0.5`, which points outwards from the centre and grows towards the edges.
2. Red from `uv + dir * Amount`, green from `uv`, blue from `uv - dir * Amount`, with `Amount` about 0.01.
3. `AppendVector` the three channels back into one colour.

**Vignette.** Lenses darken towards the corners. `d = Length(centred) * 2`, then `colour * (1 - Intensity * SmoothStep(0.5, 1.4, d))`, with `Intensity` about 0.6.

**Film grain.** Add a little random noise that changes every frame. Hash the pixel position with time, as in Notes 04: `grain = Frac(Sine(Dot(PixelPosition + Frac(Time) * 100, (12.9898, 78.233))) * 43758.5453) - 0.5`, then add `grain * GrainAmount`, with `GrainAmount` about 0.04.

**Posterisation.** `Floor(colour * Levels) / Levels` cuts the image to a few shades per channel, a quick retro look.

**Scanlines.** `0.75 + 0.25 * Sine(uv.y * Rows * 2 * Pi)` multiplied over the image, with `Rows` about 240, gives the dark lines between rows of an old screen.

Each filter is a few nodes. Put them in one material in a fixed order: UV changes first, then the samples, then the colour filters.

### 5.5 Letterbox Bars

Films are often shot wider than a monitor, at 2.39:1, and shown with black bars above and below. Games use the same bars to say "this is a cutscene".

The bar height comes from comparing the screen's shape with the target shape:

1. `bar = Saturate((1 - aspect / TargetAspect) / 2)`, with `TargetAspect` 2.39. On a 16:9 screen each bar is about 13% of the height. On a screen already wider than 2.39:1, the bar is 0.
2. Scale it by `LetterboxAmount`, 0 to 1, so the bars can slide in and out.
3. `edge = Min(uv.y, 1 - uv.y)`, the distance to the nearer of the top and bottom.
4. `mask = 1 - SmoothStep(bar - InvSize.y, bar + InvSize.y, edge)`. The `SmoothStep` across one pixel keeps the bar edge clean.
5. `Lerp(colour, black, mask)`.

Drive `LetterboxAmount` from a **Material Parameter Collection**, as in Notes 05, so a Blueprint can slide the bars in when a cutscene starts. Any HUD is drawn after post-processing, so it stays on top of the bars.

### 5.6 World Position from Depth: the Scanner Pulse

A post-process material has no mesh, yet it knows where each pixel is in the world. Depth says how far away the surface is along the view ray, and the engine combines it with the camera to rebuild the position. In a post-process material, `AbsoluteWorldPosition` returns that rebuilt position.

A scanner pulse is a ring of light that expands across the world from a point:

1. `d = Distance(AbsoluteWorldPosition, PulseOrigin)`. How far each visible surface is from the origin.
2. `r = (Time - PulseStart) * PulseSpeed`. The ring's current radius. `PulseStart` is the game time when the pulse was fired.
3. `ring = 1 - SmoothStep(0, PulseWidth, Abs(d - r))`. A band `PulseWidth` wide at radius `r`.
4. `trail = SmoothStep(r - TrailLength, r, d) * Step(d, r)`, with `TrailLength` about 400 cm. A fading glow inside the ring.
5. `fade = 1 - Saturate(r / MaxRadius)`, so the pulse dies out as it grows.
6. `colour + (ring + trail * 0.15) * fade * PulseColour * Strength`, and keep the sky unchanged.

`PulseOrigin` and `PulseStart` live in a Material Parameter Collection. A Blueprint fires a pulse with two nodes: set `PulseOrigin` to the player's location and `PulseStart` to **Get Game Time in Seconds**. The material's `Time` is the same game time, so the ring starts at radius 0 and grows by itself, with no Tick or Timeline. Run the material at `Scene Color Before Bloom` so the ring blooms like a light.

### 5.7 Edges from Normals and Depth

The depth outline in Notes 07 only finds silhouettes. A cube's edges inside its own outline have no depth jump, so they get no line. The surface normal changes sharply there, so test normals too:

1. Read `SceneTexture` **WorldNormal** at the pixel and at its four neighbours, as in Notes 07 5.8.
2. `normalEdge = Sum(1 - Dot(centre, neighbour))`. 0 on a flat or smoothly curved surface, large at a crease.
3. Combine it with the depth edge: `edge = Max(SmoothStep(DepthThreshold, DepthThreshold * 2, depthJump), SmoothStep(NormalThreshold, NormalThreshold * 2, normalEdge))`, with `NormalThreshold` about 0.4.

Normal-map detail is in the WorldNormal buffer, so a strong normal map can trigger lines across a surface. Raise `NormalThreshold` until only real creases remain.

**Depth fog tint.** The same depth read gives distance fog in one line: `Lerp(colour, FogColour, Saturate((depth - FogStart) / FogRange))`. It is a cheap way to give an underwater or dusty look without a fog actor.

### 5.8 Stencil Masks and Selective Colour

A classic effect keeps one object in full colour while the rest of the world turns grey. Custom Stencil from Notes 05 marks the object:

1. On the object, tick **Render CustomDepth Pass** and set **CustomDepth Stencil Value** to 1. The project already has **Custom Depth-Stencil Pass** set to `Enabled with Stencil`.
2. `grey = Desaturation(colour)` with **Fraction** 1, or `Dot(colour, (0.3, 0.59, 0.11))`.
3. `mask = 1 - Saturate(Abs(CustomStencil - 1))`. 1 where the stencil is exactly 1, and 0 for any other value.
4. `Lerp(grey * GreyStrength, colour, mask)`.

Put a wall in front of the object and the colour shows straight through the wall. The custom depth pass draws the whole object regardless of what is in front of it. The fix is a depth test:

5. `visible = Step(CustomDepth, SceneDepth + 1)`. The object is visible only where its own depth is no further than the nearest surface at that pixel. The extra 1 cm absorbs precision errors.
6. Use `mask * visible` in the `Lerp`.

Order matters when effects share the screen. If selective colour runs after the scanner pulse from 5.6, it greys the ring too. Put both at `Scene Color Before Bloom` and give the pulse the higher **Blendable Priority**, so the grey is applied first and the ring is drawn over it.

The same mask can tint, outline or blur a single object. Different stencil values, such as 1 for allies and 2 for enemies, let one material treat groups differently.

### 5.9 The Cost of Full-Screen Effects

A 1920 x 1080 screen has about 2.1 million pixels, and a 3840 x 2160 screen has about 8.3 million. Every node in a post-process material runs that many times per frame. Three habits keep the cost down:

- **Count the scene reads.** Each `SceneTexture` read is a memory fetch. Chromatic aberration costs three, the normal-and-depth edge detector costs ten. Reading the same buffer twice at the same UVs costs twice.
- **Merge filters.** Five separate post-process materials are five full-screen passes. One material with five filters is one pass.
- **Switch off at zero.** A filter with its strength at 0 still costs its nodes. Remove the material from the volume, or set the volume's **Blend Weight** to 0, when the effect is not in use. Use `QualitySwitch` from Notes 05 to drop expensive filters on low settings.

`stat GPU` lists the post-process passes, so you can measure each material's share of the frame.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes08/Demo/Levels/Lvl_Notes08`. The finished demo assets are in `/Game/GD/Notes08/Demo/Materials` and `/Game/GD/Notes08/Demo/Blueprints`. To reproduce the demo, create `/Game/Student/Notes08/MyWork` and save your assets there. The level has three areas, each inside its own post-process volume, plus the letterbox and scanner, which apply everywhere.

### 6.1 Part A - A Retro Screen and Cinematic Bars (Session A)

**Step 1 - Pass-through.** Create `PPM_CRT` with **Material Domain** `Post Process` and **Blendable Location** `Scene Color After Tonemapping`. Wire `PostProcessInput0` straight into `Emissive Color`. Add it to the CRT area's volume. Nothing changes, which proves the material reads and writes the image correctly.

**Step 2 - Curve and pixelate.** Add barrel distortion with `Curve` 0.15, then pixelation with `PixelSize` 4, both before the read. Draw black where the curved UVs fall off the screen. The view bulges like an old television and breaks into blocks.

**Step 3 - Colour fringes.** Replace the single read with three, for chromatic aberration with `Amount` 0.006. Red and blue fringes appear at high-contrast edges, strongest in the corners.

**Step 4 - Scanlines and vignette.** Multiply the scanlines with `Rows` 240 and the vignette with `Intensity` 0.7. Fly the camera in and out of the CRT area to toggle the look.

**Step 5 - Letterbox.** Create `PPM_Letterbox` from 5.5, and `MPC_Cinematic` with a scalar `LetterboxAmount`. `BP_LetterboxControl` slides the bars in and out when you press L in **Play**. Resize the editor viewport to show that the bars always frame a 2.39:1 picture.

### 6.2 Part B - Effects that Read the Scene (Session B)

**Step 6 - Scanner pulse.** Create `PPM_ScannerPulse` at `Scene Color Before Bloom` from 5.6, and `MPC_Scanner` with a vector `PulseOrigin` and a scalar `PulseStart`. Click **Play** and press P: `BP_ScannerControl` fires a ring from the player that sweeps over the floor, the walls and every prop, following the shape of whatever it crosses.

**Step 7 - Selective colour.** Create `PPM_SelectiveColour` from 5.8 without the depth test, and fly into its area. The marked sphere stays red in a grey world, and its colour shows through the wall in front of it. Add the depth test, and the wall hides it correctly.

**Step 8 - Crease edges.** Create `PPM_EdgeDetect` from 5.7. In its area, the cubes gain lines along every edge, not only around their outlines. Raise `NormalThreshold` to show how the lines inside textured surfaces disappear first.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| Material Domain left as Surface | The material cannot be added to a volume, or does nothing | Set **Material Domain** `Post Process` |
| Volume not unbound | The effect only shows inside a box | Tick **Infinite Extent (Unbound)**, or fly into the volume |
| Circle maths in raw UVs | Vignettes and rings are ovals | Multiply the horizontal offset by the aspect ratio |
| Colour changed instead of UVs | Distortion smears colours instead of moving the image | Offset the UVs into `PostProcessInput0` |
| Pixel size given in UVs | Blocks change size with the resolution | Build the cell count from `ViewSize` |
| Barrel distortion without a border | Stretched edge pixels fill the corners | Draw black where the UVs fall outside 0 to 1 |
| Scanner pulse in `Scene Color After Tonemapping` | The ring looks flat and does not bloom | Use `Scene Color Before Bloom` |
| Scanner ring on the sky | A ring appears painted on the sky | Exclude pixels whose depth is very large |
| Stencil without a depth test | The selected object shows through walls | Multiply the mask by `Step(CustomDepth, SceneDepth + 1)` |
| Normal edges on every texture | Lines across bumpy surfaces | Raise `NormalThreshold` |
| Many small post-process materials | A high post-process cost in `stat GPU` | Merge filters into one material |

---

## 8. Exercises

1. Add a heat-haze wobble to `PPM_CRT` that only affects the bottom third of the screen.
2. Change the scanner so it draws a faint grid on the ground inside the ring, using the triangle wave from Notes 04 on `AbsoluteWorldPosition.xy`.
3. Give two objects stencil values 1 and 2, and keep the first in colour and tint the second red in the selective colour material.
4. Make a damage effect: a red vignette whose strength comes from an MPC value that a Blueprint raises when the player is hurt and lowers over time.
5. Measure `PPM_CRT` with `stat GPU` at two screen percentages and explain the difference.

---

## 9. Reflective Questions

1. A post-process effect costs the same whether the screen shows one cube or a whole city. When does that make post-processing the cheaper way to build an effect, and when does it make it the more expensive way?
2. The letterbox bars were driven by an MPC. What would go wrong if two Blueprints, a cutscene and a menu, both tried to set `LetterboxAmount`?
3. The scanner pulse follows the ground, but a translucent water surface does not write depth. What happens to the ring when it crosses water, and which blendable location could change that?
4. Selective colour needed a depth test, while the outline in Notes 05 deliberately showed through walls. Name another effect where each behaviour is the one you want.

---

## 10. How This Builds

- **Lab 13** (`lab_13_screen_filters`) applies Session A. You build pixelation, chromatic aberration, a vignette and animated letterbox bars in one material.
- **Lab 14** (`lab_14_scanner_pulse_and_selective_colour`) applies Session B. You build a scanner pulse fired from Blueprint and a selective-colour object with the depth test.
- **Week 8** (`09_hlsl_fundamentals`) moves into HLSL. The filter maths in this week, a few lines each, is ideal for a first Custom node.
- **Week 9** writes multi-tap blurs and edge detectors as HLSL loops, where the number of samples becomes a single number.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **Aspect ratio** | The width of the view divided by its height |
| **Barrel distortion** | A bulge in the image, made by reading pixels near the edges from further out |
| **Blendable Location** | The point in the frame where a post-process material runs |
| **Blendable Priority** | The order of post-process materials that share a blendable location |
| **Chromatic aberration** | Colour fringes caused by a lens bending red, green and blue light by different amounts |
| **Film grain** | Fine random noise that changes every frame, imitating film stock |
| **Letterbox** | Black bars above and below the image that frame a wider picture |
| **Post-process material** | A material that runs once per screen pixel on the rendered image |
| **Scanner pulse** | An expanding ring drawn on the world, using the world position rebuilt from depth |
| **Selective colour** | An effect that keeps chosen objects in colour while the rest of the image is grey |
| **Vignette** | Darkening towards the corners of the frame |
