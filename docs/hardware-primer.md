# Hardware Primer: Board by Board Explanation

> This document answers one question: **How many boards are actually in the Microduck, and what does each one do?**
>
> Division of labor with the other two documents:
> [Hardware Design Reverse Engineering](hardware-teardown.md) is the **derivation process and evidence sources** (for those who need to verify),
> [Hardware Specifications Quick Reference](hardware-spec.md) is a **one-page spec sheet** (for those who already understand).
> This document is an **introductory explanation**, assuming you're just getting started with this project.
>
> All conclusions can be independently verified — the final section [How to Verify Yourself](#nine-how-to-verify-yourself-five-must-dos-before-replication) 
> explains exactly how to check.

---

## One: Start with the Big Picture: Five Modules, Only Two Need DIY

```
                    ┌─────────────────────────────────────────┐
                    │  Head (jaw_soft rigid body in MJCF)     │
                    │                                          │
   MIPI CSI ────────┤  ④ IMX219 Camera + M12 Lens             │
   (~13mm,          │        ↕ Same rigid body, no joint      │
    doesn't cross   │  ① Radxa Zero 3W    Main controller     │
    the neck)       │        ↕ Stack spacing 7.3mm, 40-pin    │
                    │  ② RPI Robot HAT    Power/Audio/Bus     │
                    │        └─ Stemma port ──► ⑤ ToF Module  │
                    │  Speaker                                 │
                    └──────────────┬──────────────────────────┘
                                   │
                    ═══════════════╪══════ Through 3-DOF neck ══════
                     Servo bus(1×) │ Battery power line(+BATT)
                                   │
                    ┌──────────────┴──────────────────────────┐
                    │  Trunk (trunk_base)                      │
                    │   Two hip_yaw XL330 — cavity mostly full│
                    │   ③ imu_to_dxl board (on servo bus)     │
                    └──────────────┬──────────────────────────┘
                                   │
                     NP-F550 battery ─┘ External mount on rear/lower
                     71mm long, ~40mm overhangs shell,
                     held by two power_support pieces from outside
```

| # | Module | One-line Description | How to Get |
|---|---|---|---|
| ① | **Radxa Zero 3W** | Brain. Runs Linux, neural net policy, vision | 🛒 Buy |
| ② | **RPI Robot HAT** | Power distribution + audio card + servo bus interface | 🏭 **Official open source, fab yourself** |
| ③ | **imu_to_dxl** | Disguises IMU as a fake servo | ✏️ **Must draw yourself** |
| ④ | **IMX219 Camera** | Eye | 🛒 Buy |
| ⑤ | **ToF Module** | 8×8 zone ranging, not lidar | 🛒 Buy |

> ⚠️ **Three Common Misconceptions**
> 1. **There is no lidar.** ToF is a multi-zone ranging module (4×4 or 8×8 zones), not radar.
> 2. **Audio is not a separate module.** Codec, amp, microphone are all on the HAT.
> 3. **Main controller is not in the trunk, it's in the head.** This is the easiest to get wrong and directly affects cable length.

---

## Two: ① Main Controller · Radxa Zero 3W

**What it does**: Runs Armbian Linux, runs Rust runtime `robotd`, runs ONNX policy inference, runs vision.
In one sentence — **it does everything except driving motors**.

| Item | Value |
|---|---|
| SoC | Rockchip **RK3566**, quad-core Cortex-A55 @1.6GHz |
| NPU | 0.8–1 TOPS INT8, node `npu@fde40000`, **disabled by default in Armbian**, needs overlay |
| Form Factor | **65 × 30 mm**, Raspberry Pi Zero form factor, 40-pin header |
| System | Armbian **headless** image |

**Why this board and not Raspberry Pi**: It's a **commercial off-the-shelf module**, not a custom carrier board — this is important,
meaning replication doesn't require making a main controller board. Device tree hardcodes `compatible = "radxa,zero-3w"`.

**Which 40-pin GPIO it uses** (extracted from official HAT schematic):

| Pin | Purpose | Linux Device |
|---|---|---|
| **GPIO14 / GPIO15** | **Dynamixel UART Tx / Rx** | `/dev/ttyS2` @ 1 Mbps |
| **GPIO02 / GPIO03** | I²C — IMU + audio + Stemma | `i2c3` @ 400 kHz |
| GPIO18 / 19 / 20 / 21 | Audio PCM Clk / FS / Din / Dout | `i2s3_2ch` |
| GPIO06 | Switch off detection | |
| GPIO22 | BMI088 interrupt on HAT (**not used in software**) | |
| GPIO00 / 01 | HAT identification I²C (reserved) | ⚠️ See HAT section |
| GPIO04/05, 08/09, 10/11 | Additional Stemma/QT I²C | |
| MIPI CSI | Camera | |

**Which config to buy**: 2G / 16G eMMC. Derivation process in [Electronics Sourcing List](electronics-sourcing.md#which-config-to-buy).
⚠️ **What config the official build actually uses, no evidence found anywhere in the codebase — unknown.**

---

## Three: ② RPI Robot HAT — One Board Does Four Things

**This is the only board with complete official open source** ([`pollen-robotics/elec_RPI_Robot_HAT`](https://github.com/pollen-robotics/elec_RPI_Robot_HAT),
Apache-2.0, includes KiCad project + Gerber + BOM + pick-and-place coordinates).

**Specifications** (Gerber measurements): 4-layer board, **65.0 × 30.9 mm**, corner radius R3.5, BOM 47 rows / 123 parts,
including 5 rows 9 parts DNP, actual placement **113 positions**.

### Function Block 1: Power Distribution

```
NP-F550 battery ──► +BATT ──┬──► Servos (direct supply, no buck)
  2S, 6.6–8.2V              │
                            └──► U9 AP63205 + L4 6.8µH ──► Q2 / U10 LM5050-1 ──► +5V
                                   buck, 2A, Fsw 1.1MHz      ideal diode + shutdown detect  │
                                                                                            ├─► Main controller 40-pin
                                                                                            ├─► PAM8406D amp
                                                                                            └─► U3 XC6206P182 → +1V8 (codec)
```

**Here's a counterintuitive fact: servos eat battery voltage directly, not 5V.**

XL330's official rating is **3.7–6.0 V, recommended 5.0 V**, but here it gets **6.6–8.2 V**.
Evidence is not speculation, the schematic directly states: J13/J14/J3/J11 four Dynamixel connectors' power pins all connect to `+BATT`,
annotated "3A max". And **the entire board has only one buck (U9) and one inductor (L4)**, that 5V rail is for the main controller
— the 2A AP63205 can't drive 15 XL330s (single stall current ~1.5A).

See [Voltage Truth](actuator-selection.md#voltage-truth-xl330-runs-overvoltage) for details.

> `+3V3` is conversely **supplied by the main controller through the 40-pin**, not generated by the HAT itself.

### Function Block 2: Servo Bus (Half-Duplex TTL)

Dynamixel / Feetech style bus servos are **single-wire half-duplex** — one wire for both transmit and receive, switched by direction control.

| Device | Function |
|---|---|
| **U6 `SN74LVC1G125`** | Tri-state buffer, transmit direction |
| **U7 `SN74LVC1G126`** | Tri-state buffer, receive direction |
| **U5 `74LVC1G08`** | AND gate, participates in direction logic |
| **U8 `SIT3088E`** | RS-485 transceiver (for 4P ports) |
| **TH1 100R** | Thermistor, overcurrent protection |

Direction controlled by `Dynamixel_dir`, official comment states **"Dynamixel_dir is based on Tx"** — direction automatically derived from transmit signal.

**Four connectors, two types**:

| Designator | Type | Who Uses |
|---|---|---|
| **J13 / J14** | **JST EH 3P** (TTL) | **Microduck's XL330 use these two**, through TH1 to `+BATT` |
| J3 / J11 | JST EH 4P (RS-485) | Driven by U8, direct `+BATT` |

> 💡 **JST EH is 2.5 mm pitch**, while Feetech servo cables are marked **5264 terminal**, also 2.5 mm.
> In the domestic market "5264" is basically the common name for 2.5mm 3P — very likely to plug directly, but shell latch details
> should be verified with physical parts.

### Function Block 3: Audio

| Device | Function | Address |
|---|---|---|
| **U2 `TLV320AIC3104`** | Audio codec, I²S data + I²C control | i2c3 `0x18` |
| **U1 `PAM8406D`** | Class D amplifier | |
| **MK1** | Onboard MEMS microphone (LCSC `C7587901`, official didn't give model) | |
| J1 | Wago screwless terminal, connects **5W speaker** | |
| J2 / J9 | Wago terminal, connects external microphone | |

Audio clock: MCLK 12 MHz fixed crystal, I²S sysclk 12.288 MHz (256 × 48 kHz).

> **If you don't need recording and speaker, this entire block can be omitted.**

### Function Block 4: Sensor Expansion

- **J5–J8**: JST SH 1.0 mm 4P (Qwiic / Stemma QT) — ToF plugs in here
- **U11 `BMI088`**: ⚠️ **Populated but software doesn't use it at all**. Device tree comment verbatim is `dormant`,
  `unused but still connected`. When outsiders say Microduck "has two IMUs" they're referring to this
- **U4 `CAT24C32`** EEPROM: ⚠️ **DNP, not populated** — so this board **is not a self-identifying HAT**,
  although the schematic says "HAT Identification: I2C add. 0x50 / Standard Class HAT+"

### Can't Buy This Board, What To Do

**Pollen doesn't sell it separately**, only sells complete robots.Two options:

1. **Faithful replication** — Take official Gerber + BOM + pick-and-place coordinates to JLCPCB. 4-layer board + SMT, starts at a few hundred yuan, with MOQ.
2. **Use off-the-shelf modules instead** — A ¥22 ST/SC half-duplex adapter board + a UBEC replaces the two essential functions "servo bus" and
   "power distribution", ToF connects directly to i2c3. Cost is loss of onboard audio. See [Electronics Sourcing List](electronics-sourcing.md#six-two-pcbs).

---

## Four: ③ imu_to_dxl — The Cleverest Move in the Entire Electronics Design

**This board does one thing: disguises the IMU as a fake servo.**

It's **STM32G031F8P6 + LSM6DSV16X** (ST six-axis IMU) + half-duplex buffer,
externally appearing as a slave on the servo bus, **ID = 200**.

### Why Go This Roundabout Way

The intuitive approach is IMU via I²C or SPI directly to main controller. But there's a problem:

**IMU data and servo data are two different paths, sampling timestamps don't align.**

Policy runs at 50 Hz, each tick needs to know simultaneously "body pose" and "15 joint angles".
If IMU goes via I²C and servos via UART, there'll be a few milliseconds of jitter between them — and jitter directly pollutes policy input.

Pollen's approach is **to make the IMU speak the servo language**, so the host sends **one `sync_read` per tick**,
and simultaneously gets back IMU pose + 15 servo positions, **timestamps naturally aligned**.

### Protocol and Registers

| Item | Value |
|---|---|
| Protocol | Dynamixel **Protocol V2** |
| ID | **200** |
| Baud Rate | **1 Mbps** (EEPROM register `baud_rate = 3`) |
| Starting Register | **124** |
| Length | **12 bytes** |

12-byte layout:

| Offset | Content |
|---|---|
| `0..6` | gyro x/y/z, `i16` little-endian raw counts, range **±500 dps**, **17.5 mdps/LSB** |
| `6..12` | SFLP quaternion x/y/z, **IEEE half-precision fp16**; `w = √(1 − x² − y² − z²)` |

**Note `w` is not transmitted** — relying on unit quaternion constraint to compute back on host side, saves 2 bytes.

**SFLP** is the **hardware sensor fusion block** inside the LSM6DSV16X chip, directly outputs game rotation quaternion
— host doesn't need to run attitude fusion itself.

### 16 Devices on the Bus Total

```
ID  10 11 12 13 14   Right leg
ID  20 21 22 23 24   Left leg
ID  30 31 32 33 34   Neck / head / jaw
ID  200              imu_to_dxl board
```

15 servos + 1 IMU board = **16 devices, one `sync_read` reads them all**.

### Replication Notes

- **Protocol and register layout fully reverse-engineered**, implement according to spec ([Hardware Design Reverse Engineering](hardware-teardown.md))
- **Connector use 2.5 mm pitch 3P** (JST EH / 5264)
- **Input capacitor voltage rating ≥25 V** — bus full charge 8.4 V, 10 V spec insufficient margin (MLCC also has DC bias derating)
- **CS pin needs pull-up** — see next section's real-world case
- **DATA line add TVS** — hot-plugging servos is normal

---

## Five: ④ IMX219 Camera

MIPI CSI interface, 8 megapixel. Device tree overlay uses `radxa-zero3-rpi-camera-v2`,
ISP config `SENSOR = imx219`.

**Two gotchas:**

1. **It's rotated 90° not inverted 180°.** Code states
   `/// 90, because the head camera is mounted a quarter turn off, and this is the one place
   that fact is written down.` Some early materials say 180, that's **alpha machine** data.
2. **Buy the shortest cable.** Camera and main controller **are in the same rigid body, separated by ~13 mm with no joint** —
   MIPI cable doesn't cross the neck. 4–15 cm sufficient, don't buy 30 cm.

> ⚠️ Radxa official docs only say "1x4-lane MIPI CSI", **doesn't give pin count or pitch**.
> But the board is Pi Zero form factor, CSI connector on same side as Micro HDMI, **very likely narrow 22-pin 0.5 mm**,
> while generic IMX219 modules come with standard 15-pin — have an adapter cable ready (a few yuan).

---

## Six: ⑤ ToF Module

**Not lidar.** It's ST FlightSense multi-zone ranging chip, outputs an **8×8 distance matrix**.

| Item | Value |
|---|---|
| Model | **VL53L5CX or VL53L8CX**, firmware auto-detects by revision ID (`0x02` = L5cx, `0x0C` = L8cx) |
| Interface | I²C, on **i2c3 @ 400 kHz** |
| Address | **`0x29` or `0x52`** (tries both candidates) |
| Physical Connection | **JST SH 1.0 mm 4P** (Qwiic / Stemma QT) on HAT |
| Frame Rate | 15 Hz |

> ⚠️ **Don't buy VL53L0X** — that's **single-point** ranging, firmware doesn't recognize it.
> **VL53L7CX is also not L8CX** (L7 is another chip with 90° FOV).

---

## Seven: What Happens in One Tick

Policy runs at 50 Hz, i.e. **every 20 ms one round**:

```
   ① One sync_read                  ttyS2 @ 1 Mbps
      ├─ ID 200  → gyro + quaternion    ┐
      ├─ ID 10~14 → Right leg 5 angles  ├ 12 bytes × 16 devices
      ├─ ID 20~24 → Left leg 5 angles   │  Timestamps naturally aligned
      └─ ID 30~34 → Neck/head/jaw 5     ┘

   ② Assemble 61-dim observation vector

   ③ ONNX policy inference (775 KiB small network) → 14-dim action

   ④ Voltage adaptation: scale × (7.4 / measured battery voltage EMA)
      — Battery drops from 8.4V to 6.5V, same PWM gives very different torque, without compensation policy drifts

   ⑤ One sync_write, send 14 target positions down
```

Side channels also running: **i2c3** ToF (15 Hz) and audio codec, **MIPI CSI** camera.

> ⚠️ **20 ms budget is tight, and it's not just Feetech's problem.**
> Official source code states: XL330 factory `return_delay_time = 250`, i.e. **500 µs turnaround delay per device**,
> "Across 16 devices that is **8 ms per tick — 40% of a 20 ms budget**".
> Official solution is to write this register **to 0**.
> Replacing with any servo, this step must be redone and physically tested.

---

## Eight: Going Feetech Route, What Changes

Many people (including this repo) are evaluating using Feetech servos to replace XL330. Impact by severity:

| Impact | Description |
|---|---|
| 🔴 **Official 9 ONNX policies will likely fail** | Servos changed, torque curve, mass, possibly even gear ratio all different. **Must retrain** (requires CUDA GPU or HF jobs) |
| 🟡 **Bus timing must be physically tested** | Feetech uses STS/SCS proprietary instruction set (includes `0x82 SYNC READ`), packet format different. **But don't assume it's necessarily slower** — official Dynamixel path also has per-device turnaround delay (see above). First thing after parts arrive is run benchmark, get an **FE-URT-1 debug board** |
| 🟡 **`imu_to_dxl` needs repositioning** | Bus protocol changed, board either adapts to emulate Feetech slave, or abandons "on bus" design for IMU via SPI direct. **Recommendation: hardware unchanged, firmware dual-protocol** — both are physically identical (half-duplex single-wire TTL / 1 Mbps / 3-wire), difference is in **packet format** and **factory defaults**, see [Hardware Design Reverse Engineering](hardware-teardown.md) "Feetech STS/SCS Protocol Comparison" |
| 🟢 **Voltage actually more suitable** | Feetech HD-1910 native 5–8.4 V, doesn't need to run overvoltage like XL330 |

---

## Nine: How to Verify Yourself: Five Must-Dos Before Replication

**This section is more important than all content above.**

Open source project materials can be outdated, can have errors, AI-generated schematics even more so. Below are verification actions
this repo summarized in practice — each corresponds to a real crash.

### 1. Pin Definitions: Check Datasheet Pin Table, One by One

**Real case:** This repo received an `imu_to_dxl` schematic PR. Drawing looked complete, but
LSM6DSV16X pins were wrong — checking **ST DS13510 Rev 1 Table 1** (page 10) revealed:

| Pin | Manual Definition | Drawn in PR |
|---|---|---|
| **12** | **CS** (chip select) | ❌ Misaligned |
| **13** | **SCL** (SPI clock) | ❌ Misaligned |
| **14** | **SDA** (SPI data) | ❌ Misaligned |
| **2 / 3** | **SDx / SCx auxiliary interface**, manual requires **connect to GND** when not used | ❌ Treated as main interface |

This board made would **not work**. Finding it only required one thing: **open manual, turn to pin table, check line by line**.

> **Drawing schematics without checking manual equals not drawing.** This has no exceptions.

### 2. Voltage Margin: Each Capacitor's Rating vs Actual Bus Voltage

Same PR, input capacitor marked **10 V**, while bus full charge **8.4 V**. Looks like margin, but:
- MLCC has **DC bias effect**, 8.4 V on 10 V rated MLCC, actual capacitance can drop more than half
- Battery, motor back-EMF will have spikes

**Rule: Voltage rating at least 2× working voltage.** 8.4 V bus, use 25 V.

### 3. Power-On State: What Level are Mode Select Pins at Power-On Instant

LSM6DSV16X's CS pin level **at power-on determines SPI or I²C mode**. If CS floats,
power-on instant is undefined — sometimes works, sometimes doesn't, hardest kind of bug to track.

**Add a 10 kΩ pull-up to 3V3**, locks SPI mode.

### 4. Run ERC, and Get to 0 Errors 0 Warnings

KiCad command line can run it:

```bash
kicad-cli sch erc --output erc.rpt your.kicad_sch
```

Common real issues: power pins have no drive source (missing `PWR_FLAG`), two Power outputs collide,
pin coordinates not on connection grid (looks connected but isn't).

### 5. Cross-Validate: Addresses and Registers in Software Source vs Hardware

Microduck's software is open source, **code is spec**. Every address in hardware can be found in code:

| Hardware | Evidence in Software |
|---|---|
| Audio codec `0x18` | `deploy/audio/i2c3-pihat.dts` |
| ToF `0x29` / `0x52` | `tof/src/main.rs` `ADDRESS_CANDIDATES` |
| IMU board ID 200 | `duck-control/src/model.rs` `IMU_DXL_ID` |
| Servo ID allocation | Same, `JOINT_IDS` |
| i2c3 400 kHz | `i2c3-pihat.dts` `clock-frequency` |
| BMI088 not used | `i2c3-pihat.dts` comment: `dormant`, `unused but still connected` |

**If it doesn't match, one side is wrong.**

---

## Appendix: Sources of All Conclusions in This Document

| Conclusion | Source |
|---|---|
| Board locations | `robot_walk.xml:235` (HAT), `:247` (main controller), both in `jaw_soft` body |
| 40-pin allocation | `ASE01187-C1_elec_RPI_Robot_HAT_SCH.pdf` GPIO table |
| HAT board specs | Gerber `Edge_Cuts.gm1` measurements; BOM.csv 47 rows / POS.csv 117 rows |
| Servos connect to `+BATT` | Schematic Dynamixel page (4/6), annotated "3A max" |
| Only one buck | BOM.csv full table only has L4 one inductor (FB1/2/3 are ferrite beads) |
| 12-byte register layout | `robotd` source code, see [Hardware Design Reverse Engineering](hardware-teardown.md) |
| Camera rotated 90° | `mediad/src/main.rs:82` |
| 8 ms turnaround delay | `duck-control/src/model.rs:84-86` |
| LSM6DSV16X pins | **ST DS13510 Rev 1, Table 1, page 10** |

> This document based on official public MJCF model, STL meshes, open source hardware projects and runtime source code derivation,
> **not physically verified**. Found errors please file issue.
