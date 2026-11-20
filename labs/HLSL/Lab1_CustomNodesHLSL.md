# Lab 1 - Introduction to Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will create a reusable Unreal **Material Function** that transforms UVs from standard Cartesian space (U, V) into **polar coordinates** (angle, radius).  You will learn how to use a **Custom Node** to write HLSL inside the Material Editor and apply this to drive radial, circular, and stylised effects.

This lab is suitable for students in Year 4 who are comfortable with basic material nodes and are ready to start authoring simple shaders via HLSL.

## Learning Outcomes

By completing this lab, you will be able to:

- Add and configure a **Custom HLSL Node** inside a Material Function.
- Convert UV coordinates into centred space.
- Compute **radius** and **angle** using HLSL's `length()` and `atan2()` functions.
- Normalise and pack polar coordinates as a `float2`.
- Apply custom UV transformations to textures to create radial or rotating effects.

## Creating the Polar UV Material Function

### Create a Material Function
1. In the Content Browser: **Right-click -> Material -> Advanced -> Material Function**
2. Name it:
   ```
   MF_PolarUV
   ```
3. Open it.

### Add a Function Input (UV)
1. Right-click -> **Function Input**
2. Name: `UV`
3. Type: **Function Input Vector 2**

### Recentre the UV Coordinates
Create a Constant2Vector:
- Value: `(0.5, 0.5)`

Add a **Subtract** node:
- A = UV
- B = (0.5, 0.5)
- Result description (optional): `CenteredUV`

This shifts (0.5, 0.5) -> (0, 0), placing the texture origin in the centre.

### Add a Custom Node for Polar Conversion

1. Add a **Custom** node
2. Set description: `PolarConvert`
3. Add input:
   - Name: `P`
   - Connect `CenteredUV` -> P
4. Set Output Type = **CMOT Float 2**

#### Paste this HLSL code:
```hlsl
float angle = atan2(P.y, P.x);      // -PI to PI
float radius = length(P);           // 0 to ~0.7 inside UV square
float angle01 = angle / (2 * PI) + 0.5; // Normalise angle to 0..1
return float2(angle01, radius);
```

`PI` is already defined by the engine's shader code, so any Custom node can use it.

This computes the polar coordinates:
- X = angle in 0..1
- Y = radius in 0..0.7

### Add a Function Output
1. Right-click -> **Function Output**
2. Name: `PolarUV`
3. Connect the Custom node -> Output.

Your Material Function is complete.

## Testing the Function in a Material

### Create a Test Material
1. Right-click -> **Material**
2. Name it:
   ```
   MM_Prop_PolarUV_Opaque
   ```

### Add a Texture Sample
Use any checker/grid or repeating texture that you have available in your content browser.

### Add the Material Function
1. Right-click -> **Material Function Call**
2. Select `MF_PolarUV`
3. Add a **TextureCoordinate** node
4. Connect:
   - TextureCoordinate -> `UV` input of MF_PolarUV
   - MF_PolarUV output -> Texture Sample **UVs**

You should now see a radial distortion of your texture.

**Checkpoint:** The texture wraps around the centre of the surface: its columns become rays from the centre and its rows become rings. A thin line runs from the centre to one edge, where the angle wraps from 1 back to 0. That line is a mip-level seam, the same one Notes 02 describes for lat-long reflections, and Notes 10 5.2 shows how to remove it with `Texture2DSampleGrad`.

## Student Tasks

### Rotating Radial Texture
1. Add a **Time** node
2. Add `Time * Speed` to the angle component (`PolarUV.R`)
3. Repack the modified angle + original radius into a float2
4. Use that as the new UVs

This creates a spinning effect.

**Checkpoint:** The texture turns around the centre of the surface, faster as `Speed` increases.

## Reflective Questions

1. Why do we centre UVs using (0.5, 0.5)?
2. Why must angle be normalised before using it as a UV?
3. Give two game effects that use polar UVs.
4. What is one advantage of Custom nodes vs pure material nodes?

## Useful Links

- [Unreal Engine Material Custom Material Expressions (Official Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/custom-material-expressions-in-unreal-engine)

- [Microsoft HLSL Reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-pguide)

- [HLSL Tutorial Series (Ben Cloward)](https://www.youtube.com/playlist?list=PL78XDi0TS4lEMvytsE_MoWEpzBcukXv9b)

## Appendix A - Useful HLSL Functions

### `length(x)`
Returns the magnitude of a vector.

### `normalize(x)`
Returns a unit vector.

### `dot(a, b)`
Dot product for angle and lighting calculations.

### `lerp(a, b, t)`
Interpolates between two values.

### `saturate(x)`
Clamps to [0, 1].

### `frac(x)`
Returns fractional part.

### `fmod(x, y)`
Floating modulo.

### `atan2(y, x)`
Angle between vectors (-PI to PI).

### `sin(x)` / `cos(x)`
Trigonometric helpers.

### `pow(a, b)`
Useful for curve shaping.
