# Lab 13 - Toon Shading with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

This is the most open-ended lab in the series, so complete the others first. In this lab you will build a **toon-shading material** in Unreal Engine using **Custom HLSL nodes**. This lab introduces three major components of a stylised shader:

1. **Edge/Crevice Masking** using derivatives (`ddx`, `ddy`).
2. **Threshold-Based Lighting** (Shadow / Midtone / Highlight).
3. **Optional Halftone Shading** for comic-style rendering.

By completing this lab, you will build a fully controllable toon shader with adjustable tone thresholds, colour ramps, softness, edge thickness, edge contrast, and optional halftone overlay.

## Learning Outcomes

You will learn to:

- Use `ddx` / `ddy` to extract curvature and screen-space edge information.
- Create controllable edge masks via Custom HLSL nodes.
- Implement toon shading using thresholding and smooth transitions.
- Blend shadow/mid/highlight colours using Softness control.
- Combine edge masks and lighting functions into a single stylised output.
- Optionally incorporate halftone-dot shading for comic rendering.

## Creating the Base Material

### Create a Material
1. Right-click -> **Material**
2. Name it:
   ```
   MM_Prop_ToonShading_Opaque
   ```
3. Set **Shading Model -> Unlit**. Toon shading replaces the engine's lighting with your own bands, so the result goes into **Emissive Color**. A Default Lit material would light your bands a second time and smooth them out again.
4. Ensure **Use Material Attributes** is *off* for simplicity.

## Creating the Edge/Curvature Mask

This section demonstrates calculating edge details by measuring change in the surface normals using derivative instructions.

### Add a Custom Node for Edge Mask
1. Add a **Custom Node** called:
   ```
   EdgeMask
   ```
2. Add inputs:
   - `NormalsWS` (Vector3) -> connect from **PixelNormalWS**
   - `Thickness` (Scalar)
   - `Contrast` (Scalar)

3. Paste this HLSL code. **Output Type:** `CMOT Float 1`.

```hlsl
float edgeDetails = length(ddx(NormalsWS)) +
                    length(ddy(NormalsWS));

edgeDetails /= (Thickness + 1.0f);
edgeDetails  = (edgeDetails - 0.5f) * Contrast + 0.5f;

return saturate(edgeDetails);
```

### Parameter Setup
- Convert `Thickness` and `Contrast` to **Parameters**.
- Add each of the parameters to a group:
  - Group: **Edges**
  - Parameters: Thickness, Contrast

### Preview
Connect `EdgeMask` output to **Emissive Color** temporarily to verify edges are appearing.

## Creating the Toon Lighting Function

We now build a thresholded lighting model to classify each pixel into:
- Shadow
- Midtone
- Highlight

### Add a Toon Lighting Custom Node
1. Add a Custom Node named:
   ```
   ToonLight
   ```
2. Inputs:
   - `NormalsWS` (Vector3) - connect **PixelNormalWS**
   - `LightDir` (Vector3) - connect a **SkyAtmosphereLightDirection** node, which gives the direction towards the level's main directional light. If your level has no Sky Atmosphere, use a `VectorParameter` such as (0.5, 0.3, 0.8) instead.
   - `Shadow` (Scalar)
   - `Mid` (Scalar)
   - `High` (Scalar)
   - `In1` (Vector3) - Shadow colour
   - `In2` (Vector3) - Midtone colour
   - `In3` (Vector3) - Highlight colour
   - `Softness` (Scalar)

3. Paste this HLSL. **Output Type:** `CMOT Float 3`.

```hlsl
float lightIntensity = max(dot(normalize(NormalsWS), normalize(LightDir)), 0.0);

float3 color;

// Guard the band widths so equal thresholds cannot divide by zero.
float highBand = max(High - Mid, 1e-4);
float midBand = max(Mid - Shadow, 1e-4);

if (lightIntensity > High)
    color = In3;
else if (lightIntensity > Mid)
    color = lerp(In2, In3, smoothstep(0, Softness, (lightIntensity - Mid) / highBand));
else if (lightIntensity > Shadow)
    color = lerp(In1, In2, smoothstep(0, Softness, (lightIntensity - Shadow) / midBand));
else
    color = In1;

return color;
```

### Grouping Parameters
Use two parameter groups:
- **Gradient** - In1, In2, In3
- **Size and Falloff** - Shadow, Mid, High, Softness

### Preview
Connect the ToonLight output directly to **Emissive Color** to preview.

## Combining Edge Mask with Lighting

We will next combine the lighting result with the inverted edge mask.

### Node Logic
1. **1 - EdgeMask**
2. Invert edge mask:
   ```
   1 - EdgeMask
   ```
3. Multiply by toon lighting output.
4. Optionally saturate.

### Build Combined Output
```
FinalColor = saturate( ToonLight * (1 - EdgeMask) );
```

Set **Emissive Color** = FinalColor.

## Optional: Halftone Pattern Overlay

### Add Halftone Custom Node
Inputs:
- `color` (Vector3)
- `uv` (Vector2)
- `tiling` (Scalar)
- `maxSize` (Scalar)

### Paste halftone code

**Output Type:** `CMOT Float 1`.

```hlsl
float luminance = dot(color, float3(0.299, 0.587, 0.114));

float dist = length(frac(uv * tiling) - 0.5);
float radius = (1.0 - luminance) * maxSize;

return lerp(1, 0.5, step(dist, radius));
```

- Multiply this output by the toon-lighting colour.
- Use screen-space UVs or texture UVs depending on intended look.

## Final Material Output

Set:
- **Emissive Color** = combined toon lighting *(with optional halftone overlay)*

An Unlit material has no Roughness or Specular inputs, because the engine does not light it.

You now have a fully functioning, stylised toon shader.

**Checkpoint:** A sphere with this material shows three flat bands of colour, shadow, midtone and highlight, arranged towards and away from the sun, with dark lines where its surface curves most sharply. Rotate the directional light and the bands move with it.

## Student Tasks

### Adjust Tone Thresholds
Explore how Shadow, Mid, High alter banding.

### Experiment with Halftone Patterns
Try screen-space vs UV-space mapping for dots.

## Reflective Questions

1. Why does toon shading benefit from thresholded lighting instead of continuous shading?
2. What advantages do `ddx` and `ddy` offer for stylised rendering?
3. How does adjusting Softness affect the shading style?
4. What visual differences occur when blending lighting and edge masks?
5. How can halftone shading enhance stylised material aesthetics?

## Useful Links

- [Unreal Engine Advanced Cel Shader](https://dev.epicgames.com/community/learning/tutorials/qZl2/unreal-engine-advanced-cel-shader)

- [Unreal Engine Material Custom Material Expressions (Official Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/custom-material-expressions-in-unreal-engine)

- [Microsoft HLSL Reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-pguide)

- [HLSL Tutorial Series (Ben Cloward)](https://www.youtube.com/playlist?list=PL78XDi0TS4lEMvytsE_MoWEpzBcukXv9b)

## Appendix A - Background: Technical Notes on Toon Shading Filters

### 1. Derivative-Based Edge Detection (`ddx`, `ddy`)
- Measures change in normals across screen space.
- High variation gives a strong edge.
- Very cheap vs full Sobel filter.
- Ideal for stylised outlines that match geometry curvature.

### 2. Thresholded Lighting Models
- Creates clear "bands" of tone.
- Can emulate anime, comic, and cel shading styles.
- Softness blends edges for smoother gradients.

### 3. Halftone Dot Functions
- Common in printed comics.
- Dot radius scales with luminance.
- Provides retro, stylised look.
- Works in screen UV or texture UV space.
