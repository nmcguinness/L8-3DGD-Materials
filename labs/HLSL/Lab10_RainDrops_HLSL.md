# Lab 10 - Animated Raindrops with Custom Nodes & HLSL
## Module: 3D Game Development

## Overview

In this lab you will create a simple procedural **animated raindrop and ripple effect** using a **Custom HLSL node** inside Unreal Engine's Material Editor.
This effect splits the surface into cells, places one drop at a random position in each cell, and grows a ripple ring from it over time, with each drop starting at its own moment. The aim is not to produce a production ready shader. It is to show how HLSL can support dynamic texture generation.

This lab builds on the Custom Node techniques introduced in **Lab 1**, the cell maths from **Lab 8** and the loop from **Lab 9**, adding randomisation and soft fading.

## Learning Outcomes

By completing this lab, you will be able to:

- Implement a multi-input Custom HLSL Node.
- Use loops (`for`) inside a Custom node.
- Generate pseudo-random positions for droplets.
- Animate ripple radius using time-based pulse cycles.
- Produce raindrop masks for use in opacity, emissive, refraction, or normal distortion.
- Integrate the mask into a surface material.

## Creating the Raindrop Material Function

### Create a Material Function
1. In the Content Browser: **Right-click -> Material -> Advanced -> Material Function**
2. Name it:
   ```
   MF_RainDrops
   ```
3. Open the function.

### Add Function Inputs
Add the following **Function Input** nodes:

1. **UV**
   - Name: `UV`
   - Type: *Vector 2*

2. **TimeInput**
   - Name: `TimeInput`
   - Type: *Scalar*

### Add a Custom Node
1. Add a **Custom** node.
2. Name it:
   ```
   RainDrops
   ```
3. Set **Output Type = CMOT Float 1**, a single mask value.
4. Add two inputs in the Details panel:
   - `uv`
   - `time`
5. Connect:
   - Function Input **UV** -> `uv`
   - Function Input **TimeInput** -> `time`

### HLSL Overview
The raindrop effect is produced by:

- Splitting the surface into cells with `floor` and `frac`, one drop per cell.
- Hashing each cell's ID to get a random position and a random start time.
- Using `frac(time / duration + rnd.y)` to give each drop a life that runs from 0 to 1 and repeats.
- Growing the ring's radius with its life and fading it out as it grows.
- Looping over the cell and its 8 neighbours, so a ring that spreads into the next cell is still drawn.
- Accumulating rings into a single output mask.

Checking 9 cells per pixel is far cheaper than looping over every drop on the surface, and it costs the same however many drops there are.

### Full HLSL Code
#### Paste this HLSL code into the Custom Node:
```hlsl
// Inputs:
//   uv   : float2 UV coordinates (0..1)
//   time : time in seconds from the Time node

float cells = 8.0;      // drops across the surface, one per cell
float duration = 1.0;   // seconds for one ripple to grow and fade
float ringWidth = 0.04; // ring thickness as a fraction of a cell

// Split the surface into cells: f is the position inside the cell, cell is its ID.
float2 p = uv * cells;
float2 cell = floor(p);
float2 f = frac(p);

float mask = 0.0;

// Check this cell and its 8 neighbours, so ripples can cross cell edges.
[unroll] for (int y = -1; y <= 1; y++)
{
    [unroll] for (int x = -1; x <= 1; x++)
    {
        float2 c = cell + float2(x, y);

        // Two random values for this cell, both 0..1: one for position, one for timing.
        float2 rnd = frac(sin(float2(dot(c, float2(12.9898, 78.233)),
                                     dot(c, float2(39.346, 11.135)))) * 43758.5453);

        // Drop centre inside the neighbouring cell, kept away from its edges.
        float2 centre = float2(x, y) + 0.2 + rnd * 0.6;

        // Life runs 0..1 over one duration, and each drop starts at its own moment.
        float life = frac(time / duration + rnd.y);

        // The ring grows to most of a cell over its life.
        float radius = life * 0.9;

        // Distance from this pixel to the drop centre, then a soft ring at the radius.
        float d = length(f - centre);
        float ring = 1.0 - smoothstep(0.0, ringWidth, abs(d - radius));

        // Fade the ring out as it grows, and add it to the mask.
        mask += ring * (1.0 - life);
    }
}

return saturate(mask);
```

## Testing the Function in a Material

### Create a Test Material
1. Create a new Material:
   ```
   MM_Prop_RainDrops_Opaque
   ```
2. Add:
   - **TextureCoordinate**
   - **Time**
   - **Multiply** (optional rain speed)
   - **Material Function Call** (MF_RainDrops)

### Connect the Nodes
- TextureCoordinate -> `UV`
- Time (or Time * speed parameter) -> `TimeInput`
- MF_RainDrops output -> one of:
  - **Emissive**
  - **Base Color multiplier**

Apply the material to a plane or mesh and observe animated ripples.

**Checkpoint:** Rings appear all over the surface, each growing from its own point and fading out at a different moment, with no grid visible and no rings cut off at cell edges.

## Student Tasks

### Adjust Density and Radius
Edit the HLSL:
- `cells`
- `duration`
- `ringWidth`

Observe how drop density, speed and ring thickness affect the rain style.

### Expose Parameters
Convert HLSL constants into **Function Inputs**:
- Cells
- Duration
- RingWidth

Then connect them to parameters in the calling material, enabling real-time art control.

### Apply to a Realistic Surface
Use the raindrop mask to:
- Distort normals
- Drive roughness

## Reflective Questions

1. Why is `frac(time / duration + rnd.y)` used to animate droplet radius?
2. What visual changes occur when increasing or decreasing the `cells` value?
3. How do smooth fades (`smoothstep`) improve the realism of the ripple?
4. Why is pseudo-randomisation important in procedural droplet generation?
5. How might this effect be extended to simulate droplets sliding down glass?
6. Why does the loop check the eight neighbouring cells as well as the pixel's own cell?

## Useful Links

- [Unreal Engine Material Custom Material Expressions (Official Docs)](https://dev.epicgames.com/documentation/en-us/unreal-engine/custom-material-expressions-in-unreal-engine)

- [Microsoft HLSL Reference](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-pguide)

- [HLSL Tutorial Series (Ben Cloward)](https://www.youtube.com/playlist?list=PL78XDi0TS4lEMvytsE_MoWEpzBcukXv9b)

## Appendix A - Useful HLSL Functions

### Math Utilities
- **abs(x)** - absolute value
- **frac(x)** - fractional part
- **floor(x)** - round down
- **ceil(x)** - round up
- **fmod(a, b)** - floating-point modulo
- **clamp(x, min, max)** - constrain range
- **saturate(x)** - clamp to [0,1]

### Trigonometry
- **sin(x)**
- **cos(x)**
- **tan(x)**

### Vector Operations
- **length(v)** - magnitude
- **distance(a, b)** - length(a - b)
- **normalize(v)** - unit vector
- **dot(a, b)** - projection / angle
- **cross(a, b)** - perpendicular 3D vector

### Interpolation Functions
- **lerp(a, b, t)** - linear interpolation
- **smoothstep(a, b, x)** - smooth falloff curve
- **step(edge, x)** - 0/1 threshold
- **pow(x, y)** - curve shaping

### Helper Functions in a Custom Node

A Custom node's code is the body of a function, so you cannot declare `hash` or `rotate` directly in it. Put them inside a struct and call them through an instance, as Notes 10 5.3 explains. **Output Type:** `CMOT Float 1`. **Inputs:** `uv` (float2).

```hlsl
struct Helpers
{
    float Hash(float2 p)
    {
        return frac(sin(dot(p, float2(12.9898, 78.233))) * 43758.5453);
    }

    float2 Rotate(float2 uv, float angle)
    {
        float s = sin(angle);
        float c = cos(angle);
        return float2(uv.x * c - uv.y * s, uv.x * s + uv.y * c);
    }
};
Helpers H;

return H.Hash(floor(uv * 8.0));
```
