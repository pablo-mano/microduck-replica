# Project Progress

**Project**: Microduck Replica
**Started**: 2026-08-28
**Last updated**: 2026-09-03 (hardware open-source status correction: HAT board officially published)
**Repository**: https://github.com/fanhao375/microduck-replica (public)

---

## Goals

Replicate Pollen Robotics' **Microduck** (25cm bipedal robot duck, $399, shipping Christmas 2026).
Official software is open source; hardware is **partly open** — **the HAT board has complete KiCad project
and production files** ([`elec_RPI_Robot_HAT`](https://github.com/pollen-robotics/elec_RPI_Robot_HAT), Apache-2.0),
but the `imu_to_dxl` board, editable mechanical CAD, whole-robot BOM and assembly documentation are not published.

> **Correction 2026-09-03**: previously recorded as "hardware not open source, no PCB schematics", **this
> judgment was wrong**. The reason was only searching the `microduck` main repo, missing the `elec_`-prefixed
> hardware repositories under the same organization.

This project reverse-engineers all information needed for mechanical replication from the officially
published **MJCF simulation model + 47 STL files**.

---

## Overall Status

| Component | Status | Notes |
|---|---|---|
| Part geometry | ✅ Complete | 47 STLs |
| Assembly relationships | ✅ Complete | Accurate to 0.1mm, exploded views produced |
| CAD assemblies | ✅ Complete | World transforms applied, ready to import |
| Joint parameters | ✅ Complete | Axes and travel for all 14 controlled joints |
| Mass and inertia | ✅ Complete | All 15 rigid bodies |
| Fastener list | ✅ Complete | Reverse-engineered from hole features, M2 system |
| Actuator selection | ✅ Complete | XL330 ×15, BAM M6 parameters |
| Bearing specs | ✅ Complete | Ø22×16×4, Ø15×10×3 |
| **Electronics** | ✅ Complete | Fully recovered from Rust source, see "Batch 6" |
| Main board selection | ✅ Complete | **Radxa Zero 3W commercial module**, not custom carrier |
| **HAT board** | ✅ Officially published | KiCad + Gerbers + BOM + pick-and-place, ready to fab (4-layer) |
| **`imu_to_dxl` board** | ✅ Self-drawn (not fabricated) | No public project exists anywhere; chip/address/protocol fully recovered, this repo provides schematic + PCB |
| **Cable routing** | ❌ Missing | No official documentation |
| **Control software** | ✅ Available | Same main board → official Rust runtime runs directly; only needs porting if changing main board |
| **Policy retraining** | ✅ Not required | Same hardware → official 9 ONNX policies work directly; only need retraining if changing body/electronics |

---

## Completed

### Batch 1 · Initial Research

- ~~Confirmed **Microduck hardware is not open source**~~: `microduck` repo is all Rust software,
  all 5 subdirectories under `docs/` are software docs, searching the entire repo finds no
  `.stl/.step/.f3d/bom` files — **this conclusion was wrong** (corrected 2026-09-03). Only
  checked the main repo, missed the `elec_`-prefixed hardware repos under the same organization;
  **the HAT board in [`elec_RPI_Robot_HAT`](https://github.com/pollen-robotics/elec_RPI_Robot_HAT)
  is fully open source** (Apache-2.0, with KiCad + Gerbers + BOM + pick-and-place).
  Lesson: when judging "whether a project has open sourced some part", must search **the entire
  organization**, not just the main repo.
- Confirmed **`microduck_rl` has 47 STLs + complete MJCF**, this is the only geometry source
- Confirmed **3D model license is CC BY-NC-SA** (non-commercial), code is Apache-2.0
- Ruled out the Open Duck Mini v2 route (see "Decision Log")

### Batch 2 · Assembly Drawings

- Used MuJoCo offscreen rendering, white skybox, 1600×2100
- 4 standard views + 2 exploded views + 1 color-coded reference view
- Exploded views offset progressively along kinematic chain (48mm per level), farther from trunk = farther exploded
- Implemented world coordinate → pixel projection for annotation leaders and layout collision avoidance
- Output: `assembly-drawings/` (7 images), script `scripts/render_assembly.py`

### Batch 3 · CAD Assemblies

- Upstream 47 STLs are all in **part local coordinate frames**, importing directly into CAD piles
  everything at the origin
- Extracted world transform for each geom from MJCF, applied and exported grouped by rigid body
- Output: `cad/` — whole robot single file (796,792 triangles) + 15 parts, units mm
- Measured whole robot envelope **144 × 141 × 264 mm** (matches official 25cm claim)
- Script `scripts/export_assembly_stl.py`

### Batch 4 · Fastener Reconstruction

- Wrote hole feature recognition: weld vertices → build face adjacency graph → split smooth surface
  patches at 35° dihedral angle → fit cylinder to each patch (axis = normal covariance minimum
  eigenvector) → project and fit circle for diameter → classify hole/boss by normal direction
- Scanned 47 parts in 2.5 seconds
- Output: `docs/fastener-reconstruction.md`, `docs/hole_analysis.json`, `scripts/analyze_holes.py`

### Batch 10 · English Documentation

- Background: when listed in awesome lists, specifically marked "In Chinese"; traffic data shows
  Google only 5 times while Bing 132, English community barely entered
- Prioritized by access count, translated `hardware-teardown.md` first (2nd most visited on site, 84 times),
  because it is **globally unique content** — the 3D anatomy page by HF staff @mishig25 (36k views)
  already covers "what the whole robot looks like", spec-type docs overlap with that, but chip
  addresses, bus protocols, 12-byte data block layout are only here
- Output: `docs/hardware-teardown.en.md` (426 lines) + `docs/actuator-selection.en.md` (459 lines),
  both cross-linked with Chinese versions
- English README's documentation index changed to use 🇬🇧 to mark which have English versions

### Batch 9 · Actuator Selection Analysis

- Answers two frequently asked questions: why servos not closed-loop steppers; can we swap to cheaper STS3215
  without changing mechanics
- **Hard data comparison**: same 15-body bipedal duck, XL330 version **737.2 g** vs STS3215 version **2107.1 g**,
  2.86× difference — Open Duck Mini v2 is "the STS3215 version answer" (42cm / 2.1kg)
- Full derivation with seven citations proving torque/speed/inertia constraints
- Cross-comparison of same-class servos, including deep assessment of Unitree S288
- Output: `docs/actuator-selection.en.md` (459 lines English + 481 lines Chinese)

### Batch 8 · Real Build Started

- Someone is actually building it: head shell, trunk shell, leg structure parts, feet printed
- **First real-world verification point**: black leg parts have **M2 screws already fitted** — the "whole
  robot is M2 system" conclusion from [Fastener Reconstruction](docs/fastener-reconstruction.md) holds on
  physical hardware
- Photo: `build-log/photos/2026-09-02-first-printed-parts.jpg`
- `BUILD-LOG.md` started, tracking print settings, assembly problems, dimensional verification

### Batch 7 · `imu_to_dxl` PCB

- Schematic validated → routed → DRC clean → fabrication outputs generated
- **45 × 22 mm, 2-layer**, R2 corner radius, two diagonal M2 mounting holes, bottom layer solid GND
- All 23 nets connected, every IC power pin has 100nF decoupling within 2.3mm
- Still **not fabricated, not validated on hardware**
- Output: complete KiCad project, schematic PDF, PCB PDF, 3D STEP, Gerbers, BOM, pick-and-place
- `hardware/imu_to_dxl/README.md` added design notes, review points, audit checklist

### Batch 6 · Full Electronics Stack Recovered

From runtime source code, fully recovered:

1. **Main board confirmed: Radxa Zero 3W** — device tree `compatible = "radxa,zero-3w"`, commercial
   off-the-shelf module, **not** a custom carrier
2. **Confirmed camera: Raspberry Pi Camera v2 (IMX219)** — overlay name explicitly mentioned
3. **Servo bus confirmed: `/dev/ttyS2`, 1 Mbps, TTL half-duplex** — not RS-485
4. **`imu_to_dxl` protocol fully recovered**: LSM6DSV16X, bus ID 200, register 124, 12-byte block,
   same `sync_read` as servos
5. HAT board I²C devices: TLV320AIC3104 @ 0x18, BMI088 (dormant), VL53L5CX/L8CX ToF @ 0x29/0x52
6. Three UART collision pitfalls: `serial-getty@ttyS2` blocks the servo bus, I²C3 collides with FUSB302,
   NPU ships disabled in Armbian
7. Battery: Sony NP-F550 (2S, 7.4V), **no fuel gauge, no ADC** — voltage read from servo reports

Output: `docs/hardware-teardown.md` (Chinese, 380 lines) + 7-diagram hardware atlas PDF

**Conclusion revised** from "mechanics copyable, electronics are a wall" to **"whole robot is reproducible"** —
main board is off-the-shelf, custom board function and protocol fully recovered.

### Batch 5 · Community Engagement

- WeChat group established, grew to 200 members in 72 hours, second group opened
- Independent verification: [@tspy](https://x.com/tspy/status/2094249218735300630) published hardware
  teardown on X (169 likes) matching this repo's conclusions exactly
- Added traffic monitoring: 36 countries/regions, 1,700+ pageviews in first week
- `docs/community-updates.md` added to track X/GitHub signals

---

## Current Work

- [ ] Translate remaining Chinese docs to English (in progress)
- [ ] Validate `imu_to_dxl` PCB on hardware (waiting for fabrication)
- [ ] Complete build log with dimensional measurements

---

## Decision Log

### Why Not Open Duck Mini v2?

- Open Duck Mini v2: 42cm tall, **2.1 kg**, STS3215 servos (larger, heavier, cheaper)
- Microduck: 25cm tall, **737 g**, XL330 servos (smaller, lighter, more expensive)
- **2.86× mass difference** from servo choice alone — not the same robot at different scales,
  fundamentally different designs
- Open Duck Mini v2 is great for budget builds and larger format; Microduck optimizes for
  compactness and low inertia
- Both are valid; chose Microduck because upstream published MJCF + STL makes geometry
  reconstruction feasible

### Why Translate Documentation?

- Traffic data shows English community hasn't arrived (Google 5× vs Bing 132×)
- Hardware teardown content is globally unique
- Awesome lists mark "In Chinese", limiting discoverability
- Goal: make unique technical content accessible, not translate everything

---

## Open Items

- [ ] English translation: `docs/hardware-primer.md`, `docs/hardware-spec.md`
- [ ] Cable routing documentation (no official source exists)
- [ ] Second printing batch verification
- [ ] Assembly instructions with photos

---

## Metrics

| Metric | Value (as of 2026-09-03) |
|---|---|
| Repository stars | (check GitHub) |
| WeChat group members | 200 + 200 (groups 1 & 2 full) |
| Pageviews (first week) | 1,700+ |
| Countries/regions reached | 36 |
| Independent verifications | 1 ([@tspy hardware teardown](https://x.com/tspy/status/2094249218735300630)) |
