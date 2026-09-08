# Community Intelligence

**Recorded**: 2026-08-31
**Data sources**: X (read via logged-in session, public posts), GitHub API, public media reports

> ⚠️ This document distinguishes **verifiable facts** from **unverified rumors**. Any entry marked 🔴
> is second-hand information; do not base decisions on it.

---

## 1. Timeline

| Date | Event |
|---|---|
| 2026-08-27 | Pollen Robotics × Hugging Face launch Microduck, $399 pre-orders open |
| 2026-08-27 | Software stack (Rust runtime + RL training) open-sourced under Apache-2.0 |
| 2026-08-30 | Community starts requesting STEP files, 3D print sources, power board schematics on GitHub |
| 2026-08-31 | Thomas Wolf: *"Microduck reached the Shopify UI limit"* |
| **Before Christmas 2026** | **First shipments (North America / Europe / UK)** |

> 📌 **Key context: as of 2026-08-31, no physical units exist in the wild.**
> All public "teardowns" so far are **inference from software and specs**, not physical disassembly.

---

## 2. Engagement Data (Verifiable)

### Launch Announcement Posts

| Account | Likes | Views |
|---|---|---|
| [@ClementDelangue](https://x.com/ClementDelangue/status/2092931447644442635) (HF CEO) | 11,974 | 4.49M |
| [@Thom_Wolf](https://x.com/Thom_Wolf/status/2092923071829049592) (HF co-founder) | 7,833 | 2.56M |
| [@pollenrobotics](https://x.com/pollenrobotics/status/2092915032052879425) | 4,761 | 1.56M |

### GitHub Repository Growth

| Repository | 2026-08-28 | 2026-08-31 |
|---|---|---|
| `pollen-robotics/microduck` | 938 ★ / 87 fork | **3,508 ★ / 418 fork** |
| `pollen-robotics/microduck_rl` | 280 ★ / 33 fork | **876 ★ / 152 fork** |

3.7× growth in three days.

### Sales

- 🔴 "Broke $1M revenue in 7 hours" — from [@DataChaz](https://x.com/DataChaz/status/2094028777634533518) (826 likes),
  **not official data, unconfirmed by Pollen**
- ✅ Thomas Wolf himself posted on 8/31 that *"Microduck reached the Shopify UI limit"* — indirectly confirms very high sales
- 🔴 A Chinese account claimed "$2.6M in 24 hours, one every 4 seconds" — **this account is also promoting meme coins, low credibility**

---

## 3. ⭐ External Verification: Independent Confirmation of Main Board Conclusion

On 2026-08-31, [@tspy](https://x.com/tspy/status/2094249218735300630) published "Microduck Physical Architecture Teardown" (169 likes):

> Actuators: 15 × Dynamixel XL330 · Main board: **Radxa ZERO 3W / Rockchip RK3566** ·
> 1 GB RAM + 32 GB · Dual 6-axis IMU · 8×8 multi-zone ToF LiDAR · Front wide-angle camera ·
> 2× NFC antenna · Battery NP-F550 · Control frequency 50 Hz

**Matches this repository's [Hardware Teardown](hardware-teardown.md) conclusions exactly**, including the
critical one: **Radxa Zero 3W**. Two independent paths, one answer.

In the same thread, @tspy's answer to the cost question also matches this repository's conclusion:
> **"Cost seems higher than direct purchase"**

### This Repository's Depth Advantage Over Community Analysis

| Layer | Public community analysis | This repository |
|---|---|---|
| Spec list | ✅ | ✅ |
| Main board model | ✅ | ✅ |
| **Bus electrical layer & protocol** | ❌ | ✅ TTL half-duplex 1 Mbps, Protocol V2 |
| **Chip I²C addresses** | ❌ | ✅ codec 0x18 / ToF 0x29 / BMI088 0x19 |
| **IMU data block layout** | ❌ | ✅ register 124, 12 bytes, byte-by-byte |
| **Assembly geometry** | ❌ | ✅ exploded views + CAD assemblies |
| **Fasteners** | ❌ | ✅ M2 system, hole-scan derived |

---

## 4. What the Community Built (First 4 Days Post-Launch)

All verifiable public projects:

| Author | Contents | Reception |
|---|---|---|
| [@onusoz](https://x.com/onusoz/status/2093763495846441348) | Trained 1.3M-param small LLM to output emotion labels, then used **Rust synthesizer** to procedurally generate R2D2-style sound effects, runs in browser | 543 ♥ |
| [@Thom_Wolf](https://x.com/Thom_Wolf/status/2092959363992326236) | vibe-coded image detection integration, robot **tracks laser pointer** | 3,065 ♥ |
| [@cdngdev](https://x.com/cdngdev/status/2093721082582933639) | Trained it to play "find needle in 5 million haystacks" game | 852 ♥ |
| Anonymous | Trained **backflip** (retweeted by Clem) | 3,714 ♥ |
| [@__Rhodium__](https://x.com/__Rhodium__/status/2093404140454265205) | Trained **breakdancing** | 610 ♥ |

> Shows the platform's playability and low barrier to customization — all done within days of getting the software stack.

---

## 5. What the Community Is Requesting on GitHub

| Issue | Request | Official response |
|---|---|---|
| [#216](https://github.com/pollen-robotics/microduck/issues/216) | **STEP files / editable CAD** | Matthieu: "This would be great but not our current priority" |
| [#208](https://github.com/pollen-robotics/microduck/issues/208) | **3D print STL sources** | Closed with: upstream RL repo already has STLs |
| [#230](https://github.com/pollen-robotics/microduck/issues/230) | **Power board schematic** (HAT) | **Later updated** — board is now open source @ [`elec_RPI_Robot_HAT`](https://github.com/pollen-robotics/elec_RPI_Robot_HAT) |

> 📌 The first two remain open. Pollen's position (paraphrased): hardware is *partially* open —
> software first, hardware CAD "maybe later".

---

## 6. Media Coverage

| Outlet | Link | Headline |
|---|---|---|
| VentureBeat | [article](https://venturebeat.com/ai/...) | "Hugging Face's $399 robot duck learns to walk with reinforcement learning" |
| TechCrunch | [article](https://techcrunch.com/...) | Pollen co-founder Matthieu Lapeyre told them **not to call it "open-source hardware" (for now)** |
| The Verge | [article](https://theverge.com/...) | "Microduck is a $399 AI duck that runs on open-source software" |

> ⚠️ All press coverage emphasizes "open-source software", but Pollen **actively clarified** that the
> hardware side is not to be called that.

---

## 7. ⚠️ Scam Warning

Several fake "Microduck pre-order" domains have appeared, some using Pollen's images and product copy.

**The only official purchase page is**: https://pollen-robotics.com/microduck/ → redirects to Seeed Studio.

When in doubt, check the official X account [@pollenrobotics](https://x.com/pollenrobotics) or
GitHub org `pollen-robotics`.

---

## 8. This Repository's Position in the Ecosystem

### What distinguishes this repo from other community work

| | Approach | Advantage |
|---|---|---|
| **HF 3D anatomy** ([@mishig25](https://huggingface.co/spaces/mishig/microduck)) | Interactive WebGL viewer (36k views), beautiful, intuitive | **Accessibility** — zero install, works on phones |
| **This repository** | Assembly relationships, exploded views, CAD exports, electronics recovery | **Technical depth** — everything needed to actually replicate |
| **Reddit / X discussions** | Specs, price comparisons, "can I build one" threads | **Community vibe check** |

All three serve different needs; this repository's niche is **the technical manual for replication**.

### Repository traffic (first week)

- 36 countries/regions
- 1,700+ pageviews
- Top sources: GitHub search, Bing (not Google — English discoverability issue)
- Visitors hitting Chinese docs without translation bounce quickly

> This motivated the English documentation effort.

---

## 9. License and Compliance Landscape

| Component | License | Commercial use |
|---|---|---|
| Software (`microduck` repo) | Apache-2.0 | ✅ Allowed |
| RL training code | Apache-2.0 | ✅ Allowed |
| **3D models** (STLs / MJCF) | **CC BY-NC-SA** | ❌ **Non-commercial only** |
| HAT board (KiCad project) | Apache-2.0 | ✅ Allowed |

> ⚠️ The CC BY-NC-SA clause **blocks commercial derivatives** of the geometry. Selling prints or
> modified versions requires negotiating with Pollen. Personal replication for learning is fine.

---

## 10. Unanswered Questions

- **Why NP-F550 battery, not 18650 packs?** (legacy from photography gear? Pollen has not said)
- **Will the `imu_to_dxl` board design be published?** (so far, no)
- **When will STEP files be released?** (Pollen: "maybe later")
- **What's the profit margin at $399?** (speculation ranges from $50 to $150 per unit; no official word)

---

## Sources

All claims in sections 1–4 are cited with X post URLs or GitHub issue numbers. Traffic data from
this repository's own analytics. License claims verified against upstream LICENSE files and
README text.

Last updated: 2026-09-03
