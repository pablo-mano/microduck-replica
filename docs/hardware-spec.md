# Microduck Hardware Spec Sheet

> 📘 Beginners should start with [Hardware Primer](hardware-primer.md) (board-by-board walkthrough); full derivation in [Hardware Teardown](hardware-teardown.md).

> One-page version. Full derivation process and evidence in [hardware-teardown.md](hardware-teardown.md).
> All data from `pollen-robotics/microduck` source code, device tree and config files — **the code is the datasheet**.

**One sentence**: Radxa Zero 3W (RK3566) running Armbian + one 1 Mbps TTL serial bus with 15
Dynamixel XL330 servos + two custom boards.

---

## System Block Diagram

```
                ┌──────────────────────────────┐
                │   Radxa Zero 3W (RK3566)     │  65 × 30 mm
                │   Armbian · kernel 6.1.115   │  Pi Zero form factor
                └──┬────────┬─────────┬────────┘
      /dev/ttyS2   │        │ 40-pin  │ MIPI CSI
      1 Mbps TTL   │        │  I2C3   │
    ┌──────────────┘        │ 400kHz  └── IMX219 camera (rotated 90°)
    │                       │
    │              ┌────────▼─────────────────┐
    │              │  Pollen RPI Robot HAT    │  65×30mm custom (open source)
    │              │   ├ TLV320AIC3104  0x18  │  audio codec
    │              │   ├ BMI088   0x19/0x68   │  soldered but unused
    │              │   └ Stemma J5 → ToF 0x29 │  VL53L5CX/L8CX
    │              │  Battery power via board  │
    │              └──────────────────────────┘
    │
════╪══════════ Dynamixel bus · Protocol V2 · 1 Mbps ══════════
    │
    ├── 15 × XL330   ID L-leg 20-24 / neck-head-mouth 30-34 / R-leg 10-14
    └── imu_to_dxl v2 board   ID 200   ← LSM6DSV16X
```

**Core design**: one bus does everything. The IMU does not use I²C; it presents as a Dynamixel
slave on the servo bus, read in **the same `sync_read`** as the 15 servos. Battery voltage also
has no fuel gauge — directly read from what the servos report as their own supply.

---

## Main Board

| Item | Value |
|---|---|
| Module | **Radxa Zero 3W** — commercial off-the-shelf, **not** a custom carrier |
| SoC | **RK3566** (4× Cortex-A55 + Mali-G52 + NPU) |
| Size | 65 × 30 mm, Pi Zero form factor, 40-pin Raspberry Pi compatible header |
| OS | **Armbian** (Debian-based), vendor kernel 6.1.115 |
| Policy inference | ONNX Runtime ≥1.23 (ships 1.28.0), `dlopen` loaded |
| NPU inference | rknn-toolkit2 — **disabled by default in Armbian**, needs overlay flash and reboot |

> Basis: device tree `compatible = "radxa,zero-3w", "rockchip,rk3566"`;
> "radxa" appears 182 times in the entire repo.

### Why the Rumor Says Raspberry Pi

1. The custom board is literally named **"RPI Robot HAT"**
2. Pi Zero form factor + 40-pin Raspberry Pi compatible header
3. Camera is **Raspberry Pi Camera v2** (IMX219)
4. **Prototype stage did use Raspberry Pi Zero 2W** — `robotd.toml` comment: 50 Hz control loop
   *"is inherited from the prototype, where it was chosen on a Raspberry Pi Zero 2W"*

The predecessor Open Duck Mini v2 **still uses Raspberry Pi Zero 2W**, because it does no vision or WebRTC streaming.

### Why Linux SoC Required, Not MCU

Must simultaneously run: ONNX policy inference @50Hz, 720p30 H.264 hardware encoding + WebRTC streaming,
NPU object detection, GStreamer congestion control (takes 7.6% of one core), Bluetooth gamepad + phone app + WiFi.
**STM32 / ESP32 cannot do this.**

---

## Actuators and Bus

| Item | Value |
|---|---|
| Motors | **Dynamixel XL330 × 15** (each ~18 g) |
| Electrical layer | **Single-wire half-duplex TTL** (3 wires: data + VDD + GND), 3.3V |
| Protocol | **Dynamixel Protocol V2** |
| Baud rate | **1 Mbps** (EEPROM `baud_rate = 3`) |
| Port | `/dev/ttyS2` = RK3566 **UART2, M0 pinmux** |
| Attached devices | **16**: 15 servos + 1 IMU board |
| Control loop | **50 Hz**, one `sync_read` covers all 16, registers 124–136 |
| Rust library | [`rustypot`](https://github.com/pollen-robotics/rustypot) |

> ❗ **Not RS-232, not RS-485.** RS-485 is differential two-wire; Dynamixel's XM/XH series use it.
> XL330 is TTL single-wire half-duplex. Searching entire repo for `rs485|rs232|direction pin` returns **zero hits** —
> no direction control GPIO in code; direction switching done by hardware automatic direction circuit (on HAT board).

### Servo ID Assignment

```
L leg   20 21 22 23 24    hip_yaw / hip_roll / hip_pitch / knee / ankle
Neck-head-mouth 30 31 32 33 34    neck_pitch / head_pitch / head_yaw / head_roll / mouth
R leg   10 11 12 13 14    (mirror of L leg)
IMU board 200
```

**The 15th servo is the mouth (ID 34, index 9).** All policies are `obs[1,61] → act[1,14]`,
action vector **skips this index** — mouth controlled by upper logic, not policy.

### Bus Reliability (Official Measured)

| Metric | Value | Source |
|---|---|---|
| Drop rate | **~8 times/minute ≈ 0.27%** | *"Measured on a bench robot at ~8 drops a minute"* |
| Official characterization | *"One dropped Dynamixel transaction is **ordinary**"* | `robotd/src/main.rs` |
| Symptom | Random small jerk (dropped read causes policy to miss one beat, return to hold pose) | Same |
| Tolerance | `COAST_TICKS = 3` uses last good sample for 60ms coast | Same |
| Tolerance ceiling | `max_consecutive_errors = 10` | `robotd.toml` |
| Loop stability | `missed=3` / 15022 ticks, maintains 50.0 Hz | `roadmap.md` |
| IMU data staleness | Has dedicated `StaleImuTracker`, comment says *"Known to happen"* | `bus.rs` |

Bus utilization estimated **~25%** (~500 bytes/tick ÷ 1 Mbps ÷ 20ms) —
**bottleneck is not bandwidth, but signal integrity and single point of failure**: single-ended 3.3V signal, 15 servos daisy-chained,
harness passes through the dynamically flexing neck, right next to motor PWM noise sources.

---

## Two Pollen Custom Boards (One Open Source, One DIY)

### ① `imu_to_dxl` v2 — Few Components, Easiest to DIY

| Item | Value |
|---|---|
| IMU chip | **LSM6DSV16X** (ST 6-axis, with SFLP hardware orientation fusion block) |
| Interface | **Dynamixel bus** (not I²C) |
| Bus ID | **200** |
| Register address | **124** |
| Control loop read | 12 bytes |
| Full diagnostic block | 20 bytes (also contains raw acceleration, sample count, status flags) |

**12-byte Block Layout**

| Bytes | Contents |
|---|---|
| `0..6` | gyro x/y/z, `i16` little-endian, ±500 dps, **17.5 mdps/LSB** |
| `6..12` | SFLP quaternion x/y/z, **IEEE fp16**; `w = √(1 − x² − y² − z²)` |

**To DIY**: LSM6DSV16X + small MCU to act as Dynamixel V2 slave
(STM32G0/G4, CH32V203, ESP32 all work) + half-duplex TTL transceiver, power from bus.

### ② Pollen RPI Robot HAT — Officially Open Source, Order Directly

**65.0 × 30.9 mm, board thickness 1.0 mm, 4-layer** (from official KiCad and Gerbers, see [BOM](../BOM.md)).
The STL mesh measurement of 65.02 × 30.02 × 0.84 is simulation approximation, **do not use for ordering**. Uses 40-pin header.

| Component | I²C address | Notes |
|---|---|---|
| **TLV320AIC3104** | `0x18` | TI audio codec, I2S data + I2C control |
| **BMI088** | `0x19` / `0x68` | **Soldered but unused** (comment says "dormant") |
| **ToF** | `0x29` | Not on board, external via **Stemma J5** |

- I²C: 40-pin **pin 3 / pin 5** (`GPIO1_A0`=SDA, `GPIO1_A1`=SCL), **400 kHz**
- Pull-up: **single pair 10k, R12/R13**
- Audio clock: MCLK **12 MHz**, I2S sysclk **12.288 MHz** (256×48kHz)
- Microphone connects **Mic3R** (mono right PGA), all other inputs off
- **Robot internal power via this board**

---

## Sensors

| Type | Model | Interface | Notes |
|---|---|---|---|
| IMU | **LSM6DSV16X** | Dynamixel bus ID 200 | On-chip SFLP outputs quaternion, host does not run fusion |
| ToF | **VL53L5CX / VL53L8CX** | I²C `0x29` @ i2c3 | 15 Hz, multi-zone ranging |
| Camera | **IMX219** (Raspberry Pi Camera v2) | MIPI CSI, I²C `0x10` | **Rotated 90°**, see [Hardware Teardown](hardware-teardown.md#seven-sensors) |

- Camera sensor locked to 1920×1080@30 mode (avoids 21 fps ceiling of boot default mode),
  ISP scaling output, default 720p30 @ 2 Mb/s
- Encoding uses **Rockchip MPP hardware encoder** → WebRTC
- HFOV ~62°

---

## Power and Battery

| Item | Value |
|---|---|
| Battery | **Sony NP-F550** (L-series camcorder battery). ⚠️ Upstream mesh name `np_f970` is misleading; measured size is F550 |
| Configuration | **2S lithium-ion**, nominal 7.2 V |
| Full (under load) | **8.2 V** |
| Empty (under load) | **6.6 V** |
| Fuel gauge | **None** |
| ADC | **None** |

> Comment quote: *"There is no fuel gauge and no ADC. The only measurement available is
> what the servos report as their own supply."*

**Voltage read directly via Dynamixel protocol from what servos report as their own supply** — i.e. "battery as seen from the bus",
sags under load, recovers when idle. So 6.6–8.2 V is **usable range under load**, not cell chemistry range.

Battery EMA (~10 second time constant) drops to 6.6 V → **auto graceful sit-down and shutdown**.

> 💡 Replication-friendly: **no need to design any battery monitoring circuit.**

---

## All Interfaces at a Glance

| Interface | Use | Parameters |
|---|---|---|
| **UART2** (`ttyS2`) | 15 servos + IMU board | 1 Mbps, TTL half-duplex |
| **I2C3** (pin 3/5, M0) | audio codec / ToF / (unused BMI088) | 400 kHz |
| **I2S3** | audio data | MCLK 12 MHz, sysclk 12.288 MHz |
| **MIPI CSI** | IMX219 camera | I²C control `0x10` |
| Bluetooth | gamepad (`padd`), phone app (`btd`) | on-board |
| WiFi | WebRTC streaming, config (`configd`) | on-board |
| USB-C | power + maskrom flashing | **PD negotiation sacrificed by overlay** |

---

## Software-Side Parameters That Must Align

| Parameter | Value | Notes |
|---|---|---|
| Control loop | **50 Hz** | Inherited from Pi prototype, **never revalidated on Radxa** |
| Policy interface | `obs[1,61] → act[1,14]` | Validated on load, 51-dim old version rejected |
| `action_scale` | 0.9 (walking) / 0.8 (skating) | |
| Position P gain | **200** | ×0.8 when standing |
| Action lowpass | head **0.5** / legs **0.7** | **Must match training**, otherwise sim-to-real degradation |
| Voltage adaptation | Off by default | `scale × (7.4 / measured EMA)`, clamped 6.0–9.5 V |

---

## Replication Checklist: Buy vs DIY

| Component | Status | Notes |
|---|---|---|
| Radxa Zero 3W | ✅ Buy direct | Commercial module |
| Dynamixel XL330 × 15 | ✅ Buy direct | **Cost center, $359–€629** (huge channel variation, see [BOM](../BOM.md)) |
| **NP-F550** battery + holder | ✅ Buy direct | Generic camcorder accessory. **Don't buy F970** |
| Raspberry Pi Camera v2 (IMX219) | ✅ Buy direct | |
| VL53L8CX module | ✅ Buy direct | Stemma/Qwiic in stock |
| **`imu_to_dxl` board** | 🔧 **Draw yourself** | Protocol fully recovered, only 3 component types |
| **HAT board** | ✅ Download official Gerbers and fab | 4-layer. Can skip entire board if no recording/speaker, but **half-duplex direction circuit requires separate adapter**, see [Electronics Sourcing](electronics-sourcing.md#six-two-pcbs) |
| Control software | ✅ Official Apache-2.0 | Same main board → **runs directly** |
| Policies | ✅ Official 9 ONNX | Hardware unchanged → work directly |

> 💰 **Cost reminder**: 15 XL330 alone cost **$359–€629** (ROBOTIS international $23.90 start, European inc-VAT €41.95 ceiling),
> already far exceeds whole robot **$399** price. $399 is volume pricing.

---

## Three Mandatory Pitfalls

1. **Armbian runs login console on UART2 by default** — `serial-getty@ttyS2`'s `agetty`
   will hold the serial port. Must `systemctl mask serial-getty@ttyS2`.
   Pollen found it using `fuser -v /dev/ttyS2`.

2. **i2c3 collides with FUSB302 pins** — RK3566's i2c3 in stock DTB runs M1 pinmux
   serving USB-C PD controller; using pin 3/5 M0 requires disabling fusb302, **loses PD negotiation**.
   Fortunately FUSB302 power-on default CC presents Rd, plain 5V charging still works.

3. **NPU disabled by default** — Armbian ships with it off; running rknn models requires flashing overlay and reboot.

---

## Conclusion

**This hardware has no irreplicable parts.** Main board is commercial module; of the two Pollen custom boards,
**HAT board is fully open source by Pollen** (KiCad + Gerbers + BOM + pick-and-place, download and order),
truly DIY-required is only `imu_to_dxl` — and it has few components, protocol fully recovered.

The real barrier is **cost** (15 XL330 more expensive than whole robot price), not technical.

> Data sources: `duck-control/src/{model,imu,bus}.rs`, `robotd/src/main.rs`,
> `deploy/robotd.toml`, `deploy/audio/*.dts`, `deploy/overlays/*.dts`,
> `tof/src/*.rs`, `mediad/src/*.rs`, `docs/design/robotd-design.md`,
> `docs/project/{media-bringup,roadmap}.md`

---

## Official Spec vs This Repository's Reverse Engineering (Cross-Validation)

Pollen published on 2026-08-27, disclosing partial specs. Table below compares **official disclosure** with
**this repository's source code reverse engineering** item by item — both corroborate each other and fill gaps.

| Item | Official disclosure | This repo reverse engineering | Conclusion |
|---|---|---|---|
| SoC | RK3566, quad A55 @1.8GHz, Mali-G52, NPU **0.8 TOPS** | RK3566 (device tree `compatible`) | ✅ Match |
| **Main board module** | **Not disclosed** | **Radxa Zero 3W** | 🔍 **RE-exclusive** |
| RAM / storage | **1 GB / 32 GB eMMC** | No evidence in source | 🆕 **Official-exclusive**: indeed one Radxa Zero 3W SKU.<br>⚠️ Replica recommended 2G/16G, see [Electronics Sourcing](electronics-sourcing.md#which-config-to-buy) |
| Battery | **NP-F550, 2600 mAh, ~1 hour runtime** | NP-F550, 2S, 8.2 V full / 6.6 V empty | ✅ Match |
| ToF | **8 × 8** LiDAR | VL53L5CX / VL53L8CX @ `0x29` | ✅ This series is indeed 8×8 |
| Camera | Front camera with indicator light | **IMX219** (Pi Cam v2), I²C `0x10`, **rotated 90°** | 🔍 RE more detailed |
| **IMU** | **2 (body + head)** | LSM6DSV16X **in use**; BMI088 source annotated **dormant / unused** | ⚠️ **Contradiction, see below** |
| Motors | 15 DOF, articulated grippable beak | 15 × Dynamixel XL330, with complete ID table | ✅ Match |
| Size/weight | Height 25 cm, width **14 cm**, < 800 g | Measured **144 × 141 × 264 mm**, **737.2 g** | ✅ Match |
| **NFC** | **Dual antenna** | **No mention anywhere** in source | 🆕 **Official-exclusive** |
| Audio | Microphone + speaker | TLV320AIC3104 @ `0x18`, mic via Mic3R | 🔍 RE more detailed |
| **Main bus** | **Not disclosed** | TTL half-duplex 1 Mbps, `/dev/ttyS2`, Protocol V2 | 🔍 **RE-exclusive** |
| Connectivity | Wi-Fi, Bluetooth | Same (Radxa on-board) | ✅ Match |

### ⚠️ The Contradiction: Official Says 2 IMUs, Software Uses 1

Official spec says **"2× IMUs (body and head)"**, but in source:

- **Only LSM6DSV16X in use** — `imu.rs` header says *"One IMU, one code path"*
- **BMI088** on HAT board marked **"dormant"** in device tree comment,
  **"(unused but still connected)"**

**Conclusion: second IMU physically soldered on board, but shipping software does not use it.** Can omit entirely for replication.

---

## Open Source Status: Official Position (2026-08 Update)

> CNX Software coverage quote:
> *"While the Microduck's software stack is open source, nothing was said about the
> mechanical and hardware design files, and **Pollen Robotics told the press not to
> refer to the robot as 'open-source hardware' (for now)**."*

**This requires careful parsing.** What Pollen rejected is calling the whole robot "open-source hardware" as a **label**,
not that no board is published — in fact **RPI Robot HAT board is fully open**
([`elec_RPI_Robot_HAT`](https://github.com/pollen-robotics/elec_RPI_Robot_HAT), Apache-2.0,
with KiCad project, Gerbers, BOM, pick-and-place). What's truly not published is the `imu_to_dxl` board,
editable mechanical CAD, whole-robot BOM and assembly documentation.

That **"(for now)"** leaves an opening, no timeline given.

> Correction 2026-09-03: this doc previously wrote "hardware not open source is Pollen's deliberate line"; too strong, now corrected.

### External Verification: Someone Independently Reached Same Main Board Conclusion

On 2026-08-31, [@tspy](https://x.com/tspy/status/2094249218735300630) on X published
"Microduck Physical Architecture Teardown" (169 likes), independently listing:

> Actuators 15 × Dynamixel XL330 · Main board **Radxa ZERO 3W / Rockchip RK3566** ·
> 1 GB RAM + 32 GB · Dual 6-axis IMU · 8×8 ToF · Front camera · 2× NFC antenna ·
> Battery NP-F550 · Control frequency 50 Hz

**Matches this repository's reverse engineering conclusions exactly**, including the critical **Radxa Zero 3W** —
two independent paths, one answer, credibility greatly increased.

> 📌 **Note: as of 2026-08-31 no physical units exist in the wild** (ships before Christmas).
> So all current "teardowns" are **inference from public software and specs**, not physical disassembly.
> This repository's depth (I²C addresses, register layout, 12-byte data blocks, bus protocol)
> exceeds spec sheets by one layer, source is reading Rust source as the spec.

In the same thread, @tspy's answer to the cost question also matches this repository: **"Cost seems higher than direct purchase"**.

### Three Issues the Community Is Pressing (As of 2026-08-31 Pollen Has Not Replied to Any)

| Issue | Request |
|---|---|
| [#175](https://github.com/pollen-robotics/microduck/issues/175) | Want **STEP files** — STL has no normal info, surfaces render as polygon blocks |
| [#173](https://github.com/pollen-robotics/microduck/issues/173) | Want **3D print source files** |
| [#174](https://github.com/pollen-robotics/microduck/issues/174) | Asking if battery can charge in-place, and directly asking **if power board schematic is in open source plan** |

Checked ~10 issues, **Pollen members have not replied to any**, but repo still updates daily.

> This also shows this repository's value: no one in the community has reverse-engineered assembly drawings from MJCF,
> and no one has systematically extracted the electronics stack from Rust source.
