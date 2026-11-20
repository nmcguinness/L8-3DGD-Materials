---
title: "Unreal Lab Folder Conventions"
module-code: "COMP I8015"
module-title: "3D Game Development"
programme: "BSc (Hons) in Computing in Games Development"
stage: 4
institution: "DkIT"
author: "Niall McGuinness"
academic-year: "2026-27"
last-updated: "2026-11-19"
doc-type: supplementary
topic_code: MAT-S2
difficulty_tier: foundation
duration: "Reference, about 5 min reading"
prerequisites: []
feeds_into: [lab_1_pbr_surface_materials]
engine_version: "UE 5.7"
---

# Unreal Lab Folder Conventions

Use this folder structure for every lab. Each lab then has one predictable home, so you can find your work, and anyone helping you can find it too.

This document covers where assets go. For what to call them, see [Unreal asset naming conventions](unreal-naming-conventions.md).

---

## 1. Where Your Work Goes

All of your work goes inside `/Game/Student`. Everything under `/Game/GD` belongs to the module: the shared textures, the shared lab level, the demo materials and the lab solutions. You read from `/Game/GD` but never change anything in it.

| Work | Folder |
|:---|:---|
| A lab | `/Game/Student/LabNN`, for example `/Game/Student/Lab03` |
| Following a notes Worked Example | `/Game/Student/NotesNN/MyWork`, for example `/Game/Student/Notes04/MyWork` |

- `NN` is the number written with two digits, so Lab 3 uses `Lab03` and Lab 10 uses `Lab10`. Two digits keep the folders in order, because a sorted list would otherwise place `Lab10` before `Lab2`.
- `/Game/Student/Lab03` is **Content > Student > Lab03** in the Content Browser and `unreal/Content/Student/Lab03` on disk.
- Keeping notes work and lab work in separate folders means a demo you follow along with in class never mixes with the lab you submit.

---

## 2. Folders for Lab N

Every lab needs the two required folders. Add an optional folder only when the lab produces an asset that belongs in it, because empty folders add clutter and Git does not store them.

```text
Content/
  Student/
    LabNN/
      Maps/                  required
      Materials/             required
        Functions/           optional
      Textures/              optional
      Meshes/                optional
      Blueprints/            optional
```

| Folder | Status | Holds | Prefixes |
|:---|:---|:---|:---|
| `Maps` | Required | Your copy of the lab level | `Lvl_` |
| `Materials` | Required | Materials, material instances, post process materials and Material Parameter Collections | `MM_`, `MI_`, `PPM_`, `MPC_` |
| `Materials/Functions` | Optional | Material Functions | `MF_` |
| `Textures` | Optional | Textures you import, or copy before changing, and render targets | `T_`, `RT_` |
| `Meshes` | Optional | Meshes you import or copy for this lab | `SM_` |
| `Blueprints` | Optional | Blueprints that drive or demonstrate an effect | `BP_` |

Materials and their instances share one folder. Every material is named for what it makes, so no two share a name and none needs a folder of its own. **Create Material Instance** also places a new instance beside its parent, so keeping them together avoids moving assets later.

If a lab produces an asset type that is not listed, add a folder named for that type directly inside `LabNN`, using PascalCase with no spaces. Spaces and special characters are error-prone in asset paths.

---

## 3. Rules

| Rule | Reason |
|:---|:---|
| Keep every asset you make for Lab N inside `LabNN` | You can review, copy or remove one lab without searching the whole project |
| Never edit assets outside `/Game/Student` | Other work depends on shared content, so one change can alter materials you did not touch |
| Save your own copy of the lab level before you place anything | Every lab opens the shared `Lvl_Lab`, and changes saved to it would appear in everyone's next lab |
| Refer to shared textures and meshes where they are, without copying them | A second copy uses extra disk space and can drift from the original |
| Copy a shared asset into `LabNN/Textures` or `LabNN/Meshes` before you change it | Settings such as sRGB belong to the asset, so changing the shared asset affects every material that samples it |
| When a lab builds on an earlier lab, copy the earlier asset into the new `LabNN` | The earlier lab stays as you finished it, so a mistake in Lab 3 cannot break your Lab 1 result |
| Name every material for what it makes, as the naming conventions describe | The name tells you what the asset is in any search, and the lab folder already tells you which lab it belongs to |
| Create folders before you create assets | Moving an asset later leaves a redirector behind, which can cause broken references (see [Common Pitfalls](unreal-naming-conventions.md#11-common-pitfalls)) |

---

## 4. Setting Up Lab N

Do this at the start of each lab, before you create any assets.

1. In the Content Browser, go to **Content > Student**. If `Student` does not exist yet, right-click **Content**, choose **New Folder** and name it `Student`.
2. Right-click `Student`, choose **New Folder** and name it `LabNN`.
3. Inside `LabNN`, create the `Maps` and `Materials` folders.
4. Open `/Game/GD/Shared/Levels/Lvl_Lab`, then choose **File > Save Current Level As** and save it as `/Game/Student/LabNN/Maps/Lvl_LabNN`. Work in your copy from now on.
5. To reuse an earlier lab's asset, drag it onto the new folder and choose **Copy Here**.

**Checkpoint:** The Content Browser shows `Student/LabNN` with `Maps` and `Materials` inside it, `Maps` contains `Lvl_LabNN`, and the level tab shows `Lvl_LabNN` as the open level.

---

## 5. Example: Lab 3

Lab 3 builds a master material and five instances, using the shared wood and steel textures. It has no Material Functions and imports nothing, so it uses only the two required folders.

```text
Content/
  Student/
    Lab03/
      Maps/
        Lvl_Lab03
      Materials/
        MI_Prop_HeatedSteel_Opaque
        MI_Prop_PolishedSteel_Opaque
        MI_Prop_StainedWood_Opaque
        MI_Prop_Steel_Opaque
        MI_Prop_Wood_Opaque
        MM_Prop_PBRMaster_Opaque
```

---

## 6. Checklist

Check these points before you finish each lab.

- [ ] The lab folder is `/Game/Student/LabNN`, with a two-digit number.
- [ ] `Maps` contains your copy of the lab level, named `Lvl_LabNN`.
- [ ] Every asset you made for the lab is inside `LabNN`, in the folder that matches its prefix.
- [ ] Every material and instance has its own name, and no folder or asset name contains a space.
- [ ] You did not change any asset outside `/Game/Student`, including `Lvl_Lab`.
- [ ] You did not change any earlier lab's assets.
- [ ] All assets are saved. Use **File > Save All**, because an unsaved asset is lost when the editor closes.
