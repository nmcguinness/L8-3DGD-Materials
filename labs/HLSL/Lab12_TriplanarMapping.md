# Lab 12 - Uniplanar, Biplanar and Triplanar Mapping with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will implement **uniplanar, biplanar and triplanar texture mapping** using a single **Custom HLSL node** in a **surface material**.

Traditional UV mapping often stretches textures badly on steep slopes or on meshes with poor UV layouts (for example, cliffs, rocks, and modular environment pieces). Triplanar mapping avoids this by **projecting the texture from multiple directions** (X, Y, Z axes) and blending them based on the surface normal.

You will:

- project a texture in world space (uniplanar),
- blend between two projections (biplanar),
- generalise this to three projections weighted by the world normal (triplanar).

The same Custom node will contain all three methods, controlled by a parameter.

## Learning Outcomes

By completing this lab, you will be able to:

- Explain why triplanar mapping is used in games to reduce texture stretching.
- Use world-space position and normal in a Custom HLSL node.
- Implement uniplanar, biplanar and triplanar projections in one HLSL function.
- Blend projected textures based on the absolute world normal.
- Expose parameters for texture scale, blend sharpness and mapping mode.

## 1. Theory - Why Triplanar Mapping?

### UV stretching on steep slopes

Standard UVs are fixed to the mesh. If a surface is very steep or vertical, and the UVs were laid out flat, then:

- textures get **squashed** when seen from the side,
- or **stretched** when the surface is almost vertical.

This is very noticeable on:

- terrain with cliffs,
- rocks and boulders,
- modular environment pieces with reused textures.

### Projecting textures instead of relying on UVs

Instead of trusting the mesh UVs, we can use **world-space coordinates** as our "UVs".

- For a **top-down projection**, we might use world X and Y as our UV axes.
- For a **side projection**, we might use X/Z or Y/Z.

This is called **planar projection**.

### From uniplanar to biplanar to triplanar

- **Uniplanar mapping**
  Project texture from a single direction (for example, top-down using world X/Y).

- **Biplanar mapping**
  Blend between two planar projections (for example, top-down and one side axis).

- **Triplanar mapping**
  Blend between **three** planar projections (X, Y, Z) based on how much the surface normal points in each direction.

Key idea:

- Take the **absolute value of the normal** (so front/back faces behave the same).
- Use the **X, Y, Z components** as blend weights for the three projections.
- Normalise the weights so they sum to 1.

This produces a seamless blend across edges, with minimal stretching.

## 2. Material Setup

Create a new material:

```text
MM_Env_TriplanarMapping_Opaque
```

Set:

- **Material Domain -> Surface**
- **Shading Model -> Default Lit** (so we can still use lighting)

You will output the triplanar colour into **Base Color**.

### Parameters

Create these parameters:

| Parameter Name | Type | Default | Description |
| :- | :- | :- | :- |
| `BaseTexture` | Texture Object Parameter | `T_Stone_Albedo` from `/Game/GD/Shared/Textures/Stone`, or any tiling rock texture | Texture to project |
| `TextureScale` | Scalar | 0.01 | World-space tiling scale: 0.01 repeats the texture once per metre |
| `BlendSharpness` | Scalar | 2.0 | Controls how sharp the triplanar blend is |
| `MappingMode` | Scalar | 0.0 | 0 = uniplanar, 1 = biplanar, 2 = triplanar |

Notes:

- `TextureScale` will be multiplied with world position.
- `MappingMode` will be used inside the HLSL to switch behaviour.

### World-space inputs

Add:

- **AbsoluteWorldPosition** node -> gives world position `float3`.
- **PixelNormalWS** node -> gives world-space normal `float3`.

You will connect:

- AbsoluteWorldPosition -> Custom node `worldPos`
- PixelNormalWS -> Custom node `worldNormal`
- BaseTexture -> Custom node `baseTex`

## 3. Custom Node Setup

Add a **Custom** node named `PlanarMapping`.

Set:

- **Output Type -> CMOT Float 3** (we output a colour)

Add these **Inputs**:

| Input Name | Type | Connect From |
| :- | :- | :- |
| `worldPos` | Float3 | AbsoluteWorldPosition |
| `worldNormal` | Float3 | PixelNormalWS |
| `baseTex` | Texture Object | `BaseTexture` parameter |
| `textureScale` | Float1 | `TextureScale` |
| `blendSharpness` | Float1 | `BlendSharpness` |
| `mappingMode` | Float1 | `MappingMode` |

Custom node inputs have no type setting: each input takes its type from what is wired into it. Wiring a Texture Object Parameter into `baseTex` makes it a texture, and Unreal also creates its sampler for you, named `baseTexSampler`. Do not add an input with that name, because it would clash with the one Unreal creates.

## 4. HLSL Code - Uniplanar, Biplanar, Triplanar

Paste the following fully commented HLSL into the `PlanarMapping` Custom node. The code has no helper function and no `main()`: a Custom node's code is already the body of a function, so it runs top to bottom and ends with `return`.

```hlsl
// Inputs:
//   worldPos      : float3 world-space position of the pixel
//   worldNormal   : float3 world-space normal of the pixel
//   baseTex       : Texture Object for the base texture
//                   (Unreal also gives you its sampler, named baseTexSampler)
//   textureScale  : scalar tiling factor applied to world-space coordinates
//   blendSharpness: controls how sharp triplanar blending is (>=1)
//   mappingMode   : 0=uniplanar, 1=biplanar, 2=triplanar

// Step 1: Prepare basic data.

// Normalise the world normal so its length is 1.
float3 n = normalize(worldNormal);

// Take the absolute value so back-facing directions behave like front-facing.
// This avoids hard seams when the normal flips sign.
float3 an = abs(n);

// Scale world position by the textureScale parameter.
// A textureScale of 0.01 repeats the texture once per metre (100 cm).
float3 p = worldPos * textureScale;

// Step 2: Build planar UVs for each axis.
// Unreal is Z-up, so:
// - X/Y plane: "top" projection
// - Y/Z plane: side projection in X direction
// - X/Z plane: side projection in Y direction
float2 uvZ = float2(p.x, p.y); // projecting along +Z/-Z (top-down)
float2 uvX = float2(p.y, p.z); // projecting along +X/-X
float2 uvY = float2(p.x, p.z); // projecting along +Y/-Y

// Step 3: Sample the texture for each projection.
// Texture2DSample is Unreal's macro for sampling a texture with its sampler.
float3 texZ = Texture2DSample(baseTex, baseTexSampler, uvZ).rgb; // top-down projection
float3 texX = Texture2DSample(baseTex, baseTexSampler, uvX).rgb; // X-facing projection
float3 texY = Texture2DSample(baseTex, baseTexSampler, uvY).rgb; // Y-facing projection

// -----------------------------
// Uniplanar mapping (mode 0)
// -----------------------------
// Use a single planar projection (top-down along Z).
float3 colorUniplanar = texZ;

// -----------------------------
// Biplanar mapping (mode 1)
// -----------------------------
// Blend between top-down (Z) and a single side projection (X).
// We use the normal's Z and X components to decide weights.
float wZ_bi = an.z;
float wX_bi = an.x;

// Normalise the two weights so they sum to 1 (avoid divide by zero).
float sum_bi = max(wZ_bi + wX_bi, 1e-4);
wZ_bi /= sum_bi;
wX_bi /= sum_bi;

// Blend the two projections using the biplanar weights.
float3 colorBiplanar = texZ * wZ_bi + texX * wX_bi;

// -----------------------------
// Triplanar mapping (mode 2)
// -----------------------------
// Blend between all three projections (X, Y, Z) using their
// absolute normal components as weights.
float3 weights = an;

// Optionally sharpen the blend by raising weights to a power.
// blendSharpness = 1.0 -> soft blend
// blendSharpness > 1.0 -> sharper, more axis-aligned
weights = pow(weights, blendSharpness);

// Normalise weights so x + y + z = 1.
float weightSum = max(weights.x + weights.y + weights.z, 1e-4);
weights /= weightSum;

// Blend the three projections using the final weights.
float3 colorTriplanar =
    texX * weights.x +
    texY * weights.y +
    texZ * weights.z;

// -----------------------------
// Select output based on mappingMode
// -----------------------------
// mappingMode:
//   0 -> uniplanar
//   1 -> biplanar
//   2 -> triplanar (default for production use)
float3 finalColor = colorTriplanar;

if (mappingMode < 0.5)
{
    finalColor = colorUniplanar;
}
else if (mappingMode < 1.5)
{
    finalColor = colorBiplanar;
}

// Return the chosen mapping colour.
return finalColor;
```

## 5. Connect the Custom Node to the Material

Connect:

- `PlanarMapping` Custom node output -> **Base Color**

Leave Metallic, Roughness, Normal, etc., at suitable defaults or plug constants as needed.

Apply the material to:

- a large cube,
- a sphere,
- a landscape or irregular mesh.

Observe the difference between **MappingMode** 0, 1 and 2.

**Checkpoint:** With `MappingMode` 0 the top of the cube is clean but its sides show long streaks. With 2 every face shows the texture at the same size, and the sphere has no stretched band around its middle.

## 6. Material Instances and Experiments

Create a Material Instance:

```text
MI_Env_TriplanarRock_Opaque
```

Test:

- `MappingMode = 0` (uniplanar)
  - Notice strong stretching on vertical sides.

- `MappingMode = 1` (biplanar)
  - Some blending between top and one side; reduces stretching on certain angles. Faces that point along Y still stretch, because this version always uses the X side projection. Production biplanar mapping picks the two strongest axes for each pixel instead.

- `MappingMode = 2` (triplanar)
  - Texture appears much more uniform across all angles.

Adjust:

- `TextureScale` to control tiling in world units.
- `BlendSharpness` from 1.0 to 8.0 and see how edges become sharper or softer.

## 7. Student Tasks

### Task 1 - Compare UV vs Triplanar

Apply:

- A standard UV-mapped material using the same texture.
- `MI_Env_TriplanarRock_Opaque` with `MappingMode = 2`.

Compare:

- cliffs or steep surfaces,
- stretched vs consistent texel density.

Make notes on when triplanar is visually superior.

### Task 2 - Add a Second Texture for Vertical Surfaces

Extend the HLSL to support:

- one texture for "top" surfaces (using Z projection),
- a second texture for vertical surfaces (X and Y projections).

Use the weights in the triplanar block to blend rock vs grass or snow vs rock.

### Task 3 - Triplanar Normal Mapping (Design Exercise)

Outline how you would:

- feed a Normal Map texture into the same Custom node,
- sample normal maps along X, Y, Z,
- reconstruct and blend them using the same weights,
- transform them into world-space or tangent-space as needed.

You do not need to fully implement this in code, but describe the steps clearly.

## 8. Reflective Questions

1. Why does triplanar mapping significantly reduce texture stretching compared to simple UV mapping on steep slopes?
2. What role do the absolute normal components play in deciding how much each projection contributes?
3. How does increasing `BlendSharpness` change the look of the triplanar blend? When might softer vs sharper blends be preferable?
4. In what situations would uniplanar or biplanar mapping still be sufficient, instead of full triplanar?
5. How could triplanar mapping be combined with procedural noise (from Lab 11) to create more varied and organic surface materials?
