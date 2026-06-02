# jigawatt — Prototype BOM (Rev 1.1: battery-primary, AC-coil contactor, SSR-driven)

**Target site:** single-phase, 230 V, 40 A whole-house feed (India).
**Controller:** Raspberry Pi Pico W (RP2040 + CYW43439 Wi-Fi). Code intended to port to ESP32 variants without ceremony.
**Pricing as of:** May 2026, INR, India retail. All prices exclude GST unless noted; assume +18% on industrial-channel quotes (Industrybuying, IndiaMART) and tax-included on hobbyist channels (Robu, Robocraze, ElectronicsComp).

**Rev 1 (vs. Rev 0):** controller (Pico W + AS3935) runs primarily from a LiFePO4 cell, not from any AC-DC output. Hi-Link modules are sacrificial — they keep the battery topped up and feed the coil rail, but the controller has no direct dependency on them. AS3935 sees a quieter DC rail, controller survives an AC-DC failure during a strike.

**Rev 1.1 (vs. Rev 1):** the contactor is now the readily-available L&K MCX 04 CS97012 with a 240 V AC coil, driven through a Fotek-style SSR-10DA solid-state relay. The HLK-10M24 (24 V coil rail) is dropped — the coil is now switched directly from line-side mains through the SSR. The MOSFET and flyback diode in the driver chain are also dropped; the SSR has internal opto-isolation and is rated for inductive AC loads. Net effect: ~₹950 cheaper, fewer parts, faster Amazon/Electrixzone availability, slightly less "elegant" drive train.

---

## Procurement log

| Date | Item | Ex-GST | Inc. GST (18%) | Source |
|---|---|---:|---:|---|
| Order #1 | SSR-10DA (generic) | 320 | ~378 | (TBD) |
| Order #1 | 12-slot distribution board (generic, indoor IP20/30 class) | 600 | ~708 | (TBD) |
| Order #1 | L&K MCX 04 CS97012BOOO 4P 40A 240V AC coil | 1,880 | ~2,219 | (TBD) |
| Order #1 | Shipping | 250 | ~295 | — |
| **Order #1 total** | | **3,050** | **₹3,599** | |

**Still to procure:** Pico W spare, AS3935 (DFRobot or CJMCU), HLK-5M05, TP5000/TP4056 + LiFePO4 26650 cell + MT3608 boost, PC817 pack, R/C kit, 63A 2P isolator (BF206300), 6A SP MCB (BB10060C), 1A fuse + holder, DIN rail, terminals, glands, wire (6 mm² + 1.5 mm²), ferrules, lugs, mounting plate for controller compartment, perfboard. Estimated remaining spend: **~₹6,000–7,000 inc. GST.**

---

## 0. Topology — where jigawatt physically sits

Most Indian consumer distribution boards are full and sealed — no spare modules for a 40 A contactor and an SPD, no room for two AC-DC modules. jigawatt therefore gets its **own enclosure**, installed **upstream of the existing DB**. The existing DB is untouched.

```
[Utility meter] → [jigawatt enclosure] → [existing DB, untouched] → [house loads]
```

Inside the jigawatt enclosure, line-to-load:

1. Service cable in (from meter / utility tail).
2. **2-pole 63 A isolator switch** — manual disconnect so you can service jigawatt without pulling the meter. This is a DPST (a.k.a. DP / ICDP) isolator; a true DPDT is a transfer-switch shape used for mains↔generator changeover and isn't what we want here.
3. **6 A SP MCB**, branched off the line side, feeding the controller's HLK-5M05 (which in turn feeds the battery charger).
4. **4-pole 40 A contactor** in the main current path, coil switched by the SSR-10DA.
5. Output to existing DB.

**(Position 3 in the chain is reserved for a Type 2 SPD if you add one later — see Section 6.)**

The single Hi-Link module (HLK-5M05), the Pico W, the AS3935, the LiFePO4 cell, the charger, the boost converter, and the SSR all live inside this same enclosure. The HLK sits on the line side of the contactor through the 6 A MCB and feeds:

- the **battery charger input** (5 V into a TP5000 LiFePO4 charger).

The **contactor coil** (240 V AC) is switched directly from the line-side mains through the SSR-10DA. The SSR's control input is driven from the Pico GPIO through a current-limit resistor; the SSR's internal opto provides ~2.5 kV galvanic isolation between the Pico-side DC drive and the mains-side AC switching.

**The Pico W and AS3935 do not run from any Hi-Link output.** They run from the LiFePO4 cell, through a small 5 V boost. The HLK exists only to keep the battery topped up. Two cascaded galvanic barriers separate the Pico from grid at all times — the HLK primary-to-secondary barrier (~3 kV) and the SSR input-to-output barrier (~2.5 kV) — neither of which is a primary power dependency.

**Four-state behaviour:**

| State | Grid | Lightning | Contactor | Battery role | What's happening |
|---|---|---|---|---|---|
| 1a | on | no | closed | trickle-charging | Normal operation. SSR conducts, 240 V AC reaches the coil, contactor closes. Charger keeps battery at float. Net continuous battery draw ≈ 0. |
| 1b | on | yes (trip) | open | trickle-charging | Pico pulls SSR drive low, SSR opens, coil de-energizes, contactor falls open. HLK is still alive on the line side; battery still charging. Controller waits out cool-off. |
| 2a | off | no | open | discharging slowly | HLK dead, SSR off (no AC to switch anyway). Contactor open by virtue of no coil current. Controller runs at ~0.25 W on battery. Sensor keeps listening. |
| 2b | off | yes | open | discharging slowly | Identical to 2a — contactor is already open because grid is gone. Strike just gets logged. |

The contactor only consumes power in state 1a, and in state 1a the grid is supplying the coil current directly via the SSR — never the battery. The battery only ever powers the controller, never the coil. That's what lets the battery shrink to a single LiFePO4 cell.

Implication for the BoM: the "enclosure" is no longer a generic project box. It's effectively a small secondary distribution board — a **DIN-modular, IP65, ~12-module enclosure**, with one half-row reserved as a low-voltage compartment for the controller PCB and the battery holder. The Hager Vector VE112L (12 modules, IP65, double-door) or an L&K SmartLine equivalent is the right shape.

---

## 1. Grid-isolation element (the contactor)

This is the only line that meaningfully changes the device's safety story.

**Rev 1.1 build choice: L&K MCX 04 CS97012BOOO, 4P, 40 A, 240 V AC coil — purchased at ₹1,880 ex-GST (₹2,219 inc. 18% GST).** Conforms to IS/IEC 60947-4-1, 4 NO power contacts, AC-1 rating 40 A, frame size Fr1. ✓ in hand.

The coil is 240 V AC, so it's switched from line-side mains through a solid-state relay (see Section 3) rather than directly from a Pico-driven MOSFET. The contactor itself is mechanically identical to the DC-coil variants — same body, same contacts, same mechanical air gap when open — only the coil winding differs. The fail-safe property is unchanged: any failure that interrupts coil current causes the contactor to open under spring tension.

**Alternative buys, ranked:**

| # | Part | Pole / Coil | Approx. price | Where | Notes |
|---|---|---|---|---|---|
| **A (Rev 1.1)** | **L&K MCX 04 CS97012** | 4P, 40 A, 240 V AC coil | ~₹1,748 | [Electrixzone](https://electrixzone.com/product/lt-lk-4-pole-power-contactor-40a-type-mcx-04-cs97012/) · [Moglix](https://www.moglix.com/l-t-mcx-04-40a-4-pole-power-contactor-cs97012/mp/msn457ml70zn9j) · [Amazon.in](https://www.amazon.in/Mcx-04-Pole-Power-Contactor-Cs97012/dp/B01H8PUMF2) · [Online Balajimart](https://www.onlinebalajimart.in/product/29495727/L-T-MCX-04-Contactor-40A-4-Pole---CS97012) | 4-pole 40 A AC-3, India-made, ubiquitous availability, ferrule-friendly terminals. Needs SSR-10DA driver (Section 3). |
| B | **ABB AF26-40-00-11** | 4P, 40 A, 20–60 V AC/DC wide-range coil | ~₹2,660 | [ABB eMart India](https://shop.in.abb.com/1sbl237001r1100/af26-30-00-11-24-60v50-60hz-20-60vdc-contactor.html) · [Bestomart](https://www.bestomart.com/products/abb-dc-type-contactor-four-pole-af26-40-00-1000022831) | Wide-range DC coil — MOSFET drives directly from a 24 V rail. Cleaner driver chain, +₹912 more than the L&K. |
| C | **Schneider TeSys LC1D40BD** | 4P, 40 A, 24 V DC coil | ~₹5,200 | [IndiaMART](https://www.indiamart.com/proddetail/schneider-contactor-lc1d40bd-2853818100455.html) · [Schneider page](https://www.se.com/in/en/product/LC1D40BW/contactor-tesys-deca-3-poles-ac3-440v-40-a-coil-wide-range-24-v-dc/) | Schneider quality. ~3× the price of the L&K. |

**Google search terms that work:**
- `L&T MCX 04 CS97012 4 pole 40A contactor`
- `ABB AF26-40-00-11 24V DC coil India price`
- `Schneider LC1D40BD 24V DC coil`

---

## 2. Brain and sensing

| Item | Qty | Approx. price (₹) | Where to buy | Search term |
|---|---:|---:|---|---|
| Raspberry Pi Pico W (you have one; buy a spare) | 1 | **650** | [Robocraze](https://robocraze.com/products/raspberry-pi-pico-w) · local hobby shops | `Raspberry Pi Pico W India 650` |
| DFRobot SEN0290 (Gravity AS3935, antenna-tuned) | 1 | 1,800–2,500 | [MakerBazar](https://makerbazar.in/products/dfrobot-gravity-lightning-distance-sensor) · [element14 India](https://in.element14.com/dfrobot/sen0290/lightning-distance-sensor-gravity/dp/3769918) · [Fab.to.Lab](https://www.fabtolab.com/dfrobot-sen0290-gravity-lightning-distance-sensor) | `DFRobot SEN0290 lightning sensor India` |
| **OR** CJMCU-3935 generic AS3935 module (budget) | 1 | 650–1,650 | [DNA Tech](https://www.dnatechindia.com/CJMCU-3935-AS-3935-lightning-sensor-module-india.html) · [Amazon.in](https://www.amazon.in/Pro3D-CJMCU-3935-AS3935-Lightning-Detection/dp/B0CVGQTW2Y) · [Flipkart](https://www.flipkart.com/prime-intact-cjmcu-3935-as3935-i2c-amp-spi-storm-lightning-detection-sensor-electronic-components-hobby-kit/p/itm8080984d740cf) | `CJMCU-3935 AS3935 lightning sensor India` |

**Recommendation:** spring for the DFRobot SEN0290. The Coilcraft MA5532-AE antenna is properly tuned and saves you an evening with a piezo lighter trying to figure out why your noise floor is wrong. The cheap CJMCU module works but I've heard enough horror stories about antenna detuning when mounted near anything metal.

---

## 3. Driver chain (logic ↔ contactor coil)

Rev 1.1 uses a 240 V AC-coil contactor, switched through a solid-state relay. Pico GPIO → 330 Ω current-limit resistor → SSR-10DA control input (3–32 V DC, ~7 mA at 3.3 V). SSR output switches line-side mains through the contactor coil. No MOSFET, no PC817, no flyback diode needed — the SSR has internal opto-isolation (~2.5 kV) and is rated for inductive AC loads.

| Item | Qty | Approx. price (₹) | Where to buy | Search term |
|---|---:|---:|---|---|
| **SSR-10DA** solid-state relay (3–32 V DC in, 24–380 V AC out, 10 A; generic clone) ✓ purchased ₹320 ex-GST / ~₹378 inc. GST | 1 | **378** (paid) | [Robu (generic clone)](https://robu.in/product/dc-to-ac-ssr-10da-solid-state-relay-module-3-32-vdc-24-380vac-10a/) · [IndiaMART (Fotek)](https://www.indiamart.com/proddetail/ssr-10da-adjustable-solid-state-relay-21578621633.html) · [Bikrampur (Fotek)](https://bikrampurtradecorporation.com/product/fotek-ssr-10da-solid-state-relay/) | `SSR-10DA solid state relay India robu` |
| Optional aluminium heatsink for SSR (not strictly needed — coil current is ~30 mA) | 1 | 60 | Robu / IndiaMART | `SSR heatsink aluminium` |
| PC817 optocoupler (DIP, pack of 5 — kept for general spares, not required in this driver chain) | 1 pack | 50 | [Robu](https://robu.in/product/pc817-dip-4-transistor-output-optocoupler-pack-of-5-ics) · [IndiaMART](https://www.indiamart.com/proddetail/pc817-optocoupler-19491205891.html) | `PC817 DIP optocoupler robu` |
| Resistors (330 R, 1 k, 10 k) and 100 nF caps | — | 80 | local kit | `resistor capacitor kit through hole` |

**On generic vs. genuine SSR-10DA:** the original Fotek-branded part is ₹1,200–1,800 from Indian dealers; the unbranded "compatible" SSR-10DA modules on Robu (same form factor, same pinout) run ₹400–600 and are entirely adequate for switching ~30 mA of contactor coil current. Buy the cheaper one for the prototype; consider the genuine Fotek for a deployed unit if you want the longer datasheet pedigree.

**Why no flyback diode?** AC coils don't store significant DC magnetic energy — they reach steady state every half-cycle. The SSR-10DA's zero-cross switching plus the contactor's built-in coil damping (shading ring) handle inductive switching transients. A flyback diode wouldn't help an AC coil anyway (it would just short half the AC waveform).

Driver-chain subtotal (actuals): **~₹510–580** (SSR purchased + PC817 pack + R/C kit; skip heatsink unless thermals say otherwise).

---

## 4. Grid-side power supply (Hi-Link module)

Just one Hi-Link in Rev 1.1, line-side, behind the 6 A controller-feed MCB. It feeds the battery charger only. The contactor coil takes mains AC directly via the SSR (Section 3), so no 24 V rail is needed.

| Item | Qty | Approx. price (₹) | Where to buy | Search term |
|---|---:|---:|---|---|
| Hi-Link **HLK-5M05** (5 V 1 A AC-DC, battery charger input) | 1 | **250** | [Robu](https://robu.in/product/hlk-5m05-5v-5w-switch-power-supply-module) · [ElectronicsComp](https://www.electronicscomp.com/hlk-5m05-hi-link-5v-5w-ac-dc-power-supply-module) · [DNA Tech](https://www.dnatechindia.com/hlk5m05-open-frame-smps-ac-dc-converter-power-supply-module-buy-in-india.html) | `HLK-5M05 Hi-Link India robu` |

**Note on Hi-Link clones:** Robu, ElectronicsComp, and DNA Tech sell genuine modules. Amazon Marketplace third-party listings are a mixed bag — I'd avoid for anything that lives in a damp box on a mains feed. Mean Well IRM-05-05 (~₹700) is the indestructible alternative if you want to spend more for sleep, but for a sacrificial line-side AC-DC the Hi-Link is entirely adequate.

Grid-side AC-DC subtotal: **~₹250** (confirmed price).

---

## 5. Battery and charger (controller-side power)

This is the controller's only power source. The HLK-5M05 from Section 4 feeds the TP5000 charger, which keeps the LiFePO4 cell topped up. The cell feeds an MT3608 boost converter that produces the 5 V rail for the Pico W. The Pico's onboard regulator then makes 3.3 V for itself and the AS3935.

LiFePO4 over Li-ion for two reasons: usable temperature range up to +60 °C (vs. ~+45 °C continuous for Li-ion before accelerated wear) and a much safer thermal failure mode. The enclosure can sit in 50 °C ambient through an Indian summer afternoon without halving the cell's life. Capacity per cell is lower (~3 Ah for a 26650 vs. 3 Ah for a Li-ion 18650), but cycle life is 5–10× higher.

Power budget: Pico W idle Wi-Fi ~70 mA + AS3935 ~3 mA at 3.3 V ≈ 0.25 W average. Single LiFePO4 26650 at 3 Ah × 3.2 V = 9.6 Wh → **~38 hours of grid-loss runtime**, covering essentially any non-substation-level outage.

| Item | Qty | Approx. price (₹) | Where to buy | Search term |
|---|---:|---:|---|---|
| LiFePO4 **26650 cell**, 3 Ah, 3.2 V (e.g., A123 ANR26650M1B or quality equivalent) | 1 | 350–500 | [IndiaMART LiFePO4 26650 listings](https://dir.indiamart.com/impcat/lifepo4-battery.html) · Robu / Amazon.in | `LiFePO4 26650 3000mAh 3.2V India` |
| 26650 single-cell battery holder (PCB-mount or panel) | 1 | 60 | Robu / Amazon.in | `26650 battery holder` |
| **TP5000** 1-cell LiFePO4 charger module (constant-current/constant-voltage, accepts 5 V input) | 1 | 150–220 | Robu / [DNA Tech](https://www.dnatechindia.com/) / Amazon.in | `TP5000 LiFePO4 charger module 3.6V` |
| **MT3608** boost converter module (1.5 → 5 V) | 1 | 50–80 | Robu / ElectronicsComp | `MT3608 boost converter module` |
| Spare LiFePO4 cell (because cells are cheap and one will die during prototyping) | 1 | 350–500 | as above | — |

**Why not a Pi-style UPS HAT?** Those are designed for the controller to draw both from the AC rail *and* fall back to battery, with the AC rail as primary. Rev 1 inverts that: battery is primary, AC is just the charger. A dedicated LiFePO4 charger does the right thing here for a third the price.

**Li-ion alternative if LiFePO4 sourcing is awkward:** swap to a Samsung 30Q 18650 (verify authenticity per [ElectronicsComp](https://www.electronicscomp.com/samsung-inr18650-30q-3000mah-5c-li-ion-battery) — counterfeits are everywhere) plus a TP4056 charger (~₹50). Saves ~₹100 but you give up the thermal margin. Acceptable for an indoor install; not great for an outdoor box in summer.

Battery + charger subtotal: **~₹960–1,360** (including the spare cell).

---

## 6. Service switchgear, surge protection, controller feed

The 63 A isolator and 6 A MCB are not negotiable. The SPD is — three tiers, pick by build profile.

### Fixed lines

| Item | Qty | Approx. price (₹) | Where to buy | Search term |
|---|---:|---:|---|---|
| L&K (L&T) **BF206300** 63 A 2-pole isolator switch (service disconnect) | 1 | 500–900 | [smartshop.lk-ea.com](https://smartshop.lk-ea.com/isolator-switch-63a-2pole-on-off-61192sib03tdyr.html) · [Moglix BF206300](https://www.moglix.com/l-t-63a-double-pole-isolators-bf206300/mp/msnl5glrrg8yk4) · [IndiaMART](https://www.indiamart.com/proddetail/l-t-63a-isolator-double-pole-2849077399333.html) | `L&T BF206300 63A 2P isolator` |
| L&K (L&T) **BB10060C** 6 A SP MCB (controller feed) | 1 | 130–200 | [ClickWik](https://clickwik.in/6a-sp-c-mcb-lt-bb10060c) · [Best of Electricals](https://www.bestofelectricals.com/lt-bb10060c-6a-sp-c-curve-10ka-mcb) | `L&T BB10060C 6A SP MCB` |
| 1 A 5×20 mm glass fuse + panel holder (in-line on logic AC feed) | 1 | 30 | local | `5x20mm glass fuse holder panel mount` |

Fixed-lines subtotal: **~₹660–1,130.**

### Surge protection — **deselected for current build**

**Current decision: skip.** The contactor's open air gap handles distant strikes (the AS3935-detectable kind) by disconnecting before any surge arrives. Near-direct strikes that propagate faster than jigawatt can react may cost a HLK-5M05 + SSR-10DA replacement (~₹950 combined) — that's the risk being accepted in exchange for the simpler, cheaper build.

The two DIN modules that would have held the SPD stay empty for now (position 3 in Section 0's chain). Adding one later is a 30-minute job: drop in the SPD, land L–N–PE to its terminals, done. No rework to anything else.

**Future upgrade paths, kept here for reference:**

| Tier | Part | Approx. price (₹) | Where to buy |
|---|---|---:|---|
| **1 (discrete MOVs)** | 2 × EPCOS **S20K275** MOV (275 V AC, 20 mm, 151 J) — one L–N, one L–PE — plus a 2 A slow-blow glass fuse upstream | 100–200 | [IndiaMART (₹25 each)](https://www.indiamart.com/proddetail/s20k275-epcos-tdk-movs-varistors-275vac-10-20mm-standard-2854956470655.html) · [Amazon.in](https://www.amazon.in/Epcos-S20K275-275Vac-350Vdc-Varistor/dp/B00BPIIQ7K) · [Sharvi Electronics](https://sharvielectronics.com/product/mov-20k275-metal-oxide-varistor/) |
| **2 (Havells Type 2 SPD)** | Havells **DHSA2NBN40320** (40 kA Type 2, 1P+N, DIN-mount with status indicator and thermal disconnect) | 975–1,513 | [IndiaMART Ahmedabad ₹975](https://www.indiamart.com/proddetail/havells-ac-surge-protection-device-spd-single-phase-20097282730.html) · [IndiaMART Kolkata ₹1,020](https://www.indiamart.com/proddetail/surge-protection-device-spd-2856359292733.html) · [Moglix](https://www.moglix.com/brands/havells/electricals/circuit-breakers-fuses/surge-protection-devices/211122000) · [Havells product page](https://havells.com/type-2-ac-surge-protection-device-dhsa2nbn40320.html) |
| **3 (Phoenix VAL-MS 230 ST)** | Phoenix Contact **VAL-MS 230 ST** Type 2 SPD, L–N + L–PE + N–PE in a single plug | 1,150–1,500 | [IndiaMART](https://www.indiamart.com/proddetail/val-ms-230-it-st-surge-protection-25391923255.html) · [RS Components India](https://in.rsdelivers.com/product/phoenix-contact/2798844/phoenix-contact-val-ms-230-st-surge-protector-230/8026542) |

Search term for later: `Type 2 SPD 40kA 1P+N India 230V`.

---

## 7. Enclosure, terminals, wiring

Because jigawatt is now a small secondary DB rather than a project-box mod, the enclosure is DIN-modular and a bit larger.

Module count budget (SPD deselected): 2 (isolator) + 1 (controller MCB) + 5 (4P contactor) = **8 modules used**. In a 12-module enclosure that leaves 4 spare — enough for a future SPD, RCBO, or generator change-over.

| Item | Qty | Approx. price (₹) | Where to buy | Search term |
|---|---:|---:|---|---|
| **12-module distribution board (generic, indoor IP20/30 class)** ✓ purchased ₹600 ex-GST / ~₹708 inc. GST. Likely Anchor / GreatWhite / IndoKopp class — fine for an indoor wall-mount install next to the existing DB | 1 | **708** (paid) | local switchgear shop / IndiaMART | `12 module distribution board indoor IP30 single phase India` |
| *Alternative for outdoor/sheltered install:* Hager Vector VE112L 12-module IP65 | (alt) | 1,800–3,000 | [Eleczo (VE112L)](https://www.eleczo.com/hager-vector-enclosure-distribution-board-1-row-of-12-module-12-module-double-door-ip-65-ref-no-ve112l.html) · [Moglix Hager DBs](https://www.moglix.com/brands/hager/electricals/distribution-boards/211135000) | `Hager Vector VE112L 12 module IP65 distribution board` |
| Small mounting plate / DIN-mounted PCB holder for the controller compartment | 1 | 200 | local fab / Robu | `DIN rail PCB holder universal` |
| Phoenix Contact UT 2.5 / generic feed-through terminals | 12 | 20–30 each | [IndiaMART](https://www.indiamart.com/proddetail/feed-through-terminal-block-uk-2-5-n-3003347-22282691212.html) | `Phoenix UT 2.5 terminal block` |
| 35 mm DIN rail, 200 mm (usually included with the enclosure; spare for the controller compartment) | 1 | 80 | local hardware | `35mm DIN rail steel` |
| Cable glands (PG21 for the 6 mm² through-path, PG9 for control) | 6 | 30 each | local | `PG21 PG9 cable gland nylon` |
| 6 mm² stranded copper wire for through-path, 5 m ✓ on hand | 1 | **0** | — | — |
| 1.5 mm² stranded copper for control, 5 m ✓ on hand | 1 | **0** | — | — |
| Service tail to jigawatt — 6 mm² 3-core flexible, 2 m (meter → jigawatt enclosure) ✓ on hand | 1 | **0** | — | — |
| Ferrules + ring lugs + bootlace assortment | 1 set | 250 | local | `bootlace ferrule kit` |
| Heat-shrink, zip ties, label sleeves | 1 set | 150 | local | `heat shrink tube assorted kit` |

Enclosure + wiring subtotal: **~₹3,500–4,800.**

---

## 8. PCB

| Item | Qty | Approx. price (₹) | Where to buy | Notes |
|---|---:|---:|---|---|
| Perfboard 9 × 15 cm (prototype rev) ✓ confirmed at ₹30 | 1 | **30** | local / Robu / IBT | First iteration lives here. |
| JLCPCB 2-layer 5-pack (after design freezes) | 1 batch | 850–1,100 landed | [JLCPCB quote](https://jlcpcb.com/quote) · [JLCPCB India guide on Zbotic](https://zbotic.in/how-to-order-pcb-from-jlcpcb-and-pcbway-india-cost-guide/) | Under ₹5,000 = usually no customs hit. |

PCB subtotal (actuals): **~₹30 (perfboard) → ~₹1,330 (perfboard + first JLCPCB run).**

---

## 9. Totals — three build profiles (Rev 1.1, with Order #1 actuals)

All three profiles use the battery-primary architecture, the L&K MCX 04 240 V AC-coil contactor (✓ in hand), and the SSR-10DA driver (✓ in hand). The differences are in the sensor, battery chemistry, AC-DC quality, and the wiring/terminals spend. All values inc. GST.

| Section | "Cheap & cheerful" | "Recommended" | "Belt and suspenders" |
|---|---:|---:|---:|
| 4P 40A 240V AC-coil contactor (L&K MCX 04 CS97012BOOO) ✓ | 2,219 | 2,219 | 3,140 (ABB AF26-40-00-11 + DC coil rail) |
| Driver chain (SSR ✓ + opto pack + R/C kit) | 510 | 510 | 1,640 (genuine Fotek SSR + heatsink) |
| Spare Pico W + sensor | 650 + 1,000 (CJMCU) = 1,650 | 650 + 2,000 (DFRobot) = 2,650 | 650 + 2,400 (DFRobot premium) = 3,050 |
| HLK-5M05 (charger input only) | 250 | 250 | 700 (Mean Well IRM-05-05) |
| Battery + charger + boost + holder + spare cell | 850 (Li-ion 30Q + TP4056) | 1,100 (LiFePO4 26650 + TP5000) | 1,400 (premium LiFePO4 + holder + spares) |
| 63 A 2P isolator + 6 A MCB + fuse (fixed, est. inc. GST) | 780 | 945 | 1,335 |
| SPD (deselected — see Section 6 for future upgrade options) | 0 | 0 | 0 |
| DB enclosure ✓ (terminals included in DB purchase, wiring ✓ on hand) | 708 | 708 | 2,740 (Hager IP65 + premium terminals) |
| PCB (perfboard initially) ✓ ₹30 | 30 | 30 | 1,330 (perfboard + first JLCPCB run) |
| Misc (heatshrink, ferrules, postage) | 470 | 590 | 825 |
| Order #1 shipping (already paid) | 295 | 295 | 295 |
| **Total** | **~₹7,762** | **~₹9,297** | **~₹16,455** |

**Recommended build (~₹9,297) gets you:** L&K MCX 04 4P 40 A AC-coil contactor (✓) switched through an SSR-10DA (✓), DFRobot SEN0290 lightning sensor with a properly tuned Coilcraft antenna, a sacrificial Hi-Link AC-DC feeding the battery charger only, a LiFePO4 26650 cell with TP5000 charger as the controller's only direct power source, ~38 hours of grid-loss runtime, 63 A manual isolator for service, all sitting in the **purchased 12-module indoor distribution board (✓)** that mounts next to your existing DB without touching it. SPD slot left empty for now (4 spare modules available for it).

**Cheap & cheerful (~₹9,635)** is the prototype-iteration build — CJMCU-3935 sensor instead of DFRobot, Li-ion 30Q + TP4056 instead of LiFePO4 + TP5000. Upgrade those when you're ready to put the unit on a wall for keeps.

**Order #1 paid:** ₹3,599 (contactor + SSR + DB + shipping inc. GST). **Remaining spend for Recommended build:** ~₹5,698.

**Recommended is now under ₹10k.** Drop from the Rev 1.1 estimate of ₹11,570 to ₹9,297 came from a stack of small actuals beating the estimates:

| What | Saved (₹) |
|---|---:|
| Pico W actual ₹650 vs. ₹913 budgeted | 263 |
| DB ₹708 actual vs. ₹2,300 Hager budget; terminals included | 1,592 |
| Wiring on hand (6 mm² + 1.5 mm² + service tail) | 800 |
| HLK-5M05 ₹250 vs. ₹450 budgeted | 200 |
| Perfboard ₹30 vs. ₹160 budgeted | 130 |
| MCX 04 ₹2,219 inc. GST vs. ₹1,825 Pulstronix | **−394** (cost more) |
| **Net** | **~+2,591** |


**Cost change vs. Rev 1 "Recommended" (₹14,400):** about ₹1,810 cheaper. Drops the HLK-10M24 (~₹500), swaps ABB AF26 for L&K MCX 04 (~₹912 saved), adds the SSR-10DA (~₹500), drops the MOSFET and flyback (~₹70 saved), swaps Phoenix SPD for Havells (~₹130 saved), plus the recomputed switchgear subtotal. The architecture is slightly less "elegant" — there's a mains-switching SSR in the drive chain rather than a clean DC MOSFET — but the SSR's internal opto provides the galvanic isolation we'd otherwise build from discrete parts.

The README's original BoM of ~₹5,500 was real for a bare-bones board on a bench but didn't carry the SPD, the battery, the DB-style enclosure, the service isolator, the AC-DC, or any spares-for-iteration. The "Recommended" column is what a usable, field-deployable unit-one actually costs to build.

---

## 10. Where to put each order

Practical ordering plan, to minimise shipping rounds:

1. **One Robu.in cart:** Pico W spare, AS3935 (DFRobot SEN0290), Hi-Link HLK-5M05, SSR-10DA module (generic clone), PC817 pack (spares), TP5000 charger, MT3608 boost, 26650 cell holder, perfboard, resistor/cap kit. Free shipping above ₹500. ~3-day delivery in metros.
2. **One Electrixzone or Amazon.in order:** L&K MCX 04 CS97012 4P 40A contactor at ~₹1,748. Electrixzone has it at sale price with free shipping above ₹2,000 — add the SSR-10DA to that cart to clear the threshold if you'd rather get genuine Fotek through them.
3. **One Industrybuying or local switchgear shop trip:** BF206300 63 A 2P isolator, BB10060C MCB, Hager VE112L enclosure, DIN rail, terminals, ferrules, cable glands. Faster from a local shop, ~5 days from Industrybuying.
4. **Surge protection — three paths depending on build profile:**
   - **Tier 1 (MOVs):** add S20K275 varistors and a 2 A glass fuse to the Robu cart above. Add ₹150 total.
   - **Tier 2 (Havells):** IndiaMART quote request, or order on Moglix. The IndiaMART listings (Ahmedabad / Kolkata) are often ~30% cheaper than Moglix retail. Stocked widely.
   - **Tier 3 (Phoenix):** IndiaMART quote request. Stocked in tier-1 cities.
5. **One IndiaMART / Robu order for cells:** 2 × LiFePO4 26650 cells (one for the build, one spare). Verify provenance — knockoffs exist here too.
6. **JLCPCB:** wait until perfboard rev of the schematic is bench-tested.

---

## 11. What's deliberately not in the BOM

- **Earthing rod / earth pit work.** The whole point of jigawatt is mains-side disconnection, not surge clamping. If your house earthing is poor, fix that *before* deploying this. A pit-and-strip earthing job is a separate ₹4,000–10,000 line that doesn't belong here.
- **Lightning rod / air terminal.** Different device class. jigawatt protects equipment *by disconnection*; an air terminal protects the *structure*. Not redundant, not interchangeable.
- **Whole-house surge protector (Type 1+2 panel SPD).** Recommended, but lives on the consumer unit, not on jigawatt. Belongs in a separate panel-upgrade project.

---

*BOM rev 0, May 2026. Update when you re-order anything and the price has moved more than 15%.*
