---
title: "Lab 18 - HLSL Dissolve Rewrite"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-20"
doc-type: lab
session: 9
topic_code: MAT-09
difficulty_tier: advanced
duration: "30 min"
prerequisites: [10_applied_hlsl, lab_17_loops_sampling_and_pom, 05_runtime_materials_and_performance]
feeds_into: []
engine_version: "UE 5.7"
---

# Lab 18 - HLSL Dissolve Rewrite

```yaml
week: 9
session: B
duration: 30 min
prerequisites:
  - "Notes 10, sections 5.6 to 5.10"
  - "Notes 05, section 5.5 (the dissolve)"
assets_needed:
  - "/Game/GD/Shared/Levels/Lvl_Lab"
  - "/Game/GD/Notes05/Demo/Materials/MM_Prop_Dissolve_Masked"
```

---

## 1. Overview

You rewrite the Week 4 dissolve, the burning-edge effect from Notes 05, in HLSL, twice. The first rewrite keeps the engine's `Noise` node and moves only the threshold and edge maths into a Custom node, which you prove matches the original exactly. The second replaces the `Noise` node too, with the fBm from Lab 17. Then you compare all three versions on cost and on readability, and decide which one you would ship.

---

## 2. What You Will Learn

| Outcome | Step |
|:---|:---|
| Read an existing node graph and its generated HLSL | 2 |
| Return two results from one Custom node, emissive colour and an opacity mask | 3 |
| Prove a rewrite matches with a difference check | 3 |
| Replace an engine node with your own HLSL, and handle the differences | 4 |
| Compare node count, instructions and samplers, and judge readability | 5 |

---

## 3. Before You Start

- [ ] You have read sections 5.7 to 5.10 of Notes 10, because Step 5 asks you to apply its HLSL-or-nodes table.
- [ ] You remember how the dissolve works from Notes 05: noise is compared with a threshold that rises with `Dissolve`, pixels below it are clipped by **Opacity Mask**, and a thin band just above it glows.
- [ ] You can find `MM_Prop_Dissolve_Masked` in `/Game/GD/Notes05/Demo/Materials`.

---

## 4. Plan

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "15px", "lineColor": "#64748b", "edgeLabelBackground": "#ffffff"}}}%%
flowchart LR
    accTitle: Lab 18 plan
    accDescr: Copy the Week 4 dissolve and read its graph and generated code. Rewrite version one keeps the Noise node and moves the threshold and edge maths into a Custom node returning a float4 of emissive colour and opacity mask, checked against the original with a difference view. Rewrite version two replaces the Noise node with an fBm loop in a struct. Finally compare nodes, instructions and samplers across the three versions and judge readability.

    orig["<b>Week 4 dissolve</b><br/>nodes"]
    v1["<b>Rewrite 1</b><br/>Noise node + Custom"]
    v2["<b>Rewrite 2</b><br/>all HLSL"]
    cmp{{"Compare<br/>cost and readability"}}

    orig --> v1 --> v2 --> cmp

    classDef build fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0f172a
    classDef check fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#1f2937
    class orig,v1,v2 build
    class cmp check
```

| Step | Time |
|:---|:---|
| 1 - Set up | 4 min |
| 2 - Read the graph and its code | 4 min |
| 3 - Rewrite 1: the maths in code | 9 min |
| 4 - Rewrite 2: the noise in code | 8 min |
| 5 - Compare | 5 min |

---

## 5. Steps

### Step 1 - Set Up

1. Open `/Game/GD/Shared/Levels/Lvl_Lab`. Choose **File > Save Current Level As** and save it as `/Game/Student/Lab18/Maps/Lvl_Lab18`, then work in your copy, so the shared level stays unchanged for everyone, as the [lab folder conventions](../supplementary/unreal-lab-folder-conventions.md) explain.
2. Create `/Game/Student/Lab18/Materials`. Copy `MM_Prop_Dissolve_Masked` into it twice with **Copy Here**, and rename the copies `MM_Prop_DissolveHLSL_Masked` and `MM_Prop_DissolveFBMHLSL_Masked`.
3. Place three upright **Plane** shapes, **Rotation** X -90, scaled to 2.4. Apply the original from `/Game/GD/Notes05/Demo/Materials` to the first and your two copies to the others.
4. Create a material instance of each, set `Dissolve` to 0.5 in all three, and apply the instances instead, so you can compare them half-dissolved.

**Checkpoint:** Three identical steel planes, each half eaten away into ragged holes with glowing orange edges.

### Step 2 - Read the Graph and Its Code

1. Open `MM_Prop_DissolveHLSL_Masked`. Follow the graph from the `Noise` node to **Opacity Mask** and **Emissive Color**, and write the maths down as two lines: one for the mask and one for the glow.
2. Choose **Window > Shader Code > HLSL Code** and search for `1.2`. Find the generated lines for the threshold.
3. Note the node count and the **Stats** instruction and sampler counts.

**Checkpoint:** You have written `mask = noise - (Dissolve * 1.2 - 0.1)` and `glow = (1 - SmoothStep(0, EdgeWidth, mask)) * EdgeColour * EdgeStrength`, and recorded 23 nodes, about 288 instructions and 5 samplers.

### Step 3 - Rewrite 1: the Maths in Code

1. Add a **Custom** node, `Dissolve`, with **Output Type** `CMOT Float 4`, and inputs `Noise`, `Dissolve`, `EdgeWidth`, `EdgeColour` and `EdgeStrength`. Wire the existing `Noise` node and the four parameters into them.
2. Paste this code. RGB carries the glow and A carries the mask, so one node returns both. **Output Type:** `CMOT Float 4`. **Inputs:** `Noise` (float), `Dissolve` (float), `EdgeWidth` (float), `EdgeColour` (float3), `EdgeStrength` (float).

   ```hlsl
   float threshold = Dissolve * 1.2 - 0.1;
   float d = Noise - threshold;
   float edge = 1.0 - smoothstep(0.0, EdgeWidth, d);
   return float4(edge * EdgeColour * EdgeStrength, d);
   ```

3. Check the glow: `Abs` of the original glow node minus a `ComponentMask` RGB of the Custom node, times 100, into a temporary `Emissive Color`. Then check the mask the same way with the A channel. Apply each time, with `Dissolve` at 0.5 in the material's preview.
4. When both checks are black, wire `ComponentMask` RGB into `Emissive Color` and `ComponentMask` A into **Opacity Mask**, and delete the old maths nodes. Keep the `Noise` node, the parameters and the three steel textures. Apply and save.

**Checkpoint:** Both difference checks are black. After the swap, the second plane is identical to the first, and the graph has about 16 nodes with the same instruction count.

### Step 4 - Rewrite 2: the Noise in Code

1. Open `MM_Prop_DissolveFBMHLSL_Masked`. Build the Custom node as in Step 3, but delete the `Noise` node, its `Multiply`, its `AppendVector` and the constants feeding it, and add a `UV` input from a `TextureCoordinate` instead of `Noise`.
2. Paste this code, which makes the noise itself with the struct and loop from Lab 17. **Output Type:** `CMOT Float 4`. **Inputs:** `UV` (float2), `Dissolve` (float), `EdgeWidth` (float), `EdgeColour` (float3), `EdgeStrength` (float).

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

   float2 p = UV * 10.0;
   float noise = 0.0;
   float amp = 0.5;
   [unroll] for (int o = 0; o < 4; o++)
   {
       noise += N.Value(p) * amp;
       p *= 2.0;
       amp *= 0.5;
   }
   noise /= 0.9375;
   noise = saturate((noise - 0.5) * 2.0 + 0.5);

   float threshold = Dissolve * 1.2 - 0.1;
   float d = noise - threshold;
   float edge = 1.0 - smoothstep(0.0, EdgeWidth, d);
   return float4(edge * EdgeColour * EdgeStrength, d);
   ```

3. Wire RGB into `Emissive Color` and A into **Opacity Mask**. Delete the old maths nodes. Apply and save.
4. Compare the third plane with the first. Then comment out the line `noise = saturate((noise - 0.5) * 2.0 + 0.5);` by starting it with `//`, apply, and compare again. Put the line back.

**Checkpoint:** The third plane dissolves in the same way, with a different but similar pattern of holes. Without the contrast line, almost the whole plane disappears at 0.5. A masked material keeps only pixels whose mask is above its **Opacity Mask Clip Value**, 0.3333 by default, and fBm values bunch around 0.5, so very few pixels clear the threshold until the contrast is stretched.

### Step 5 - Compare

1. Fill in this table from **Stats** and the graphs.

| Version | Nodes | Instructions | Samplers | Lines of HLSL |
|:---|:---|:---|:---|:---|
| Original | | | | 0 |
| Rewrite 1 | | | | |
| Rewrite 2 | | | | |

2. Under the table, write three sentences: which version you would ship, which an artist could change most easily, and which you would choose if the same noise were needed in ten other materials.

**Checkpoint:** Your table shows about 288 instructions for the original and rewrite 1, about 283 for rewrite 2, one sampler fewer for rewrite 2 because the `Noise` node reads a texture internally, and far fewer nodes in each rewrite. Your sentences refer to the HLSL-or-nodes table in Notes 10 5.10. Reference solutions are in `/Game/GD/Notes10/Lab18/Materials`.

---

## 6. Debugging and Pitfalls

| Symptom | Likely cause | Fix |
|:---|:---|:---|
| The plane is fully solid at any `Dissolve` | A is not wired to **Opacity Mask**, or the material is not Masked | Wire the A mask, and check **Blend Mode** is `Masked` |
| The plane vanishes completely | The mask is inverted | `d = noise - threshold`, not the other way round |
| The glow covers the whole surface | `1.0 -` missing before the `smoothstep` | `edge = 1.0 - smoothstep(0.0, EdgeWidth, d)` |
| The difference check shows a faint pattern | The Custom node reads a different noise or a different UV scale | Wire the same `Noise` node that the original used |
| Rewrite 2 disappears too early | The contrast stretch was left out | Keep `noise = saturate((noise - 0.5) * 2.0 + 0.5);` |
| Rewrite 2 holes are huge | The noise scale is too low | Keep `UV * 10.0`, or raise it |
| Error about the struct | The `};` after the struct is missing | End the struct with `};` |

---

## 7. Production Checklist

| Check | Why |
|:---|:---|
| Both rewrites are in `Lab18/Materials`, and the original is unchanged | You can compare the three at any time |
| Rewrite 1 passed both difference checks | It is proven identical, not just similar |
| Glow and mask come from one node through RGB and A | One node, one place to read the effect |
| All four parameters are still parameter nodes | The Week 4 Blueprint and material instances keep working |
| Your comparison table and three sentences are saved with the lab | The decision is recorded, with the evidence |

---

## 8. Extension

1. **Shared noise.** In a C++ copy of the project with the `GDShaders` plugin from Notes 10 5.6, replace the struct in rewrite 2 with `/Plugin/GDShaders/GDNoise.ush` and one call to `GDFBM`.
2. **Drive it from Week 4.** Use rewrite 2 on `BP_DissolveProp` from `/Game/GD/Notes05/Demo/Blueprints` and check the Blueprint still dissolves it, since it only sets the `Dissolve` parameter.
3. **World-space dissolve.** Feed `AbsoluteWorldPosition` into rewrite 2 instead of UVs, divided by a scale, so the holes stay the same size on objects of any size.
4. **Directional burn.** Add the object's local height to the threshold so the dissolve starts at the top and burns downwards, with the noise still breaking up the edge.
5. **Debug view.** Add an Additional Output carrying `noise`, wire it through `DebugScalarValues`, and read the noise values at the burning edge.

---

## 9. Reflective Questions

1. Rewrite 1 cost exactly the same as the original. What did you gain from it, if anything?
2. Rewrite 2 removed a sampler but changed the look. When is a small change in look acceptable in exchange for cost, and who should decide?
3. The original's noise came from an engine node you cannot easily read. Is that a reason to prefer your own HLSL noise, or a reason to keep the node?
4. Across this block you have built effects as nodes, as HLSL and as a mix. What will you reach for first on your next project, and why?

---

## 10. How This Builds

- This is the final lab of the block. It uses the dissolve from Notes 05, the noise from Notes 04, the Custom node from Notes 09 and the loops and helpers from Notes 10.
- The judgement in Step 5 is the one you will make on every shader you write: build it in the way that is cheapest to run, change and understand, and measure rather than assume.
