# Lab 6 - Lens Distortion (Brown-Conrady Model) with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will implement a **lens distortion post-process** effect using the **Brown-Conrady radial distortion model** in a **Custom HLSL node** in Unreal Engine.

Real camera lenses bend light in a non-linear way. This creates characteristic **barrel / fisheye** or **pincushion** distortion, especially near the image edges. We will approximate this behaviour using the Brown-Conrady polynomial model and apply it as a full-screen post process.

You will:

- Use a polynomial distortion formula to warp screen-space UVs.
- Implement the Brown-Conrady model with coefficients *k1*, *k2*, *k3*.
- Understand how these coefficients shape barrel vs pincushion distortion.
- Apply intensity and zoom controls to tune the effect.

## Background - Brown-Conrady Radial Distortion Model

Many camera calibration pipelines use a polynomial model to approximate radial lens distortion. A common 3-coefficient version is:

```text
D(r) = 1 + r^2 ( k1 + r^2 ( k2 + r^2 k3 ) )
```

Where:

- `r` is the **radial distance** from the distortion centre (usually the optical centre of the lens, in normalised screen space).
- `k1`, `k2`, `k3` are **distortion coefficients**:
  - `k1` controls the main barrel (positive) or pincushion (negative) curvature.
  - `k2` shapes distortion more strongly near the mid-outer regions of the image.
  - `k3` mostly affects the extreme edges and corners.

To distort a point, we:

1. Convert the UV into a centred coordinate (so the middle of the screen is `(0, 0)`).
2. Compute `r^2` as the squared length of that coordinate.
3. Evaluate the polynomial `D(r)`.
4. Multiply the centred coordinate by `D(r)` to push it **outwards** (barrel / fisheye) or **inwards** (pincushion).
5. Convert back to standard UV space in `[0, 1]` and sample the scene there.

## Learning Outcomes

By completing this lab, you will be able to:

- Implement the Brown-Conrady radial distortion model in HLSL.
- Use a Custom node to warp screen UVs and re-sample `PostProcessInput0`.
- Control lens distortion using intensity and zoom parameters.
- Explain the role of the `k1`, `k2`, `k3` coefficients in shaping the distortion.

## 1. Create a Post Process Volume

1. Drag a **Post Process Volume** into your level.
2. In the Details panel:
   - Enable **Infinite Extent (Unbound)** -> *true*.
3. Under **Post Process Materials**, add an **Array Element** and set its type to **Asset Reference**.
4. You will assign the lens distortion material here later.

## 2. Create the Lens Distortion Material

1. In the Content Browser create a new **Material**:

   ```text
   PPM_LensDistortion
   ```

2. Open the material and set:
   - **Material Domain -> Post Process**

We will output our final colour to **Emissive Color**.

## 3. Provide Screen UVs to the Custom Node

The Brown-Conrady model operates on normalised screen coordinates. We will pass these into the Custom node.

1. Add a **ScreenPosition** node.
2. Use its **ViewportUV** output. The node has two outputs, **ViewportUV** and **PixelPosition**, and no mode to set.
3. ViewportUV is a `float2` in `[0, 1]` for the current pixel, which is the UV the SceneTexture node expects.
4. You will connect this UV into the Custom node input called `uv`.

## 4. Add the Custom Node - Brown-Conrady Lens Distortion

The code below is commented line by line. The Custom node will:

- Take `uv` as input.
- Apply the Brown-Conrady distortion.
- Output **distorted UVs** as a `float2`.

### 4.1 Custom Node Setup

1. Add a **Custom** node:
   - **Name:** `LensDistort`
   - **Output Type:** `CMOT Float 2`

2. In the node's **Inputs** list, add:

| Input Name | Type | Connect From |
| :- | :- | :- |
| `uv` | Float2 | `ScreenPosition (ViewportUV)` output |

(For this version, intensity, zoom, and coefficients are stored as constants inside the HLSL. Students will later promote them to parameters.)

### 4.2 Commented HLSL Code

Paste the following into the Custom node's **Code** field:

```hlsl
// === Brown-Conrady Lens Distortion Custom Node ===
// Input:
//   uv : float2 viewport UV in [0, 1]

// distortion intensity
// Controls how much of the distortion model we apply.
// 0.0 = no distortion, 1.0 = full distortion.
float intensity = 1.0;

// zoom control
// Values > 1.0 zoom in (magnify the image),
// values < 1.0 zoom out.
float zoom = 1.0;

// define k1, k2, k3 distortion coefficients.
// These shape the barrel / pincushion distortion curve.
// Positive values give barrel distortion, negative values give pincushion.
float k1 = 0.2;
float k2 = 0.05;
float k3 = 0.0;

// define cx, cy optical centre as an offset.
// This allows us to shift the distortion centre if needed.
// (0,0) means centred exactly on the middle of the image.
float2 centerOffset = float2(0, 0);

// normalise uvs so that (0.5,0.5) becomes (0,0).
// - First subtract 0.5 to centre the coordinates.
// - Multiply by 2 to scale from [-0.5,0.5] into [-1,1].
// - Subtract centerOffset to apply any optical centre shift.
// - Divide by zoom to zoom the image in or out.
float2 centerUV = ((uv - 0.5) * 2.0 - centerOffset) / zoom;

// squared radial distance from the distortion centre.
// rSq = x^2 + y^2; using dot product for convenience.
float rSq = dot(centerUV, centerUV);

// polynomial distortion model (Brown-Conrady):
// model = 1 + r^2 (k1 + r^2 (k2 + r^2 * k3));
// Here rSq = r^2, so we nest rSq multiple times.
float model = 1.0 + rSq * (k1 + rSq * (k2 + rSq * k3));

// blend between no distortion (1.0) and the full model using intensity.
float distort = 1.0 + (model - 1.0) * intensity;

// fit to screen: evaluate the same model at a corner (rSq = 2)
// and divide by it, so the corners of the screen sample the corners of the image
// instead of reading far outside it.
float modelCorner = 1.0 + 2.0 * (k1 + 2.0 * (k2 + 2.0 * k3));
float fit = 1.0 + (modelCorner - 1.0) * intensity;

// apply distortion to our centred UVs,
// then convert back from [-1,1] to [0,1] by *0.5 and +0.5.
float2 distortedUV = centerUV * (distort / fit) * 0.5 + 0.5;

// output result (distorted UVs in [0,1]).
return distortedUV;
```

## 5. Sample the Scene with Distorted UVs

Now that the Custom node outputs distorted UVs, we will use them to sample the final rendered scene.

1. Add a **SceneTexture** node.
2. Set **SceneTexture Id -> PostProcessInput0** (this is the final scene colour).
3. Connect the `LensDistort` Custom node output to the **UVs** input of the SceneTexture node.
4. Connect the **Color** output of the SceneTexture node to the material's **Emissive Color**.

Your material now renders the scene through a distorted lens.

## 6. Assign the Material to the Post Process Volume

1. Return to your level and select the **Post Process Volume**.
2. Under **Post Process Materials**, assign:

   ```text
   PPM_LensDistortion
   ```

3. Set **Blend Weight = 1.0**.

Press **Play** and observe the fisheye / pincushion distortion depending on the signs and magnitudes of `k1`, `k2`, `k3`.

**Checkpoint:** Straight lines in the scene bow outwards away from the centre, like a wide-angle lens, and the image still fills the whole screen with no smeared edges. The fit-to-screen lines at the end of the code keep the corners of the screen on the corners of the image.

## 7. Student Tasks

### Task 1 - Explore Barrel vs Pincushion Distortion

Open the Custom node and experiment with:

- Positive and negative `k1` values (e.g. `0.2` vs `-0.2`).
- Smaller / larger `k2` and `k3` values.

Questions:

- Which values give a **fisheye / barrel** look?
- Which give a **pincushion** look?
- How do `k2` and `k3` change the shape of distortion towards the edges?

### Task 2 - Promote Intensity and Zoom to Material Parameters

Modify the HLSL and Custom node so that:

- `intensity` is passed in from a **Scalar Parameter** `DistortionIntensity`.
- `zoom` is passed in from a **Scalar Parameter** `Zoom`.

This allows artists or Blueprint to change distortion strength and zoom at runtime.

Hints:

- Add `DistortionIntensity` and `Zoom` parameters in the material.
- Add `intensity` and `zoom` as **inputs** to the Custom node and remove the hard-coded values.

### Task 3 - Centre Offset Experiment (Optional)

Use `centerOffset` to shift the distortion centre:

- Try `float2(0.1, 0)` or `float2(-0.2, 0.05)`.
- Observe how the distortion appears to "pull" more strongly towards one side of the frame.

Think about possible use cases: broken lens, helmet-cam, body-cam footage, etc.

## 8. Reflective Questions

1. Why do we convert UVs so that the **centre of the screen is (0, 0)** before applying the distortion model?
2. How does the polynomial form `1 + r^2 (k1 + r^2 (k2 + r^2 k3))` allow us to control distortion at different radii?
3. What visual differences do you observe when only `k1` is non-zero compared to when `k2` and `k3` are also used?
4. Why might a game developer want the distortion **intensity** and **zoom** to be controllable at runtime?
5. How could this effect be combined with chromatic aberration, vignette, or pixelation to create a distinctive visual style?
