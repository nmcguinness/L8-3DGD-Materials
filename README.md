---
title: "Materials in Unreal Engine 5.7"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-27"
engine_version: "UE 5.7"
---

# Materials in Unreal Engine 5.7

A two-block set of notes on building materials in Unreal Engine. Block A (weeks 1 to 7) covers node-based materials in the Material Editor. Block B (weeks 8 and 9) covers writing HLSL in Custom nodes and external shader files.

---

## 1. Learning Outcomes

On completing the block, you will be able to:

1. Build physically based materials from texture sets, applying the correct colour space, compression and channel packing.
2. Choose blend modes, shading models and material domains to suit a given surface or effect.
3. Structure materials for reuse using master materials, instances, material functions and layered blending.
4. Drive material parameters at runtime from Blueprint, per instance and globally.
5. Build vertex, depth-aware, post-process and composite effects.
6. Profile material cost and optimise expensive materials.
7. Write, debug and organise HLSL in Custom nodes and external `.ush` files, and justify when HLSL is the better choice over nodes.

---

## 2. Repo Structure

| Path | Contents |
|:---|:---|
| `notes/` | Notes, `nn_` |
| `labs/` | Labs, `lab_1_` to `lab_18_` |
| `labs/HLSL/` | Short HLSL practice labs, `Lab1_` to `Lab13_` (see section 4) |
| `assets/images/` | Screenshots and diagrams used by the notes and labs |
| `assets/hlsl/` | `.ush` helper files and HLSL snippets for Block B |
| `supplementary/` | Reference documents that support the notes and labs (see section 4) |
| `interactive/` | Browser-based demos linked from the notes (see section 4) |
| `unreal/` | The UE demo project used by every demo and lab |

---

## 3. Contents

Documents are published week by week, so links to later weeks resolve once each week is released.

| Nr | Topic | Notes | Lab / Exercises |
|:---|:---|:---|:---|
| 1 | PBR Foundations and the Material Editor | [01_pbr_and_material_editor](notes/01_pbr_and_material_editor.md) | [lab_1_pbr_surface_materials](labs/lab_1_pbr_surface_materials.md)<br>[lab_2_blend_modes_and_shading_models](labs/lab_2_blend_modes_and_shading_models.md) |
| 2 | Reuse and Layering | [03_master_materials_and_layering](notes/03_master_materials_and_layering.md)<br>Extra reading: [02_texture_packing_and_spherical_coordinates](notes/02_texture_packing_and_spherical_coordinates.md) | [lab_3_master_material_instances](labs/lab_3_master_material_instances.md)<br>[lab_4_material_functions_and_blending](labs/lab_4_material_functions_and_blending.md) |
| 3 | Procedural Textures and UV Maths | [04_procedural_textures_and_uv_maths](notes/04_procedural_textures_and_uv_maths.md) | [lab_5_procedural_tiles_and_bricks](labs/lab_5_procedural_tiles_and_bricks.md)<br>[lab_6_procedural_wood](labs/lab_6_procedural_wood.md) |
| 4 | Runtime Materials, Screen-Space and Performance | [05_runtime_materials_and_performance](notes/05_runtime_materials_and_performance.md) | [lab_7_runtime_driven_effects](labs/lab_7_runtime_driven_effects.md)<br>[lab_8_post_process_decals_and_profiling](labs/lab_8_post_process_decals_and_profiling.md) |
| 5 | Geometry and Scene-Aware Materials | [06_geometry_and_scene_aware_materials](notes/06_geometry_and_scene_aware_materials.md) | [lab_9_world_position_offset_and_parallax](labs/lab_9_world_position_offset_and_parallax.md)<br>[lab_10_depth_effects_and_flipbooks](labs/lab_10_depth_effects_and_flipbooks.md) |
| 6 | Composite Effects | [07_composite_effects](notes/07_composite_effects.md) | [lab_11_water_and_hologram](labs/lab_11_water_and_hologram.md)<br>[lab_12_toon_shading_and_landscape](labs/lab_12_toon_shading_and_landscape.md) |
| 7 | Screen-Space Post-Processing | [08_screen_space_post_processing](notes/08_screen_space_post_processing.md) | [lab_13_screen_filters](labs/lab_13_screen_filters.md)<br>[lab_14_scanner_pulse_and_selective_colour](labs/lab_14_scanner_pulse_and_selective_colour.md) |
| 8 | HLSL Fundamentals via the Custom Node | [09_hlsl_fundamentals](notes/09_hlsl_fundamentals.md) | [lab_15_custom_node_translation](labs/lab_15_custom_node_translation.md)<br>[lab_16_procedural_patterns](labs/lab_16_procedural_patterns.md) |
| 9 | Applied HLSL | [10_applied_hlsl](notes/10_applied_hlsl.md) | [lab_17_loops_sampling_and_pom](labs/lab_17_loops_sampling_and_pom.md)<br>[lab_18_hlsl_dissolve_rewrite](labs/lab_18_hlsl_dissolve_rewrite.md) |

---

## 4. Supplementary Material

These resources sit outside the weekly sequence. Use them alongside the notes when a topic needs more depth than the class time allows.

| Resource | Type | Use it for |
|:---|:---|:---|
| [Unreal asset naming conventions](supplementary/unreal-naming-conventions.md) | Reference | Naming every asset you create in the demos and labs, so others can identify its type and purpose without opening it |
| [Unreal lab folder conventions](supplementary/unreal-lab-folder-conventions.md) | Reference | Setting up the folders for each lab, so every asset has one predictable location |
| [Noise in Unreal materials](supplementary/unreal-material-noise.md) | Reading | Choosing between blue, Perlin, Simplex, Voronoi and fBm noise, and deciding when to bake noise into a texture |
| [Material node glossary](supplementary/unreal-material-node-glossary.md) | Reference | Looking up any material node used in the node-based notes and labs: what it does, its pins and where it is typically used |
| [Spherical Coordinates Explorer](interactive/spherical_coordinates_explorer.html) | Interactive demo | Seeing spherical to Cartesian conversion, lat-long mapping and two-channel normal packing. Used in the Week 2 extra reading |
| [Texel Path Explorer](interactive/texel_path_explorer.html) | Interactive demo | Following a texture sample through the GPU cache to video memory. Used in the Week 2 extra reading |
| [Blend Modes Explorer](interactive/blend_modes_explorer.html) | Interactive demo | Comparing the blend modes in Unreal and Unity: the equation behind each one, how each treats the same texture, and which ones depend on draw order |
| [Texture Filtering Demo](interactive/texture_filtering_demo.html) | Interactive demo | Comparing nearest, bilinear, trilinear and anisotropic filtering on a receding plane, with mip-level tinting, a texel inspector and the texel reads each mode costs |
| [Noise Types and Octaves Demo](interactive/noise_types_demo.html) | Interactive demo | Comparing white, pink, brown, blue and violet noise by their power spectra, and building fBm from octaves with lacunarity, gain, turbulence and ridged variants |

GitHub shows the source of an HTML file rather than running it, so open the interactive demos in a browser from your local copy of the repository.

### 4.1 HLSL Practice Labs

Thirteen short labs in `labs/HLSL/` build HLSL skills in small steps, each adding one new idea to the last. Use them as extra practice alongside Weeks 8 and 9, in this order.

| Lab | Effect | New skill | Use after |
|:---|:---|:---|:---|
| [Lab 1](labs/HLSL/Lab1_CustomNodesHLSL.md) | Polar UVs | A first Custom node: inputs, output type, `atan2` and `length` | Notes 09, Session A |
| [Lab 2](labs/HLSL/Lab2_Vignette.md) | Vignette | A first post-process Custom node: buffer and viewport UVs, `SceneTextureLookup` | Notes 09, Session A |
| [Lab 3](labs/HLSL/Lab3_Pixelation.md) | Pixelation | Snapping UVs to a grid with `floor` | Notes 09, Session A |
| [Lab 4](labs/HLSL/Lab4_ChromaticAberration.md) | Chromatic aberration | Three samples at different UVs, one channel from each | Notes 09, Session A |
| [Lab 5](labs/HLSL/Lab5_HeatDistortion_Simple.md) | Heat distortion | Returning UVs instead of colour, animated with time | Notes 09, Session A |
| [Lab 6](labs/HLSL/Lab6_LensDistortion.md) | Lens distortion | A polynomial model applied to centred UVs | Notes 09, Session A |
| [Lab 7](labs/HLSL/Lab7_DepthFog.md) | Depth fog | Combining scene colour and depth from SceneTexture nodes | Notes 09, Session A |
| [Lab 8](labs/HLSL/Lab8_ProceduralPatterns_Waves.md) | Procedural patterns | Stripes, checkers, tiled shapes and a loop of wave rows | Notes 09, Session B |
| [Lab 9](labs/HLSL/Lab9_SobelEdgeDetection.md) | Sobel edges | A fixed loop over nine neighbouring pixels | Notes 10, Session A |
| [Lab 10](labs/HLSL/Lab10_RainDrops_HLSL.md) | Raindrops | Hashing cell IDs and looping over neighbouring cells | Notes 10, Session A |
| [Lab 11](labs/HLSL/Lab11_ProceduralNoise_fBm.md) | Noise and fBm | Helper functions in a struct, and an octave loop | Notes 10, Session A |
| [Lab 12](labs/HLSL/Lab12_TriplanarMapping.md) | Triplanar mapping | Sampling a texture inside a Custom node from three projections | Notes 10, Session A |
| [Lab 13](labs/HLSL/Lab13_ToonShading.md) | Toon shading | Derivative edges, banded lighting and a halftone overlay. The most open-ended lab | Notes 10, Session B |

---

## 5. Getting Help

1. Ask during the lab, because most material problems are faster to diagnose with the graph on screen.
2. Check the lab's Debugging and Pitfalls table and the notes' Common Pitfalls table. They cover the failures seen most often.
3. Post on the Moodle forum. Include a screenshot of the graph and of the result, so others can reproduce the problem.
4. Report errors in the notes or labs on the Moodle forum, naming the file and section.
