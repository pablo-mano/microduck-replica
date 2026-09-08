# imu_to_dxl Netlist

> For manual wiring. Component designators consistent with `microduck-replica/BOM.md`.
> Status: Design proposal, not validated with physical hardware.

## Component List (with LCSC Part Numbers)

| Designator | Component | Package | LCSC # |
|---|---|---|---|
| U1 | STM32G031F8P6 | TSSOP-20 | C529334 |
| U2 | LSM6DSV16XTR | LGA-14 | C5267406 |
| U3 | SN74LVC2G241DCUR | VSSOP-8 | C10430 |
| U4 | HT7533-1 (30V input, 100mA) | SOT-89 | C14289 |
| F1 | PPTC MF-NSMF020X-2 (24V / 200mA hold) | 1206 | C210358 |
| D1 | TVS SMF12A | SOD-123FL | C2943870 |
| R1, R2, R4, R5 | 10kΩ | **0603** | C25804 |
| R6 | 150Ω | **0603** | C22808 |
| D2 | BZT52C5V1 5.1V zener | **SOD-123** | C151588 |
| C1–C4, C8 | 100nF 50V | **0603** | C14663 |
| C5 | 4.7µF / 16V X5R | 0603 | C19666 |
| C6 | 10µF / 25V X5R | 0805 | C15850 |
| C7 | 10µF / 10V | 0603 | C19702 |
| J1, J2 | B3B-EH-A (3P **2.5mm**, Dynamixel / official HAT wire) | Through-hole | C160259 |
| **J4, J5** | **B3B-PH-K-S (3P 2.0mm vertical, mates with Feetech `AMP2.0-3P`)** | Through-hole | TBD |
| J3 | SWD+UART 6P header PZ254V-11-06P | Through-hole | C492405 |

## Netlist (23 Networks)

### VDD_BUS — Servo Bus Raw Voltage (2S LiPo 6.0–8.4V)
J1.2 · J2.2 · **J4.2** · **J5.2** · F1.1

> Four connectors' `VDD_BUS` routed with **direct copper strip**: `J1.2 → J2.2 → J5.2 → J4.2` entire path 0.76mm.

### VDD_F — After Fuse
F1.2 · D1 cathode · C6.1 · C8.1 · U4.2 (Vin)

### +3V3
U4.3 (Vout) · C7.1 · U1.4 (VDD/VDDA) · C4.1 · C5.1 · R2.2 ·
U2.5 (Vdd_IO) · U2.8 (Vdd) · C1.1 · C2.1 · R1.2 ·
U3.8 (VCC) · C3.1 · J3.4 · **R7.2**

### GND
D1 anode · C6.2 · C8.2 · U4.1 (GND) · C7.2 ·
U1.5 (VSS/VSSA) · C4.2 · C5.2 ·
U2.2 (SDx) · U2.3 (SCx) · U2.6 (GND) · U2.7 (GND) · C1.2 · C2.2 ·
U3.4 (GND) · C3.2 · J1.1 · J2.1 · J3.1 · **J4.3** · **J5.3** · C9.2 · X1.2

### Reset
| Network | Connection Points |
|---|---|
| NRST | U1.6 (PF2-NRST) · R2.1 |

R2 other end (R2.2) connects to +3V3, forms 10k pull-up.

### Four Pull-up Resistors (R1/R2/R4/R5, Other Ends All Connect to +3V3)

| Designator | Pulls What | Why |
|---|---|---|
| **R1** | `SPI_CS` | During power-on and reset PA4 is high-Z, CS floating may be misinterpreted by IMU as I²C mode, or bus noise as commands. **Must have** |
| **R2** | `NRST` | Chip has internal ~40kΩ weak pull-up, external 10kΩ improves noise immunity. Enhancement |
| **R4** | `DE` | `1OE#` **low enables transmit**. During reset PA1 high-Z, if this pin floats low, transmit buffer turns on, board feeds data onto bus, colliding with 15 servos. **Must have** |
| **R5** | `DXL_DATA` | When bus idle no driver, `2A` this CMOS input floating causes shoot-through current, UART may interpret noise as start bit. Board defines idle level itself, doesn't depend on host |

> R4 and R5 added 2026-09-06 per community review comments. Previously only CS had pull-up,
> **same reasoning not generalized to DE, this was oversight**.

### IMU (SPI 4-wire + 2 Interrupt Lines)
| Network | U1 (STM32) | U2 (LSM6DSV16X) | Other |
|---|---|---|---|
| SPI_CS   | 11 · PA4 | 12 · CS       | R1.1 (R1.2 to +3V3 pull-up) |
| SPI_SCK  | 12 · PA5 | 13 · SCL      | |
| SPI_MISO | 13 · PA6 | 1 · SDO/SA0   | |
| SPI_MOSI | 14 · PA7 | 14 · SDA      | |
| IMU_INT1 | 7 · PA0  | 4 · INT1      | |
| IMU_INT2 | 15 · PB0/PA8 | 9 · INT2  | |

### Half-Duplex Bus (U3 = SN74LVC2G241 Dual Buffer)
| Network | Connection Points |
|---|---|
| MCU_TX   | U1.9 (PA2 · USART2_TX) · U3.2 (1A) |
| MCU_RX   | U1.10 (PA3 · USART2_RX) · U3.3 (2Y) |
| DE       | U1.8 (PA1 · USART2_DE) · U3.1 (1OE#) · **R4.1** |
| DXL_DATA | U3.6 (1Y) · U3.5 (2A) · **R5.1** · **R6.1** — buffer side |
| **DXL_BUS** | **R6.2** · **D2 cathode** · J1.3 · J2.3 · **J4.1** · **J5.1** — connector side |
| **RX_EN** | **U1.1 (PB7/PB8)** · **U3.7 (2OE)** · **R7.1** — receive buffer enable, added 2026-09-07 |

`U3.7 (2OE)` originally tied directly to +3V3, changed 2026-09-07 to **`RX_EN` network**: via **R7 10kΩ pulled up to
+3V3**, while running a line to **U1.1 (PB7/PB8)**.

**Default behavior unchanged** — During MCU reset PB7 is high-Z, R7 pulls `2OE` high, receive buffer remains on as usual,
firmware writes zero lines same as before. This is **reserved**: When wanting to disable echo during transmission, configure PB7 as
push-pull output pulling low, while enabling internal pull-up on `MCU_RX` pin to avoid `2Y` high-Z causing RX float.

### Debug Port J3 (6P, `PZ254V-11-06P` / `C492405`)

| Pin | Network | Source |
|---|---|---|
| 1 | GND | |
| 2 | SWDCLK | U1.19 (PA14) |
| 3 | SWDIO | U1.18 (PA13) |
| 4 | **UART_TX** | **U1.16** `PA11[PA9]` → remapped to **PA9** = `USART1_TX` |
| 5 | **UART_RX** | **U1.17** `PA12[PA10]` → remapped to **PA10** = `USART1_RX` |
| 6 | +3V3 | |

First 4 pins order unchanged, **existing 4-pin SWD cable plugged into 1–4 still works**.

Pins 16/17 remapping via `SYSCFG_CFGR1` (DS12992 Rev 3, Table 12 footnote 4).

**Why no USB**: STM32G031 **has no USB peripheral** (full datasheet search for USB yields zero hits, Development support
only mentions "serial wire debug (SWD)"). Adding USB requires additional CH340/CP2102, not cost-effective on 45×22mm board.
Debug using SWD is sufficient:

| Method | Requires What | Notes |
|---|---|---|
| **RTT** (recommended) | Only J3 pins 1–4 | `probe-rs` supports RTT over CMSIS-DAP, directly reads target RAM, **doesn't halt CPU**, no impact on 50Hz control loop |
| UART printf | J3 pins 4/5 + any USB-TTL | Via USART1 |
| Semi-hosting semihosting | Only J3 | Each printf halts CPU, control loop crashes, **don't use** |
| ~~SWO / ITM~~ | — | **Cortex-M0+ doesn't have**, M3 and above only |

### Servo Interfaces — Two Types Coexist, ⚠️ **Pinouts Are Reversed**

Board has four servo bus connectors, two pairs, three wires parallel-connected (same electrical node),
**but two groups' pin order completely reversed** — because the two families' wire order really are different.

**`J1` / `J2` — `B3B-EH-A`, 2.5mm, Mates with Dynamixel / Official HAT Wire Harness**

| Pin | Network |
|---|---|
| 1 | GND |
| 2 | VDD_BUS |
| 3 | **DXL_BUS** |

**`J4` / `J5` — `B3B-PH-K-S`, 2.0mm, Mates with Feetech HD-1910 Wire Harness**

| Pin | Network |
|---|---|
| 1 | **DXL_BUS** |
| 2 | VDD_BUS |
| 3 | **GND** |

> ⚠️ **Never copy networks from `J1` to `J4`.** Feetech "HD-1910-C001 Serial Specification" A/0
> page 3/7 items 6-7 states wire end is `1=Signal / 2=Vcc / 3=GND`, head-to-tail reversed from Dynamixel side.
> Fortunately **2.0mm and 2.5mm don't mate**, physically prevents wrong connection, only "drawn wrong" remains possible.
> All four connectors' pinouts verified by reading back from PCB netlist 2026-09-08, all correct.
>
> Also note **`AMP2.0` is 2.0mm, `JST EH` is 2.5mm**, Feetech wire doesn't mate with `J1`/`J2`,
> vice versa — which servo family you use, plug into that group.

> 📌 **Pin 3 previously incorrectly recorded in this table as `DXL_DATA`**. After 2026-09-06 adding `R6` 150Ω series resistor,
> buffer side (`DXL_DATA`) and connector side (`DXL_BUS`) became two networks, connector has `DXL_BUS`.
> Corrected 2026-09-08.

## Floating Pins

- U1: **3 (PC15-OSC32_OUT), 20 (PB3/PB4/PB5/PB6)**
  - Pin 3 routed to pad, forms active oscillator reserved position with pin 2 (default DNP), see "Clock" below
  - Pin 20 is only completely idle GPIO (multiplexes PB3/PB4/PB5/PB6 four ports, most flexible), reserved for future use
  - Pin 1 (PB7/PB8) originally idle, repurposed 2026-09-07 as `RX_EN`
  - Pin 2 (PC14-OSC32_IN) connects to `OSC_IN`, pins 16/17 (PA9/PA10) connect to `UART_TX`/`UART_RX`
    — previously incorrectly listed as floating in this table, corrected 2026-09-07
- U2: 10 (OCS_Aux), 11 (SDO_Aux) — auxiliary SPI not used, ST DS13510 allows floating; conservative approach is OCS_Aux to Vdd_IO

**U2.2 (SDx) and U2.3 (SCx) must tie to ground**, cannot float — ST DS13510 Rev 1, Table 1.

## Clock: Use Internal HSI16, Crystal Position Reserved DNP

**This package cannot connect passive crystal.** DS12992 Rev 3, Table 12 (p.36): TSSOP20 pin 2
(`PC14-OSC32_IN`) has additional function `OSC_IN`, but pin 3 (`PC15-OSC32_OUT`) only has `OSC32_OUT`,
**no `OSC_OUT`**; `PF0`/`PF1` not brought out in this package at all. Passive crystal needs two pins to oscillate.

| Approach | Feasibility |
|---|---|
| Passive high-speed crystal | ❌ Missing `OSC_OUT` |
| Active oscillator via HSE bypass (inject into pin 2, pin 3 can be `OSC_EN`) | ✅ |
| 32.768kHz LSE passive crystal | ✅ But cannot supply clock to PLL, can only calibrate HSI16 |

HSI16 accuracy (Table 41, p.64): Factory −0.75%/+0.5%, 0–85°C temp drift ±1%, voltage drift ±0.1%
→ worst case about **−1.85% / +1.55%**. 1 Mbps UART typical tolerance ±3%, sufficient but not much margin.

**Approach**: Pins 2, 3 routed to pads, reserved 3225 active oscillator position + decoupling position, **default DNP**.
If actual test reveals communication problems, populate 16MHz active oscillator.

## Enable Polarity Notes

`1OE#` low enables transmit buffer, `2OE` tied high keeps receive buffer always on.

- PA1 is USART2's hardware DE pin, configure `CR3.DEP = 1` (DE active low), RX/TX switching handled by hardware
- Receive always on means during transmission RX receives own echo, firmware discards based on transmission length
- Alternative is `2OE` also tied to DE (disable receive buffer during transmission), but then during transmission MCU's RX pin floats, false start bit may occur when re-enabled, not recommended
- **2026-09-07 improvement**: `2OE` no longer hard-tied to +3V3, instead via R7 10kΩ pull-up while routed to `U1.1 (PB7/PB8)`.
  Default still always on (behavior unchanged), but firmware has option — when wanting to disable receive, pull PB7 low and enable
  internal pull-up on MCU RX pin to resolve floating. More flexible than tying to DE: whether to disable, when to disable, firmware decides

## HD-1910-C001 Actual Parameters (Feetech Official Product Page, Pre-order Period)

Feetech specially made for duckling, page title reads "**Open-source Duckling Servo** — Serial bus servo, custom developed specifically for duckling AI toy".
Servo body marking: **"3-pin interface · TTL serial bus"**.

| Item | Value |
|---|---|
| Voltage range | **5V – 8.4V** |
| Stall torque | **10 kg·cm = 0.98 N·m** |
| Rated load | **3.0 kg·cm = 0.294 N·m** |
| No-load speed | 0.09 s/60° (110 RPM) |
| **Idle current** | **21 mA** |
| **No-load current** | **140 mA** |
| **Rated current** | **500 mA** |
| **Stall current** | **1.8 A** |
| Dimensions | 34 × 20 × 23 mm |
| Weight | 22.5 ± 2 g |
| Housing | PA + Fiber |

Dimensions same class as XL330 (20×34×26mm), mechanically approximately interchangeable; torque nearly double XL330-M288 (0.52 N·m).

> ⚠️ Don't apply electrical parameters from `HL-2915-C002` spec to this — that's 12V (9–14V) model.
> Only universally applicable are connector definition and protocol sections.

### ⚠️ J1↔J2 Through Path Current

J1 / J2 three wires parallel, **if this board hangs in middle of chain, all downstream servo current flows through board's connectors and copper**:

| Condition | Single | ×15 |
|---|---|---|
| Idle | 21 mA | 0.3 A |
| No-load turning | 140 mA | 2.1 A |
| **Rated load** | **500 mA** | **7.5 A** |
| Stall | 1.8 A | 27 A |

But **JST EH rated current only 3A**.

> F1 (200mA PPTC) **not affected** — it only series in this board's self-power path
> (`VDD_BUS → F1 → VDD_F → U4`), not in J1→J2 through path.

**Choose one:**

1. **Hang this board at chain end** (leaf node), only use J1, leave J2 empty or connect only 1 servo — flows through board only about 12mA. **Recommended**
2. If must place in chain, on PCB route `VDD_BUS` and `GND` from J1 to J2 with **wide copper** (≥2mm or direct pour),
   and verify downstream servo count doesn't exceed 3A

## Feetech Interface Definition (Official Spec Confirmed)

**Source**: Feetech "HL-2915-C002 Serial Specification" A/0, 2026-02-27, page 4 items 6-7.

```
6-7  Connector and Cable
     Type              AMP-3
     Material          PP
     Length            15CM / 25CM
     Pin Definition    1  GND
                       2  Vcc
                       3  Signal/TTL
```

**`1=GND / 2=Vcc / 3=Signal`, identical to Dynamixel → J1/J2 current connections correct.**

Corroborating evidence: In `HD-1910-C001.stp` assembly, connector part name is **`AMP-3P`**, and assembly contains
`HL-2915-C002-20260409_ASM` — two models share platform, interface definition universal.

### Other Usable Items from Same Spec

| Item | Value | Significance |
|---|---|---|
| 7-2 Protocol type | Half Duplex Asynchronous, **8bit / 1stop / No Parity** | Consistent with this board |
| 7-4 Baud rate | 38400 ~ 1 Mbps, **factory default 1 Mbps** | Same speed as Dynamixel |
| 7-3 ID range | 0–253, **factory default ID 1** | 15 servos need individual ID changes; this board uses 200 no conflict |
| 12 **Signal high level** | **2V ~ 5V** | See below |
| 12 Signal low level | 0.0V ~ 0.45V | |
| 6-6 Angle sensor | 12 Bits magnetic encoder | Magnetic encoder ⇒ **little-endian byte order** |

**Signal level check both directions**:

1. This board drives bus — `SN74LVC2G241` at 3.3V supply outputs about 3.3V, falls within servo's required 2–5V high level window ✅
2. Servo drives bus — it may output to **5V**; 74LVC at 3.3V supply **input tolerates 5.5V** ✅
   (Choosing LVC rather than LV/HC precisely for this)

> ⚠️ This spec is **HL-2915-C002 (12V, 9–14V)**, not HD-1910 (5–8.4V).
> **Electrical parameters not interchangeable** — supply range, stall current, overvoltage protection thresholds all different.
> Universal only **connector definition and protocol** sections.

### Background: Feetech Different Series Have Different Physical Layers

Official series memory table manuals (<http://doc.feetech.cn>, interface `getQr/{q}`) original text:

| Series | Level | Default Baud |
|---|---|---|
| **STS** (magnetic encoder) | **TTL single bus** | **1M** |
| SCS (potentiometer) | TTL single bus | 1M / 500k |
| SMS (magnetic encoder) | **RS485** | 115200 |
| HLS (magnetic encoder) | Not specified | 1M |
| SHC | CANopen | 125k |

"Feetech = TTL single-wire" **only applies to STS/SCS/HL branch**, SMS is RS485. Before switching series, check its manual first.

## Protocol Auto-Detection (No Jumpers, No DIP Switches)

Two families' packet headers naturally distinguishable, discrimination point is **bytes 3, 4**:

| | Header | Following | Checksum |
|---|---|---|---|
| Dynamixel V2 | `FF FF` | **`FD 00`** | CRC-16 (poly 0x8005) |
| Feetech STS/SCS | `FF FF` | `ID` + `LENGTH` | `~(ID+LENGTH+INST+params)` |

Unambiguous: Feetech packet byte 3 is ID (can be 0xFD), but **byte 4 LENGTH never 0**
(minimum 2 = instruction + checksum). So `FF FF FD 00` can only be V2.

```
State: Unknown / Locked V1 / Locked V2

When unknown, received FF FF starting packet:
    if byte[2]==0xFD && byte[3]==0x00:
        Parse as V2, calculate CRC-16 → pass records V2 vote
    else:
        Parse as V1, calculate ~sum   → pass records V1 vote
    3 consecutive consistent votes → lock

After locked: Only parse per locked protocol, reply also uses same
Fallback: 100 consecutive packets all parse failures → unlock re-detect
```

All following based on **Feetech official "Servo SCS Communication Protocol" original text**:

- Frame format: **1 start bit + 8 data bits + no parity + 1 stop bit**, total 10 bits
- Header: Two consecutive `0xFF`
- ID range **0~253** (`0x00`~`0xFD`), **broadcast ID = 254 (`0xFE`)**
- Data length **Length = parameter count N + 2**, so **minimum 2, never 0** ← basis for unambiguous discrimination
- Checksum = `~(ID + Length + Instruction + Param1 + … + ParamN)`, over 255 take low byte
- Sync read `0x82`, sync write `0x83`, PING `0x01`, read `0x02`, write `0x03`
- ⚠️ **Multi-byte parameter byte order by servo type**: "Potentiometer type servos **big-endian** (high byte first), magnetic encoder type servos **little-endian** (low byte first)"

Three key points:

1. **Must lock, cannot judge per packet** — bus has noise, some packet corrupted looking like `FD 00` causes oscillation
2. **Reply uses same protocol** — not only parsing branches, packet assembly too
3. **Fallback unlock** — when changing servo brands no need to re-flash firmware

Slave ID both sides use **200**, no conflict with 15 servos (typically numbered 1–15).

> Feetech STS's `Return Delay Time` defaults 0, and manual notes "Not functional on STS" —
> register not effective, reply delay entirely decided by firmware.

## Package Size: All Resistors/Capacitors 0603

All resistors/capacitors use **0603**, D2 uses **SOD-123**, not using 0402/SOD-323.

Rationale not hand-soldering — board has LGA-14 LSM6DSV16X, must use hot air anyway.
**Real rationale is debug period rework**: Series resistor trying 150Ω to 33Ω, removing some pull-up, adding decoupling,
0603 with iron takes seconds to change. First version unvalidated board will definitely need several changes.

Trade-off: 11 components occupy additional about 16mm², less than 2% board area. And these 0603 values all in JLCPCB **basic library**
(10kΩ `C25804` · 150Ω `C22808` · 33Ω `C23140` · 100nF `C14663`), cheaper material cost than 0402 extended library.

> C6 (10µF/25V) stays 0805; C5 (4.7µF), C7 (10µF) originally 0603.

## Data Line Protection (Following HAT's Approach)

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
