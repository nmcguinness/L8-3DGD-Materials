---
title: "HLSL Fundamentals via the Custom Node"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: notes
session: 8
topic_code: MAT-08
difficulty_tier: intermediate
duration: "2 x 30 min theory, 2 x 20 min demo"
prerequisites: [03_master_materials_and_layering, 04_procedural_textures_and_uv_maths, 08_screen_space_post_processing]
feeds_into: [lab_15_custom_node_translation, lab_16_procedural_patterns, 10_applied_hlsl]
engine_version: "UE 5.7"
---

# HLSL Fundamentals via the Custom Node

---

## 1. Key Concepts

1. **Every material graph is already HLSL.** The Material Editor writes shader code from your nodes, and you can read it.
2. **A Custom node is a function body.** Its inputs are the function's parameters, and it must `return` a value of its **Output Type**.
3. **Types follow the wires.** An input is a `float` if a scalar is wired to it and a `float3` if a colour is, so check what you connect.
4. **Swizzles are free.** `.xy`, `.rgb` and `.zyx` reorder or pick channels at no cost, exactly like `ComponentMask` and `AppendVector`.
5. **Most nodes have an HLSL twin with the same name.** `Lerp` is `lerp`, `Saturate` is `saturate`, `Frac` is `frac`. Learn the few that differ.
6. **HLSL is not automatically faster.** The compiler sees the same maths either way. What HLSL buys you is fewer nodes and code you can read.
7. **Translate, then verify.** Compare the Custom node's output with the node graph's before you delete anything.
8. **One Custom node can return several results.** Pack them into a `float4` and split them with a `ComponentMask`.
9. **Keep inputs explicit.** Pass UVs, time and parameters in as inputs, so the node is reusable and its behaviour is visible on the graph.
10. **Anti-alias in code too.** `fwidth` gives you the same clean edges as the `DDX` and `DDY` nodes from Notes 04.

---

## 2. Framing Question

The floor tiles from Notes 04 took 36 nodes and a large, careful layout. Every one of those nodes is a line or two of code that the editor writes for you behind the scenes. What does that code look like, can you write it yourself in a Custom node, and when is writing code better than wiring nodes?

---

## 3. What You Will Learn

| Outcome | Where |
|:---|:---|
| Find and read the HLSL that a material graph generates | 5.1 |
| Use HLSL scalar and vector types, constructors and swizzles | 5.2, 5.3 |
| Translate common nodes into HLSL intrinsics | 5.4 |
| Set up a Custom node: code, output type, inputs and returns | 5.5 |
| Compare the cost of a graph and its HLSL with the Stats panel | 5.6 |
| Write stripes, checkers and tiled patterns in HLSL | 5.7, 5.8 |
| Write signed distance fields for circles, boxes and rings, and combine them | 5.9 |
| Write a hash and value noise in HLSL | 5.10 |
| Return several results from one Custom node | 5.11 |

---

## 4. Why This Matters

Node graphs are good at showing data flow, but they grow fast. A pattern that fits on a few lines of paper becomes dozens of nodes, and changing it means rewiring half the graph. Code is shorter, easier to compare, easier to paste between projects and easier to discuss in a code review.

You do not need to give up nodes. Most production materials mix both: nodes for textures, parameters and outputs, and Custom nodes for dense maths. Knowing HLSL also lets you read the shaders the engine generates, which is the only way to understand some bugs and costs. This week starts with the language and the Custom node. Week 9 adds loops, texture sampling and shared code files.

---

## 5. Core Ideas

Sections 5.1 to 5.6 are Session A, the language and the Custom node. Sections 5.7 to 5.11 are Session B, procedural patterns in HLSL.

### 5.1 Reading the Generated HLSL

Open any material and choose **Window > Shader Code > HLSL Code**. A panel shows the shader the editor generated for the platform you select. Most of it is a fixed template shared by every material. The part written from your graph is in a function named `CalcPixelMaterialInputs`.

Inside it, each node becomes one or more lines assigned to local variables named `Local0`, `Local1` and so on, for example:

```hlsl
MaterialFloat2 Local3 = (Local2 * Material.PreshaderBuffer[1].x);
MaterialFloat2 Local4 = frac(Local3);
```

`MaterialFloat2` is the engine's name for `float2`, and `Material.PreshaderBuffer` holds parameter values sent from the CPU. Use the panel's search box to find a constant you know is in the graph, such as `43758.5453`, and read the lines around it.

Two other windows matter this week. **Window > Stats** shows the instruction count of the base pass pixel shader, and **Window > Platform Stats** shows the cost on several platforms at once.

### 5.2 Types and Constructors

HLSL is a typed C-like language. The types you need are:

| Type | Holds | Matches |
|:---|:---|:---|
| `float` | One number | `Constant`, `ScalarParameter`, one channel |
| `float2` | Two numbers | `TextureCoordinate`, `Constant2Vector` |
| `float3` | Three numbers | Colours, positions, normals |
| `float4` | Four numbers | Colour with alpha, packed results |
| `half`, `half2` ... | Lower precision numbers | Used on mobile to save registers. Desktop GPUs usually treat `half` as `float` |
| `int`, `bool` | Whole numbers, true or false | Loop counters and tests |

A constructor builds a vector: `float3(1.0, 0.5, 0.0)`, or `float4(colour, 1.0)` to add a fourth channel to a `float3`. Arithmetic on vectors works per channel, so `uv * 4.0` scales both channels, and a scalar is repeated across every channel it meets.

Write numbers with a decimal point, `1.0` rather than `1`, so every value is clearly a `float`.

### 5.3 Swizzling

After a vector, `.x .y .z .w` or `.r .g .b .a` select channels in any order and any combination:

| Code | Result | Node equivalent |
|:---|:---|:---|
| `uv.x` | The first channel | `ComponentMask` R |
| `colour.rgb` | The first three channels | `ComponentMask` RGB |
| `p.yx` | The two channels swapped | Two masks and an `AppendVector` |
| `v.xxx` | One channel copied three times | A mask wired into a `float3` |
| `float4(colour.rgb, rough)` | Three channels plus one | `AppendVector` |

Swizzling costs nothing, because the GPU reads channels in any order anyway. Do not mix the two letter sets in one swizzle: `.xg` is an error.

### 5.4 From Nodes to Intrinsics

Most maths nodes are **intrinsic functions**, built into HLSL with the same name in lower case:

| Node | HLSL | Notes |
|:---|:---|:---|
| `Add`, `Subtract`, `Multiply`, `Divide` | `+ - * /` | Per channel |
| `Lerp` | `lerp(a, b, t)` | |
| `Saturate` | `saturate(x)` | Clamp to 0 to 1 |
| `Clamp` | `clamp(x, lo, hi)` | |
| `SmoothStep` | `smoothstep(lo, hi, x)` | Same order as the node's **Min**, **Max**, **Value** |
| `Step` | `step(y, x)` | 1 where `x >= y`, same order as the node's **Y** and **X** pins |
| `Frac`, `Floor`, `Abs` | `frac`, `floor`, `abs` | |
| `Dot`, `Length`, `Distance` | `dot`, `length`, `distance` | |
| `Power` | `pow(base, exp)` | `base` must not be negative |
| `Arctangent2` | `atan2(y, x)` | Returns radians, from -pi to pi |
| `DDX`, `DDY` | `ddx`, `ddy` | `fwidth(x)` is `abs(ddx(x)) + abs(ddy(x))` |
| `Min`, `Max`, `OneMinus` | `min`, `max`, `1.0 - x` | |

The one that catches everyone is `Sine`. The `Sine` and `Cosine` nodes have a **Period** setting, 1 by default, which means one full wave per unit: the node computes `sin(x * 2 * pi)`. HLSL `sin` and `cos` take radians, a full wave every 6.283185. When you translate a `Sine` node, check its **Period**: if it is 1, write `sin(x * 6.283185)`, and if it is 6.283185, as in the hash from Notes 03, write `sin(x)`.

### 5.5 The Custom Node

Add a **Custom** node from the palette. Its settings in the **Details** panel are:

| Setting | Meaning |
|:---|:---|
| **Code** | The HLSL, written as the body of a function. It must `return` a value |
| **Output Type** | `CMOT Float 1`, `CMOT Float 2`, `CMOT Float 3` or `CMOT Float 4`, the type you return |
| **Description** | The title shown on the node, for example `HeightBlend` |
| **Inputs** | A list of names. Each becomes an input pin and a variable in the code |

Each input's type comes from what is wired into it, so the same Custom node can take a `float` or a `float3` without changing the code, as long as the maths still makes sense. Every input must be connected, because an unconnected input has no type and the material fails to compile.

The **Code** box is small. For anything longer than a line or two, write the code in a text editor and paste it in.

When the code has an error, the **Stats** panel shows the compiler's message with a line number from the generated shader, not from your code. Read the message, then look for the same mistake in your few lines: a missing semicolon, a misspelt input name or a wrong type.

Here is the height blend from Notes 03 5.9 as one Custom node.

**Output Type:** `CMOT Float 1`. **Inputs:** `Weight` (float, the splat map's R), `Height` (float, the inverted stone height), `BlendContrast` (float).

```hlsl
return saturate((2.0 * Weight + Height - 1.5) * BlendContrast + 0.5);
```

Nine nodes become one line, and the formula from the notes is now written exactly as it reads.

### 5.6 What HLSL Costs

It is tempting to assume that code is faster than nodes. The numbers from this week's demo materials say otherwise:

| Material | Nodes | Pixel shader instructions |
|:---|:---|:---|
| `MM_Env_MossyStoneFloor_Opaque` (Notes 03, nodes) | 31 | 267 |
| `MM_Env_HeightBlendHLSL_Opaque` (Custom node) | 23 | 267 |
| `MM_Env_FloorTiles_Opaque` (Notes 04, nodes) | 36 | 215 |
| `MM_Env_FloorTilesHLSL_Opaque` (Custom node) | 9 | 216 |

The instruction counts match because the same maths reaches the same compiler. Most of each count is the fixed cost of a lit material, so the pattern itself is a small part of the total.

What changes is the graph: a quarter of the nodes, in one place, readable as text. Custom nodes have real downsides too. The editor cannot preview the inside of the node, cannot fold constants into or out of it, and cannot help when the code is wrong. Use HLSL where the maths is dense and stable, and keep nodes where artists need to see and change the flow.

### 5.7 Stripes and Checkers in Code

The patterns from Notes 03 5.7 and Notes 04 5.2 translate line by line.

**Stripes.** **Output Type:** `CMOT Float 1`. **Inputs:** `UV` (float2), `Count` (float).

```hlsl
return step(0.5, frac(UV.x * Count));
```

**Checker.** **Output Type:** `CMOT Float 1`. **Inputs:** `UV` (float2), `Count` (float).

```hlsl
float2 cell = floor(UV * Count);
return fmod(cell.x + cell.y, 2.0);
```

**Anti-aliased rotated stripes.** The triangle wave and `fwidth` from Notes 04 give clean edges at any distance. **Output Type:** `CMOT Float 1`. **Inputs:** `UV` (float2), `Angle` (float, in turns), `Count` (float).

```hlsl
float a = Angle * 6.283185;
float x = UV.x * cos(a) - UV.y * sin(a);
float s = x * Count;
float tri = abs(frac(s) - 0.5) * 2.0;
float w = fwidth(s) * 2.0;
return smoothstep(0.5 - w, 0.5 + w, tri);
```

Local variables such as `x`, `s` and `w` name each step, which a node graph can only do with comments.

### 5.8 Frac Tiling and Cell IDs

`frac(UV * Count)` gives the position inside each cell, and `floor(UV * Count)` gives the cell's ID, as in Notes 03. In code you usually centre the cell position so that 0 is the middle of the cell:

```hlsl
float2 cell = floor(UV * Count);
float2 p = frac(UV * Count) - 0.5;
```

`p` now runs from -0.5 to 0.5 in every cell, so a shape drawn around (0, 0) appears once in every cell, and `cell` can choose a different random value, colour or shape for each one.

### 5.9 Signed Distance Fields in Code

The distance fields from Notes 04 5.7 are each one or two lines. Each returns a negative value inside the shape, 0 on its edge and a positive value outside.

| Shape | HLSL, with `p` centred on the shape |
|:---|:---|
| Circle of radius `r` | `length(p) - r` |
| Box of half-size `b` | `float2 q = abs(p) - b; length(max(q, 0.0)) + min(max(q.x, q.y), 0.0)` |
| Ring of radius `r` and half-width `w` | `abs(length(p) - r) - w` |
| Rounded box | The box distance minus the corner radius |

Combine shapes by combining distances:

| Operation | HLSL |
|:---|:---|
| Union, either shape | `min(d1, d2)` |
| Intersection, both shapes | `max(d1, d2)` |
| Subtraction, the first minus the second | `max(d1, -d2)` |

`abs(p)` mirrors a point into one quarter of the cell, so one circle at `abs(p) - 0.36` appears in all four corners at once. Turn any distance into a clean mask with `fwidth`:

```hlsl
float aa = fwidth(d);
float fill = 1.0 - smoothstep(-aa, aa, d);
```

Here is a complete pattern that tiles the surface and puts a random circle, box or ring in each cell.

**Output Type:** `CMOT Float 3`. **Inputs:** `UV` (float2), `Count` (float), `Background` (float3), `ShapeColour` (float3).

```hlsl
float2 cell = floor(UV * Count);
float2 p = frac(UV * Count) - 0.5;
float pick = frac(sin(dot(cell, float2(12.9898, 78.233))) * 43758.5453);
float dCircle = length(p) - 0.3;
float2 q = abs(p) - 0.25;
float dBox = length(max(q, 0.0)) + min(max(q.x, q.y), 0.0);
float dRing = abs(length(p) - 0.3) - 0.06;
float d = pick < 0.33 ? dCircle : (pick < 0.66 ? dBox : dRing);
float aa = fwidth(d);
float fill = 1.0 - smoothstep(-aa, aa, d);
return lerp(Background, ShapeColour, fill);
```

The `? :` operator chooses between two values, like an `If` node. All three distances are still computed in every pixel, which is cheap here, but worth remembering when the branches are expensive.

### 5.10 Hash and Value Noise

The hash from Notes 03 is one line:

```hlsl
float h = frac(sin(dot(cell, float2(12.9898, 78.233))) * 43758.5453);
```

Value noise from Notes 04 5.8 hashes the four corners of each cell and blends them with a smooth curve.

**Output Type:** `CMOT Float 1`. **Inputs:** `UV` (float2), `Scale` (float).

```hlsl
float2 p = UV * Scale;
float2 i = floor(p);
float2 f = frac(p);
float a = frac(sin(dot(i, float2(12.9898, 78.233))) * 43758.5453);
float b = frac(sin(dot(i + float2(1.0, 0.0), float2(12.9898, 78.233))) * 43758.5453);
float c = frac(sin(dot(i + float2(0.0, 1.0), float2(12.9898, 78.233))) * 43758.5453);
float d = frac(sin(dot(i + float2(1.0, 1.0), float2(12.9898, 78.233))) * 43758.5453);
float2 u = f * f * (3.0 - 2.0 * f);
return lerp(lerp(a, b, u.x), lerp(c, d, u.x), u.y);
```

The hash is written out four times. A Custom node's code is a function body, so you cannot declare a `Hash` function inside it. Week 9 shows two ways round this: a struct with member functions, and a shared `.ush` file.

### 5.11 Several Results From One Node

A Custom node has one return value, but a `float4` holds four numbers. The floor tiles return a colour and a roughness together.

**Output Type:** `CMOT Float 4`. **Inputs:** `UV` (float2), `TileCount` (float), `Grout` (float), `TileColourA` (float3), `TileColourB` (float3), `GroutColour` (float3).

```hlsl
float2 p = UV * TileCount;
float2 f = frac(p);
float2 cell = floor(p);
float edge = min(min(f.x, 1.0 - f.x), min(f.y, 1.0 - f.y));
float aa = fwidth(p.x);
float tile = smoothstep(Grout - aa, Grout + aa, edge);
float rnd = frac(sin(dot(cell, float2(12.9898, 78.233))) * 43758.5453);
float3 colour = lerp(GroutColour, lerp(TileColourA, TileColourB, rnd), tile);
float roughness = lerp(0.9, 0.2, tile);
return float4(colour, roughness);
```

On the graph, a `ComponentMask` RGB feeds `Base Color` and a `ComponentMask` A feeds `Roughness`. The parameters stay as ordinary parameter nodes wired into the inputs, so material instances work exactly as before.

---

## 6. Worked Example

This is the in-class demo. Open `/Game/GD/Notes09/Demo/Levels/Lvl_Notes09`. Each node graph stands next to its HLSL version. The finished demo materials are in `/Game/GD/Notes09/Demo/Materials`. To reproduce the demo, create `/Game/Student/Notes09/MyWork` and save your assets there.

### 6.1 Part A - From Graph to Code (Session A)

**Step 1 - Read the generated code.** Open `MM_Env_MossyStoneFloor_Opaque` from `/Game/GD/Notes03/Demo/Materials`. Choose **Window > Shader Code > HLSL Code** and search for `1.5`. The lines around it are the height blend from Notes 03, written as `Local` variables. Note the instruction count in **Stats**: 267.

**Step 2 - Replace the chain.** Duplicate the material into `MyWork` as `MM_Env_HeightBlendHLSL_Opaque`. Delete the nine nodes between the splat map and the three `Lerp` alphas. Add a Custom node with the height blend code from 5.5, set **Output Type** to `CMOT Float 1`, add the inputs `Weight`, `Height` and `BlendContrast`, and wire it to all three `Lerp` alphas.

**Step 3 - Compare.** The two floors are identical. **Stats** still says 267 instructions, and the graph has 23 nodes instead of 31.

**Step 4 - Break it.** Rename an input to `weight` with a lower-case w and apply. Read the error in **Stats**, then fix it. HLSL names are case-sensitive.

### 6.2 Part B - Patterns in Code (Session B)

**Step 5 - Floor tiles.** Build `MM_Env_FloorTilesHLSL_Opaque` from 5.11, with the same parameter defaults as the Notes 04 version. Place both side by side: they match, at 215 and 216 instructions, with 36 nodes against 9.

**Step 6 - Shapes.** Build `MM_Prop_SDFShapesHLSL_Opaque` from 5.9. Change `Count` and watch every cell pick its own shape. Swap `min` for `max` in a test union to show intersection.

**Step 7 - Value noise.** Build `MM_Prop_ValueNoiseHLSL_Opaque` from 5.10, then point out the four copies of the hash. They are the motivation for Week 9.

---

## 7. Common Pitfalls

| Pitfall | Symptom | Fix |
|:---|:---|:---|
| No `return` in a multi-line Custom node | Compile error | End the code with `return` |
| Output Type does not match the return | Compile error, or channels missing | Return the type set in **Output Type** |
| Input left unconnected | Compile error naming the input | Wire every input, even to a constant |
| Input name misspelt or wrong case | "undeclared identifier" error | Use exactly the name in **Inputs** |
| Missing semicolon | Error on a line that looks correct | End every statement with `;` |
| `Sine` node translated as `sin(x)` | The pattern repeats at the wrong rate | Check the node's **Period** and multiply by 6.283185 when it is 1 |
| `step` arguments reversed | The mask is inverted | `step(y, x)` is 1 where `x >= y` |
| Integer literals | Unexpected rounding in division | Write `1.0`, not `1` |
| Mixed swizzle sets | Compile error on `.xg` | Use `xyzw` or `rgba`, not both |
| Assuming HLSL is faster | No gain in Stats | Measure, and choose HLSL for clarity |

---

## 8. Exercises

1. Open the HLSL view for one of your Week 3 materials and find the lines produced by its `Frac` and `Floor` nodes.
2. Write a Custom node that returns a checker in two colours, with `ColourA` and `ColourB` as `float3` inputs.
3. Translate the radar sweep from Notes 04 into one Custom node and compare instruction counts.
4. Combine a circle and a box with subtraction so the box cuts a notch out of the circle.
5. Return a `float4` from the shape pattern with the shape mask in A, and use it to make the shapes shiny and the background rough.

---

## 9. Reflective Questions

1. The HLSL versions cost the same instructions as the node versions. Given that, how would you decide between them on a team where artists maintain the materials?
2. A Custom node hides its maths from the graph. What could an artist no longer do with the floor tiles once they are code?
3. The value noise repeats the hash four times. What problems does copy-pasted code cause, and how have you solved them in other languages?
4. HLSL inputs take their type from the wires. When is that convenient, and when could it hide a mistake?

---

## 10. How This Builds

- **Lab 15** (`lab_15_custom_node_translation`) applies Session A. You translate three supplied graphs into Custom nodes and prove each one matches.
- **Lab 16** (`lab_16_procedural_patterns`) applies Session B. You write a riveted metal panel from two distance fields and value noise.
- **Week 9** (`10_applied_hlsl`) adds loops, texture sampling inside Custom nodes, helper functions through structs and shared `.ush` files, and debugging.

---

## 11. Glossary

| Term | Meaning |
|:---|:---|
| **Constructor** | Code that builds a vector from values, such as `float3(1.0, 0.0, 0.0)` |
| **Custom node** | A material node that runs HLSL code you write |
| **HLSL** | High-Level Shading Language, the shader language Unreal's materials compile to |
| **Intrinsic function** | A function built into HLSL, such as `lerp`, `frac` or `dot` |
| **Output Type** | The type a Custom node returns, from `CMOT Float 1` to `CMOT Float 4` |
| **Signed distance field** | A function giving the distance to a shape's edge, negative inside |
| **Swizzle** | Selecting or reordering vector channels with letters, such as `.xy` or `.bgr` |
| **Ternary operator** | `condition ? a : b`, which chooses between two values |
