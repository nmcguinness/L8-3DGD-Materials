# Lab 8 - Procedural Patterns with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will create your first **procedural textures** using only maths and HLSL - no image textures. Procedural patterns are:

- resolution-independent
- memory-light
- easy to recolour and animate
- ideal for stylised or technical art

You will implement:

- UV-based **stripes**, **checkerboard**, and **radial masks**
- **sine wave based patterns** suitable for wire, cables, and chain-link fencing

A single material will expose parameters that allow artists to experiment with different looks from a small amount of clean HLSL.

## Learning Outcomes

By completing this lab, you will be able to:

- Use UV maths to generate repeating procedural patterns.
- Implement conditional pattern selection in a Custom HLSL node.
- Build a sine-wave based pattern suitable for fences, wires, and weaves.
- Expose parameters for frequency, amplitude, thickness, and tiling.

## 1. Create the Base Material

1. In the Content Browser create a Material:

   ```text
   MM_Prop_ProceduralPatterns_Opaque
   ```

2. Set:
   - **Material Domain -> Surface**
   - **Shading Model -> Unlit**

3. You will output colour to **Emissive Color**.

## 2. Provide UVs and Base Parameters

Add a **TextureCoordinate** node for basic 0-1 UVs.

Create the following **parameters**:

| Parameter Name | Type | Default | Description |
| :- | :- | :- | :- |
| `UVScale` | Scalar | 5.0 | Overall tiling for basic patterns |
| `LineWidth` | Scalar | 0.25 | Stripe / checker line thickness |
| `Pattern` | Scalar | 0.0 | Pattern selector (0=stripes, 1=checker, 2=radial) |
| `ColorA` | Vector | (0, 0, 0) | Base colour |
| `ColorB` | Vector | (1, 1, 1) | Pattern colour |

Multiply `TextureCoordinate` by `UVScale` and connect this to the `uv` input of the first Custom node.

## 3. Custom Node 1 - Basic Patterns (Stripes, Checker, Radial)

Add a **Custom** node named `ProceduralPatternsBasic`.

- **Output Type:** `CMOT Float 1`

Inputs:

| Input Name | Type | Connect From |
| :- | :- | :- |
| `uv` | Float2 | Scaled UVs from TextureCoordinate x UVScale |
| `lineWidth` | Float1 | `LineWidth` parameter |
| `patternID` | Float1 | `Pattern` parameter |

### 3.1 HLSL Code (Basic Patterns)

Paste this HLSL into the Custom node:

```hlsl
// Inputs:
//   uv        : float2 UV coordinates (scaled in the material graph)
//   lineWidth : thickness of stripes and checkerboard lines
//   patternID : selects which pattern to output (0=stripes, 1=checker, 2=radial)

// Initialise the output mask to 0 (black).
float mask = 0.0;

// ------------------------------
// Pattern 0: Vertical Stripes
// ------------------------------
// frac(uv.x) returns the fractional part of uv.x in the range [0,1).
// step(a, x) returns 0 when x < a, 1 when x >= a.
// We want a narrow "on" band where frac(uv.x) is less than lineWidth.
float stripe = step(frac(uv.x), lineWidth);

// ------------------------------
// Pattern 1: Checkerboard
// ------------------------------
// floor(uv.x) and floor(uv.y) give us integer cell indices.
// Adding them together and taking modulo 2 alternates between 0 and 1.
float cx = floor(uv.x);
float cy = floor(uv.y);

// fmod(x, 2) gives the remainder after dividing by 2.
// This produces a 0/1 pattern across integer grid cells.
float checker = fmod(cx + cy, 2.0);

// Here we use step(0.5, checker) so that values < 0.5 -> 0, >= 0.5 -> 1.
checker = step(0.5, checker);

// ------------------------------
// Pattern 2: Radial Mask
// ------------------------------
// Use frac(uv) so the pattern repeats in every cell,
// then shift so that the centre of each cell is at (0, 0).
float2 centred = frac(uv) - float2(0.5, 0.5);

// Compute the distance from the centre (radius).
float radius = length(centred);

// Use smoothstep to create a soft circular mask.
// For radius <= 0.3 -> 0 (inside the circle).
// For radius >= 0.5 -> 1 (outside the circle).
float radial = smoothstep(0.3, 0.5, radius);

// ------------------------------
// Select Pattern Based on patternID
// ------------------------------
// We compare patternID against ranges to decide which pattern to output.
// Assume patternID values 0, 1, or 2.
if (patternID < 0.5)
{
    // 0 -> vertical stripes
    mask = stripe;
}
else if (patternID < 1.5)
{
    // 1 -> checkerboard
    mask = checker;
}
else
{
    // 2 -> radial mask
    mask = radial;
}

// Return final mask in the range [0,1].
return mask;
```

## 4. Colouring the Basic Patterns

1. Add a **Lerp (LinearInterpolate)** node.
2. Connect:
   - **A:** `ColorA`
   - **B:** `ColorB`
   - **Alpha:** output of `ProceduralPatternsBasic`
3. Connect the Lerp output -> **Emissive Color**.

At this stage you have a procedural stripes/checker/radial material.

**Checkpoint:** With `Pattern` at 0 the surface shows 5 vertical stripes, at 1 a 5 by 5 checkerboard, and at 2 a 5 by 5 grid of soft dark circles, one in every tile.

## 5. Material Instance - Basic Patterns

Create a Material Instance:

```text
MI_Prop_PatternsBasic_Opaque
```

Test:

- `Pattern = 0` -> stripes
- `Pattern = 1` -> checkerboard
- `Pattern = 2` -> radial

Change `UVScale`, `LineWidth`, and swap `ColorA` / `ColorB` to see the effect.

## 6. Extending to Wave-Based Patterns (Fencing, Wire, Weave)

Now we extend the lab to generate **wave-based patterns** suitable for:

- chain-link fences
- metal wire mesh
- stylised fabric/weave patterns

The idea is that **everything starts from a single sine wave**:

- We control its **amplitude** (height) and **frequency** (how many cycles).
- We then **duplicate** the wave vertically using a loop.
- We optionally **flip** every second wave to produce a woven or chain-link look.

We will implement this in a second Custom node so the logic is easy to explain.

## 7. Additional Parameters for Wave Patterns

Add these parameters to the material:

| Parameter Name | Type | Default | Description |
| :- | :- | :- | :- |
| `WaveAmplitude` | Scalar | 0.1 | Height of the sine wave, as a fraction of one tile |
| `WaveFrequency` | Scalar | 3.0 | Whole wave cycles across one tile in U |
| `WaveThickness` | Scalar | 0.02 | Visual thickness of the wire |
| `WaveCopies` | Scalar | 6.0 | Number of stacked wave rows in each tile |
| `WaveMirrorStep` | Scalar | 2.0 | Flip every Nth row (for weave) |
| `WaveOffset` | Scalar | 0.0 | Phase offset along U (animation) |

## 8. Custom Node 2 - Wave Pattern Generator

Add a second **Custom** node named `WavePattern`.

- **Output Type:** `CMOT Float 1`

Connect the **same scaled UVs** as before (TextureCoordinate x UVScale) to the input `uv` of this node, and add the following inputs:

| Input Name | Type | Connect From |
| :- | :- | :- |
| `uv` | Float2 | Scaled UVs |
| `amplitude` | Float1 | `WaveAmplitude` |
| `frequency` | Float1 | `WaveFrequency` |
| `thickness` | Float1 | `WaveThickness` |
| `copies` | Float1 | `WaveCopies` |
| `mirrorStep` | Float1 | `WaveMirrorStep` |
| `offsetU` | Float1 | `WaveOffset` |

### 8.1 HLSL Code - Wave-Based Fence / Chain-Link Pattern

Paste this HLSL into the `WavePattern` Custom node:

```hlsl
// Inputs:
//   uv         : float2 UV coordinates (scaled in the material)
//   amplitude  : vertical height of each sine wave, as a fraction of one tile
//   frequency  : number of whole wave cycles across one tile in U
//   thickness  : visual thickness of the "wire"
//   copies     : how many stacked rows of waves we draw in each tile
//   mirrorStep : flip every Nth row to create a woven look
//   offsetU    : phase offset along U (useful for animation)

// PI is already defined by the engine, so we can use it directly.

// Start with an empty mask (black).
float result = 0.0;

// Convert 'copies' and 'mirrorStep' to integers for loop and modulus.
// floor() ensures we use whole numbers even if parameters are fractional.
int numCopies = max((int)floor(copies), 1);
int mirrorEvery = max((int)floor(mirrorStep), 1); // avoid zero

// Work inside one tile: frac(uv.y) repeats 0..1 in every tile,
// so the rows repeat with the UV scale instead of filling only the first tile.
float v = frac(uv.y);

// Loop over each copy (row) of the wave.
for (int i = 0; i < numCopies; ++i)
{
    // Spread rows evenly through the tile, with a half-row gap at each edge,
    // so rows in neighbouring tiles are spaced the same as rows within a tile.
    float baseV = ((float)i + 0.5) / (float)numCopies;

    // Decide whether this row should be flipped.
    // If (i % mirrorEvery) == 0 -> flip direction, else normal.
    int modVal = i % mirrorEvery;
    float waveDirection = (modVal == 0) ? -1.0 : 1.0;

    // Compute the horizontal position along the wave.
    // One full cycle is 2 * PI, so whole-number frequencies tile cleanly.
    // We add offsetU to allow sliding/animation.
    float u = uv.x + offsetU;
    float wavePhase = u * frequency * 2.0 * PI;

    // Compute the vertical offset of the sine wave at this U.
    // sin(wavePhase) returns -1..1; we scale by amplitude.
    float waveHeight = amplitude * sin(wavePhase * waveDirection);

    // Combine the base vertical position with the wave height
    // to get the curve's centre line in V.
    float curveV = baseV + waveHeight;

    // Measure how close the current pixel's V is to the curve centre.
    float distanceToCurve = abs(v - curveV);

    // Convert distance to a thickness mask.
    // The smaller the distance, the stronger the mask.
    float wire = 1.0 - smoothstep(thickness, thickness * 2.0, distanceToCurve);

    // Accumulate this row's contribution into the result.
    // Using max ensures overlapping rows do not exceed 1.0.
    result = max(result, wire);
}

// Return the final wave-based mask (0..1).
return result;
```

## 9. Blending Between Basic and Wave Patterns

There are a few options. A simple approach is:

1. Add a new **Scalar Parameter**: `UseWave` (0 or 1).
2. Use a Lerp to blend between the Basic mask and Wave mask.

Steps:

1. Add a **Scalar Parameter** `UseWave` (default 0).
2. Add a **Lerp** node:
   - A: output of `ProceduralPatternsBasic`
   - B: output of `WavePattern`
   - Alpha: `UseWave`
3. Feed the result of this Lerp into the colour Lerp's **Alpha** instead of the Basic mask alone.

Now:

- `UseWave = 0` -> use the original stripe/checker/radial patterns.
- `UseWave = 1` -> use the wave-based fence pattern.
- Values in between allow blending.

## 10. Material Instance - Wave Patterns

Create another Material Instance:

```text
MI_Prop_PatternsWave_Opaque
```

Suggested experiments:

- `WaveAmplitude`: 0.05-0.15
- `WaveFrequency`: 2-5 (whole numbers tile cleanly)
- `WaveCopies`: 4-10
- `WaveThickness`: 0.01-0.03
- `WaveMirrorStep`: 1 (all same direction), 2, or 3 (woven look)
- `UseWave`: 1

Try colours:

- Dark background (`ColorA`) and bright wire (`ColorB`) for fences.
- Two bright colours for more abstract weave patterns.

**Checkpoint:** With `UseWave` at 1 and the suggested values, every tile shows rows of wires, and alternate rows are flipped so they cross into a chain-link diamond pattern that continues seamlessly from tile to tile.

## 11. Student Tasks - Wave Patterns

### Task 1 - Animate the Fence

Drive `offsetU` with a Time node from the material graph to create a sliding wave:

- e.g. `offsetU = Time * 0.1`

Observe how the fence appears to "flow" or vibrate.

### Task 2 - Diagonal Fence

Modify the HLSL to use a combination of `uv.x` and `uv.y` for the wave phase (for example, `uv.x + uv.y`) to create diagonal or skewed fences.

### Task 3 - Layered Weave

Use two calls to `WavePattern` with different parameters (amplitude, copies, mirrorStep) and combine them with `max()` to create a more complex woven fabric pattern.

## 12. Reflective Questions

1. How does changing `WaveFrequency` affect the apparent "tightness" of the fence or weave?
2. What is the visual effect of increasing `WaveCopies` without adjusting `UVScale`?
3. How does `WaveMirrorStep` help create a more woven or chain-link look compared to all waves having the same direction?
4. Why do we use `smoothstep` instead of a hard `step` when converting distance to a thickness mask?
5. In a real game, where could wave-based procedural patterns be useful beyond chain-link fencing (for example, cables, fabric details, UI backgrounds)?
