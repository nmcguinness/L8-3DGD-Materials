---
title: "Materials in Unreal Engine 5.7"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-10-23"
engine_version: "UE 5.7"
---

# Materials in Unreal Engine 5.7

A seven-week block on building materials in Unreal Engine 5.7. Weeks 1-5 cover node-based materials in the Material Editor. Weeks 6-7 cover writing HLSL in Custom nodes and external shader files.

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
| `notes/` | Weekly notes, `01_` to `07_` |
| `labs/` | Labs, `lab_1_` to `lab_14_` |
| `assets/images/` | Screenshots and diagrams used by the notes and labs |
| `assets/hlsl/` | `.ush` helper files and HLSL snippets for Weeks 6-7 |
| `supplementary/` | Reference documents that support the notes and labs (see section 4) |
| `interactive/` | Browser-based demos linked from the notes (see section 4) |
| `unreal/` | The UE 5.7 demo project used by every demo and lab |

---

## 3. Contents

Documents are published week by week, so links to later weeks resolve once each week is released.

| Nr | Topic | Notes | Lab / Exercises |
|:---|:---|:---|:---|
| 1 | PBR Foundations and the Material Editor | [01_pbr_and_material_editor](notes/01_pbr_and_material_editor.md) | [lab_1_pbr_surface_materials](labs/lab_1_pbr_surface_materials.md)<br>[lab_2_blend_modes_and_shading_models](labs/lab_2_blend_modes_and_shading_models.md) |
| 2 | Reuse and Layering | [02_master_materials_and_layering](notes/02_master_materials_and_layering.md)<br>Extra reading: [02_texture_packing_and_spherical_coordinates](notes/02_texture_packing_and_spherical_coordinates.md) | [lab_3_master_material_instances](labs/lab_3_master_material_instances.md)<br>[lab_4_material_functions_and_blending](labs/lab_4_material_functions_and_blending.md) |
| 3 | Runtime Materials, Screen-Space and Performance | [03_runtime_materials_and_performance](notes/03_runtime_materials_and_performance.md) | [lab_5_runtime_driven_effects](labs/lab_5_runtime_driven_effects.md)<br>[lab_6_post_process_decals_and_profiling](labs/lab_6_post_process_decals_and_profiling.md) |
| 4 | Geometry and Scene-Aware Materials | [04_geometry_and_scene_aware_materials](notes/04_geometry_and_scene_aware_materials.md) | [lab_7_world_position_offset_and_parallax](labs/lab_7_world_position_offset_and_parallax.md)<br>[lab_8_depth_effects_and_flipbooks](labs/lab_8_depth_effects_and_flipbooks.md) |
| 5 | Composite Effects | [05_composite_effects](notes/05_composite_effects.md) | [lab_9_water_and_hologram](labs/lab_9_water_and_hologram.md)<br>[lab_10_toon_shading_and_landscape](labs/lab_10_toon_shading_and_landscape.md) |
| 6 | HLSL Fundamentals via the Custom Node | [06_hlsl_fundamentals](notes/06_hlsl_fundamentals.md) | [lab_11_custom_node_translation](labs/lab_11_custom_node_translation.md)<br>[lab_12_procedural_patterns](labs/lab_12_procedural_patterns.md) |
| 7 | Applied HLSL | [07_applied_hlsl](notes/07_applied_hlsl.md) | [lab_13_loops_sampling_and_pom](labs/lab_13_loops_sampling_and_pom.md)<br>[lab_14_hlsl_dissolve_rewrite](labs/lab_14_hlsl_dissolve_rewrite.md) |

---

## 4. Supplementary Material

These resources sit outside the weekly sequence. Use them alongside the notes when a topic needs more depth than the class time allows.

| Resource | Type | Use it for |
|:---|:---|:---|
| [Unreal asset naming conventions](supplementary/unreal-naming-conventions.md) | Reference | Naming every asset you create in the demos and labs, so others can identify its type and purpose without opening it |
| [Noise in Unreal materials](supplementary/unreal-material-noise.md) | Reading | Choosing between blue, Perlin, Simplex, Voronoi and fBm noise, and deciding when to bake noise into a texture |
| [Spherical Coordinates Explorer](interactive/spherical_coordinates_explorer.html) | Interactive demo | Seeing spherical to Cartesian conversion, lat-long mapping and two-channel normal packing. Used in the Week 2 extra reading |
| [Texel Path Explorer](interactive/texel_path_explorer.html) | Interactive demo | Following a texture sample through the GPU cache to video memory. Used in the Week 2 extra reading |

GitHub shows the source of an HTML file rather than running it, so open the interactive demos in a browser from your local copy of the repository.

---

## 5. Getting Help

1. Ask during the lab, because most material problems are faster to diagnose with the graph on screen.
2. Check the lab's Debugging and Pitfalls table and the notes' Common Pitfalls table. They cover the failures seen most often.
3. Post on the Moodle forum. Include a screenshot of the graph and of the result, so others can reproduce the problem.
4. Report errors in the notes or labs on the Moodle forum, naming the file and section.

---
