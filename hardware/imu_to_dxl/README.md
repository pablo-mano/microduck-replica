# imu_to_dxl Schematic + PCB · Review Requested

> ⚠️ **This is a third-party replica, not an official design.**
> The official design has never publicly released the schematic, Gerber files, or mechanical dimensions for this board (zero hits across the web, see
> [Hardware Reverse Engineering](../../docs/硬件方案逆向.md)). This directory is the result of reverse engineering based on the MJCF model, official runtime source code,
> and the open-source HAT project, combined with independent component selection. **This has not been fabricated or validated with physical hardware.**
> Please open an [issue](https://github.com/fanhao375/microduck-replica/issues) if you find any problems.

![imu_to_dxl Schematic](../../assets/hw/imu_to_dxl-原理图.png)

| File | Description |
|---|---|
| [`imu_to_dxl-原理图.pdf`](imu_to_dxl-原理图.pdf) | Vector PDF, zoom for details |
| [`netlist.md`](netlist.md) | Network-by-network pin listing for direct verification |
| [`imu_to_dxl.eprj2`](imu_to_dxl.eprj2) | JLCPCB EDA Pro project, can be opened and modified directly |
| [`../../BOM.md`](../../BOM.md) | Complete BOM with LCSC part numbers and selection rationale for each component |

---

## What This Board Does

The official Microduck has 15 servos connected to a single Dynamixel half-duplex bus. This small board
**acts as the 16th device on the same bus**, presenting itself as a Dynamixel slave (ID 200).
The main controller requests attitude data from it by reading registers.

The onboard IMU uses the **LSM6DSV16X**, which has a **SFLP hardware fusion block** inside the chip
that directly outputs game rotation quaternions — attitude calculation doesn't consume main controller processing power or bus bandwidth.

```
Main Controller (Raspberry Pi/Taishan Pi)
  │  1 Mbps half-duplex single-wire
  ├── Servos ×15
  └── imu_to_dxl (this board, ID 200)
         ├── LSM6DSV16X   SPI 4-wire + 2 interrupt lines
         ├── STM32G031F8P6 packet RX/TX / IMU reading
         └── SN74LVC2G241 half-duplex transceiver buffer
```

## Core Component Selection and Rationale

| Designator | Component | Why This Choice |
|---|---|---|
| **U1** | STM32G031F8P6 | ① USART has **hardware DE** (RS485 Driver Enable) — at 1 Mbps, one bit is only 1 µs; software direction control jitter is on this order of magnitude. In contrast: **STM32F401 datasheet has no "Driver Enable" anywhere**, only software control<br>② Same manufacturer as IMU, ST's `lsm6dsv16x-pid` driver and SFLP examples are ready-made |
| **U2** | LSM6DSV16XTR | Onboard SFLP fusion, outputs quaternions directly |
| **U3** | SN74LVC2G241DCUR | Dual buffer for single-wire half-duplex. Channel 1 left-to-right, channel 2 right-to-left, perfect for routing. With 3.3V supply **input tolerates 5.5V**, can handle 5V signal from servos |
| **U4** | HT7533-1 | **30V withstand** — it survives even if the first two protection stages fail |
| **F1** | PPTC MF-NSMF020X-2 | **24V / 200mA hold**. In case of board fault, disconnects itself first without dragging down the entire servo bus |
| **D1** | TVS SMF12A | Cutoff 12V > fully charged 8.4V, clamp 19.9V < C6's 25V rating |
| **C6** | 10µF / **25V** | Connected to 8.4V bus. MLCCs have DC bias derating; 6.3V/10V margins are insufficient |
| **C5** | 4.7µF | Combined with C4's 100nF per ST requirements — **DS12992 Rev 3, Figure 13** explicitly marks `VDD/VDDA` with `1 × 100 nF + 1 × 4.7 μF` |

## Several Potentially Controversial Decisions

### 1. `2OE` Tied High, Receives Echo During Transmission

`U3.1 (1OE#)` is connected to DE to control transmit buffer, `U3.7 (2OE)` **tied directly to 3.3V, receive buffer always enabled**.

- Trade-off: During transmission, MCU receives its own transmitted bytes on RX; firmware discards them based on transmission length
- Benefit: PA3 is always driven, **never floating**

Alternative is connecting `2OE` to DE as well (disable receive during transmit), but then RX floats during transmission,
potentially creating false start bits when re-enabled. I chose the former. **Open to counterarguments**.

### 2. No Crystal, Only Footprint Provided

TSSOP-20 package **does not bring out `OSC_OUT`** (DS12992 Rev 3, Table 12: pin 2 `PC14` has additional function
`OSC_IN`, pin 3 `PC15` only has `OSC32_OUT`), so **passive crystal cannot be connected at all**.

HSI16 accuracy (Table 41): factory −0.75%/+0.5%, 0–85°C temp drift ±1% → worst case about **−1.85%/+1.55%**.
1 Mbps asynchronous UART typical tolerance ±3%, sufficient but not much margin.

Board provides **X1 (16MHz active oscillator, HSE bypass) + C9 + R3** footprints, **DNP by default**.
Additionally, firmware can use timer input capture to measure bit period from the host, adjusting `HSITRIM` (0.2–0.4%/step),
reducing error to within 0.3%, at no cost.

### 3. Retain Both J1/J2, Handle Current with Copper Pour

J1 and J2 have three pins each, connected in parallel. **If this board is in the middle of the chain, all downstream servo current flows through the board's connectors and copper traces.**

Feetech HD-1910-C001 official specs: rated **500mA** / stall **1.8A**.

But this constraint **is not unique to this board** — servos are already daisy-chained, their own connectors carry the same current.
The HAT has **4 servo interfaces**, 15 servos distributed across 4 branches, max 4 per branch, about **2A** per branch rated,
within 3A. If the official design accepts this, this board using the same connector is no worse.

**Therefore retain both connectors**: want to hang at the end, don't populate J2; want to insert in the middle, populate it. Flexibility at nearly zero cost.
On PCB, route `VDD_BUS` and `GND` from J1 to J2 with **wide copper** (≥2mm or direct copper pour).

> One review comment suggested "leave only one 3P connector". That's valid when this board is definitely a leaf node,
> but **if it's the first node in the chain** (HAT output enters this board first, then exits to servos), both are required.
> Official topology cannot be confirmed; retaining both is safer.

### 4. Protocol Auto-Detection by Software, No Jumpers/DIP Switches

Feetech and Dynamixel have identical physical layers (half-duplex single-wire / 1 Mbps / 3-wire / GND-VCC-DATA),
difference is only in packet format:

| | Header | Following | Checksum |
|---|---|---|---|
| Dynamixel V2 | `FF FF` | **`FD 00`** | CRC-16 |
| Feetech SCS | `FF FF` | `ID` + `LENGTH` | `~(ID+LEN+INST+params)` |

Feetech official protocol document states **Length = parameter count N + 2**, so minimum is 2, **never 0**;
ID range 0–253. → `FF FF FD 00` can only be V2, **unambiguous discrimination**.

Firmware votes on first 3 packets to lock protocol, then follows locked protocol for RX/TX; unlocks for re-detection after 100 consecutive parse failures.
More reliable than DIP switches (which become vibration failure points in a walking robot).

### 5. J3 Uses 6P, Brings Out UART printf as Well

`1=GND 2=SWDCLK 3=SWDIO 4=UART_TX 5=UART_RX 6=+3V3`, first 4 pins order unchanged,
**existing 4-pin SWD cable plugged into 1–4 still works**.

UART comes from U1 pins 16/17 (`PA11[PA9]` / `PA12[PA10]`), remapped via `SYSCFG_CFGR1` to
`USART1_TX/RX`.

**No USB**: G031 has no USB peripheral (datasheet Development support only mentions "serial wire debug (SWD)"),
adding USB requires an additional CH340/CP2102, not cost-effective on a 45×22mm board.
Debug via SWD + RTT (`probe-rs` supports RTT over CMSIS-DAP without halting CPU).

> Cortex-M0+ **has no SWO/ITM**, don't expect SWO printf.

---

## Board Outline and Mounting Holes

| Item | Value |
|---|---|
| Board outline | **45 × 22mm**, 2-layer board |
| Corners | **R2 rounded** |
| Mounting holes | **2 × M2 non-plated holes φ2.2mm**, diagonal |
| Hole positions (origin at bottom-left) | MH1 (2.667, 2.413), MH2 (42.672, 19.685) mm |
| Hole spacing | X 40.00mm / Y 17.27mm / diagonal 43.57mm |
| Hole to edge | MH1 **1.31mm** / MH2 **1.22mm** (JLCPCB requires ≥0.4mm) |

Midpoint of two holes (22.67, 11.05) differs from board center (22.50, 11.00) by 0.17mm, **180° rotation still aligns** (0.34mm deviation).

> ⚠️ **Hole positions changed on 2026-09-08.** Original design was MH1 (5.5, 3.5) / MH2 (39.5, 18.5), X spacing 34mm,
> deliberately aligned with original `banana_pcb_locker` printed part's two positioning ear spacing.
> **When adding `J4`/`J5`, MH2's clearance circle landed right on the only empty strip that could fit through-hole components on the right side**,
> so both holes were moved toward corners, X spacing changed to 40mm.
> **Trade-off is no longer compatible with original latch — mechanical side needs synchronized pressure bar change.**
> Anyway the inference in reverse engineering doc that "latch presses `imu_to_dxl`" was only speculation
> (see [Hardware Reverse Engineering](../../docs/硬件方案逆向.md)), hard constraint is only "IMU rigidly connected to `trunk_base`".

Board outline file: `imu_to_dxl-板框-45x22-R2.dxf` (units mm, 8 segments closed: 4 straight edges + 4 90° arcs).

> **Why separate DXF**: This version of JLCEDA Pro's primitive interface only accepts copper layers,
> adding lines/arcs to board outline layer (layer 11) all report "parameter incorrect", changing layers/document sources are all blocked,
> so board outline goes through DXF import (`File → Import → DXF`, target layer select **board outline layer**, units mm, origin align 0,0).
> Two M2 holes not in DXF — they're already built in PCB as actual NPTH pads.

## PCB: Layout and Routing

![imu_to_dxl PCB Top Assembly View](../../assets/hw/imu_to_dxl-PCB.png)

*Routing diagram, rendered from actual project data (top red / bottom blue / pads yellow / GND vias green / signal vias purple, blue background is bottom GND pour, purple box is `J4`/`J5`).*
*Status 2026-09-08: 29 components · 227 trace segments · 31 vias · 172 teardrops, includes `R7`, `J4`, `J5`, `X1`/`R3`/`C9` changed to actual placement.*
*⚠️ `.eprj2` / PDF / STEP in directory are still 09-06~09-07 version, pending re-export.*

**45 × 22mm, 2-layer board, 1 oz copper**, R2 rounded corners, two diagonal M2 non-plated holes.
All SMD components on top; 3 through-hole connectors on two edges: `J1` `J2` (servo bus) on bottom edge,
`J3` (SWD + UART) on top edge.

### Zoning

| Zone | Contents | Why |
|---|---|---|
| Bottom edge | `J1` `J2` bus connectors (JST EH 2.5mm) | Cable harness from trunk routing, exit nearby |
| Bottom-right | **`J5` Feetech bus out** (2.0mm) | Same side as `J1`/`J2`, four connectors' `VDD_BUS` directly connected via copper strip |
| Right edge | **`J4` Feetech bus in** (2.0mm) | Only empty strip on entire board that fits through-hole component; MH2 moved away to make room for it |
| Right side | `F1` → `D1` → `C6`/`C8` → `U4` → `C7` | Power chain one path to the end, isolated from digital section |
| Center | `U3` buffer + `D2` clamp + `R6` series resistor | Between MCU and bus connectors, shortest RX/TX path |
| Upper-center | `U1` MCU, `U2` IMU and respective decoupling | |
| Left side | `X1` / `R3` / `C9` clock position | Changed to actual placement, corner location doesn't occupy prime space |
| Top edge | `J3` debug header | Insertion direction outward |

### Routing

| Item | Value |
|---|---|
| Components | **29** (including R7 added 09-07, J4/J5 added 09-08) |
| Traces | **227 segments** (top 197 / bottom 30) |
| Trace width | Signals mainly 0.25mm; **bus through 0.76mm** |
| Vias | **31, φ0.6 / hole 0.3mm** |
| Teardrops | 172 |
| Copper pour | **Bottom entire GND plane**, top no pour |

Bottom 30 trace segments, **ground plane basically intact** — this provides return path for SPI and 1 Mbps half-duplex bus.
Top layer GND pads each via down to ground plane nearby.

### Review Results · 2026-09-08

In addition to built-in JLCPCB DRC, used script to re-rasterize by **actual pad shapes** (circular / oval / rectangular),
independently verified once more:

| Check Item | Result |
|---|---|
| JLCPCB DRC | **0 violations** |
| **Connector pinout** | **All four correct**: `J1`/`J2` = `GND`/`VDD_BUS`/`DXL_BUS`; `J4`/`J5` = `DXL_BUS`/`VDD_BUS`/`GND` (**reversed**, see pinout section below) |
| Network connectivity | **All 23 networks connected** (raster connectivity + vias/through-holes cross-layer, including bottom copper pour) |
| Minimum spacing between nets | **0.152mm = 5.98 mil** (precise geometry, non-raster). JLCPCB 2-layer minimum 0.127mm, 20% margin |
| Copper to edge | Minimum **0.37mm** |
| Hole to hole (wall to wall) | Minimum **0.751mm**, requirement ≥0.4mm |
| Hole to edge | MH1 **1.31mm** / MH2 **1.22mm** |
| M2 clearance | MH1 clearance circle 2.5mm contains only one **GND** via (1.81mm) — screw pressing on ground is still ground, harmless; MH2 nearest 4.59mm |
| Via specs | φ0.6 / hole 0.3mm, meets JLCPCB 2-layer minimum |
| Decoupling distance | `U1.4`→`C4` 2.27, `U2.8`→`C1` 2.00, `U2.5`→`C2` 2.01, `U3.8`→`C3` 1.96, `X1.1`→`C9` 2.39 mm, all <2.4mm; `U4.3`→`C7` 2.94mm. ⚠️ `U4.2` LDO input side 100nF `C8` is **6.64mm** (loose, but transient-managing 10µF `C6` at 4.29mm, and LDO is linear no switching) |

**Current capacity verification**: `VDD_BUS` through path between four connectors
`J1.2 → J2.2 → J5.2 → J4.2` **entirely 0.76mm**, 1 oz copper ΔT=10K about 2.0A, ΔT=30K about 3.2A.
JST EH single contact rated 3A, **trace and connector basically same magnitude**, widening trace doesn't raise overall chain limit.
Also has 4.7mm of 0.51mm segment on `J1.2 → F1.1` path that **only feeds this board itself**,
protected by 200mA hold PPTC, sufficient.

> **2026-09-08 caught and fixed one issue**: When routing `DXL_BUS` to `J4`/`J5`, one φ0.3 via
> landed directly over `J2.3`'s φ1.0 through-hole (0.001mm center distance). JLCPCB DRC reported
> `Hole to Hole (DXL_BUS): J2_3 ↔ e75` — drilling is mechanical process, not allowed to overlap even same net.
> `J2.3` itself is through-hole, top-bottom already connected, via was redundant, deleted with connectivity unchanged, DRC cleared.

**Still unvalidated**: **No fabrication, no physical board**. All above are geometry and rule-level verification,
cannot replace actual testing.

### TODO

- Before ordering, run **component standardization** once (schematic DRC reported "component properties mismatch supplier part number")
- Silkscreen review: whether designators cover pads, whether `U1` pin 1 / `D1` `D2` cathode / `J1` pin 1 markings are complete
- **Connector height**: `J1`/`J2`/`J3` are JST EH / 2.54 header through-hole, body about **8mm**. If trunk cavity
  is indeed only 6.65mm, need right-angle `S3B-EH-A`, both footprint and BOM must follow — compare exported STEP in
  assembly to determine.
  **`J4`/`J5` don't need worry**: `B3B-PH-K-S` body about **6.0mm**, fits in 6.65mm cavity
- **Board-side connector model to be confirmed**: Feetech spec only states wire end is `AMP2.0-3P`, **board end model not given**.
  This board temporarily uses `B3B-PH-K-S` (JST PH, 2.0mm vertical). Pad array is three holes 2.0mm spacing,
  compatible with other 2.0mm series, **changing series later only changes footprint, not pads, no re-routing**
- **`.eprj2` / PCB PDF / schematic PDF / STEP pending re-export** — these four files in directory still 09-06~09-07 version

## Please Focus Review on These Areas

1. **`2OE` always-on trade-off** (item 1 above) — echo vs RX floating, which is worse
2. **J1/J2 current capacity and copper width** (item 3) — is 2mm sufficient
3. **HSI16 running 1 Mbps margin** (item 2) — anyone with actual test experience please comment
4. **Three stages of overvoltage protection sufficient** — 8.4V bus + servo back-EMF
5. **Connectors**: see below "⚠️ Feetech pinout varies by model, and is reversed". `J4`/`J5` two 2.0mm connectors
   already placed in schematic and PCB (2026-09-08), but **exact board-side connector series not physically compared yet**
   — Feetech spec only states wire end `AMP2.0-3P`, board end model pending manufacturer confirmation

## ⚠️ Feetech Pinout Varies by Model, and Is Reversed

**This is the most potentially fatal point of this board, both manufacturer specs are correct individually — the two models really are reversed.**

| | Connector | Pin 1 | Pin 2 | Pin 3 |
|---|---|---|---|---|
| **`HD-1910-C001`** (selected by this repo) | **`AMP2.0-3P`** (2.0mm) | **Signal/TTL** | Vcc | **GND** |
| `HL-2915-C002` (different model, don't copy) | `AMP-3` | **GND** | Vcc | **Signal/TTL** |
| This board `J1`/`J2` (JST EH 2.5mm, same as official HAT) | `B3B-EH-A` | GND | VDD_BUS | DXL_BUS |
| This board `J4`/`J5` (2.0mm, mates with Feetech) | `B3B-PH-K-S` | **DXL_BUS** | VDD_BUS | **GND** |

All four connectors' pinouts verified by reading back from PCB netlist, all correct (2026-09-08).
**Two pitches don't mate (2.0 vs 2.5mm), physically prevents wrong connection, only "drawn wrong" remains possible.**

Source: Feetech "HD-1910-C001 Serial Specification" **A/0, 2026-09-07** page 3/7 items 6-7;
Feetech "HL-2915-C002 Serial Specification" A/0, 2026-02-27 page 4/8 items 6-7.
Both are specs sent by manufacturer to customer for confirmation, **this repo per convention does not redistribute PDFs**, please request from Feetech if needed.

> This repo previously only recorded the `HL-2915` line but recommended `HD-1910` as substitute model, **that was a trap**,
> corrected 2026-09-08 per A/0 new spec.
>
> ⚠️ **`HL-2915-C002` is 9–14V (typical 12V) servo, cannot hang on this design's 2S bus (6.6–8.2V)**
> — it only starts working at 9V. Using it as "Feetech interface convention" sample was expedient, don't follow for selection.
>
> ⚠️ **Pitch also mismatched**: `AMP2.0` is 2.0mm, `JST EH` is 2.5mm, **don't mate**.
> Fortunately this actually provides insurance — physically prevents wrong connection, only "drawn wrong" remains possible.

## Confirmed / Unvalidated

**Confirmed (primary sources)**

- **HD-1910-C001** interface `1=Signal/TTL · 2=Vcc · 3=GND`, connector `AMP2.0-3P`, wire length 15±0.5 cm
  — Feetech "HD-1910-C001 Serial Specification" A/0 (2026-09-07) page 3/7 items 6-7
- HD-1910-C001 runs **TTL 3-pin bus**, **4–8.4V**, stall 9/12/15 kg·cm @4.8/6/7.4V,
  rated current **500/690/900mA**, stall current **1.2/1.6/2.0A**, idle 20mA, 12 bit magnetic encoder,
  gear ratio 1/320 — same spec page 2/7, 3/7
- **Motor has three built-in electronic protections** (same page 4/7 7-11):
  **Overcurrent** (>0.5A sustained 2s shuts output, threshold and duration customizable, **factory default OFF**) ·
  **Overvoltage** (>10V or <4V protection, range customizable, **factory default OFF**) ·
  **Overheat** (>80℃ shuts torque output, **factory default ON**).
  **Should enable first two before installation** — anomaly detection done in motor, host doesn't need to poll
- Feedback six items complete: input voltage / load / working current / working speed / working temperature / position (same 7-10).
  Official runtime indeed uses these: `present_current` read every tick with position, voltage+temperature about once per second,
  and **voltage is the only battery metering method for entire system** (`duck-control/src/{bus,model}.rs`)
- Factory default run mode is **mode 4 "pure position PD (Sim2Real)"**, targeting RL deployment (same 7-12)
- Packet format, `Length = N+2`, checksum algorithm, sync read `0x82` — Feetech official "Servo SCS Communication Protocol"
- STM32G031 no HSE pins, HSI16 accuracy, `100 nF + 4.7 µF` decoupling requirement — ST DS12992 Rev 3
- Schematic connectivity — verified net-by-net with script, no floating pins, no shorts, no duplicate net names

**Unvalidated**

- **Entire board never fabricated**, no physical hardware
- Board outline 45 × 22mm and M2 hole X spacing 34mm reverse-engineered from printed part `banana_pcb_locker`, not official dimensions
- MCU model is this repo's selection recommendation, **not reverse-engineered** — impossible to recover what MCU official board used

---

## Review Log

### 2026-09-07 · Three Layout/Selection Suggestions (All Not Adopted, Rationale Recorded)

Reviewer looked at drawings and made three suggestions: switch resistors/capacitors to 0402, place IMU at board center, move LDO away from IMU. Measured data for each,
**conclusion is maintain current state**, but rationale for each worth recording — especially items 2 and 3 actually point to the same thing.

**① Switch resistors/capacitors to 0402 — Not switching, rationale not about "can hand-solder"**

Reviewer added "don't torture yourself soldering, group buy at JLCPCB assembly saves hassle". This suggestion itself is correct, but switching to 0402
decision criteria isn't here: this directory's 2026-09-06 entry first sentence stated "**rationale is not that 0402 can't be hand-soldered**"
— board has LGA-14 LSM6DSV16X (bottom pads 3×2.5mm), must use hot air anyway.

Real rationale is **debug period rework**: `R6` series resistor needs trying 150Ω to 33Ω, pull-up may need removal, decoupling may need addition.
**First version unvalidated board will definitely need several changes**, this doesn't change because of JLCPCB assembly — assembly factory only does
first pass, changes still need your own iron.

Trade-off side: 16 resistors/capacitors switching to 0402 saves **12.5mm², 1.3% of board area**, and this board **only uses 36%
of area**, not lacking space at all. 0402's 10k/100nF at JLCPCB are also basic library, material cost applies to both sides equally,
doesn't constitute rationale.

**② IMU at board center · ③ LDO away from IMU — Direction both correct, but magnitude different**

Measure first:

```
Board center (22.5, 11.0) = midpoint of two M2 holes (designed this way initially)
U2 (IMU)  7.47mm from board center, 3.00mm from two screw connection line
U3 (buffer) 0.41mm from board center      ← Most rigid position occupied by logic chip
U2 ↔ U4 (LDO) center distance 6.37mm
```

**Thermal magnitude**: Board total current about 5–10mA (G031 about 3mA + LSM6DSV16X about 0.65mA + buffer about 1mA
+ pull-up), LDO dropout 8.4 − 3.3 = 5.1V → **dissipation about 51mW** (worst case fully charged). SOT-89 on small copper
θJA about 100–160°C/W → junction temp rise **5–7°C**; 6.4mm away with entire bottom ground copper spreading, reaching IMU
location **temp difference about 1–3°C**. LSM6DSV16X gyro zero-bias temp drift about 0.05°/s/°C → **about 0.1°/s offset**.
During walking gyro is tens of °/s magnitude, and trunk has 15 servos drawing ampere-level current, heat sources two to three orders of magnitude larger than 51mW.
**Direction correct, but not dominant term.**

**Really worth moving is the other aspect of item ②**: Board center happens to be midpoint of two screws, most deformation-resistant position on entire board,
and currently sitting there is `U3` which doesn't care about position. Swapping `U2`/`U3` achieves three goals —
IMU from board center 7.47 → 0.4mm, while distance from LDO 6.37 → **11.3mm** (doubled).

**Why not moving this time**: Board already DRC 0 violations, fully routed; swapping means center area re-arrangement + re-routing
SPI×4, 2 interrupts, buffer's DE/TX/RX/DXL_DATA. Based on above magnitude estimation, benefit is "more robust" rather than
"solving known problem". **Save for after fabrication with actual measured data to decide** — if measured gyro zero-bias drifts significantly with LDO load,
or vibration noise is excessive, v2 change together.

**A more important reminder**: IMU's position and orientation **in the robot** is fixed by trained model, much more important than 7mm offset on board. That's determined by how board mounts in trunk, already verified and recorded in
[`docs/硬件方案逆向.md`](../../docs/硬件方案逆向.md): `trunk = [+raw_z, +raw_y, −raw_x]`,
+90° around Y axis. Moving position on board doesn't affect this.

### 2026-09-07 · Reserve Control Line for `2OE` (R7 + RX_EN)

Someone suggested leaving a line to MCU for "that EN pin". Refers to `U3.7 (2OE)` — receive buffer output enable,
previously hard-tied to +3V3, receive always on. **Adopted.**

**Changes**

| | |
|---|---|
| Disconnect | `U3.7 (2OE)` from `+3V3` direct connection |
| Add | **R7 = 10kΩ 0603** (`C25804`, same part as R1/R2/R4/R5, no new part number) |
| New net | **`RX_EN` = `U1.1 (PB7/PB8)` · `U3.7 (2OE)` · `R7.1`** |
| Retain | `R7.2 → +3V3` (pull-up) |

**Why choose U1.1**: U1 available are pin 1 (PB7/PB8), pin 3 (PC15), pin 20 (PB3/PB4/PB5/PB6).
Pin 3 is PC15, shared with LSE on G0, drive capability limited; pin 20 multiplexes four ports most flexible, reserve it.
Pin 1 is regular GPIO, and same bottom row as `DE`/`MCU_TX`/`MCU_RX`, routes same channel.

**Default behavior completely unchanged** — During MCU reset PB7 is high-Z, R7 pulls `2OE` high, receive buffer remains on as usual.
Firmware writes zero lines, identical to before change. This is **reserved**: When wanting to eliminate echo during transmission, configure PB7 as push-pull
output pulling low, while enabling internal pull-up on `MCU_RX` pin to avoid `2Y` high-Z causing RX float —
this solves the concern that previously rejected "`2OE` tied to DE" scheme, using independent GPIO + internal pull-up.

**Post-change review**: Components 26→27, nets 23→24, `RX_EN` = 3 pins, `+3V3` removed `U3.7`, `R7.2` replaces.
JLCPCB DRC **0 violations**, minimum spacing between nets **0.160mm** (pre-change 0.153),
other items consistent with 2026-09-07 first review.

**Correction by the way**: This directory's `netlist.md` previously incorrectly listed pins 2, 16, 17 as floating
(they connect to `OSC_IN`, `UART_TX`, `UART_RX` respectively), corrected. Now truly idle are only pins 3 and 20.

### 2026-09-07 · Three Questions About U3 Buffer (Two Not Adopted, One Drawing Method Changed)

Reviewer checked SN74LVC2G241 datasheet, raised three points about U3 connection. Verified each — netlist read from
PCB current state (PCB nets exported from schematic, equivalent to schematic connection):

```
U3 pin1  1OE#  → DE          pin5  2A   → DXL_DATA
U3 pin2  1A    → MCU_TX      pin6  1Y   → DXL_DATA
U3 pin3  2Y    → MCU_RX      pin7  2OE  → 3.3V
U3 pin4  GND   → GND         pin8  VCC  → 3.3V

DXL_DATA  4 pins: R5.2 R6.1 U3.5 U3.6     ← does not include U3.7
DE        3 pins: R4.2 U1.8 U3.1
MCU_TX    2 pins: U1.9 U3.2
MCU_RX    2 pins: U1.10 U3.3
```

**① "A is input, Y is output, so pin3 should connect DXL_DATA, pin5 should connect MCU_RX" — Premise correct, conclusion reversed**

Datasheet original text is correct: device is two independently-enabled 1-bit line drivers, data direction always **A → Y**.
But swapping wires per this is wrong:

- `MCU_RX` is **the network to be driven by buffer** (into MCU receive pin), must land on **output pin Y**
  → currently pin3 (2Y) ✅
- `DXL_DATA` is **the network to be read in** (servos transmit on bus), must land on **input pin A**
  → currently pin5 (2A) ✅

After suggested swap, channel2 becomes `MCU_RX(2A) → DXL_DATA(2Y)`, meaning from MCU's receive pin
sending data onto bus — RX is input pin doesn't output level; and `1Y` with `2Y` would drive
`DXL_DATA` simultaneously, two drivers directly fighting.

Current division is standard half-duplex connection: **channel1 = transmit** (`MCU_TX → bus`, controlled by DE),
**channel2 = receive** (`bus → MCU_RX`, always on). **Not adopted.**

**② "2OE should have series pull-up resistor, otherwise pull-up too strong" — Electrically invalid, but drawing method needs change**

`2OE` is CMOS logic input, input current magnitude ±1µA, directly tying to VCC is standard datasheet usage.
Series resistor has no benefit, instead introduces interference coupling to high-Z input.

"Pull-up too strong" concern only applies when this node is **data line** — it's not:
`DXL_DATA` network only has 4 pins (`R5.2` `R6.1` `U3.5` `U3.6`), **does not include `U3.7`**.
Bus pull-up is `R5 = 10kΩ`, value also appropriate.

**But this reminder has value**: This misunderstanding arose because in drawing "2OE's 3.3V symbol" happens to sit right
above `DXL_DATA` node, visually looks like connected as one piece. **Listed as pending change: Move that 3.3V symbol away,
route around**, eliminate ambiguity. Drawing method issue, not changing electrical connection.

**③ "For default conduction, 1OE# should tie low" — Default non-conduction is intentional, not adopted**

`1OE#` active low, `R4` pulls it **high**, meaning channel1 (`MCU_TX → bus`)
**default disabled**. This is intentional:

- During power-on until firmware runs, and during MCU failure/reset, this board **must not drive bus**.
  Bus is shared by 15 servos; once held, entire chain communication fails.
- If `1OE#` permanently tied low, `MCU_TX` constantly feeds bus, idle high level holds bus dead,
  servos can never return packets — half-duplex fails.

During transmission MCU pulls `DE` **low**. STM32's USART hardware Driver Enable supports this polarity,
`CR3.DEP = 1` one bit solves it, no need for external inverter.

> This item aligns with 2026-09-06 item ① (community feedback requiring DE pull-up) direction —
> That time adding R4 was precisely to ensure "default non-driving bus", cannot reverse here.

**TODO**: In schematic move `U3.7`'s 3.3V symbol away, avoid visual connection with `DXL_DATA` node.

### 2026-09-06 · Community Feedback (Three Items, Two Already Changed)

**① DE pin needs pull-up — Changed, this was oversight**

`U3.1 (1OE#)` is low-enable for transmit. During power-on and reset **PA1 is high-Z**, this pin floats;
**if floats low, transmit buffer turns on, this board feeds data onto bus, colliding with all 15 servos.**

Added **R4 10kΩ, DE → 3.3V**.

Previously adding R1 pull-up on `SPI_CS` was exactly same reasoning, **but didn't generalize to DE**, truly an oversight.

**② Two `DXL_DATA` should connect together and pull up — Changed**

Electrically already same network (`U3.5 2A` / `U3.6 1Y` / `J1.3` / `J2.3`),
but drawing shows as two separate short lines each with label, **looks disconnected**, drawing method not intuitive enough. Changed to explicit connection node.

Pull-up item also correct: When bus idle no driver, `2A` this CMOS input floating causes shoot-through current,
UART may also interpret noise as start bit — this board should define idle level itself, shouldn't depend on host.
Added **R5 10kΩ, DXL_DATA → 3.3V**.

**③ Leave only one 3P connector — Not adopted, rationale see item 3 above**

This suggestion applies when board is definitely leaf node; but if board is first node in chain must have both connectors,
and official topology cannot be confirmed. Retain both, handle current with copper width.

Post-change review: `DE` changed from 2 pins to 3 pins, `DXL_DATA` changed from 4 pins to 5 pins,
entire drawing no floating pins, no shorts, no duplicate net names.

### 2026-09-06 · Cross-Check Against Official HAT Review (Add Two Protection Stages)

Read [`pollen-robotics/elec_RPI_Robot_HAT`](https://github.com/pollen-robotics/elec_RPI_Robot_HAT)
`dynamixel.kicad_sch` component-by-component, compared with this board.

**HAT's TTL Half-Duplex Circuit**

```
                        ┌── R31 10k ── +3V3
   IO_15 (RX) ◄── U6 ───┘                    ┌── R32 10k ── +3V3
              (1G125, OE# active low)         │
                        ▲                     ▼
   IO_14 (TX) ──┬── U7 ──────────────────────► ● ── R33 150R ──┬── TH1 100R ── J13.3 / J14.3
                │  (1G126, OE active high)                  D4 5V1
                └── R27 10k ── Q1 (PNP) ── Dynamixel_dir        ⏚
```

**Item-by-Item Comparison**

| | Official HAT | This Board | |
|---|---|---|---|
| RX/TX buffer | 1G125 + 1G126 two chips | SN74LVC2G241 one chip | Same logic |
| Direction control | Auto-derived from TX (Q1 + RC) | **USART2 hardware DE** | This board better, see below |
| Bus pull-up | R32 10kΩ | R5 10kΩ | Consistent |
| **Data line series resistor** | R33 150Ω + TH1 100Ω | ❌ None → **Added R6 150Ω** | |
| **Data line clamp** | D4 5V1 | ❌ None → **Added D2 5V1 (same part `C151348`)** | |
| Power protection | None (`+BATT` direct to connector) | F1 PPTC + D1 TVS | This board better |

**Why direction control is better on this board**: HAT derives direction from TX, relies on RC timing to return to receive state,
**releasing early truncates last stop bit** — this is common disease of TX-derived direction schemes.
This board uses `USART2`'s hardware DE, `DEAT`/`DEDT` adjustable in 1/16 bit increments, no such issue.

**Data line topology after completion** (same structure as HAT):

```
U3.6 (1Y) ──┬─ DXL_DATA ─┬── R5 10k → 3.3V
U3.5 (2A) ──┘            │
                         └── R6 150Ω ──┬── DXL_BUS ── J1.3 / J2.3
                                       │
                                    D2 5V1
                                       ⏚
```

| Component | Function |
|---|---|
| **R6 150Ω** | Source series resistor. Limits short-circuit current, softens edges for EMI reduction. At 1 Mbps delay about 3% bit time |
| **D2 5V1** | Clamp. Cathode to `DXL_BUS`, anode to ground |

**Pull-up stays on buffer side** (R5 before R6): This way even without bus connected, `U3.5 (2A)` this CMOS input
has defined level, won't float. Consistent with HAT's `R32` position.

> Official HAT uses `R33 150R + TH1 100R` series total 250Ω, plus `D4 5V1` clamp
> (`dynamixel.kicad_sch`). **D2 same model as HAT's D4, different package** — HAT is SOD-323 (`C151348`), this board uses SOD-123 (`C151588`) for easier rework.
> This board only uses 150Ω, because HAT side already has 250Ω, too much series on bus slows edges.

## Three Stages of Overvoltage Protection

1. **F1** PPTC 200mA hold / 24V — Current limit, prevent short circuit
2. **D1** TVS SMF12A — Clamp, prevent servo back-EMF spikes
3. **U4** HT7533-1 withstand 30V — Even if first two stages fail, LDO itself survives

## Layout Highlights

- 2-layer board, 45 × 22mm, SMD components all on top
- 3 connectors (J1/J2/J3) lined on same edge, through-hole
- Two M2 mounting holes, spacing 34mm (compatible with original `banana_pcb_locker`)
- C1/C2 close to U2's Vdd / Vdd_IO pins, traces as short as possible
- C4 (100nF) with C5 (4.7µF) close to U1's pin 4 — ST DS12992 Fig.13 requires "as close as possible to pins, or on PCB back directly opposite pin"
- DXL_DATA traces widened, away from SPI
