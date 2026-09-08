# Mechanical Sourcing List (Domestic China / Taobao)

> **Snapshot date: 2026-09-04.** Prices and sales volumes change, links may expire, verify pricing before ordering.
> Electronic/electrical components (main controller / camera / ToF / power / PCB / cables) see [Electronics Sourcing List](电控采购清单.md).
>
> Quantities from MJCF `robot_walk.xml` mesh instance counts and hole feature reverse derivation, **derived values not official drawings**.

---

## One: Bearings (14 Total, Two Specifications)

Quantity directly counted from `robot_walk.xml` mesh references, six MJCF variants all identical counts.
**Specifications are from measured mesh geometry, not from filenames**:

| Specification | Quantity | Measured (OD / ID / Width) | Mesh |
|---|---|---|---|
| **Ø22 × 16 × 4** | **11** | 22.0 / **16.0** / 4.0 mm | `seeed_bearing__configuration__22x16x4` |
| **Ø15 × 10 × 3** | **3** | 15.0 / **10.0** / 3.0 mm | `seeed_bearing__configuration_default` |

| Shop | Description | Price | Link |
|---|---|---|---|
| **NBZH 永天轴承** (16-year shop) | Stainless steel ultra-thin wall `ET2216ZZ` / `MR16224` / `SET2216` / `DDA2216`, 16×22×4 | **¥2.2** | [taobao 539024647147](https://item.taobao.com/item.htm?id=539024647147) |
| **鑫燚轴承** (7-year shop) | NSK miniature bearings, **same link has both 10×15×3 and 16×22×4** | **¥5** | [taobao 670727787832](https://item.taobao.com/item.htm?id=670727787832) |

> 💡 The 鑫燚 link can buy both specs together in one order, convenient.

> ⚠️ **`bearing_roll` is not a bearing, nor a purchased part.** MJCF also has a mesh appearing 2× called
> `bearing_roll`, measured is **23 × 3 × 40 mm, flat plate with Ø≈18 center hole** (volume 625 mm³, fill rate 23%),
> and from same Onshape Part Studio as `trunk_base` / `yaw2roll` / shells
> (elementId `d6fcdccc8b25aaa256e7e213`), while the two real bearings are in another standard parts library with config strings.
> It's a **bearing cap** for the Ø22 bearing on the hip_roll shaft, **must print, don't buy**. See [Printed Parts](#four-printed-parts).

---

## Two: Fasteners (M2 System, ~325 Pieces)

Entire robot M2-sized holes (Ø1.9–2.5 mm, wrap angle ≥300°) **weighted sum by assembly usage totals 237**:
60 on servo bodies (15 × 4), 21 on bearings/PCBs/battery and other standard parts, ~156 on structural parts.
Below recommends 325 pieces, giving **1.37× margin** on 237 hole positions.

| Specification | Recommended Quantity | Use |
|---|---|---|
| M2×4 socket head cap screw | 60 | Thin-wall positions |
| **M2×6 socket head cap screw** | **80** (workhorse) | |
| M2×8 socket head cap screw | 40 | Hole depth 3–5 mm |
| M2×12 socket head cap screw | 15 | Few deep holes |
| M2 nut | 50 | Non-tapped locations |
| **M2 heat-set insert** | **60** | Recommended for prints, much stronger than direct tapping |
| M2.5×6 | 20 | Few Ø2.7 hole positions |

Derivation process see [Fastener Reverse Derivation](紧固件反推.md).

### Screw Kits

| Shop | Description | Price | Sales | Link |
|---|---|---|---|---|
| **广州信邦电子** (10-year shop, free shipping) | **600 pcs** box stainless steel flat/countersunk socket head **M2/M2.5/M3** combo, **includes wrenches** | **¥26.8** | 42 | [taobao 842110292995](https://item.taobao.com/item.htm?id=842110292995) |
| 宏顺通五金工厂店 (8-year shop) | 600 pcs M2/M2.5/M3 countersunk socket head + nuts + tools | ¥23 | 2 | [taobao 842400332284](https://item.taobao.com/item.htm?id=842400332284) |
| 广州信邦电子 (10-year shop, free shipping) | 304 stainless steel M2–M8 button head socket + nuts + washers kit | ¥19.8 | 71 | [taobao 888186264852](https://item.taobao.com/item.htm?id=888186264852) |

> 💡 **Prioritize kits explicitly containing M2.** Many "M2M3M4M5M6M8 complete" have M2 in title but actually smallest is M3 —
> check SKU before ordering. This project **M2 is absolute workhorse**, M3 basically unused.

### Heat-Set Inserts + Tips

| Shop | Description | Price | Sales | Link |
|---|---|---|---|---|
| **深圳市榕誉微电子** (6-year shop) | **400 PCS M2 + M3** heat-set brass inserts, dual-channel knurled | **¥22** | 200+ | [taobao 1000673642588](https://item.taobao.com/item.htm?id=1000673642588) |
| 深圳市榕誉微电子 (6-year shop) | 80/220 PCS M2–M6 heat-set brass inserts | ¥18 | 1000+ | [taobao 922318148975](https://item.taobao.com/item.htm?id=922318148975) |
| 天卓五金官方旗舰店 (free shipping) | 土八 heat-set inserts M1–M8, buy by spec | ¥0.77 up | 40k+ | [tmall 809364062256](https://detail.tmall.com/item.htm?id=809364062256) |

> ⚠️ **Don't forget tips.** Heat-set inserts need **dedicated soldering iron tips**, regular tips can't press straight, will tilt.

| Shop | Description | Price | Sales | Link |
|---|---|---|---|---|
| **烙铁工具** (7-year shop) | 3D heat-set insert tip kit M2–M8, fits **936 / T12 / T65** | **¥13** | 800+ | [taobao 902798112263](https://item.taobao.com/item.htm?id=902798112263) |
| 外茂优品 (free shipping) | Similar M2–M8 tip kit | ¥11.5 | 200+ | [taobao 966212633086](https://item.taobao.com/item.htm?id=966212633086) |

### Thread Locker (Metal-to-Metal Screw Positions Anti-Loosening)

Servo output shafts, metal-to-metal joints recommend medium-strength thread locker.

| Shop | Description | Price | Sales | Link |
|---|---|---|---|---|
| **LOCTITE 乐泰旗舰店** | Henkel Loctite **243** medium-strength removable threadlock | **¥19.11** | 10k+ | [tmall 653848839737](https://detail.tmall.com/item.htm?id=653848839737) |
| LOCTITE 乐泰企业店 | 243 / 263 threadlock | ¥18.4 | 700+ | [taobao 663963768247](https://item.taobao.com/item.htm?id=663963768247) |
| 工业胶厂家直销 (free shipping) | Loctite 243 / 242 / 263 | ¥16.8 | 600+ | [taobao 925965306702](https://item.taobao.com/item.htm?id=925965306702) |

> ⚠️ **Beware prices under ¥15.** Search results have ¥8, ¥10 "Loctite 243", genuine won't reach this price.
> Look for LOCTITE official stores or authorized distributors.

---

## Three: Consumables

| Material | Usage | Where Used |
|---|---|---|
| **PLA / PETG** | ~300–500 g | Most structural parts |
| **TPU** | Small amount | `jaw_soft_软下巴`, `soft_mouth_top_软嘴顶部` — from naming and use recommend flexible material |

> Print process and tolerance evaluation not yet done, see [PROGRESS.md](../PROGRESS.md) risk list.
> **Simulation STL ≠ printable engineering parts** — only guarantees exterior and inertia, no fit tolerances, threaded holes, heat-set insert seats.

---

## Four: Printed Parts

**30 types / 41 pieces.**

**9 types need multiple copies:**

| Part | Print How Many |
|---|---|
| `leg_腿部` | **×4** |
| `hip_l_髋部左` | ×2 |
| `neck_颈部` | ×2 |
| `power_support_电源支架` | ×2 |
| `sole_left_左脚底` | ×2 |
| `sole_right_右脚底` | ×2 |
| `upper_leg_rigidity_plate_上腿加固板` | ×2 |
| `yaw2roll_偏航转横滚` | ×2 |
| **`bearing_roll_横滚轴承压盖`** | **×2** ← See erratum below |
| Other 21 types | 1 each |

> ⚠️ **Erratum (2026-09-04)**: `bearing_roll` was previously categorized by this repo as "Standard Parts - No Print Needed", **this is wrong**.
> Evidence: ① Measured 23×3×40 mm flat plate with Ø≈18 center hole, not any standard bearing shape (real bearings are bodies of revolution);
> ② From same Onshape Part Studio as `trunk_base` / `yaw2roll` / shells;
> ③ In assembly same body and location as `yaw2roll`, is a cover plate attached to its side.
> Chinese translation "轴承滚轮" (bearing roller) also wrong — it's neither bearing nor roller, changed to "横滚轴承压盖" (roll bearing cap).
> Print count therefore corrected from 29 types / 39 pieces to **30 types / 41 pieces**.

⚠️ **Don't confuse left/right**: `upper_leg_left` and `upper_leg_right` are mirrored parts
(CoM **±0.006766 m = ±6.77 mm**), must print both;
`ankle_left`/`ankle_right`, `sole_left`/`sole_right`, `foot_left`/`foot_right` likewise.

---

## Five: Don't Need to Buy (In Official Model But Other Categories)

| Mesh | What Is It | How to Handle |
|---|---|---|
| `xl330` ×15 | Servos | See [Actuator Selection](执行器选型.md) |
| `np_f970` | NP-F battery (**actually F550**) | See [Electronics Sourcing List](电控采购清单.md#five-power) |
| `elec_rpi_robot_hat_pcb` | HAT circuit board | Fabricate, see [Electronics Sourcing List](电控采购清单.md#six-two-pcbs) |
| `pcb__raspberry_pi_zero_2_w` | Main controller placeholder | Buy Radxa Zero 3W |
| `lens` / `m12_lens_holder` | Lens and lens holder | ⚠️ Both from same Onshape Part Studio,<br>this repo categorized `lens` as standard part, `m12_lens_holder` as printed part, **this classification is questionable**, awaiting physical verification |
| `speaker` | Speaker | 5W, connects to HAT's Wago terminal |

---

## Unverified Items

- All prices, sales, shop information and link validity (2026-09-04 snapshot)
- Screw lengths estimated from hole depth, **are ranges not measured values**; FDM print shrinkage also changes actual hole diameter
  (Ø2.2 typically undersized 0.1–0.3 mm)
- TPU parts judgment from naming and use, official didn't annotate materials
- `lens` / `m12_lens_holder` purchase/print classification
