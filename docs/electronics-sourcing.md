# Electronics Sourcing List (Domestic China / Taobao)

> **Snapshot date: 2026-09-04.** Prices and sales volumes change, links may expire, verify pricing before ordering.
> Mechanical parts (bearings / fasteners / consumables) see [Mechanical Sourcing](mechanical-sourcing.md). List](机械采购清单.md).
>
> This list doesn't include endorsements, only search result records. Shop information (sales, years) are displayed values at time of scraping.

---

## First Get Clear: Where Are All the Boards

This directly affects what cable specs you buy, **easy to get wrong**:

```
Head (jaw_soft rigid body in MJCF)
 ├── Raspberry Pi form factor main controller  ← Main controller in head, not trunk
 ├── RPI Robot HAT                             ← Stacked 7.3 mm above controller
 ├── IMX219 camera + M12 lens                  ← ~13 mm from controller center, same rigid body, no joint between
 └── Speaker

Trunk (trunk_base)
 ├── Two hip_yaw XL330                         ← Cavity mostly occupied by these two
 ├── trunk_base bottom plate
 └── NP-F battery — external mount on rear/lower body, 71 mm long with ~40 mm overhanging shell,
      held by two power_support pieces from outside

Through 3-DOF neck: servo bus + battery +BATT power line
```

**Evidence**: `robot_walk.xml:235` `mesh="elec_rpi_robot_hat_pcb"`, `:247` `mesh="pcb__raspberry_pi_zero_2_w"`
both geoms in `jaw_soft` body; two 65×30 boards X/Y fully aligned, Z spacing 7.3 mm, standard Pi + HAT stack.

> ⚠️ **So MIPI cable doesn't cross neck, buy shortest sufficient.** Real cable fatigue risk is **servo bus and power lines crossing neck** —
> those are current-carrying lines, risk more worthy of attention than signal cables. And official **hasn't published any wiring documentation**.

---

## One: Main Controller · Radxa Zero 3W

### Which Config to Buy

**Conclusion: 2G / 16G eMMC.** Derivation basis:

**Storage Side**

| Item | Volume | Basis |
|---|---|---|
| Armbian headless rootfs | ~2 GB | `deploy/README.md:144` confirms headless image; **2 GB is external experience value, no repo evidence** |
| **ONNX Runtime** | **~20 MB** | `docs/design/robotd-design.md:420` verbatim: "ONNX Runtime is a board prerequisite … **~20 MB** in every artifact would enlarge every update for nothing"—**board-level preinstalled, not in release packages** |
| **GStreamer stack** (`mediad` uses) | **~100 MB** | `docs/design/updater-design.md:901` "the GStreamer stack for `mediad`, which is **around 100 MB of apt**" |
| RKNN runtime `.so` | Unknown | No repo evidence, don't make up numbers |
| daemon artifacts + retained one old version | ~10¹ MB scale | `Cargo.toml:79` "**model artifacts will dwarf a few MB of binary**"; `keep_previous = 1` |
| **9 locomotion policy ONNX** — each **793,685 B ≈ 775 KiB** | **7 MB** | Measured. `alpha_stand` / `alpha_walking` / `alpha_sitstand` / `alpha_ground_pick` / `ball_kick_left` / `ball_kick_right` / `roller` / `roller_crouch` / `roulade` |
| `duck_detect.onnx` | 10.0 MB | Measured 10,477,940 B |
| `pet_detect.onnx` | 19 KB | Measured 20,201 B |
| journald cap `SystemMaxUse=200M` | **Doesn't consume disk under default config** | `/var/log` is zram device (`deploy/README.md:14/402/437`). But README:454 provides fallback: after disabling `armbian-zram-config` logs fall to eMMC |
| **Total** | **≈ 3 – 4 GB** | |

→ **8 GB eMMC sufficient, 16 GB comfortable, 64 GB pointless.**

> 💡 **Update mechanism doesn't need dual rootfs.** But **don't call it A/B** — upstream `docs/design/updater-design.md:46`
> specifically clarified: "Update granularity | **Application-level (not A/B image)** | Only daemon + model
> change; OS is static. A/B (RAUC/Mender) would be over-engineering."
> Actually **version directory + `current` symlink atomic switch (`rename(2)`) + health gate + rollback**,
> `keep_previous = 1` only retains one old daemon. (`updater.toml` `tag_prefix` actual value is `daemon-v`.)

**Memory Side**

| Item | Usage |
|---|---|
| Armbian headless | ~250 MB (external experience value) |
| robotd + ONNX Runtime + RKNN | Small — policy model only 775 KiB/each |
| Camera / GStreamer buffer + `duck_detect` | ⚠️ **`duck_detect.onnx` runs CPU not NPU** (`robotd.toml:324` "a `.rknn` runs on the NPU, an `.onnx` on the CPU"; NPU version `.rknn` not in repo). And **RK3566's NPU and CPU share DDR, no dedicated VRAM** |
| ⚠️ `/var/log` zram device — **eats RAM not disk** | Need reserve |

→ **1 GB can run but tight, 2 GB comfortable, 8 GB pure waste.**

> ⚠️ **Official published is 1 GB / 32 GB, but that doesn't mean replication should also buy this tier.**
> Official 2026-08-27 release page lists **1 GB / 32 GB eMMC** (exactly one Radxa Zero 3W SKU,
> also reverse-confirms main controller judgment); but `microduck/` source code nowhere mentions board SKU — source code inherently doesn't reflect this.
> **By above volume derivation, replication recommends 2G/16G**: 1 GB needs to share memory with zram log device, runs tight.

### Channels — Domestic E-commerce Prices All Inflated

Official distributor [ALLNET China](https://shop.allnetchina.cn/products/copy-of-radxa-zero-3w) real pricing (USD, ~¥7.2/$):

| Config | Price | Approx |
|---|---|---|
| 1G / no eMMC | $18 | ¥130 |
| 1G / 8G eMMC | $22 | ¥158 |
| **2G / 16G eMMC** | **$32.90** | **¥237** ⭐ |
| 4G / 32G eMMC | $50 | ¥360 |
| 8G / 32G eMMC | $70 | ¥504 |

(With headers +$1. ALLNET doesn't stock 8G/64G spec.)

Domestic available:

| Shop | Config | Price | Link |
|---|---|---|---|
| findboard旗舰店 | 8G+64G "package 6" (includes camera/power/32G card/HDMI cable/heatsink) | **¥1469** | [tmall 752703346347](https://detail.tmall.com/item.htm?id=752703346347) |
| findboard旗舰店 | 8G+64G bare board | ¥1149 | Same as above |
| 鹭控自动化工控 | Not specified (negotiate price) | ¥690 | [taobao 1044607515575](https://item.taobao.com/item.htm?id=1044607515575) |

**3–6× markup.** Recommend going through ALLNET China, AliExpress or Radxa official channels; domestic can also ask
[mcuzone 杭州野芯科技店](https://item.taobao.com/item.htm?id=1055301489376) (made complete Zero 3W expansion board set, likely has bare board source).

### Don't Buy Packages

| Component | Worth It |
|---|---|
| Camera (Radxa's own IMX219, sold separately ¥129.99) | ❌ Generic IMX219 ¥32.8 available |
| 32G microSD | ❌ With eMMC won't use |
| Power adapter | ❌ Robot is battery-powered, only for bench debugging |
| microHDMI cable | ❌ Microduck runs headless |
| Heatsink | ✅ Useful, but buy separately [¥19.99](https://item.taobao.com/item.htm?id=788387010174) |

---

## Two: Main Controller Alternative · LicheePi Taishan (Evaluated, **Doesn't Fit**)

Also **RK3566**, stable domestic source, often asked about. **But conclusion is can't use.**

**Real price** (true price after clicking "2G+16G single board" SKU; list page ¥22 / ¥45 / ¥195 all lowest SKU bait):

| Channel | Config | Real Price | Link |
|---|---|---|---|
| RK开发板 (9-year shop) | 2G+16G single board | **¥428** (+¥10 shipping) | [taobao 1017038620459](https://item.taobao.com/item.htm?id=1017038620459) |
| 电子元件shop (12-year shop, 46 purchases) | 2G+16G single board | **¥480.8** (free shipping) | [taobao 800894187326](https://item.taobao.com/item.htm?id=800894187326) |
| 立创商城官方 | 2G+16G | **¥580** (was ¥228, raised for DDR price increase) | [item.szlcsc.com/20558171](https://item.szlcsc.com/20558171.html) |
| **Compare: Radxa Zero 3W** | 2G/16G | **≈¥237** | ALLNET China |

**Four disadvantages:**

| | Taishan Pi | Radxa Zero 3W |
|---|---|---|
| **Does it fit** | ❌ **Doesn't fit** — Main controller installed **in head**, head shell overall only 58.8 × 91.8 × 122.7 mm,<br>controller slot in X direction only 30 mm, with HAT stacked above. Taishan **70 × 45** no space | ✅ 65 × 30, designed for this slot |
| 2G+16G real price | ¥428–580 | **≈¥237** |
| **Armbian image** | ❌ Official deployment scripts entire suite based on Radxa Armbian image<br>(`overlay_prefix=rk35xx`, `armbian-zram-config`, netplan, updater), need rewrite | ✅ Direct use |
| **Device tree overlay** | ❌ All need rewrite — 40-pin pinmux is **board-level mapping**.<br>Microduck depends on `/dev/ttyS2`, `i2c3 @400kHz`, `i2s3_2ch`, MIPI CSI | ✅ Ready-made |
| **HAT mechanical compatibility** | ❌ 70×45 vs Pi Zero 65×30, header positions don't match, **HAT needs redraw** | ✅ Both Pi Zero form factor |
| SoC / NPU / rknn | ✅ Both RK3566, kernel and NPU work reusable | ✅ |
| WiFi | AP6212 (WiFi4 / BT4.x) | WiFi6 / BT5.4 |

**But its open source materials worth keeping** (don't need to buy board to download): complete schematic + PCB in **JLCPCB EDA format**,
same RK3566's power tree, DDR routing, AP6212 connection all directly viewable — when drawing `imu_to_dxl`
or redrawing HAT later is ready-made high-quality reference.

- [LicheePi Open Source Hardware Platform · Taishan Pi Project](https://oshwhub.com/li-chuang-kai-fa-ban/li-chuang-tai-shan-pai-kai-fa-ban)
- [Documentation Download Center](https://openkits-wiki.easyeda.com/zh-hans/tspi-rk3566/download-center.html)

---

## Three: Camera · IMX219 (MIPI CSI)

Camera and main controller **both in head, same rigid body**, `setup-board.sh:748` uses overlay
`radxa-zero3-rpi-camera-v2`, `setup-rkaiq.sh:147` `SENSOR = imx219`.

> ⚠️ **It's rotated 90° not inverted 180°.** `mediad/src/main.rs:82`:
> "**90, because the head camera is mounted a quarter turn off**, and this is the one place
> that fact is written down." 180° in `docs/project/media-bringup.md:472` is **alpha machine** obsolete data.

| Shop | Price | Sales | Link |
|---|---|---|---|
| DECXIN旗舰店 | ¥32.8 | 100+ | [tmall 775872575316](https://detail.tmall.com/item.htm?id=775872575316) |
| 德创信摄像头 | ¥32.8 | 27 | [taobao 771618094322](https://item.taobao.com/item.htm?id=771618094322) |
| 盛成威科技 (11-year shop) | ¥38 | 84 | [taobao 596521821696](https://item.taobao.com/item.htm?id=596521821696) |
| 亚博智能 (14-year shop, has technical docs) | ¥44 | 15 | [taobao 709927597121](https://item.taobao.com/item.htm?id=709927597121) |
| 树莓派零售商 (free shipping, 48h ship) | ¥68 | 100+ | [taobao 760446989892](https://item.taobao.com/item.htm?id=760446989892) |
| findboard旗舰店 (Radxa official version) | ¥129.99 | 9 | [tmall 874789708454](https://detail.tmall.com/item.htm?id=874789708454) |

### Cable: 15-pin ⇄ 22-pin Adapter (May Need, **Buy Shortest**)

⚠️ **Official manual doesn't give connector spec.** `Radxa ZERO 3W Product Brief` Rev 1.10 §6.3 verbatim only:

> "The Radxa ZERO 3W is equipped with a **1x4-lane MIPI CSI** connector for camera integration.
> This interface is designed to be backward-compatible with standard industrial camera peripherals."

No pin count, no pitch. But Zero 3W is Raspberry Pi Zero form factor, CSI connector on same side as Micro HDMI, layout consistent with Pi Zero,
**very likely narrow 22-pin 0.5 mm**, while generic IMX219 modules come with **standard 15-pin 1.0 mm**.

> 💡 **Buy 15 cm or shorter sufficient** — camera and main controller in same rigid body, separated ~13 mm,
> cable **doesn't cross neck**. Don't buy 30 cm.

| Shop | Description | Length | Price | Link |
|---|---|---|---|---|
| **汇博视捷科技工厂店** | 15-to-22Pin FFC, **4 cm short version** | 4cm | **¥1.88** | [taobao 654755002992](https://item.taobao.com/item.htm?id=654755002992) |
| 摄像头生产厂家 (11-year shop) | Pi Zero Camera Cable | 16cm | ¥1.88 | [taobao 819560127110](https://item.taobao.com/item.htm?id=819560127110) |
| **Tytion** (15-year shop) | Title explicitly states **15P 1.0 to 22P 0.5** | 15cm | ¥2.5 | [taobao 678057486394](https://item.taobao.com/item.htm?id=678057486394) |
| 树莓派精品销售商 (13-year shop, free shipping) | Official cable, 135/200/300/500 mm options | Multiple | ¥5.8 | [taobao 784739658496](https://item.taobao.com/item.htm?id=784739658496) |

---

## Four: ToF Ranging · VL53L8CX (8×8 Multi-Zone)

Firmware supports both **VL53L5CX and VL53L8CX**, auto-detects by revision ID
(`tof/src/sensor.rs:70-100`: `0x0C = L8cx` / `0x02 = L5cx`).
I²C address candidates **`0x29` or `0x52`** (`tof/src/main.rs:87`), on i2c3 @ 400 kHz.
On HAT via **JST SH 1.0 mm 4P** (Qwiic / Stemma QT) connectors J5–J8.

| Shop | Model | Price | Sales | Link |
|---|---|---|---|---|
| 森科源电子 | VL53L8CX | ¥70 | 1 | [taobao 1025775539663](https://item.taobao.com/item.htm?id=1025775539663) |
| 深圳市渱雯电子 | VL53L8CX | ¥74.98 | 0 | [taobao 977907234039](https://item.taobao.com/item.htm?id=977907234039) |
| 都会明武电子 (free shipping, 20k repeat customers) | VL53L7CX / **L8CX** | ¥75 | 100+ | [taobao 942554129061](https://item.taobao.com/item.htm?id=942554129061) |
| 深圳优信电子 (tech docs, free shipping, B2B) | VL53L8CX | ¥78.5 | 31 | [taobao 967946231167](https://item.taobao.com/item.htm?id=967946231167) |
| 勇泰发电子 (11-year shop, free shipping) | VL53L8CX | ¥84.8 | 14 | [taobao 939165242465](https://item.taobao.com/item.htm?id=939165242465) |
| 归吉星旗舰店 | ST official P-NUCLEO-53L8A1 eval kit | ¥98 | 1 | [tmall 725486284128](https://detail.tmall.com/item.htm?id=725486284128) |

> ⚠️ **Don't buy VL53L0X.** Searching "VL53L5CX" mostly returns **VL53L0X — single-point ranging**, firmware doesn't recognize.
> Also note **VL53L7CX ≠ VL53L8CX** (L7 is another chip with 90° FOV), some links mix L7/L8 in one listing, check SKU carefully.
> Seen one ¥8 "VL53L5CX" with title also stating "79GHz radar sensor" — keyword-stuffed junk.

---

## Five: Power

| Item | Shop | Price | Sales | Link |
|---|---|---|---|---|
| **NP-F battery holder (for power extraction)** | 腾壶旗舰店 | ¥17.8 | 99 | [tmall 658975825526](https://detail.tmall.com/item.htm?id=658975825526) |
| **NP-F power adapter plate** (Type-C fast charge) | 拓普赛旗舰店 | ¥19 | 58 | [tmall 926961598924](https://detail.tmall.com/item.htm?id=926961598924) |
| NP-F550 battery body | 唯卓仕 (China 3C certified, with charge indicator) | ¥52.8 first order | 900+ | Search "唯卓仕 NP-F550" |

> 💡 **These two exactly fill upstream's missing piece.** Official only provided printed part `power_support` (×2),
> **no contact model**, power extraction solution originally needed solving yourself — directly buy ready-made adapter plate.
>
> ⚠️ **It's NP-F550 not F970.** Upstream mesh filename `np_f970` is misleading: measured bounding box
> **38.6 × 20.6 × 70.8 mm** is F550 dimensions (Sony official spec: F550 = 38.4×20.6×70.8,
> F970 = 38.4×**60.0**×70.8). Full repo grep only 2 hits, **both NP-F550**
> (`robotd-design.md:572`, `duck-control/src/model.rs:108` "NP-F550, 2S Li-ion"), F970 zero hits.
>
> Battery **external mount on rear/lower body**, 71 mm long with ~40 mm overhanging trunk shell lower edge, held by two
> 83.5 mm long `power_support` pieces from outside.

---

## Six: Two PCBs

### RPI Robot HAT — Can't Buy, Only Fabricate; But You May Not Need It

**Official doesn't sell separately.** Pollen only sells complete Microduck robots, HAT has no retail page. Not found anyone replicated on oshwhub.

**① Faithful Replication** — Official provided complete production files (Gerber + BOM + pick-and-place, Apache-2.0).
Measurement verification: **4-layer board** (`F_Cu / In1_Cu / In2_Cu / B_Cu`), board outline **65.0 × 30.9 mm**, corner **R3.5**,
BOM **47 rows / 123 parts** (including 5 rows 9 parts DNP), actual placement **113 positions**. Plus SMT costs hundreds of yuan up with MOQ.

**② Use Off-the-Shelf Modules Instead — Under ¥50**

HAT actual functions (checked against schematic line by line):

| Function | Still Need? | Off-the-Shelf Alternative |
|---|---|---|
| **Servo bus half-duplex TTL** — U5 `74LVC1G08` + U6 `SN74LVC1G125` + U7 `SN74LVC1G126`,<br>controlled by `DynUART_Tx`/`DynUART_Rx`/`Dynamixel_dir` | ✅ Essential | See table below |
| **Power distribution** — `+BATT` direct to servos; U9 `AP63205` + L4 6.8µH → `+5V` to 40-pin for controller;<br>U3 `XC6206P182` → `+1V8` for codec | ✅ Essential | Off-the-shelf UBEC (5V/3A or higher) |
| **Audio** — U2 `TLV320AIC3104` (i2c3 0x18 + i2s3_2ch), U1 `PAM8406D` amp,<br>MK1 MEMS mic, Wago terminals for 5W speaker and external mic | ❓ Depends on needs | **If no recording/speaker entire block omitted** |
| **Sensor expansion** — J5–J8 JST SH 1mm 4P (Qwiic/Stemma) for ToF | ✅ Needed | Connect directly to i2c3, few wires |
| Onboard U11 `BMI088` | ❌ **Populated but software doesn't use** (`i2c3-pihat.dts:11` "dormant", `:31` "unused but still connected") | — |
| U4 `CAT24C32` EEPROM | ❌ **DNP not populated** — so this board **is not self-identifying HAT** | — |

**Servo connectors**: J13 / J14 = **JST EH 3P** (TTL, through TH1 100R thermistor to `+BATT`);
J3 / J11 = **JST EH 4P** (RS-485, driven by U8 `SIT3088E`, direct `+BATT`), schematic annotated **"3A max"**.

**Half-duplex bus adapter boards** (directly applicable for Feetech route):

| Shop | Description | Price | Link |
|---|---|---|---|
| Fashion脖子 (13-year shop, 90 purchases) | Serial bus servo driver board, **fits ST/SC series** | **¥22** | [taobao 1005305041207](https://item.taobao.com/item.htm?id=1005305041207) |
| 众灵科技企业店 (200+ purchases) | ZLink USB/TTL debug board + bus servo adapter | ¥22.5 | [taobao 570100064201](https://item.taobao.com/item.htm?id=570100064201) |
| 芯板坊 (7-year shop) | Waveshare serial bus servo driver board, fits ST/SC | ¥24 | [taobao 954634564409](https://item.taobao.com/item.htm?id=954634564409) |
| 灵影智能企业店 (200+ purchases) | ST/SC Feetech bus servo TTL driver board `TTL_Adapter_(A)` | ¥26 | [taobao 983866781632](https://item.taobao.com/item.htm?id=983866781632) |

> 💡 **Especially worth considering for Feetech route.** `imu_to_dxl` needs redraw, policies need retraining, device tree also needs changes —
> HAT also should be re-evaluated. ¥22 adapter board + a UBEC replaces two essential functions "servo bus" and "power distribution".
> Cost is loss of onboard audio.

### imu_to_dxl — DIY Design

STM32G031F8P6 + LSM6DSV16X + half-duplex buffer, externally appears as bus slave (Dynamixel side ID 200).
Protocol and register layout fully reverse-engineered, see [Hardware Design Reverse Engineering](硬件方案逆向.md).

- **Fabrication cost**: 2-layer board, JLCPCB ~¥25–40 (not including assembly)
- **Dual protocol**: To support both Feetech and Dynamixel, **hardware unchanged, firmware branches** —
  both physically identical (half-duplex single-wire TTL / 1 Mbps / 3-wire), difference only in packet format
  (V2 header `FF FF FD 00` + CRC-16; Feetech `FF FF` + `~sum`)
- ✅ **Connector and pinout confirmed by official spec** — Feetech《HL-2915-C002 Serial Spec》A/0 (2026-02-27)
  page 4 items 6-7: connector **`AMP-3`**, **`1=GND / 2=Vcc / 3=Signal/TTL`**, consistent with Dynamixel.
  `HD-1910-C001.stp` connector part name is `AMP-3P`, and assembly contains `HL-2915-C002_ASM`, two models share platform.
  <br>Same spec also confirms: half-duplex async, 8bit/1stop/no parity, **factory default 1 Mbps**, ID 0–253 (default 1),
  signal high level **2–5 V** (this board 74LVC output 3.3 V within window; input tolerates 5.5 V also handles servo's 5 V)
  <br>⚠️ This spec is HL-2915-C002 (**12 V**), electrical parameters not applicable to HD-1910 (5–8.4 V)
  <br>⚠️ "Feetech = TTL single-wire" only applies to STS/SCS/HL — official manual states **SMS series uses RS485, default 115200**
- **Connector housing still needs physical test fit** — Feetech cables marked **5264 terminal**, official HAT uses **JST EH**,
  both **2.5 mm pitch** but different series, latches and housing profiles differ. After servo and cables arrive test once,
  if doesn't fit change J1/J2 from `B3B-EH-A` (`C160259`) to 5264-3P.
  PR #12 version schematic used JST PH 2.0 mm, **pitch is wrong, must change**
- **C6 (LDO input cap) voltage rating 10 V → 25 V** — bus full charge 8.4 V, 10 V margin insufficient (MLCC also has DC bias derating)
- **C5 use 4.7 µF not 100 nF** — ST DS12992 Rev 3 Fig.13 requires for `VDD/VDDA`
  `1 × 100 nF + 1 × 4.7 µF`, two identical 100 nF parallel don't work. Select LCSC basic library `C19666`
- **U1 pins 2, 3 leave active crystal footprint (DNP)** — TSSOP20 doesn't expose `OSC_OUT`, passive crystal can't connect;
  HSI16 accuracy worst −1.85%/+1.55% (Table 41), sufficient for 1 Mbps but margin not large
- **DATA line add TVS** — hot-plugging servos is normal
- **Protocol auto-detect by software, no DIP switches/jumpers** — both headers start `FF FF`, Dynamixel V2 follows with `FD 00`, Feetech follows with `ID`+`LENGTH` (LENGTH always ≥ 2, so `FD 00` unambiguous). First 3 packets vote to lock, then transmit/receive per locked protocol. DIP switches in walking robot are vibration failure points, and this board configured once for life
- **J3 use 6P not 4P** — extra pins 4/5 are `USART1_TX/RX` (U1 pins 16/17 remapped), plug in ¥5 USB-TTL to see printf. G031 has no USB peripheral, adding USB port needs another bridge chip, not worth it

---

## Seven: Cables and Tools

| Item | Shop | Description | Price | Link |
|---|---|---|---|---|
| **Servo bus cables** | 深圳飞特舵机工厂店 | 3-pin 4-pin **5264 connector** | **¥1.8** | [taobao 616460581906](https://item.taobao.com/item.htm?id=616460581906) |
| Servo cables | 鸿翊电子舵机工厂店 | Feetech serial bus, 3-pin/4-pin 5264 terminal | ¥2.5 | [taobao 769633088382](https://item.taobao.com/item.htm?id=769633088382) |
| Servo cables (multiple lengths) | 松甲科技 (300+ purchases) | Female-to-female **5264-3P**, 10/15/20/30/50/70 cm | ¥2 | [taobao 587191439333](https://item.taobao.com/item.htm?id=587191439333) |
| **FE-URT-1 debug board** | 深圳飞特舵机工厂店 (official) | USB to 485/TTL | **¥45** | [taobao 603181554943](https://item.taobao.com/item.htm?id=603181554943) |
| URT-2 debug board | 深圳飞特舵机厂家 | Similar | ¥45 | [taobao 575365901461](https://item.taobao.com/item.htm?id=575365901461) |
| URT debug board (third-party) | 云驱者科技企业店 | Similar | ¥40 | [taobao 712781832610](https://item.taobao.com/item.htm?id=712781832610) |

> **Debug board is essential** — first thing after servos arrive is physically test bus timing.
>
> ⚠️ **This isn't only a Feetech issue.** Official Dynamixel path also is **one response delay per device** —
> `duck-control/src/model.rs:84-86` states XL330 factory `return_delay_time = 250`,
> i.e. **500 µs/device**, "Across **16 devices** that is **8 ms per tick — 40% of a 20 ms budget**",
> official solution is write this register to 0 (in `EXPECTED_REGISTERS`).
> **Feetech side already confirmed**: STS/SCS has `SYNC READ` (`0x82`), and Return Delay Time factory is 0 —
> see [Hardware Design Reverse Engineering · Feetech STS/SCS Protocol Comparison](../docs/硬件方案逆向.md). So more important to physically test, can't assume it's necessarily slower.

---

## Eight: Pitfalls

1. **Don't buy long camera cables** — Main controller and camera both in head, same rigid body, MIPI cable doesn't cross neck. 4–15 cm sufficient.
2. **Camera is rotated 90° not 180°** — see above.
3. **VL53L0X masquerading as VL53L5CX** — search results mostly single-point L0X, firmware doesn't recognize; L7CX also not L8CX.
4. **Radxa bare board markup** — Official distributor $22–70 (¥158–504), Taobao available ¥690–1469. Change channels.
5. **Taishan Pi doesn't fit** — Not only expensive, head physically has no room for 70×45.
6. **¥8 Feetech STS3215 / ¥22 Taishan Pi** — Both SKU bait, click in lowest tier is accessories.
7. **Loctite 243 under ¥15 beware fakes** (see [Mechanical Sourcing List](机械采购清单.md)).

---

## Unverified Items

- All prices, sales, shop information and link validity (2026-09-04 snapshot)
- Whether Pollen sells HAT separately
- Zero 3W's CSI connector is actually 15P or 22P (official docs didn't specify, this document marks as "very likely 22P")
- Taishan Pi specifications, pricing, open source project content
