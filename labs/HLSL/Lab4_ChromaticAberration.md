# Lab 4 - Chromatic Aberration Post-Process with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab, you will implement a **chromatic aberration** effect as a **Post-Process Material** using a **Custom HLSL node** in Unreal Engine.

Chromatic aberration is a lens effect where different colour channels (red, green, blue) are refracted slightly differently, producing subtle colour fringes near the edges of the screen. It is widely used in games to:

- add cinematic flavour
- enhance impact moments (e.g. taking damage, explosions)
- give a slightly "dreamlike" or "unstable" feel

You will create this effect by:

- computing a direction from the screen centre to each pixel
- offsetting **R**, **G**, and **B** samples by slightly different amounts
- blending everything using a user-controlled **intensity** parameter

This lab builds on your earlier post-process work (vignette, pixelation) and further develops your understanding of **screen-space sampling**.

## Learning Outcomes

By completing this lab, you will be able to:

- Implement a post-process chromatic aberration effect using HLSL.
- Compute radial directions in screen space using UV coordinates.
- Offset red and blue channels independently from green.
- Control aberration strength with artist-friendly parameters.
- Understand the visual impact of per-channel UV sampling.

## 1. Create a Post Process Volume

1. Drag a **Post Process Volume** into your level.
2. In the Details panel:
   - Enable **Infinite Extent (Unbound)** -> *true*.
3. Under **Post Process Materials**, add an **Array Element**.
   - Set the type to **Asset Reference**.
4. You will assign the chromatic aberration material here later.

## 2. Create the Chromatic Aberration Material

1. In the Content Browser, create a new Material:

   ```text
   PPM_ChromaticAberration
   ```

2. Open the material and set:
   - **Material Domain -> Post Process**

You will output the final colour to **Emissive Color**.

## 3. Helper Custom Node - Screen UVs

As in Lab 2, you will use a small Custom node to get the buffer UV for sampling, and the **ViewportUV** of a `ScreenPosition` node for the shape of the effect.

1. Add a **Custom** node:
   - Name: `GetScreenUVs`
   - Output Type: `CMOT Float 2`

2. In the Code field, paste:

   ```hlsl
   return GetSceneTextureUV(Parameters);
   ```

This returns the current pixel's buffer UV, for `SceneTextureLookup`.

###  Access the Scene Texture
1. Add a **SceneTexture** node.
2. Set **Scene Texture Id -> PostProcessInput0**

This gives you access to the final rendered scene images your filter will operate on. You will wire this node into an input called `sceneTexture` that the code never reads. That wire matters: `SceneTextureLookup(uv, 14, ...)` only works when a SceneTexture node for that texture is in the graph, because the node is what tells Unreal to make the texture available to the shader. Do not delete it.

Add a **ScreenPosition** node as well, for its **ViewportUV** output.

## 4. Add Chromatic Aberration Parameters

Add the following **Scalar Parameters** to control your effect:

| Parameter Name | Default | Description |
| :- | :- | :- |
| `AberrationStrength` | `0.004` | Base strength of RGB separation |
| `EdgeBoost` | `1.0` | Extra scaling toward the edges (0 = uniform) |
| `CenterDeadZone` | `0.05` | Radius around the centre with no effect |

You can optionally group these under a "ChromaticAberration" group when working with a Material Instance.

## 5. Main Custom Node - Chromatic Aberration Logic

You will now create a **Custom HLSL node** that:

1. Reads the screen UV.
2. Computes a radial direction from the screen centre.
3. Computes an offset amount that grows toward the screen edges.
4. Samples red, green, and blue channels at slightly different UVs.
5. Outputs the combined colour.

### 5.1 Add the Custom Node

1. Add a new **Custom** node:
   - Name: `ChromaticAberration`
   - Output Type: `CMOT Float 3`

2. Add the following **Inputs**:

| Input Name | Connect |
|:-|:-|
| `sceneTexture` | Output of `SceneTexture` |
| `uv` |  Output of `GetScreenUVs` |
| `viewportUV` | `ViewportUV` output of `ScreenPosition` |
| `aberrationStrength` | `AberrationStrength` parameter |
| `edgeBoost` |  `EdgeBoost` parameter |
| `centerDeadZone` | `CenterDeadZone` parameter |

### 5.2 HLSL Code

Paste this fully commented HLSL code into the Custom node's Code field:

```hlsl
// Inputs:
//   uv                 : float2 buffer UV from GetSceneTextureUV, used for sampling
//   viewportUV         : float2 viewport UV in [0, 1] from ScreenPosition, used for the shape
//   aberrationStrength : base scalar strength of channel separation
//   edgeBoost          : extra scaling towards screen edges
//   centerDeadZone     : radius near the centre with no effect

// Step 1: Compute centred UV so that (0.5, 0.5) is the origin.
float2 centred = viewportUV - float2(0.5, 0.5);

// Step 2: Compute distance from the centre (radius).
float dist = length(centred);

// Step 3: Compute a direction vector from the centre to this pixel.
// To avoid division by zero, use a very small epsilon when normalising.
float epsilon = 1e-6;
float2 dir = centred / max(dist, epsilon);

// Step 4: Compute a radial factor that is 0 inside the dead zone and rises to 1 at the edges.
float radialFactor = saturate((dist - centerDeadZone) / (0.5 - centerDeadZone));

// Step 5: Final offset magnitude. It is 0 inside the dead zone, grows towards the edges,
// and edgeBoost makes it grow faster near the edges.
// edgeBoost = 0 -> linear increase
// edgeBoost = 1 -> stronger increase towards the edges
float offsetAmount = aberrationStrength * radialFactor * (1.0 + edgeBoost * radialFactor);

// Step 6: Compute separate UVs for each colour channel.
// We offset red and blue in opposite directions along the radial dir,
// while green stays at the original UV (for stability).
float2 uvR = uv + dir * offsetAmount;
float2 uvG = uv;
float2 uvB = uv - dir * offsetAmount;

// Step 7: Clamp UVs so we do not sample outside the screen region.
uvR = clamp(uvR, 0.0, 1.0);
uvB = clamp(uvB, 0.0, 1.0);

// Step 8: Sample the scene colour for each channel from PostProcessInput0.
// 14 = PostProcessInput0 (scene colour).
float3 colR = SceneTextureLookup(uvR, 14, false).rgb;
float3 colG = SceneTextureLookup(uvG, 14, false).rgb;
float3 colB = SceneTextureLookup(uvB, 14, false).rgb;

// Step 9: Assemble the final colour from one channel of each sample.
float3 finalColor;
finalColor.r = colR.r;
finalColor.g = colG.g;
finalColor.b = colB.b;

// Step 10: Return the combined chromatically aberrated colour.
return finalColor;
```

## 6. Connect Output to Emissive Color

Connect the **ChromaticAberration** Custom node output (Float3) to **Emissive Color** of the material.

This ensures the effect is applied as a post-process replacement of the final scene colour.

## 7. Assign the Material to the Post Process Volume

1. Return to your level and select the **Post Process Volume**.
2. Under **Post Process Materials**, assign:

   ```text
   PPM_ChromaticAberration
   ```

3. Set **Blend Weight = 1.0**.

Press **Play** to see subtle RGB separation near the edges of the screen.

**Checkpoint:** Red and blue fringes appear along high-contrast edges towards the sides of the view, and the centre of the view is perfectly sharp. Raise `CenterDeadZone` to 0.2 and the sharp area grows.

## 8. Student Tasks

### Task 1 - Tuning the Effect

Create a **Material Instance** of `PPM_ChromaticAberration` and experiment with:

- `AberrationStrength`: try values between `0.001` and `0.01`
- `EdgeBoost`: try values between `0.0` and `2.0`
- `CenterDeadZone`: try values between `0.0` and `0.2`

Questions:

- When does the effect feel subtle and cinematic?
- When does it become distracting or uncomfortable?

### Task 2 - Impact Moment Effect (Design Exercise)

Imagine a situation where chromatic aberration increases briefly when:

- the player takes damage
- a powerful weapon fires
- a time-warp or slow-motion event occurs

Sketch (or outline in pseudocode / Blueprint description) how you would:

- animate `AberrationStrength` over time for a short burst
- possibly combine it with **vignette** or **camera shake**

### Task 3 - Directional Aberration (Optional Extension)

Modify the HLSL to support a **directional** aberration:

- Instead of using the radial direction `dir` based on `centered`,
  use a constant direction (e.g. `float2(1, 0)` for horizontal only).
- Expose that direction as a parameter from the material graph (a Vector2) if desired.

Consider:

- How does a purely horizontal aberration feel compared to a radial one?
- Which would be more suitable for a "camera glitch" vs a "lens edge" effect?

## 9. Reflective Questions

1. Why do we compute a **direction from the screen centre** for chromatic aberration?
2. How does the `CenterDeadZone` parameter affect the usability of the effect in UI-heavy games?
3. Why might we choose to leave the **green channel** at the original UV while shifting red and blue?
4. What visual differences do you observe when `EdgeBoost` is set to `0.0` vs a high value like `2.0`?
5. In what gameplay situations might chromatic aberration enhance immersion, and in which might it be a distraction?
