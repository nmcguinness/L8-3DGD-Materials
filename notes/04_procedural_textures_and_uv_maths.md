---
title: "Procedural Textures and UV Maths"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-19"
doc-type: notes
session: 3
topic_code: MAT-03
difficulty_tier: intermediate
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [01_pbr_and_material_editor, 03_master_materials_and_layering]
feeds_into: [lab_5_procedural_tiles_and_bricks, lab_6_procedural_wood, 09_hlsl_fundamentals]
engine_version: "UE 5.7"
---

# Procedural Textures and UV Maths

---

## 1. Key Concepts

1. **A UV is just a number pair.** Anything you can do to a number, you can do to a UV, so every procedural pattern starts by transforming UVs and ends by measuring something.
2. **Centre before you rotate or measure.** Subtract 0.5 first, because rotation, polar angles and distances all assume the origin is the middle of the surface, not its corner.
3. **`Frac` repeats, `Floor` identifies.** Position inside a cell drives the shape, and the cell ID drives the variation between cells.
4. **Never `Step` an edge you will see.** A hard step aliases and crawls as the camera moves, so soften every visible edge by exactly one pixel with `SmoothStep` and the screen-space derivatives.
5. **Measure distance, then decide.** A distance field gives you fill, outline, glow and bevel from one number, so compute the distance once and derive everything else from it.
6. **Combine shapes with `Min` and `Max`.** On distance fields, `Min` is a union and `Max` is an intersection, so complex shapes are built from simple ones without branches.
7. **Hash for variety, never for noise.** A hash gives each cell an unrelated value, and only smooth interpolation between hashed corners turns that into noise.
8. **Add octaves, halve the amplitude.** fBm layers the same noise at double the frequency and half the strength, so detail appears at every scale without one layer dominating.
9. **Warp the input, not the output.** Adding noise to the UVs before a pattern bends its shape, which is how rings become wood grain and stripes become marble.
10. **Expose the numbers that matter.** A procedural material is only reusable when its counts, widths and colours are parameters, so an artist can change them without opening the graph.

---

## 2. Framing Question

Last week you built a UV rotator from scratch, and in Week 2 you split UVs into cells with `Frac` and `Floor`. Both treat UVs as numbers you can change. If the numbers can be changed, can they also be the picture? How much of a brick wall, a floor or a plank of wood can you draw with maths alone, and what do you gain by not using a texture?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Remap UVs into gradients and ramps with `Lerp` and `SmoothStep` | 5.1 |
| Rotate and repeat UVs to draw stripes, and anti-alias their edges with `DDX` and `DDY` | 5.2, 5.3 |
| Build tiles and bricks from `Frac`, `Floor`, edge distance and a per-cell hash | 5.4, 5.5 |
| Convert UVs to polar coordinates and animate a pattern around a point | 5.6 |
| Draw circles, boxes and rings as signed distance fields and combine them | 5.7 |
| Build value noise from a hash, layer it into fBm, and compare it with the `Noise` node | 5.8, 5.9 |
| Warp UVs with noise to turn regular patterns into natural ones | 5.10 |
| Combine cells, hashes, distance, noise and time into finished effects | 5.11 |

---

## 4. Why This Matters

A **procedural texture** is a pattern computed in the shader rather than read from an image. It has no resolution, so it stays sharp at any distance. It uses no texture memory. Every number in it can become a parameter, so one material makes a hundred variations. Games use procedural patterns for hazard markings, UI rings and radar sweeps, tile and brick variation, and the noise that drives dissolves, wetness and wind, which you meet in later weeks.

The cost moves from memory to arithmetic, so procedural is not always the right choice. A photographic surface such as moss is still best as a texture. Knowing UV maths lets you choose, and it is also the vocabulary of shader code: Week 7 rewrites this week's graphs in HLSL, line for line.

---

## 5. Core Ideas

Sections 5.1 to 5.5 are Session A, patterns on a grid. Sections 5.6 to 5.11 are Session B, patterns around a point, noise, and four effects that combine everything.

### 5.1 UVs as Numbers

`TextureCoordinate` outputs two numbers per pixel: **U** across the surface from 0 to 1, and **V** down it from 0 to 1. Split them with `ComponentMask`, R for U and G for V. Wired straight into `Base Color`, U is a black-to-white ramp. That ramp is the raw material for everything else.

Three operations reshape a ramp:

| Operation | Nodes | Effect |
|:---|:---|:---|
| Remap | `Lerp` with the ramp on **Alpha** | Turns 0-1 into any two colours or values |
| Sharpen | `SmoothStep` with **Min** and **Max** | 0 below Min, 1 above Max, a smooth S-curve between |
| Repeat | `Multiply` by `n`, then `Frac` | `n` copies of the ramp, each 0 to 1 |

`SmoothStep` is the workhorse of procedural graphs. Narrow its Min-to-Max range and the transition becomes an edge. Widen it and the edge becomes a glow or a fade. Station 1 of the explorer lets you drag Min and Max and watch this happen.

### 5.2 Rotating and Repeating: Stripes

Hazard stripes are a repeated ramp that runs diagonally. You rotate the UVs first, then repeat them, reusing last week's rotator:

1. Centre: `TextureCoordinate -> Subtract` (0.5, 0.5). The origin moves to the middle of the surface, so rotation turns around the centre rather than the corner.
2. Rotate by angle `a`: `U' = U cos a - V sin a`. For stripes you only need this one rotated axis.
3. Repeat: `U' -> Multiply` by `StripeCount`.

A first attempt is `Frac -> Step` with **Const Y** 0.5, as in Notes 03 5.7. It works, but every edge is a hard staircase that shimmers as the camera moves. The next section fixes that.

A better shape for stripes is a **triangle wave**, which rises from 0 to 1 and falls back again instead of jumping:

`t = Abs(Frac(x) - 0.5) * 2`

`t` is 0 at the middle of each cell and 1 at its edges, and it never jumps, which matters for the anti-aliasing below. Thresholding `t` at 0.5 gives stripes of equal width, and moving the threshold changes the ratio of the two colours.

### 5.3 Clean Edges With Derivatives

An edge drawn with `Step` is either on or off in each pixel, so a diagonal edge becomes a staircase. Close up you see the steps. Far away, many stripes fall inside one pixel, and the pattern shimmers or breaks into false patterns called **moire**.

The fix is to blur each edge by exactly one pixel. To do that you need to know how much your pattern value changes across one pixel. The GPU shades pixels in 2 x 2 groups, so it can tell you:

| Node | Returns |
|:---|:---|
| `DDX` | How much the input changes from this pixel to the next one across the screen |
| `DDY` | How much it changes from this pixel to the next one down the screen |

The sum of their absolute values, `Abs(DDX(x)) + Abs(DDY(x))`, is the change across one pixel. Shader programmers call it **fwidth**. Use it as the half-width of the `SmoothStep`:

1. `x` is the repeated, rotated coordinate from 5.2, before `Frac`.
2. `w = (Abs(DDX(x)) + Abs(DDY(x))) * 2`. The factor 2 matches the slope of the triangle wave, which changes twice as fast as `x`.
3. `stripe = SmoothStep(0.5 - w, 0.5 + w, t)`, with `Min` = `0.5 - w`, `Max` = `0.5 + w` and `Value` = `t`.

Close up, `w` is tiny and the edge stays sharp. Far away, `w` grows, the stripes blend to an even grey, and the shimmer disappears. Take the derivatives of the smooth `x`, not of `Frac(x)`, because `Frac` jumps at every cell edge and its derivative there is huge. This is the same reason texture bombing needed explicit derivatives in Notes 03. Station 2 of the explorer draws stripes with large pixels: set the count to 60 and switch between `Step` and `SmoothStep` to see the moire appear and disappear.

### 5.4 Tiles and Grout

Floor tiles are cells from Notes 03 5.7 with a gap around each one. Multiply UVs by the tile count `n`, then:

- `f = Frac(UV * n)` is the position inside the tile, 0 to 1 on each axis.
- `id = Floor(UV * n)` is which tile you are in.

The distance from the nearest tile edge is the smallest of four distances, one to each side:

`d = Min(Min(f.x, 1 - f.x), Min(f.y, 1 - f.y))`

`d` is 0 on the tile edge and 0.5 at its centre. Grout is wherever `d` is less than the grout half-width `g`:

`tile = SmoothStep(g - w, g + w, d)`, with `w` from the derivatives of `UV * n`.

`tile` is 1 on the tile and 0 in the grout. Use it twice: as the `Lerp` alpha between grout colour and tile colour, and between a high grout roughness and a low tile roughness. One mask drives every PBR input, which is what makes a procedural surface read as real.

**A different tile every time.** Feed `id` into the hash from Notes 03 5.7 to get a random value per tile, then use it to vary each tile's tint slightly: `Lerp(TileColourA, TileColourB, hash)`. The variation is fixed per tile, so it does not flicker.

### 5.5 Bricks: Offset Rows and Aspect

Bricks are tiles with two changes.

**Offset rows.** Every other row shifts half a brick sideways. Compute the row number before you take `Frac`:

1. `p = UV * (BricksAcross, BricksDown)`. Use `Multiply` with a `Constant2Vector`, or an `AppendVector` of two parameters.
2. `row = Floor(p.y)`.
3. `offset = Frac(row * 0.5)`. This is 0 on even rows and 0.5 on odd rows.
4. `p.x = p.x + offset`. Rebuild `p` with `AppendVector`.
5. Continue as for tiles, with `Frac(p)` and `Floor(p)`.

Hash `Floor(p)` after the offset, so each brick, not each grid square, gets its own value.

**Aspect.** A brick is about twice as long as it is tall. If `BricksAcross` is half of `BricksDown` on a square surface, each cell is twice as wide as it is tall, so a mortar width of 0.05 in cell space is twice as thick on the long sides as on the short ones. Correct it by measuring the horizontal distance in the same units as the vertical one:

`d = Min(Min(f.x, 1 - f.x) * Aspect, Min(f.y, 1 - f.y))`

where `Aspect` is the brick's width divided by its height in world space, here 2. The mortar is now the same thickness all round.

### 5.6 Polar Coordinates

Some patterns are arranged around a point rather than along axes: rings, a radar sweep, a sunburst, a clock face. For these, convert UVs to **polar coordinates**, a distance and an angle.

1. Centre: `c = UV - 0.5`, so `c` runs from -0.5 to 0.5.
2. Radius: `r = Length(c) * 2`. It is 0 at the centre and 1 at the middle of each edge.
3. Angle: `a = Arctangent2(c.y, c.x)`. Wire `c.y` to **Y** and `c.x` to **X**. The result runs from -pi to pi.
4. Normalise: `a01 = a / 6.283185 + 0.5`, so the angle runs from 0 to 1 once round the circle.

This is the opposite of last week's rotator. The rotator took an angle and produced new UVs. Polar conversion takes UVs and produces the angle. Rotating a polar pattern is now one `Add`: add a value to `a01` before you use it, and wrap it with `Frac`.

Every grid technique from Session A works on `(a01, r)` instead of `(U, V)`:

- **Rings:** the triangle wave of `r * RingCount`.
- **Sectors:** the triangle wave of `a01 * SectorCount`.
- **A radar sweep:** `Frac(a01 - Time * Speed)` is a ramp that rotates with time. Raise it to a high power, for example `Power` 8, and only the leading edge stays bright, leaving a fading trail behind it.

The angle has a seam where it wraps from 1 back to 0, along the negative X axis. A pattern that repeats a whole number of times round the circle hides the seam. A pattern that does not shows a hard line there.

### 5.7 Signed Distance Fields

A **signed distance field** (SDF) stores, for every point, the distance to the edge of a shape: negative inside, positive outside, 0 on the edge. You have already met one: the tile distance `d` in 5.4. The common shapes, with `p` the centred UV:

| Shape | Distance | Nodes |
|:---|:---|:---|
| Circle, radius `R` | `Length(p) - R` | `Length`, `Subtract` |
| Ring, radius `R`, half-thickness `T` | `Abs(Length(p) - R) - T` | `Length`, `Subtract`, `Abs`, `Subtract` |
| Box, half-size `B` | `q = Abs(p) - B`, then `Length(Max(q, 0)) + Min(Max(q.x, q.y), 0)` | `Abs`, `Subtract`, `Max`, `Length`, `Min`, `ComponentMask` |

One distance gives several looks, all from `SmoothStep` with the derivative width `w` of the distance:

| Look | Graph |
|:---|:---|
| Filled shape | `1 - SmoothStep(-w, w, d)` |
| Outline of width `L` | `1 - SmoothStep(L - w, L + w, Abs(d))` |
| Soft glow | `Saturate(1 - d / GlowSize)` |

Station 5 of the explorer shows these distances as a coloured field with contour lines, and lets you drag a second shape across the first. Shapes combine without any branches:

| Combination | Graph |
|:---|:---|
| Union, either shape | `Min(d1, d2)` |
| Intersection, both shapes | `Max(d1, d2)` |
| Subtraction, the first minus the second | `Max(d1, -d2)` |

A target is the union of two rings and a filled circle. A rounded button is a box with `R` subtracted from its distance. Move a shape by subtracting an offset from `p`, and scale it by dividing `p` before the distance and multiplying the result after.

### 5.8 Value Noise From a Hash

A hash gives every cell an unrelated random value, so on its own it draws a grid of flat squares. **Value noise** makes it smooth by blending between the random values at the four corners of each cell:

1. `i = Floor(p)` and `f = Frac(p)`, as in 5.4.
2. Hash the four corners: `h00 = hash(i)`, `h10 = hash(i + (1, 0))`, `h01 = hash(i + (0, 1))`, `h11 = hash(i + (1, 1))`.
3. Smooth the blend: `s = f * f * (3 - 2 * f)`. This S-curve has zero slope at 0 and 1, so neighbouring cells meet without a crease.
4. Blend across, then down: `Lerp(Lerp(h00, h10, s.x), Lerp(h01, h11, s.x), s.y)`.

That is four hashes, which is a lot of graph. Build it once as a Material Function, `MF_Math_ValueNoise`, with a `UVs` input and a `Result` output, and build the hash itself as `MF_Math_Hash`, so the noise calls it four times instead of repeating it.

Unreal also has a built-in `Noise` node. Its **Function** setting chooses among several noise types, with a quality-versus-cost trade-off listed in the node's tooltip. Building value noise yourself once shows you what the node does, and the shader code in Week 7 depends on knowing it.

### 5.9 fBm: Detail at Every Scale

One layer of noise looks like soft blobs, all the same size. Natural surfaces have large variation, smaller variation on top, and smaller still. **Fractional Brownian motion** (fBm) adds several copies of the same noise, called **octaves**, each at double the frequency and half the amplitude of the last:

`fbm(p) = noise(p) * 0.5 + noise(p * 2) * 0.25 + noise(p * 4) * 0.125 + noise(p * 8) * 0.0625`

Four octaves are usually enough. Each one adds a noise function call, so fBm costs as many times as the single noise as it has octaves. The amplitudes add up to just under 1, so the result stays in the 0-1 range.

The built-in `Noise` node does this for you with its **Levels** setting: each extra level is one more octave.

### 5.10 Domain Warping

So far noise has been the output: you looked at it directly. Used on the input instead, noise bends any pattern into a natural shape. This is **domain warping**:

`pattern(p + Strength * noise(p))`

Add noise to the UVs, or to the polar radius, before the pattern reads them. Straight stripes become marble veins. Perfect rings become wood grain:

1. `r` from 5.6.
2. `r' = r + WarpStrength * fbm(UV * NoiseScale)`.
3. `rings = TriangleWave(r' * RingCount)`, the triangle wave from 5.2.
4. `Lerp(LightWood, DarkWood, rings)` for colour, and a second `Lerp` for roughness.

`WarpStrength` controls how wavy the rings are. Stretch the noise input, for example by multiplying `UV` by (1, 8), and the rings smear into the long grain of a cut plank. The pattern is the same rings as before, so all the realism comes from where the noise moved them.

### 5.11 Putting It Together

Every technique in this week combines with the others. The four effects below each use only nodes from 5.1 to 5.10, and the lab Extensions give the full graph for each.

| Effect | Techniques combined | The idea that makes it work |
|:---|:---|:---|
| **Starfield** | Cells, three hashes, circle distance, `Time` | Each cell hashes a random position, size and brightness for one star, and a fourth hash switches some cells off, so the grid disappears. A `Sine` of `Time` offset by the hash makes each star twinkle out of step with its neighbours |
| **Hexagonal shield** | Two offset grids, edge distance, polar radius, `Time` | A hexagon grid is two rectangular grids offset by half a cell, and each pixel keeps whichever cell centre is nearer. A ring whose radius grows with `Frac(Time)` pulses outwards and brightens the edges it crosses |
| **Molten lava** | 3D noise, `Time`, a thin band around a threshold | Feeding `Time` into the third axis of the noise position animates the noise as if it were boiling. Lava glows only where the noise is close to 0.5, which draws thin, moving veins between plates of dark crust |
| **Topographic map** | Noise as height, colour bands, anti-aliased contour lines | The noise value is treated as terrain height. `SmoothStep` thresholds turn it into water, lowland and highland, and the triangle wave of height times a count draws clean contour lines, exactly like the stripes in 5.3 |

None of these needs a texture, and each is under a hundred nodes. What changes is not the maths but what you decide the numbers mean.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes04/Demo/Levels/Lvl_Notes04`. The finished demo materials are in `/Game/GD/Notes04/Demo/Materials`, on labelled meshes. Each is named for the pattern it draws, such as `MM_Prop_HazardStripes_Opaque`, because in a procedural material the technique is what tells one material from another. To reproduce the demo, create `/Game/Student/Notes04/MyWork` with a `Functions` subfolder, and save each asset there under the name its step gives.

No textures are used in this example. Every pattern comes from `TextureCoordinate`.

### 6.1 Part A - Patterns on a Grid (Session A)

**Step 1 - A ramp.** In `MyWork`, create `MM_Prop_HazardStripes_Opaque`. Wire `TextureCoordinate -> ComponentMask (R) -> Base Color` and apply it to a plane. Show the black-to-white ramp, then put it through `Multiply` (4) and `Frac`, then through `SmoothStep` with **Min** 0.4 and **Max** 0.6. Each change is one node, and each one changes what the number means.

**Step 2 - Rotated stripes.** Delete the ramp nodes. Build:

1. `TextureCoordinate -> Subtract`, with a `Constant2Vector` (0.5, 0.5) on **B**.
2. Split the result with two `ComponentMask` nodes into `cx` (R) and `cy` (G).
3. A `ScalarParameter` `StripeAngle`, default 0.785 (45 degrees in radians), into a `Cosine` and a `Sine`, each with **Period** 6.283185.
4. `x = cx * cos - cy * sin`, with two `Multiply` nodes and a `Subtract`.
5. `x -> Multiply` by a `ScalarParameter` `StripeCount`, default 10.
6. Triangle wave: `Frac -> Subtract` (0.5) `-> Abs -> Multiply` (2). Call this `t`.
7. `t -> Step` with **Const Y** 0.5, `Step -> Lerp.Alpha`. `VectorParameter` `ColourA` (black) `-> Lerp.A`, `VectorParameter` `ColourB` (yellow, 1, 0.75, 0) `-> Lerp.B`, `Lerp -> Base Color`.

Apply, and fly the camera away from the plane. The far stripes shimmer and break into moire.

**Step 3 - Clean the edges.** Take the output of step 5, before `Frac`, and wire it into `DDX` and `DDY`. `Abs` each, `Add` them, `Multiply` by 2: this is `w`. Replace `Step` with `SmoothStep`: **Min** = `0.5 - w`, **Max** = `0.5 + w`, **Value** = `t`. Fly away again. The far stripes fade to an even orange-black instead of shimmering, and the near edges are still sharp.

**Step 4 - Tiles.** In `MyWork`, create `MM_Env_FloorTiles_Opaque`. Build:

1. `TextureCoordinate -> Multiply` by a `ScalarParameter` `TileCount`, default 6. Call this `p`.
2. `f = Frac(p)`. Split into `fx` and `fy` with `ComponentMask`.
3. Edge distance: `Min(fx, OneMinus(fx))`, `Min(fy, OneMinus(fy))`, then `Min` of the two. Call this `d`.
4. `w` from `DDX` and `DDY` of `p`, as in step 3, without the factor 2. `DDX` of a two-component value returns two components, so `ComponentMask` R of the summed result.
5. `tile = SmoothStep(Grout - w, Grout + w, d)`, with a `ScalarParameter` `Grout`, default 0.03.
6. `tile -> Lerp.Alpha` with grout colour (0.3, 0.3, 0.3) on **A** and a tile colour on **B**, into `Base Color`. Then a second `Lerp` with 0.9 on **A** and 0.2 on **B** into `Roughness`.

Apply it to a floor plane. The tiles are glossy and the grout is matte.

**Step 5 - Vary each tile.** `Floor(p)` into the hash graph from Notes 03 5.7, then `Lerp` two close tile colours, (0.8, 0.78, 0.72) and (0.62, 0.6, 0.55), with the hash as **Alpha**, and wire it into the tile colour input from step 4. Every tile now has its own shade, and the grid stops looking printed.

### 6.2 Part B - Patterns Around a Point (Session B)

**Step 6 - Polar coordinates.** In `MyWork`, create `MM_FX_RadarSweep_Opaque`. With nothing selected, set **Shading Model** to `Unlit`, because a radar screen emits its own light. Build `c`, `r` and `a01` from 5.6. Show `r` and then `a01` on `Emissive Color`, one at a time: a radial gradient, then a ramp that wraps once round the centre, with its seam along the left edge.

**Step 7 - The sweep.** Build:

1. `Time -> Multiply` by a `ScalarParameter` `SweepSpeed`, default 0.25.
2. `Frac(a01 - that)`, then `Power` with **Exp** 8. This is the beam with its trailing fade.
3. Rings: triangle wave of `r * 5`, then `SmoothStep` with **Min** 0.9 and **Max** 1 for thin bright lines.
4. `Add` the beam and the rings, `Multiply` by a green `VectorParameter` `ScreenColour`, then `Multiply` by `1 - SmoothStep(0.98, 1, r)` so nothing draws outside the circle.
5. Into `Emissive Color`.

Apply it to a plane and press **Play** or enable **Realtime** in the viewport. The beam sweeps round, leaving a fading trail over fixed range rings.

**Step 8 - A target from distance fields.** In `MyWork`, create `MM_Prop_SDFTarget_Opaque`. With `p` the centred UV:

1. Outer ring: `Abs(Length(p) - 0.4) - 0.04`.
2. Inner ring: `Abs(Length(p) - 0.24) - 0.04`.
3. Bullseye: `Length(p) - 0.1`.
4. Union: `Min` of all three.
5. Fill with the derivative width of the distance, `1 - SmoothStep(-w, w, d)`, and `Lerp` white to red.

Show the outline version from 5.7 by wiring `Abs(d)` instead of `d`. Then subtract a box from the target with `Max(d, -box)` to cut a slot through all three rings.

**Step 9 - Value noise and fBm.** In `MyWork/Functions`, create `MF_Math_Hash` with a `Function Input Vector 2` `Cell` and an output `Result`, containing the hash from Notes 03 5.7. Then create `MF_Math_ValueNoise` following 5.8, calling `MF_Math_Hash` four times. In `MyWork`, create `MM_Prop_ValueNoiseFBM_Opaque`, call `MF_Math_ValueNoise` with `UV * 8`, and show it on `Base Color`.

Add three more calls at `UV * 16`, `UV * 32` and `UV * 64`, weight them 0.5, 0.25, 0.125 and 0.0625, and `Add` them: fBm. Then create `MM_Prop_NoiseNode_Opaque` with a single built-in `Noise` node, with **Function** set to `Value - ALU` and **Levels** 4, fed from `UV * 8` appended with 0, and show that it gives a similar result in one node. Click **Stats** on each version and compare the instruction counts.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| Rotating without centring | The pattern swings around the corner of the surface | `Subtract` (0.5, 0.5) before the rotation |
| `Sine` or `Cosine` left at the default **Period** | Rotation angle is wrong by a factor of 2 pi | Set **Period** to 6.283185 so the input is in radians |
| `Step` on a visible edge | Jagged staircase close up, shimmer and moire at a distance | `SmoothStep` with the width from `DDX` and `DDY` |
| Derivatives taken after `Frac` | A bright or blurred line along every cell edge | Take `DDX` and `DDY` of the smooth value before `Frac` |
| Brick offset applied after `Frac` | Rows do not shift, or bricks split in half | Compute the row with `Floor` and add the offset before `Frac` |
| Mortar thicker on the long sides | Uneven joints | Multiply the horizontal edge distance by the brick aspect |
| `Arctangent2` inputs swapped | The pattern is mirrored and rotated by 90 degrees | Y into **Y**, X into **X** |
| Polar pattern with a non-whole repeat count | A hard seam along the left edge | Use a whole number of repeats around the circle |
| Hash used as noise | A grid of flat random squares | Blend the four corner hashes with the smoothed `Lerp` |
| Too many octaves | Instruction count climbs with little visible change | Stop at four octaves, or use the `Noise` node's **Levels** |

---

## 8. Exercises

1. Write down the value of `Frac(Floor(row) * 0.5)` for rows 0 to 5, and explain why it produces the brick offset.
2. Build a checkerboard with clean edges using the triangle wave on U and V and the derivative width, rather than `Fmod`.
3. Draw a rounded rectangle as an SDF: a box distance with a radius subtracted. Make the radius a parameter and check that a radius of 0 gives a sharp box.
4. Turn the radar into a clock face: twelve thin sector lines from the angle, and a ring from the radius.
5. Using the `Noise` node, warp a set of horizontal stripes into marble veins. Record which parameter changes the vein waviness and which changes their spacing.

---

## 9. Reflective Questions

1. The tile mask in 5.4 drives both colour and roughness. Which other PBR inputs could the same mask drive, and what would each add?
2. A procedural brick wall has no texture memory cost. When would you still choose a scanned brick texture instead?
3. The derivative width makes far stripes fade to grey. Is that a bug or the correct result? Compare it with what a mipmapped texture does at a distance.
4. Domain warping makes regular patterns look natural. Which surfaces in a game would you never try to make this way, and why?

---

## 10. How This Builds

- **Lab 5** (`lab_5_procedural_tiles_and_bricks`) applies Session A. You build a brick wall with offset rows, even mortar, a random tint per brick and mortar-driven roughness. Its Extension builds the starfield and the hexagonal shield from 5.11.
- **Lab 6** (`lab_6_procedural_wood`) applies Session B. You build wood rings from the polar radius, warp them with noise, and compare the result with the Lab 1 wood texture. Its Extension builds the molten lava and the topographic map from 5.11.
- **Week 4** drives a dissolve with noise and an emissive edge, using the `SmoothStep` edge and the noise from this week.
- **Week 5** uses a procedural height to push vertices and to offset texture lookups for parallax.
- **Week 8** rewrites these patterns in HLSL in a Custom node, so you can compare the graph and the code for the same result.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **Domain warping** | Adding noise to the coordinates a pattern reads, so the pattern bends instead of the result being darkened or lightened |
| **fBm** | Fractional Brownian motion: several octaves of noise added together, each at double the frequency and half the amplitude |
| **fwidth** | The sum of the absolute screen-space derivatives of a value, `Abs(DDX) + Abs(DDY)`, which is how much it changes across one pixel |
| **Moire** | False wavy patterns that appear when a repeating pattern is finer than the pixel grid |
| **Octave** | One layer of noise in fBm |
| **Polar coordinates** | A position described as a distance from a centre and an angle around it |
| **Procedural texture** | A pattern computed in the shader rather than read from an image |
| **Signed distance field (SDF)** | A function giving the distance to a shape's edge: negative inside, positive outside, zero on the edge |
| **Triangle wave** | A repeating ramp that rises from 0 to 1 and falls back to 0, with no jump |
| **Value noise** | Smooth noise made by blending random values hashed at the corners of each grid cell |
