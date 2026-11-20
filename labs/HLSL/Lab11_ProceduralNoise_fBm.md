# Lab 11 - Procedural Noise Textures with fBm (Stone, Clouds, Marble)
## Module: 3D Game Development

## Overview

In this lab you will build **fully procedural textures** using **noise** and **fractal Brownian motion (fBm)** inside a Custom HLSL node. You will not use any image textures. Instead, you will:

- implement simple 2D value noise in HLSL
- build fBm by layering several octaves of noise
- use noise and fBm to generate stylised
  - stone / rock
  - clouds / fog
  - marble-like streaks

This lab builds on Notes 04, which covered noise and randomness, and focuses specifically on **fBm for texturing**.

## Learning Outcomes

By completing this lab, you will be able to:

- Implement 2D value noise and a simple fBm loop in HLSL.
- Use fBm as a mask to blend between different colours and material properties.
- Create at least two distinct procedural materials (stone and clouds) without image textures.
- Extend the basic pattern with a simple marble-style domain warp.

## What is fBm and why do we use it?

You have already seen how a single octave of noise produces a **blobby** pattern with a single "scale" of detail. Real surfaces (rock, clouds, terrain) have **structure at many scales**:

- large shapes (mountain forms)
- medium shapes (boulders, cloud lumps)
- fine detail (grains, cracks, wisps)

Fractal Brownian motion (fBm) is a simple way to approximate this by **adding several scaled versions of the same noise function**.

Key ideas:

- Each layer is called an **octave**.
- Each octave uses a higher **frequency** (more detail).
- Each octave uses a smaller **amplitude** (less contribution).

Typical pattern:

- Frequency multiplier (lacunarity) about 2.0
- Amplitude multiplier (gain) about 0.5

Simple pseudo-code for fBm:

```text
value = 0
amplitude = 0.5
frequency = 1.0

repeat for N octaves:
    value += amplitude * noise(uv * frequency)
    frequency *= 2.0       // more detail each octave
    amplitude *= 0.5       // smaller contribution each octave
```

Result:

- Single-octave noise -> soft, simple shapes.
- fBm -> richer, more natural textures suitable for rock, clouds, etc.

You will implement this directly in HLSL.

## 1. Create the Procedural Noise Material

1. In the Content Browser create a new Material:

   ```text
   MM_Prop_ProceduralNoise_Opaque
   ```

2. Set:
   - Material Domain -> Surface
   - Shading Model -> Unlit

3. You will output the final colour to **Emissive Color** so that the lighting does not affect the procedural texture.

## 2. Provide UVs and Base Parameters

Add a **TextureCoordinate** node for basic 0-1 UVs.

Create the following **parameters**:

| Parameter Name | Type | Default | Description |
| :- | :- | :- | :- |
| `BaseScale` | Scalar | 4.0 | Base tiling of the noise |
| `Octaves` | Scalar | 4.0 | Number of fBm noise layers |
| `Lacunarity` | Scalar | 2.0 | Frequency multiplier per octave |
| `Gain` | Scalar | 0.5 | Amplitude multiplier per octave |
| `MarbleAmount` | Scalar | 0.0 | 0=cloud/stone, 1=strong marble veins |
| `TimeScale` | Scalar | 0.0 | 0 = static, >0 = animated clouds |
| `ColorA` | Vector | (0.3, 0.3, 0.35) | Base colour (dark rock / sky) |
| `ColorB` | Vector | (0.8, 0.8, 0.85) | Highlight colour (stone veins / cloud tops) |

Multiply the TextureCoordinate by `BaseScale` and connect that into the Custom HLSL node as `uv`.

## 3. Add the Custom HLSL Node

Add a **Custom** node named `NoiseFBM`.

- Output Type -> `CMOT Float 1`

Add the following **Inputs**:

| Input Name | Type | Connect From |
| :- | :- | :- |
| `uv` | Float2 | TextureCoordinate x BaseScale |
| `octaves` | Float1 | `Octaves` parameter |
| `lacunarity` | Float1 | `Lacunarity` parameter |
| `gain` | Float1 | `Gain` parameter |
| `marbleAmount` | Float1 | `MarbleAmount` parameter |
| `timeValue` | Float1 | `Time` node x `TimeScale` |

Add a **Time** node and multiply it by `TimeScale` before connecting to `timeValue`.

## 4. HLSL Code - Noise + fBm + Simple Marble

Paste the following into the `NoiseFBM` Custom node. The code is fully commented for teaching. The helper functions sit inside a struct, because a Custom node's code is itself the body of a function and HLSL does not allow a function inside a function. Calling them through an instance, `N.fbm(...)`, is the struct trick from Notes 10 5.3.

```hlsl
// Inputs:
//   uv           : float2 UV coordinates (scaled in the material)
//   octaves      : number of noise layers (fBm octaves)
//   lacunarity   : frequency multiplier per octave
//   gain         : amplitude multiplier per octave
//   marbleAmount : blends between pure fBm (0) and marble-like pattern (1)
//   timeValue    : time input for animation (e.g., moving clouds)

// A Custom node's code is the body of a function, and HLSL does not allow
// a function inside a function. A struct can contain functions, though,
// so the helpers live in a struct and we call them through one instance of it.
struct NoiseFunctions
{
    // ---------------------------
    // Hash function for 2D value noise
    // ---------------------------
    // Converts a 2D position into a pseudo-random value in [0,1].
    float hash(float2 p)
    {
        // Dot product with fixed large constants to mix the coordinates
        float h = dot(p, float2(127.1, 311.7));
        // Use sin + large multiplier, then frac to keep in [0,1]
        return frac(sin(h) * 43758.5453123);
    }

    // ---------------------------
    // 2D Value Noise
    // ---------------------------
    // Produces smooth noise in [0,1] by interpolating between
    // random values at the corners of the integer grid cell.
    float valueNoise(float2 p)
    {
        // Get integer cell coordinates (bottom-left corner of the cell)
        float2 i = floor(p);
        // Get fractional part within the cell (0..1)
        float2 f = frac(p);

        // Random values at the four corners of the cell
        float v00 = hash(i + float2(0.0, 0.0)); // bottom-left
        float v10 = hash(i + float2(1.0, 0.0)); // bottom-right
        float v01 = hash(i + float2(0.0, 1.0)); // top-left
        float v11 = hash(i + float2(1.0, 1.0)); // top-right

        // Smooth the fractional coordinates using a Hermite curve (smoothstep-like)
        float2 u = f * f * (3.0 - 2.0 * f);

        // Bilinear interpolation:
        // Interpolate horizontally on bottom and top edges
        float bottom = lerp(v00, v10, u.x);
        float top    = lerp(v01, v11, u.x);

        // Interpolate vertically between bottom and top
        return lerp(bottom, top, u.y); // in [0,1]
    }

    // ---------------------------
    // fBm (Fractal Brownian Motion)
    // ---------------------------
    // Stacks multiple octaves of valueNoise with increasing frequency
    // and decreasing amplitude to create richer patterns.
    float fbm(float2 p, int numOctaves, float lacunarity, float gain)
    {
        float value = 0.0;     // accumulated noise value
        float amplitude = 0.5; // starting amplitude
        float frequency = 1.0; // starting frequency

        // Loop over octaves
        for (int i = 0; i < numOctaves; ++i)
        {
            // Sample value noise at current frequency, scaled by amplitude
            value += valueNoise(p * frequency) * amplitude;

            // Increase frequency and decrease amplitude for the next octave
            frequency *= lacunarity;
            amplitude *= gain;
        }

        return value; // typically in [0,1] range, but may exceed slightly
    }
};

// One instance of the struct, so we can call N.fbm(...) below.
NoiseFunctions N;

// ---------------------------
// Main Body
// ---------------------------

// Convert the float octaves input into an integer number of octaves.
// floor() ensures fractional values are safe.
int numOctaves = max((int)floor(octaves), 1);

// Optionally animate UVs over time for cloud-like motion.
// We offset UV slightly by timeValue in one direction.
float2 animatedUV = uv + float2(timeValue * 0.05, timeValue * 0.02);

// Compute base fBm noise for the animated UVs.
float baseFBM = saturate(N.fbm(animatedUV, numOctaves, lacunarity, gain));

// ---------------------------
// Marble variant (simple domain warp)
// ---------------------------
// To get a marble-like effect, we distort a coordinate using fBm
// and feed it through a sine function. This creates wavy bands
// similar to marble veins.

// Multiply uv.x to control how many bands appear.
float bands = 6.0;
float baseCoord = uv.x * bands;

// Use the fbm to warp the base coordinate.
float warpStrength = 2.0;
float warpedCoord = baseCoord + baseFBM * warpStrength;

// Compute sine-based pattern for marble.
// sin() outputs -1..1; we remap to 0..1.
float marble = sin(warpedCoord) * 0.5 + 0.5;

// Optionally sharpen contrast for stronger veins.
marble = smoothstep(0.2, 0.8, marble);

// ---------------------------
// Blend between fBm noise and marble pattern
// ---------------------------
// marbleAmount = 0 -> pure fBm (good for stone/clouds)
// marbleAmount = 1 -> pure marble (veiny pattern)
float finalMask = lerp(baseFBM, marble, saturate(marbleAmount));

// Return final mask (0..1) to be used as a colour blend factor.
return finalMask;
```

## 5. Colouring the Noise Pattern

You now have a single scalar mask in [0,1] coming from `NoiseFBM`. Use it to blend between two colours.

1. Add a **Lerp (LinearInterpolate)** node.
2. Connect:
   - A -> `ColorA`
   - B -> `ColorB`
   - Alpha -> output of `NoiseFBM`
3. Connect Lerp output -> **Emissive Color**.

At this point the material is fully procedural and uses no images.

**Checkpoint:** The surface shows soft grey cloud-like noise with detail at several scales. Change `Octaves` from 1 to 6 and finer detail appears with each step.

## 6. Creating Material Instances

Create several Material Instances of `MM_Prop_ProceduralNoise_Opaque`, for example:

- `MI_Prop_NoiseStone_Opaque`
- `MI_Prop_NoiseClouds_Opaque`
- `MI_Prop_NoiseMarble_Opaque`

Suggested starting values:

| Instance | BaseScale | Octaves | Lacunarity | Gain | MarbleAmount | TimeScale | Notes |
| :- | :- | :- | :- | :- | :- | :- | :- |
| Stone | 6.0 | 4.0 | 2.0 | 0.5 | 0.0 | 0.0 | Static, detailed stone |
| Clouds | 3.0 | 5.0 | 2.0 | 0.5 | 0.0 | 0.3 | Soft, animated clouds |
| Marble | 4.0 | 4.0 | 2.0 | 0.5 | 1.0 | 0.0 | Strong marble bands |

Experiment by changing colours:

- Stone: dark greys to light greys or browns.
- Clouds: dark blue for sky, near-white for highlights.
- Marble: coloured veins (e.g., green, blue, or gold).

**Checkpoint:** The three instances look clearly different: grainy stone, soft clouds that drift when you click **Simulate**, and wavy marble veins.

## 7. Student Tasks

### Task 1 - Build a Rock Material

Configure `MI_Prop_NoiseStone_Opaque` with:

- No animation (`TimeScale = 0`)
- 3-5 octaves
- Moderate BaseScale (e.g., 6-10)

Add details:

- Slight colour variation (brown or mossy tint).
- Optional: output the `NoiseFBM` result to the Roughness channel (inverted or remapped) to suggest rough vs smooth patches.

### Task 2 - Animated Cloud Layer

Configure `MI_Prop_NoiseClouds_Opaque` with:

- Several octaves (4-6)
- Low BaseScale (large soft shapes)
- Non-zero `TimeScale` to animate clouds slowly

Stretch the UVs in one direction (e.g., multiply `uv.y` by 0.5 in the material graph) to create more elongated cloud formations.

### Task 3 - Marble Variant

Configure `MI_Prop_NoiseMarble_Opaque` with:

- `MarbleAmount = 1`
- 3-4 octaves
- BaseScale ~ 4-6

Experiment with:

- Number of bands (edit `bands` constant in HLSL).
- Different colours for veins vs base.
- Strong contrast for an exaggerated stylised look.

## 8. Reflective Questions

1. How does increasing the number of octaves affect the look of the texture? At what point does it become too noisy?
2. What is the visual effect of changing `Lacunarity` vs changing `Gain` in the fBm loop?
3. Why does adding fBm-based warping before the sine function produce marble-like patterns instead of simple stripes?
4. How does the `TimeScale` parameter change your perception of clouds or fog compared to a static pattern?
5. In a real production game, where might procedural noise-based textures be more appropriate than image textures, and where might image textures still be preferred?
