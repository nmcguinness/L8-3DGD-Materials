# Lab 2 - Screen-Space Vignette with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab, you will implement a **screen-space vignette** effect as a **Post-Process Material** using a **Custom HLSL node** in Unreal Engine.

A vignette darkens (or tints) the edges of the screen, drawing the player's eye toward the centre. This is a very common camera/post effect in modern games and is mathematically simple, making it ideal for early HLSL practice.

The vignette will be created by:

- computing the **distance from the screen centre** for each pixel,
- using `smoothstep` to create a soft radial falloff,
- blending this radial mask with the original scene colour.

## Learning Outcomes

By completing this lab, you will be able to:

- Create a **Post-Process Material** that modifies the final rendered image.
- Use a **Custom HLSL node** to compute a radial mask based on UV coordinates.
- Apply `smoothstep` to create a soft vignette transition.
- Blend the vignette with the scene colour using an intensity parameter.
- Expose vignette radius and softness as artist-friendly controls.

## 1. Create a Post Process Volume

1. Drag a **Post Process Volume** into your level.
2. In the Details panel:
   - Enable **Infinite Extent (Unbound)** -> *true*.
3. Under **Post Process Materials**, add an **Array Element**.
   - Set the type to **Asset Reference**.
4. You will assign the vignette material here later.

## 2. Create the Vignette Material

1. In the Content Browser, create a new Material:

   ```text
   PPM_Vignette
   ```

2. Open the material and set:
   - **Material Domain -> Post Process**

Only **Emissive Color** will be used as the output.

## 3. Helper Custom Node - Screen UVs

You will use a tiny Custom node to obtain the screen UV of each pixel. `GetSceneTextureUV` is an engine function that returns the UV `SceneTextureLookup` expects, which is a **buffer UV**. In the editor the render buffer can be larger than the viewport, so buffer UVs do not run exactly from 0 at one edge of the viewport to 1 at the other. You will use them for sampling, and the **ViewportUV** of a `ScreenPosition` node for the shape of the effect.

1. Add a **Custom** node:
   - **Name:** `GetScreenUVs`
   - **Output Type:** `CMOT Float 2`

2. In the **Code** field, paste:

   ```hlsl
   return GetSceneTextureUV(Parameters);
   ```

This returns the current pixel's buffer UV, for `SceneTextureLookup`.

###  Access the Scene Texture
1. Add a **SceneTexture** node.
2. Set **Scene Texture Id -> PostProcessInput0**

This gives you access to the final rendered scene images your filter will operate on. You will wire this node into an input called `sceneTexture` that the code never reads. That wire matters: `SceneTextureLookup(uv, 14, ...)` only works when a SceneTexture node for that texture is in the graph, because the node is what tells Unreal to make the texture available to the shader. Do not delete it.

Add a **ScreenPosition** node as well. Its **ViewportUV** output runs from (0, 0) at the top-left of the viewport to (1, 1) at the bottom-right.

## 4. Add Vignette Parameters

Add the following **Scalar Parameters** to control the effect:

| Parameter Name      | Default | Description                           |
| :- | :- | :- |
| `VignetteRadius`    | `0.4`   | Radius where darkening begins        |
| `VignetteSoftness`  | `0.25`  | How softly the vignette fades out    |
| `VignetteIntensity` | `1.0`   | Strength of the vignette effect      |

You may group these parameters visually in the material instance under a **Vignette** group (optional, for organisation).

## 5. Main Custom Node - Vignette Logic

You will now create a **Custom HLSL node** that:

1. Reads the screen UV.
2. Computes the radial distance from the centre.
3. Uses `smoothstep` to calculate a vignette mask.
4. Samples the scene colour.
5. Blends the scene colour with the vignette using an intensity parameter.

### 5.1 Add the Custom Node

1. Add a new **Custom** node:
   - **Name:** `Vignette`
   - **Output Type:** `CMOT Float 3`

2. In the **Inputs** list for this node, add:

| Input Name | Connect |
|:-|:-|
| `sceneTexture` | Output of `SceneTexture` |
| `uv`                | Output of `GetScreenUVs`            |
| `viewportUV`        | `ViewportUV` output of `ScreenPosition` |
| `vignetteRadius`    | `VignetteRadius` parameter          |
| `vignetteSoftness`  | `VignetteSoftness` parameter        |
| `vignetteIntensity` | `VignetteIntensity` parameter       |

### 5.2 HLSL Code

Paste the following **fully commented** HLSL code into the Custom node's **Code** field:

```hlsl
// Inputs:
//   uv                : float2 buffer UV from GetSceneTextureUV, used for sampling
//   viewportUV        : float2 viewport UV in [0, 1] from ScreenPosition, used for the shape
//   vignetteRadius    : scalar radius where darkening begins
//   vignetteSoftness  : scalar width of the soft edge
//   vignetteIntensity : scalar intensity of the vignette effect

// Step 1: Move the viewport UV so that (0.5, 0.5) is treated as the origin (screen centre).
// After this, the centre is (0, 0) and corners are roughly (-0.5, -0.5) to (0.5, 0.5).
float2 centredUV = viewportUV - float2(0.5, 0.5);

// Step 2: Compute distance from the screen centre.
// This is the radius in screen space.
float dist = length(centredUV);

// Step 3: Use smoothstep to create a radial falloff.
// We define an inner radius (vignetteRadius) and an outer radius
// (vignetteRadius + vignetteSoftness).
// - For dist <= vignetteRadius, the smoothstep output is 0 (no darkening).
// - For dist >= vignetteRadius + vignetteSoftness, the output is 1 (full darkening).
float edge = smoothstep(vignetteRadius, vignetteRadius + vignetteSoftness, dist);

// Step 4: Convert this "edge" into a vignette factor that is 1 at the centre
// and approaches 0 towards the edges.
float vignetteFactor = 1.0 - edge;

// Step 5: Sample the original scene colour from PostProcessInput0.
// 14 = PostProcessInput0. SceneTextureLookup expects buffer UVs, so we pass uv here.
float3 sceneColor = SceneTextureLookup(uv, 14, false).rgb;

// Step 6: Compute a final darkening factor based on vignetteIntensity.
// - When vignetteIntensity = 0, factor is 1.0 (no change).
// - When vignetteIntensity = 1, factor is vignetteFactor (full vignette).
float finalFactor = lerp(1.0, vignetteFactor, vignetteIntensity);

// Step 7: Apply the vignette by multiplying the scene colour by the final factor.
float3 finalColor = sceneColor * finalFactor;

// Step 8: Return the final vignetted scene colour.
return finalColor;
```

## 6. Connect Output to Emissive Color

Connect the **Vignette** Custom node output (Float3) -> **Emissive Color**.

Your material now applies a vignette to the entire scene as a post-process effect.

## 7. Assign the Material to the Post Process Volume

1. Go back to your level and select the **Post Process Volume**.
2. Under **Post Process Materials**, assign:

   ```text
   PPM_Vignette
   ```

3. Set **Blend Weight = 1.0**.

Press **Play** to see the vignette darkening the screen edges.

**Checkpoint:** The corners of the view darken smoothly while the centre is unchanged, and the dark ring stays centred however you resize the viewport. Set `VignetteIntensity` to 0 and the image returns to normal.

## 8. Student Tasks

### Task 1 - Tuning Radius and Softness

Use a **Material Instance** of `PPM_Vignette` and experiment with:

- `VignetteRadius` values between `0.2` and `0.6`.
- `VignetteSoftness` values between `0.1` and `0.4`.

Questions to consider:

- How does a small radius with high softness look compared to a larger radius with low softness?
- Which settings feel more subtle vs more dramatic?

### Task 2 - Dynamic Vignette for Low Health (Design Exercise)

Imagine a game where the vignette becomes stronger as the player's health decreases.

- How could you drive `VignetteIntensity` from a Blueprint (no need to implement fully, just outline the idea)?
- What intensity values might you use for **full health** vs **critical health**?

### Task 3 - Experimental Colour Vignette (Optional Extension)

Extend the vignette to apply **colour tinting** to the edges:

1. Add a **Vector Parameter** called `VignetteTint` (default: black or dark red).
2. Modify the HLSL so that:
   - the edges are tinted by `VignetteTint`,
   - the centre remains close to the original scene colour.

Hint: you could lerp between `sceneColor` and `VignetteTint * sceneColor` using the same `edge` or `1 - vignetteFactor`.

## 9. Reflective Questions

1. Why do we subtract `0.5` from the UVs before computing the distance?
2. How does `smoothstep` help create a smooth transition for the vignette compared to a hard threshold?
3. What artefacts might appear if the vignette radius or softness values are set poorly (for example, radius too close to 0 or 1)?
4. How does changing `VignetteIntensity` differ visually from changing `VignetteRadius`?
5. In what situations might a vignette improve player focus or readability, and in what situations might it be distracting?
