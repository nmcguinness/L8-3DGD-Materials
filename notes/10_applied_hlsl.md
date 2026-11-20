---
title: "Applied HLSL"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: notes
session: 9
topic_code: MAT-09
difficulty_tier: advanced
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [09_hlsl_fundamentals, 06_geometry_and_scene_aware_materials]
feeds_into: [lab_17_loops_sampling_and_pom, lab_18_hlsl_dissolve_rewrite]
engine_version: "UE 5.7"
---

# Applied HLSL

---

## 1. Key Concepts

1. **A loop multiplies its body.** Eight iterations of three samples is twenty-four samples, whatever the code looks like.
2. **Stats counts a `[loop]` once.** An unrolled loop shows its true size, a dynamic loop does not, so measure dynamic loops with `stat GPU`.
3. **Sample textures through a Texture Object input.** The Custom node gives you the texture and a matching sampler named after the input.
4. **Inside a dynamic loop, choose the mip level yourself.** Use `Texture2DSampleLevel` or `Texture2DSampleGrad`, because the GPU cannot work out reliable derivatives there.
5. **Wrap helper functions in a struct.** A Custom node's code is a function body, and a struct with member functions is the only way to declare functions inside it.
6. **Shared code belongs in a `.ush` file.** One copy of the noise, included everywhere, beats ten pasted copies that drift apart.
7. **Pass time and view data in as inputs.** `Time`, `CameraVector` and `AbsoluteWorldPosition` are nodes for a reason: they keep the code independent of engine internals.
8. **Read the first compile error only.** Later errors are usually caused by the first one.
9. **Hunt NaNs with a bright colour.** A NaN spreads through every calculation it touches, so make it visible where it starts.
10. **Use HLSL where it earns its place.** Loops, dense maths and shared helpers suit code. Parameters, textures and anything an artist tweaks suit nodes.

---

## 2. Framing Question

Last week every Custom node did straight-line arithmetic, the same work as a few nodes. The things that are truly hard to build from nodes are different: a loop that marches a ray through a height map, a blur that reads twenty-five neighbours, and a noise function you write once and use in fifty materials. How do you build those, how do you know what they cost, and what do you do when the screen turns black and the compiler says nothing useful?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Write `[unroll]` and `[loop]` loops and predict their cost | 5.1 |
| Sample textures inside a Custom node, choosing the mip level where needed | 5.2 |
| Declare helper functions with the struct trick | 5.3 |
| Pass time, view and position data into code | 5.4 |
| Write parallax occlusion mapping as an HLSL loop | 5.5 |
| Share HLSL between materials with `.ush` files in a plugin | 5.6 |
| Read and fix compile errors | 5.7 |
| Find and fix NaNs | 5.8 |
| Visualise intermediate values | 5.9 |
| Decide between HLSL and nodes | 5.10 |

---

## 4. Why This Matters

Production shaders are full of loops: blurs, parallax, multi-tap outlines, octaves of noise. A node graph cannot express a loop at all, and the engine's own functions that loop, such as `ParallaxOcclusionMapping`, are written in HLSL underneath. When you can write a loop, you can build these effects yourself and tune their cost.

Shared code matters as soon as a project has more than a few materials. Noise, distance fields and colour maths get copied into every material that needs them, and the copies drift apart. A `.ush` file fixes that, and it is how engine and studio shader code is organised.

Debugging is the skill that separates a shader you can ship from one you cannot. The tools are simple, and the habits are what this week teaches.

---

## 5. Core Ideas

Sections 5.1 to 5.5 are Session A, loops, sampling and helpers. Sections 5.6 to 5.10 are Session B, shared code and debugging.

### 5.1 Loops and Their Cost

HLSL has the usual `for` loop. Two attributes before it tell the compiler how to build it:

- **`[unroll]`** copies the body once per iteration. The count must be known when the shader compiles, such as a literal `5`. There is no loop at run time, so it is fast, but the shader grows with every iteration.
- **`[loop]`** keeps a real loop. The count can come from an input, and the loop can stop early with `break`. Each iteration costs a little extra for the counter and the test.

A 5 by 5 box blur reads 25 texels and averages them:

**Output Type:** `CMOT Float 3`. **Inputs:** `UV` (float2), `Tex` (Texture Object), `Radius` (float, the distance between taps in UV units).

```hlsl
float3 sum = 0.0;
[unroll] for (int y = -2; y <= 2; y++)
{
    [unroll] for (int x = -2; x <= 2; x++)
    {
        sum += Texture2DSample(Tex, TexSampler, UV + float2(x, y) * Radius).rgb;
    }
}
return sum / 25.0;
```

The **Stats** panel tells two different stories for the two kinds of loop. Measured on blurs of three sizes:

| Blur | `[unroll]` instructions | `[loop]` instructions |
|:---|:---|:---|
| 3 x 3, 9 taps | 218 | 203 |
| 5 x 5, 25 taps | 263 | 203 |
| 7 x 7, 49 taps | 330 | 203 |

The unrolled count grows with the taps, because the body really is repeated. The dynamic count stays at 203, because **Stats** counts the loop body once, whatever the iteration count. The 49-tap loop is not as cheap as the 9-tap one. It does five times the work at run time, which only `stat GPU` or a GPU profiler shows. This is the same undercount you met with parallax occlusion mapping in Notes 06.

### 5.2 Sampling Textures in Code

Wire a **Texture Object** node, not a **Texture Sample**, into a Custom node input named, for example, `Tex`. The code then has two names available: `Tex`, the texture, and `TexSampler`, its sampler. Three functions read it:

| Function | Mip level | Use it |
|:---|:---|:---|
| `Texture2DSample(Tex, TexSampler, uv)` | Chosen automatically from screen derivatives | Ordinary straight-line code and `[unroll]` loops |
| `Texture2DSampleLevel(Tex, TexSampler, uv, mip)` | The one you pass, such as 0 | Dynamic loops, and effects that must stay sharp |
| `Texture2DSampleGrad(Tex, TexSampler, uv, ddxUV, ddyUV)` | From derivatives you pass | Dynamic loops and broken UVs that still need correct filtering |

Inside a `[loop]`, neighbouring pixels may run different numbers of iterations, so the GPU cannot compare them to find derivatives. `Texture2DSample` there may still compile, as it does on DirectX 12, but the mip level it picks is undefined, which shows as shimmering or seams, and stricter compilers reject it outright. Compute `ddx(UV)` and `ddy(UV)` before the loop and use `Texture2DSampleGrad`, or pick a level with `Texture2DSampleLevel`.

`Texture2DSampleGrad` also fixes the seam from Notes 02. A lat-long reflection's `u` jumps from 1 to 0 at one meridian, the derivative there is huge, and the GPU picks the smallest mip, which shows as a thin line. Measure the derivative of a second copy of `u` that jumps somewhere else, and use whichever is smaller:

**Output Type:** `CMOT Float 3`. **Inputs:** `UV` (float2, the lat-long coordinates), `Tex` (Texture Object).

```hlsl
float2 uv = UV;
float2 dx = ddx(uv);
float2 dy = ddy(uv);
float2 uvAlt = float2(frac(uv.x + 0.5), uv.y);
float2 dxAlt = ddx(uvAlt);
float2 dyAlt = ddy(uvAlt);
if (abs(dxAlt.x) + abs(dyAlt.x) < abs(dx.x) + abs(dy.x))
{
    dx.x = dxAlt.x;
    dy.x = dyAlt.x;
}
return Texture2DSampleGrad(Tex, TexSampler, uv, dx, dy).rgb;
```

### 5.3 Helper Functions with the Struct Trick

Last week's value noise wrote the hash out four times, because a function body cannot contain another function. It can contain a struct, and a struct can have member functions. Declare the helpers in a struct, make one instance, and call them through it:

**Output Type:** `CMOT Float 1`. **Inputs:** `UV` (float2), `Scale` (float), `Seconds` (float, from a `Time` node).

```hlsl
struct GDNoise
{
    float Hash(float2 p)
    {
        return frac(sin(dot(p, float2(12.9898, 78.233))) * 43758.5453);
    }

    float Value(float2 p)
    {
        float2 i = floor(p);
        float2 f = frac(p);
        float2 u = f * f * (3.0 - 2.0 * f);
        float a = Hash(i);
        float b = Hash(i + float2(1.0, 0.0));
        float c = Hash(i + float2(0.0, 1.0));
        float d = Hash(i + float2(1.0, 1.0));
        return lerp(lerp(a, b, u.x), lerp(c, d, u.x), u.y);
    }
};
GDNoise N;

float2 p = UV * Scale + Seconds * float2(0.3, 0.1);
float sum = 0.0;
float amp = 0.5;
[unroll] for (int o = 0; o < 5; o++)
{
    sum += N.Value(p) * amp;
    p *= 2.0;
    amp *= 0.5;
}
return sum / 0.96875;
```

This is fBm from Notes 04 5.9, five octaves in a loop. The amplitudes add up to 0.96875, so dividing by it keeps the result between 0 and 1. Each octave costs about 17 instructions: measured, 1 octave is 208, 3 are 241, 5 are 275 and 8 are 325.

### 5.4 Time, View and Position as Inputs

Shader code can read engine values such as the current time directly, but their names change between engine versions and they hide what the node depends on. Pass them in instead:

| Data | Node to wire in | Typical input name |
|:---|:---|:---|
| Time in seconds | `Time` | `Seconds` |
| Direction to the camera | `CameraVector` | `ViewWS`, or `ViewTS` after a `Transform` to tangent space |
| Position of the pixel | `AbsoluteWorldPosition` | `WorldPos` |
| Position of the object | `ObjectPositionWS` | `ObjectPos` |
| Screen position | `ScreenPosition` | `ScreenUV` |

The node's inputs then say exactly what it needs, and you can test it by wiring a constant into `Seconds` to freeze the animation.

### 5.5 Parallax Occlusion Mapping in a Loop

Notes 06 used the engine's `ParallaxOcclusionMapping` function. In HLSL the idea fits in one loop. Starting at the surface, step along the view ray into the height map until the ray is below the surface, and use the UV where that happens:

**Output Type:** `CMOT Float 2`. **Inputs:** `UV` (float2), `ViewTS` (float3, `CameraVector` through a `Transform` from World to Tangent space), `HeightTex` (Texture Object, white is high), `HeightRatio` (float, about 0.05).

```hlsl
float3 v = normalize(ViewTS);
float2 dx = ddx(UV);
float2 dy = ddy(UV);
const int STEPS = 24;
float layerStep = 1.0 / STEPS;
float2 uvStep = -v.xy / max(v.z, 0.1) * HeightRatio * layerStep;
float2 uv = UV;
float layer = 0.0;
float depth = 1.0 - Texture2DSampleGrad(HeightTex, HeightTexSampler, uv, dx, dy).r;
[loop] for (int i = 0; i < STEPS; i++)
{
    if (layer >= depth)
    {
        break;
    }
    uv += uvStep;
    layer += layerStep;
    depth = 1.0 - Texture2DSampleGrad(HeightTex, HeightTexSampler, uv, dx, dy).r;
}
return uv;
```

Wire the returned UV into the colour, normal and ORM samples. Line by line:

- `v.xy / v.z` is how far the ray moves across the surface per unit of depth. At grazing angles `v.z` is small and the ray travels far, and `max(v.z, 0.1)` stops it running off to infinity.
- The loop takes up to 24 steps, each one layer deeper, and `break`s as soon as the layer passes the height map's depth.
- Each step reads the height map with `Texture2DSampleGrad`, using derivatives taken before the loop, as 5.2 requires.

`MM_Env_POMStoneHLSL_Opaque` reports 236 instructions against 246 for the engine function, but both numbers undercount, because each counts its loop once. Real cost is 24 samples per pixel at grazing angles and far fewer looking straight down, because of the `break`.

### 5.6 Shared Code in `.ush` Files

A **`.ush`** file holds HLSL functions that any number of materials can include. The repository keeps two in `assets/hlsl/`:

- `GDNoise.ush`: `GDHash`, `GDValueNoise` and `GDFBM(p, octaves)`.
- `GDShapes.ush`: `GDCircle`, `GDBox`, `GDRing`, `GDUnion`, `GDIntersect`, `GDSubtract` and `GDFill(d)`.

Each function's name starts with `GD` so it cannot clash with the engine's own functions.

A Custom node includes a file by its **virtual path**, such as `/Plugin/GDShaders/GDNoise.ush`, listed under **Include File Paths**. Unreal only knows where to find that path if something maps it to a folder on disk, and that mapping needs a small C++ plugin module that runs before shaders compile. This is a one-time step for a project. It needs a C++ compiler, which is why it is a demonstration rather than a lab.

The plugin has three files plus the shader folder:

```text
Plugins/GDShaders/
    GDShaders.uplugin
    Shaders/
        GDNoise.ush
        GDShapes.ush
    Source/GDShaders/
        GDShaders.Build.cs
        Private/GDShadersModule.cpp
```

`GDShaders.uplugin` declares the module and loads it at `PostConfigInit`, early enough for the shader compiler:

```json
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "GD Shaders",
    "Description": "Shared HLSL helpers for COMP I8015 materials.",
    "Category": "Rendering",
    "CanContainContent": false,
    "Modules": [
        {
            "Name": "GDShaders",
            "Type": "Runtime",
            "LoadingPhase": "PostConfigInit"
        }
    ]
}
```

`GDShaders.Build.cs` lists the engine modules the code uses:

```csharp
using UnrealBuildTool;

public class GDShaders : ModuleRules
{
    public GDShaders(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
        PublicDependencyModuleNames.AddRange(new string[] { "Core", "Projects", "RenderCore" });
    }
}
```

`GDShadersModule.cpp` maps the virtual path to the plugin's `Shaders` folder when the module starts:

```cpp
#include "Modules/ModuleManager.h"
#include "Interfaces/IPluginManager.h"
#include "Misc/Paths.h"
#include "ShaderCore.h"

class FGDShadersModule : public IModuleInterface
{
public:
    virtual void StartupModule() override
    {
        const FString ShaderDir = FPaths::Combine(
            IPluginManager::Get().FindPlugin(TEXT("GDShaders"))->GetBaseDir(), TEXT("Shaders"));
        AddShaderSourceDirectoryMapping(TEXT("/Plugin/GDShaders"), ShaderDir);
    }
};

IMPLEMENT_MODULE(FGDShadersModule, GDShaders)
```

With the plugin built and enabled, a Custom node that lists `/Plugin/GDShaders/GDNoise.ush` under **Include File Paths** can call the helpers directly.

**Output Type:** `CMOT Float 1`. **Inputs:** `UV` (float2), `Scale` (float). **Include File Paths:** `/Plugin/GDShaders/GDNoise.ush`.

```hlsl
return GDFBM(UV * Scale, 5);
```

The editor does not notice when a `.ush` file changes. After editing one, run the console command `recompileshaders changed`, or re-apply the material, to rebuild the shaders that include it.

### 5.7 Reading Compile Errors

When a Custom node does not compile, the **Stats** panel lists errors with a file and line number. The line numbers refer to the generated shader, not to your few lines, so read the message, not the number:

| Message contains | Usual cause |
|:---|:---|
| `undeclared identifier` | A misspelt input or variable name, or wrong case |
| `cannot implicitly convert` | Assigning a `float3` to a `float`, or returning the wrong type |
| `expected ';'` | A missing semicolon on the line before |
| `not all control paths return a value` | A `return` inside an `if` with no `return` after it |
| `gradient instruction used in a loop` | `Texture2DSample` inside a `[loop]` on a strict compiler. Use `Texture2DSampleLevel` or `Texture2DSampleGrad` |
| An internal error naming `MainPS` | Often an `[unroll]` whose count is not a literal |

Fix the first error and apply again. The others usually disappear with it. To see the code exactly as compiled, choose **Window > Shader Code > HLSL Code** and search for `CustomExpression`, which is the generated name of each Custom node's function.

### 5.8 NaNs

A **NaN**, "not a number", is the result of an impossible calculation, such as `sqrt` of a negative number, `0.0 / 0.0`, or `pow` of a negative base. Any maths with a NaN gives a NaN, so one bad value spreads through everything that uses it, and the pixel usually shows as black, or as flickering black squares once bloom and temporal effects spread it to neighbours.

Make them visible:

**Output Type:** `CMOT Float 3`. **Inputs:** `UV` (float2), `ShowNaN` (float, 1 to highlight NaNs).

```hlsl
float x = pow(UV.x - 0.5, 2.2);
float3 colour = x.xxx * 4.0;
if (ShowNaN > 0.5 && any(isnan(colour)))
{
    return float3(1.0, 0.0, 1.0);
}
return colour;
```

`MM_Prop_NaNCheckHLSL_Opaque` is this code. Half the plane turns magenta, exactly where `UV.x - 0.5` is negative. The fixes are the usual guards: `pow(abs(x), e)` or `pow(max(x, 0.0), e)`, `sqrt(max(x, 0.0))`, `x / max(y, 0.0001)`, and `normalize` only of vectors that cannot be zero length.

### 5.9 Visualising Intermediate Values

A shader cannot print, so you show values as colour instead:

- **Return early.** Put `return someValue.xxx;` part-way through the code to see one variable. This is how Lab 16 checked each layer.
- **Map the range.** Values far above 1 all show as white, and negative values as black. Show `frac(x)` to see the pattern of a large value, `x * 0.01` to bring it into range, or `x < 0.0 ? float3(1, 0, 0) : x.xxx` to flag negatives in red.
- **Compare with a difference.** `abs(a - b) * 100` turns any mismatch bright, as in Lab 15.
- **Read exact numbers.** The engine's `DebugScalarValues` material function, in `/Engine/Functions/Engine_MaterialFunctions02/Utility`, draws a value as digits on the surface. Wire an intermediate result into it through an **Additional Output** on the Custom node.

### 5.10 HLSL or Nodes?

| Choose HLSL when | Choose nodes when |
|:---|:---|
| The effect needs a loop | Artists need to see and change the flow |
| The maths is dense and stable, such as noise or distance fields | The graph is mostly textures and parameters |
| The same helper is used in many materials | The material uses features with their own nodes, such as Material Functions, static switches or Substrate |
| You need exact control of sampling, such as mip levels in a loop | You want live previews of every step |

Most real materials mix both: parameters and textures as nodes feeding one or two Custom nodes that do the heavy maths. The cost is rarely the deciding factor, as the measurements in Notes 09 and here show.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes10/Demo/Levels/Lvl_Notes10`. The finished demo materials are in `/Game/GD/Notes10/Demo/Materials`. To reproduce the demo, create `/Game/Student/Notes10/MyWork` and save your assets there.

### 6.1 Part A - Loops, Sampling and Parallax (Session A)

**Step 1 - A blur in a loop.** Build `MM_Prop_TextureBlurHLSL_Opaque` from 5.1 with `T_Wood_Albedo` from `/Game/GD/Shared/Textures/Wood` on a **Texture Object** node. Compare it with the sharp wood beside it. Change `[unroll]` to `[loop]` and watch **Stats** drop from 263 to 203 instructions while the work stays the same.

**Step 2 - Helpers in a struct.** Build `MM_Prop_FBMNoiseHLSL_Opaque` from 5.3 with a `Time` node wired into `Seconds`. The noise drifts slowly. Change the octave count and read the cost in **Stats** each time.

**Step 3 - Parallax in a loop.** Build `MM_Env_POMStoneHLSL_Opaque` from 5.5 with the `T_Stone_*` textures and a `TextureCoordinate` tiled 4 times. The two floor patches compare it with the Notes 06 version, which uses the engine function. Fly low over both: the cobbles stand up in the same way.

### 6.2 Part B - Shared Code and Debugging (Session B)

**Step 4 - The plugin.** Show the plugin structure from 5.6 in a C++ copy of the project, with the `.ush` files copied from `assets/hlsl/` into `Plugins/GDShaders/Shaders`. Build it, enable it, and replace the struct in the fBm material with an include of `GDNoise.ush` and one line, `return GDFBM(UV * Scale, 5);`.

**Step 5 - Break it on purpose.** Delete a semicolon, misspell an input, and return a `float3` from a `CMOT Float 1` node, reading the error each time.

**Step 6 - Find a NaN.** Show `MM_Prop_NaNCheckHLSL_Opaque` with `ShowNaN` at 0, where half the plane is simply black, then at 1, where it turns magenta. Fix it with `pow(max(UV.x - 0.5, 0.0), 2.2)`.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| `Texture2DSample` inside a `[loop]` | Shimmering or seams, or a compile error about gradients | Use `Texture2DSampleLevel` or `Texture2DSampleGrad` |
| `[unroll]` with a count from an input | Compile error | Use a literal count, or `[loop]` |
| Trusting Stats for a `[loop]` | A heavy effect looks cheap | Measure with `stat GPU` |
| Texture Sample wired into a Custom input instead of a Texture Object | "undeclared identifier" for the sampler | Use a **Texture Object** node |
| Sampler name guessed | "undeclared identifier" | The sampler is the input name followed by `Sampler` |
| Helper function declared directly in the code | Compile error | Put it inside a struct |
| POM with no limit on `v.z` | Smears to infinity at grazing angles | `max(v.z, 0.1)` |
| `.ush` edited but nothing changes | Old shader still in use | `recompileshaders changed`, or re-apply the material |
| `pow` of a negative base | Black or flickering pixels | `pow(max(x, 0.0), e)` |
| Several helper files defining the same name | "redefinition" error | Prefix every shared function, as with `GD` |

---

## 8. Exercises

1. Write a 9-tap blur as a `[loop]` whose radius comes from an input, and measure it at 1080p with `stat GPU` against the `[unroll]` version.
2. Add a fifth octave to the fBm, then change the loop count to an input, and explain what the compiler now has to do differently.
3. Extend the POM loop with a final interpolation between the last two steps, so the surface stops looking stepped at grazing angles.
4. Write a four-neighbour outline from Notes 08 as a loop over an array of offsets, `const float2 offs[4] = { ... };`.
5. Add a guard to the NaN demo so it shows the correct curve on both sides of the centre.

---

## 9. Reflective Questions

1. Stats said a 49-tap loop cost the same as a 9-tap loop. How would you explain the difference to a designer who only looks at Stats?
2. The struct trick lets you write helpers inside one material. When is that enough, and when do you need a `.ush` file?
3. Shared shader code is powerful and risky: one change to `GDNoise.ush` changes every material that includes it. How would you manage that on a team?
4. Look back at the materials you have built this block. Which would you rewrite in HLSL, and which would you keep as nodes?

---

## 10. How This Builds

- **Lab 17** (`lab_17_loops_sampling_and_pom`) applies Session A. You extend your Week 8 riveted panel with struct helpers, an fBm loop, a sampled texture and blinking lights. POM is in the Extension.
- **Lab 18** (`lab_18_hlsl_dissolve_rewrite`) applies Session B. You rewrite the Week 4 dissolve as a Custom node and compare cost and readability with the node version.
- This is the last week of the block. The skills carry into any engine: Unity's Shader Graph Custom Function node takes HLSL directly, and Godot's shading language uses the same ideas in a GLSL-like syntax.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **`[loop]`** | An attribute that keeps a real loop at run time, so the count can vary |
| **`[unroll]`** | An attribute that copies a loop's body once per iteration when the shader compiles |
| **Include File Paths** | The Custom node setting that lists `.ush` files to include |
| **NaN** | "Not a number", the result of an impossible calculation, which spreads to every value it touches |
| **Shader directory mapping** | A link from a virtual path such as `/Plugin/GDShaders` to a folder on disk |
| **Struct trick** | Declaring helper functions as members of a struct inside a Custom node |
| **`.ush` file** | An Unreal shader header holding HLSL functions for inclusion in other shaders |
| **Virtual path** | A shader file path that starts at a mapped root, such as `/Engine` or `/Plugin/GDShaders` |
