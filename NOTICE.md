# Attribution and Sources

This repository is a **third-party reconstruction study of Microduck**, not affiliated with or endorsed by Pollen Robotics.

## Upstream Sources

| Source | Author | License |
|---|---|---|
| [pollen-robotics/microduck_rl](https://github.com/pollen-robotics/microduck_rl) | Pollen Robotics | Code: Apache-2.0; **3D models: CC BY-NC-SA** (upstream README writes "BY-SA-NC") |
| [pollen-robotics/microduck](https://github.com/pollen-robotics/microduck) | Pollen Robotics | Apache-2.0 |

Microduck is a commercial product by Pollen Robotics with **partly open** hardware:

- **The RPI Robot HAT board is fully published by Pollen** ([`elec_RPI_Robot_HAT`](https://github.com/pollen-robotics/elec_RPI_Robot_HAT), Apache-2.0, with KiCad project and production files)
- **The `imu_to_dxl` board, editable mechanical CAD, whole-robot BOM and assembly documentation** are not published

All geometry in this repository comes from the **MJCF simulation model and STL meshes** publicly
released in the upstream `microduck_rl` repository; electronics conclusions come from the source code,
device tree and configuration files in the `microduck` repository.

> Correction (2026-09-03): this file previously stated "its hardware is not open source" — that statement was incorrect. The HAT board is open source.

## Derivative Content in This Repository

The following content is generated or organized from the publicly released upstream MJCF + STL, and is
considered a **derivative work under CC BY-NC-SA**, and is therefore released under the same license:

| Path | Contents | Relationship to upstream |
|---|---|---|
| `assembly-drawings/` | All rendered views and exploded diagrams | Rendered from upstream MJCF + STL |
| `cad/` | "Assembled" STLs with world transforms applied | Merged from upstream STL by kinematic hierarchy |
| `print/` | 46 individual part STLs | **Direct redistribution** of upstream STL, renamed and categorized |
| `docs/hole_analysis.json` | Hole geometry analysis data | Computed from upstream STL |

> ⚠️ **About `print/`**: this repository's principle is "don't rehost upstream code", but `print/` is a
> **deliberate exception** — printable parts are what replica builders need most, and they need to be able
> to click through and view them individually on the web page. CC BY-NC-SA explicitly permits this kind of
> redistribution, conditional on attribution, share-alike, and non-commercial use; this section serves as
> that declaration. Upstream baseline: `pollen-robotics/microduck_rl` @ `2fa62b8` (2026-07-28, `assets/`
> at the time of fetch).

**Original content from this project:**

| Path | Contents | License |
|---|---|---|
| `scripts/` | Rendering, export, hole analysis scripts | **Apache-2.0** |
| `tools/stl_viewer.html` | Zero-dependency WebGL STL viewer | **Apache-2.0** |
| `docs/*.md`, `*.md` | All documentation and reverse-engineering analysis text | **CC BY-NC-SA 4.0** |
| `build-log/` photos | Physical build photos | **CC BY-NC-SA 4.0**, photographed and authorized by project participants |
| `assets/` | Cover images and community group QR codes | Same as above |

> Upstream source code snippets quoted in the documentation (comments, constants, register definitions)
> come from `pollen-robotics/microduck` and follow its **Apache-2.0** license; citations include file paths.

## License Name Clarification

The canonical name is **CC BY-NC-SA 4.0** (Attribution - NonCommercial - ShareAlike).
The upstream README writes "BY-SA-NC"; when this repository uses that spelling, it refers to the same license.

> It should be noted: the upstream `microduck_rl` repository's `LICENSE` file itself only contains Apache-2.0,
> and **the CC clause appears only in one line of text in its README**. This repository's use of the stricter
> CC BY-NC-SA for 3D models is the conservative approach — if upstream clarifies it as pure Apache-2.0, this
> repository will relax accordingly.

## Compliance Statement

All conclusions in this repository come from:

- **Source code and device tree** publicly released by Pollen Robotics (`pollen-robotics/microduck`, Apache-2.0)
- **Simulation model and meshes** publicly released (`pollen-robotics/microduck_rl`)
- **KiCad project** publicly released (`pollen-robotics/elec_RPI_Robot_HAT`, Apache-2.0)
- Device manuals and specification pages publicly available from manufacturers

**No non-public materials were used, no physical units were disassembled, no unpublished design files were accessed.**
The firmware for the `imu_to_dxl` board is not in any open source repository; this repository only reconstructs
its observable behavior on the bus side.

## Non-Commercial Statement

The upstream 3D models use **CC BY-NC-SA**, and derivative works **must not be used for commercial purposes**.
This repository is for learning, research and personal replication only.

## How to Attribute

> Based on Pollen Robotics' Microduck model (CC BY-NC-SA)
> https://github.com/pollen-robotics/microduck_rl
