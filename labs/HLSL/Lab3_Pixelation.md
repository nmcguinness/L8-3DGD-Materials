# Lab 3 - Pixelation Post-Process with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab, you will implement a **pixelation post-process effect** in Unreal Engine using a **Custom HLSL node**.
Pixelation reduces the effective screen resolution by **quantising UV coordinates**. This produces a blocky "retro" look similar to old handheld consoles or stylistic indie games.

This lab is a natural continuation of earlier post-process labs and reinforces:

- screen-space UV manipulation,
- sampling `PostProcessInput0`,
- writing simple and readable HLSL,
- exposing artist-friendly parameters such as pixel grid size.

This is deliberately a *simple* HLSL effect suitable for early shader learners.

## Learning Outcomes

By completing this lab, you will be able to:

- Use a Post-Process Material for full-screen effects.
- Quantise UVs in HLSL to create pixelation.
- Understand how sampling frequency changes visual output.
- Expose scalar parameters to control pixel resolution.
- Use `SceneTextureLookup()` to fetch the rendered scene.

## 1. Create a Post Process Volume

1. Drag a **Post Process Volume** into your level.
2. In **Details**:
   - Set **Infinite Extent (Unbound)** -> *true*.
3. Under **Post Process Materials**, add an **Array Element** and set the type to **Asset Reference**.
4. You will assign the pixelation material here later.

## 2. Create the Pixelation Material

1. In the Content Browser:
   **Right-click -> Material**
   Name it:
   ```
   PPM_Pixelate
   ```
2. Open the material.
3. Set:
   - **Material Domain -> Post Process**

Only **Emissive Color** will be active.

## 3. Helper Custom Node: Screen UVs

Add a Custom node that returns correct per-pixel screen UVs.

1. Add a **Custom** node:
   - **Name:** `GetScreenUVs`
   - **Output Type:** `CMOT Float 2`

2. Paste this code:

```hlsl
return GetSceneTextureUV(Parameters);
```

This returns the current pixel's buffer UV, the UV `SceneTextureLookup` expects (see Lab 2).

###  Access the Scene Texture
1. Add a **SceneTexture** node.
2. Set **Scene Texture Id -> PostProcessInput0**

This gives you access to the final rendered scene images your filter will operate on. You will wire this node into an input called `sceneTexture` that the code never reads. That wire matters: `SceneTextureLookup(uv, 14, ...)` only works when a SceneTexture node for that texture is in the graph, because the node is what tells Unreal to make the texture available to the shader. Do not delete it.

## 4. Add Material Parameters

Add the following **Scalar Parameter**:

### `PixelCount`
- Default: `160`
- Meaning: number of pixels across the screen width.
- Higher values = more detail.
- Lower values = more blocky.

Add a second parameter for the vertical pixel count:

### `PixelCountY`
- Default: `90`
- Meaning: number of pixels down the screen. 160 by 90 keeps the blocks square on a 16:9 screen.

## 5. Main Custom Node - Pixelation Logic

We will create a Custom HLSL node that:

1. Receives the screen UV.
2. Quantises it to a pixel grid.
3. Samples the scene at the quantised UV.

### Add a Custom Node

1. Add a new **Custom** node:
   - **Name:** `Pixelate`
   - **Output Type:** `CMOT Float 3`

2. Add these **Inputs**:

| Input Name | Connect |
|:-|:-|
| `sceneTexture` | Output of `SceneTexture` |
| `uv` | Output of `GetScreenUVs` |
| `pixelCount` | Parameter `PixelCount` |
| `pixelCountY` | Parameter `PixelCountY` |

### HLSL Code

Paste the following into the Custom node:

```hlsl
// Inputs:
//   uv          : float2 buffer UV from GetSceneTextureUV
//   pixelCount  : number of horizontal pixels
//   pixelCountY : number of vertical pixels

// Quantise UVs to a pixel grid.
// Multiply by pixel count to scale UV to "pixel space".
float2 scaledUV = uv * float2(pixelCount, pixelCountY);

// Floor the scaled UV to snap it to the corner of its "pixel",
// then add 0.5 so we sample the centre of the block instead of its corner.
float2 pixelUV = floor(scaledUV) + 0.5;

// Convert back into standard 0..1 UV space.
float2 quantisedUV = pixelUV / float2(pixelCount, pixelCountY);

// Clamp for safety.
quantisedUV = clamp(quantisedUV, 0.0, 1.0);

// Sample the final rendered scene at the quantised UV.
// 14 = PostProcessInput0 (the scene colour).
float3 sceneColor = SceneTextureLookup(quantisedUV, 14, false).rgb;

// Output the pixelated scene colour.
return sceneColor;
```

## 6. Connect to Emissive Color

Connect the **Pixelate** Custom node output (Float3) -> **Emissive Color**.

Your material now replaces the final rendered image with the pixelated version.

## 7. Assign the Material in the Post Process Volume

1. Select the **Post Process Volume**.
2. Under **Post Process Materials**, assign:
   ```
   PPM_Pixelate
   ```
3. Ensure **Blend Weight = 1.0**.

Press **Play** - the scene should now appear blocky.

**Checkpoint:** The whole view is made of square blocks, 160 across and 90 down, and each block shows the colour from its centre.

## 8. Student Tasks

### Task 1 - Experiment with Pixel Counts
Try different values for:
- `PixelCount`: 80, 120, 160, 240
- `PixelCountY`: 45, 68, 90, 135

Observe:
- How does the "effective resolution" change?
- What values feel "retro handheld"?
- What values feel "stylistically modern"?

### Task 2 - Non-Uniform Pixelation
Try different X and Y resolutions to create:
- stretched pixels
- tall CRT-style pixels
- Game Boy-style squarer pixels

### Task 3 - Pixelation Mask (Advanced Extension)
Fade pixelation only:
- near the screen edges
- in bright areas
- or based on depth

Use `SceneTexture: SceneDepth` and `smoothstep()`.

## 9. Reflective Questions

1. Why does flooring UVs create a blocky pixel effect?
2. How does increasing `pixelCount` affect sharpness and detail?
3. What visual artefacts occur if UVs are not clamped?
4. How could you animate pixelation strength over time for a glitch effect?
5. How might pixelation be used as a gameplay tool (e.g., for status effects)?
