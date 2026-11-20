# Lab 9 - Post-process Materials - Sobel Edge Detection with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will implement a **Sobel edge-detection filter** inside a **Custom HLSL node** for use in a **Post-Process Material** in Unreal Engine. This technique allows you to highlight the edges of objects in your scene and is useful for toon shading, stylised rendering, technical visualisation, and debugging. It is the first lab in the series with a loop: the Custom node reads nine neighbouring pixels in a `for` loop.

## Learning Outcomes

By completing this lab, you will be able to:

- Create a Post-Process Material in Unreal Engine.
- Use SceneTexture nodes to access final rendered pixels.
- Write and structure a Custom HLSL node.
- Implement Sobel convolution using 3x3 neighbourhood sampling.
- Control intensity and behaviour of the resulting edge mask.
- Integrate the edge mask into larger rendering pipelines.

## Creating the Post-Process Material

### Create a Post Process Volume
1. Drag a **Post Process Volume** into your level.
2. Enable:
   - **Infinite Extent (Unbound)** -> *true*

This ensures the post process applies to the entire scene.

### Add a Post Process Material Entry
1. In **Post Process Materials**, add an **Array Element**.
2. Set its type to **Asset Reference**.
3. You will assign the Sobel material here later.

## Creating the Sobel Filter Material

### Create the Material
1. Right-click -> **Material**
2. Name it:
   ```
   PPM_SobelEdges
   ```
3. Open the material.
4. Set:
   - **Material Domain -> Post Process**

Only the **Emissive Color** pin will be active.

### Access the Scene Texture
1. Add a **SceneTexture** node.
2. Set **Scene Texture Id -> PostProcessInput0**

This gives you access to the final rendered scene images your filter will operate on. You will wire this node into an input called `sceneTexture` that the code never reads. That wire matters: `SceneTextureLookup(uv, 14, ...)` only works when a SceneTexture node for that texture is in the graph, because the node is what tells Unreal to make the texture available to the shader. Do not delete it. Its **InvSize** output, the size of one pixel in UV units, is used below to step to the neighbouring pixels.

## Creating a Helper Custom Node for Screen UVs

### Add a Custom Node for UVs
1. Add a **Custom** node.
2. Set Output Type to **CMOT Float 2**
3. Enter:
   ```hlsl
   return GetSceneTextureUV(Parameters);
   ```

This returns the buffer UV that `SceneTextureLookup` expects, as in Lab 2.

## Main Sobel Custom Node

### Add the SobelFilter Custom Node
1. Add a **Custom** node named:
   ```
   SobelFilter
   ```
2. Set **Output Type = CMOT Float 1**
3. Add inputs:
   - `sceneTexture` -> Float3 (connect SceneTexture node)
   - `UV` -> Float2 (connect UV helper)
   - `InvSize` -> Float2 (connect the SceneTexture node's **InvSize** output)
   - `Intensity` -> Float1 (scalar parameter controlling strength)

### Sobel Operator Overview

Sobel filters use two 3x3 kernels:

#### Horizontal Sobel
```
-1  0  1
-2  0  2
-1  0  1
```

#### Vertical Sobel
```
-1 -2 -1
 0  0  0
 1  2  1
```

These matrices are combined in the tutorial into a single float2 kernel representing (horizontal, vertical) values.

### Full HLSL Code
#### Paste this code into the Custom Node:

```hlsl
// Sobel edge detection over PostProcessInput0
// Inputs:
//   UV        : float2 buffer UV from GetSceneTextureUV(Parameters)
//   InvSize   : float2 size of one pixel in UV units, from the SceneTexture node's InvSize
//   Intensity : scalar multiplier for edge strength

// 3x3 Sobel kernel encoded as float2 (Gx, Gy) pairs
float2 sobelMatrix[9] =
{
    float2(-1, -1), float2(0, -2), float2(1, -1),
    float2(-2,  0), float2(0,  0), float2(2,  0),
    float2(-1,  1), float2(0,  2), float2(1,  1)
};

// Accumulated gradient in X/Y
float2 gradients = float2(0.0, 0.0);

[unroll]
for (int i = 0; i < 9; ++i)
{
    int row = i / 3;
    int col = i % 3;

    // One-pixel offsets in UV space
    float2 offsetUV = UV + float2(col - 1, row - 1) * InvSize;

    // 14 = PostProcessInput0 (scene colour) in UE post-process materials.
    // Convert the colour to luminance so edges in every colour are found, not only in red.
    float3 sampleColor = SceneTextureLookup(offsetUV, 14, false).rgb;
    float sampleIntensity = dot(sampleColor, float3(0.299, 0.587, 0.114));

    gradients += sobelMatrix[i] * sampleIntensity;
}

// Scale by the intensity parameter
gradients *= Intensity;

// Edge magnitude (greyscale)
return length(gradients);
```

## Testing the Edge Filter

### Connect the Output
- Connect the Custom Node output directly to **Emissive Color**.
- Optionally multiply the output before Emissive to adjust brightness.

### Try the Material
1. Assign **PPM_SobelEdges** to your Post Process Volume.
2. Set **Blendable Location** if needed in the materials master node in the Details pane.

You should now see outline edges around all geometry in the scene.

**Checkpoint:** The view turns black with white lines along every edge where brightness changes: silhouettes, but also texture detail and shadow edges. Raise `Intensity` and fainter edges appear.

## Student Tasks

### Adjust Edge Sensitivity
Modify:
- `Intensity` input
Record how outlines change visually.

### Composite with Original Image
Use:
- **Add** (glowing edges)
- **Multiply** (darkened edges)
- **Lerp** (controlled blend)

### Experiment with Edge Thickness
Multiply the `InvSize` offset by 2 to sample a wider neighbourhood and draw thicker lines.

## Reflective Questions

1. Why does Sobel require sampling neighbouring pixels rather than the pixel alone?
2. How does screen resolution influence edge results?
3. Why do we return `length(gradients)` instead of separate X/Y channels?
4. What is the purpose of using `SceneTextureLookup` with index 14?
5. How can Sobel filtering be used stylistically in games?

## Useful Links

- [Unreal Engine Material Custom Material Expressions (Official Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/custom-material-expressions-in-unreal-engine)

- [Microsoft HLSL Reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-pguide)

- [HLSL Tutorial Series (Ben Cloward)](https://www.youtube.com/playlist?list=PL78XDi0TS4lEMvytsE_MoWEpzBcukXv9b)

## Appendix A - Background: Top 3 Edge Detection Algorithms

### Sobel Operator
Used in this lab.
- Measures gradient in X and Y direction.
- Smooths noise slightly due to kernel weighting.
- Good for toon outlines and stylised rendering.
- Balanced performance and quality.

### Prewitt Operator
Simpler version of Sobel.
- Uniform kernel weights (no 2s in the centre rows).
- Slightly more sensitive to noise.
- Faster but less precise.
- Ideal for lightweight devices or cheaper post-processing.

### Laplacian of Gaussian (LoG)
A second-derivative operator.
- Computes edges by detecting zero-crossings.
- Isotropic: responds equally to all edge directions.
- Produces clean, closed outlines.
- More expensive due to Gaussian blur + Laplacian pass.
- Excellent for anime/toon-style thick outlines.
