# Vehicle Interface Tester rev1 pre-order review — 2026-10-28 12:00 (third pass: PCB + fabrication)

Inputs (working tree, nothing committed since `721353f`):
- `Vehicle_Interface_Tester.PcbDoc` saved 11:35:56, parsed directly (Board6/Tracks6/Vias6/Pads6/Regions6/Rules6/Texts6 streams).
- DRC report 11:35:52 (4 s **before** the PcbDoc save), 0 violations / 0 waived.
- Schematic PDF 11:55, Protel netlist 11:56, BOM 11:56, `Gerber.zip` 11:57 (Gerbers + NC drill, generated while this review ran; the 11:36 set had **no layers plotted** — fixed by Jared at 11:56).
- Riverdi DS_RVT70HSSNWC00-B Rev 1.4 (P2/P6/P8/P10/P11 tables), firmware `vehicle_interface_tester.ioc`.
- Datasheets: STM32WB1MMC DS14096, onsemi MMBT3906WT1G, TJA1027, SQ2309ES, BAT54W, SMAJ26CA, MCZ1210AH (UpRev `Datasheets/`). TPS5450 / TPS54302 still not on disk.
- Fab: JLCPCB capability page read 2026-10-28 (numbers quoted below). LCSC identities checked online: C22436137, C970698, C194379, C20345389, C21934755.
- Checklist: `orienter_PCB/docs/hw_review_checklist.md` §1–§9. Earlier passes: `..._2026-10-21.md`, `..._2026-10-21_1647.md`, `..._DUT_coverage_2026-10-27.md`.

## Headline — three gates before paying

| | |
|---|---|
| **G1** | **USB VBUS decoupling and clamp are orphaned in the schematic.** This morning's rename `VBUS` → `USB_VBUS` (ECO 11:34/11:35) moved J3-1, J7 VBUS pads, R6 and R7, but **C4 (100 nF) and D7 (6.8 V zener) stayed on `VBUS`** — a two-pin net with no source. The 11:56 netlist still shows it. The PcbDoc currently has C4-1/D7-1 on `USB_VBUS` (good by luck), so **PCB ≠ schematic**; the next ECO will disconnect them. Fix: change the net label on the C4/D7 stub in `USB_Host.SchDoc` to `USB_VBUS`, ECO (expect no PCB change), re-run DRC, re-export netlist/BOM/PDF/Gerbers. |
| **G2** | **No board outline on a layer JLCPCB reads.** The 168 × 107 mm profile is only in `Vehicle_Interface_Tester.GM` (Altium "Profile" plot, 0.0254 mm stroke). `.GM1` is empty (header only) and `.GKO` holds **only the 13 keep-out circles** (7 × 3.0 mm around the mounting holes, 4 × 1.0 mm at the J10/J11 pegs, 2 × 1.575 mm). JLCPCB takes the outline from GKO or GM1 — it would either reject the zip or treat the keep-out circles as the outline. Fix: Design → Board Shape → *Create Primitives From Board Shape* onto Mechanical 1 (or rename `.GM` → `.GM1` in the zip), **drop `.GKO`** and the 20 `GM2…GM29` mechanical layers from the zip, re-zip. |
| **G3** | **Order with "Via Covering = Epoxy Filled & Capped".** All 148 vias are tented in Altium (via mask expansion 0, tent top+bottom), but 25 via barrels sit ≤ 0.07 mm from an SMD pad edge, inside that pad's mask opening: R14-1 (hole 0.02 mm *inside* the pad), MD1-61 (0.004), 18 MD1 ground ring pads (0.02–0.04), D6-2 (0.067), J4-5/6 (0.069), J3-5 (0.098). Open barrels there wick paste on reflow (LGA ground pads, a 0402 pull-up). JLCPCB says filled & capped is the default for 6+ layers — confirm the dropdown shows it before paying; otherwise move those vias ≥ 0.15 mm off the pad edges. |

Plus three decisions that are Jared's, not defects: Q1 the 8-layer stack, Q2 J1 is a female socket (board-to-board, not a cable), Q3 R11/R12/D8 now in the BOM against the sheet's "DNP" note. See below.

---

## Findings

| # | Section | Item | Verdict | Number / reasoning | Fix |
|---|---------|------|---------|--------------------|-----|
| **G1** | 6, 8 | **`VBUS` orphan (C4, D7)** | **FAIL** | Netlist 11:56: `VBUS` = {C4-1, D7-1}; `USB_VBUS` = {J3-1, J7-A4_B9, J7-B4_A9, R6-1, R7-1}. PcbDoc: C4-1 and D7-1 on `USB_VBUS`, `VBUS` net exists with no pads. ECO logs 11:34:06 and 11:35:17 list only J3/J7/R6/R7. | Rename the stub net label; ECO; DRC; re-export. |
| **G2** | 9 | **Outline layer** | **FAIL** | `.GM` = profile (53-vertex rounded rect, ±84.0 × ±53.5 mm, R5). `.GM1` = 17 lines, no primitives. `.GKO` = D10 0.254 mm + D156 3.0 mm circles only. JLCPCB outline sources: GKO / GM1. | Profile to GM1; remove GKO and GM2–GM29 from the zip. |
| **G3** | 9 | **Via barrels inside pad mask openings** | **FAIL unless filled & capped** | Pad mask expansion 0.051 mm (JLC-library pads, manual) or 0.07 mm (rule, 2.7559 mil); via mask expansion rule 0 + tented. Barrel-to-pad-edge: R14-1 −0.02, MD1-61 0.004, MD1 GND pads 0.02/0.041/0.065, D6-2 0.067, J4-5/6 0.069, J3-5 0.098, D5-2 0.135, C19-2/D4-2/Q3-2/U2-9 0.15. | Order option "Epoxy Filled & Capped" (0.15–0.55 mm vias OK; ours are 0.3/0.5). |
| **Q1** | 9 | **8-layer stack inherited from the orienter template** | **DECIDE** | V9 stack: Top / GND / Signal / GND / Power(3V3) / GND / Signal / Bottom, outer 1.378 mil, inner 0.598 mil, ≈1.47 mm. Inner routing: Mid2 63 segments, Mid4 22, Mid6 4; 3V3 plane on Mid6. Rules still carry SDRAM/LCD matched-length, PCIe/Newhaven clearances, `U5` short-circuit exception — harmless leftovers. 168 × 107 mm at 8 layers is the cost driver of this order. | Keep (order 8L, 1.6 mm, 1 oz; no impedance control needed) or re-route on 4L. Jared's call. |
| **Q2** | 8 | **J1 is a female socket, Riverdi P8 is a male header** | **NOT CHECKED (mechanical)** | J1 = Hong Cheng HC-PM127-4.3H-2x20PS, C22436137: "Female Headers, SMD, vertical, 2x20, 1.27 mm, 4.3 mm". Riverdi P8 = "1.27 mm 2x20 PIN HEADER", mates with CBC1402D100 (IDC). So J1 mates **directly** onto P8 (tester top side faces the Riverdi back), not through a ribbon. | Confirm in `vehicle_interface_tester_CAD/Vehicle_Interface_Tester.SLDASM`: pin-1 mirroring of a face-to-face mate, and that top-side parts (J2–J5 right-angle PicoBlades, MD1, U2) clear the Riverdi's back-side parts at the 4.3 mm + pin-length stack height. |
| **Q3** | 6 | **K-line master pull-up R11/R12/D8** | **FAIL (consistency)** | Sheet 6: "MASTER MODE OPERATION (DNP ON THIS BOARD)". BOM 11:56: R11, R12 (2K, 2 pcs) and D8 (3 pcs with D1/D9) **are ordered**; the 10-21 BOM excluded them. If fitted, the bus has 1K (tester, from 14 V) ‖ 1K (DUT R34/R35) = 500 Ω: dominant current (13.6 − 1)/500 = 25 mA (TJA1027 limit ≥ 40 mA, OK) and R11/R12 dissipate 13.6²/2000 = **92 mW each during dominant vs 100 mW 0603 rating** (≈50 % duty → fine, but no margin held low). | Set R11/R12/D8 to no-BOM (as on the UpRev R25) so BOM = sheet, or change the note. |
| N8 | 9 | **Component clearance not in the batch DRC** | **NOT CHECKED** | Rule `ComponentClearance` 10 mil exists but the 11:35 DRC did not run it. Bounding-box (pads + silk) overlaps on the same side: R18/Q4 −0.63 mm, L3/C23 −0.65, L3/C22 −0.35, J6/D3 −1.49, J7 vs R6/R7/D5/D6/D7 −0.01…−0.3 (parts under the raised USB-C body), MD1/C5/C6 −0.17, U1/D3 0.08, Q1/D1 0.025. Most will be silk-outline overlaps, but the J6, J7 and L3 ones involve tall bodies. | Enable Component Clearance in the DRC run (3D bodies) or eyeball in 3D before order. |
| N9 | 9 | **Mask dams on MD1** | **noted** | 0.50 mm rows with 0.35 mm pads → gap 0.15 − 2 × 0.051 = **0.048 mm**; 0.45 mm columns (pads narrowed to 0.25) → 0.098 mm. JLCPCB minimum 0.10 mm (green). Expect ganged openings along the module rows; same situation accepted on the UpRev board (0.098). USB-C J7 0.5-pitch pins: 0.2 − 2 × 0.07 = 0.06 → ganged, standard. Mask-sliver rule still disabled (0.102 mm), so DRC is blind here. | Accept; or set MD1 mask expansion to 0.025 mm to recover 0.10 dams on the rows. |
| N10 | 9 | **Silkscreen** | **minor** | Text heights 1.4 / 1.52 / 2.0 / 5.0 mm (≥ 1.0 OK). 4 top-overlay and 4 bottom-overlay segments at 0.08 / 0.1016 mm are below JLCPCB's 0.15 mm (library outlines). | Cosmetic. |
| N11 | 7 | **`.ioc` still has Riverdi defaults on every tester pin** | **firmware** | PE4 TIM15_CH1N, PD11 LPTIM2_IN2, PK1 TIM1_CH1, PB10 LPTIM2_IN1, PD12/PD13 I2C4, PA3 TIM5_CH4, PJ10 TIM1_CH2N, PB11/PH4 I2C2. Only PA9/PA10 (USART1) and PF10 (USB1_EN) match the tester. Reset state is safe: PK1 is an input until init, R20 100K holds Q4 off → 14 V off. | Reassign per the 10-27 coverage table before bring-up. Not an order blocker. |
| N12 | 6 | **J9 (OBD-II) not in the BOM** | **noted, OK** | J9 absent from the BOM rows; MTG1–7 absent. Same policy as the UpRev J2: hand-installed at assembly. J6 barrel (C3096082) **is** in the BOM and is THT on the bottom. | Order two-sided SMT + THT on the bottom; keep an OBD-II J1962F on hand. |
| N13 | 6 | **BOM text** | **cosmetic** | C17 Package column still "D6.3xL7.7mm" — part C970698 = DMBJ RVT1H221M1010, D10 × 10.2 mm, matches the footprint. U4 footprint has every pad duplicated (12 pads, 6 positions) — harmless. L1 symbol pins "1,2,4,4" (carried). | — |
| — | 9 | **DRC age** | **re-run** | DRC 11:35:52, PcbDoc 11:35:56; same editing session, 0/0. | Re-run after G1's ECO anyway. |

### Closed since the 10-21 second pass (netlist 11:56)

| Item | Evidence |
|---|---|
| G1 D4 orientation | D4-1 (cathode) on `NetC2_1` (REF), D4-2 (anode) on GND. |
| G2 3.3 V buck input | U4-3, C19–C22, R26-2 on `24V_SW`; fuse removed. |
| G3 `3V3_D` orphan | R13/R14/R15-2 on `3V3`. |
| G4 CC1 | R6 and R7 both 56K from `USB_VBUS` to CC1/CC2. |
| G5 BLE UART | `BLE_TX` (MD1-26 PB5) → J1-27 = PA10 USART1_RX; `BLE_RX` (MD1-28 PA3) → J1-25 = PA9 USART1_TX. |
| F6 BOOT0 | `NetMD1_22` = MD1-22, R10-1; R10-2 GND; R9 gone. |
| J3-4, J10/J11 STDC14, K-line on PB10/PB11 | as in the 10-27 record (J1-12 LIN_TXD, J1-34 LIN_RXD, J3-4 GND). |

### Section 1/2/3 — unchanged from 10-21 (re-read, same values)

14 V: 1.221 × (1 + 21/2) = 14.04 V. 3.3 V: 0.596 × (1 + 68/15) = 3.298 V. UVLO 1.2 × 322/22 ≈ 17.6 V. OV cut-off 2.495 × (1 + 24.9/2.49) = 27.4 V. U3-5 ENA open = enabled (U3-2/3 NC). LEDs 0.9 mA (D11, 13K from 14 V) and 1.9 mA (D13). TPS5450/TPS54302 limits still from memory (datasheets absent).

### Section 5 — pinouts re-verified this pass

| Part | Verdict | Evidence |
|---|---|---|
| Q2 MMBT3906WT1G, SC-70 | PASS | onsemi DS: 1 BASE, 2 EMITTER, 3 COLLECTOR. Netlist: Q2-1 `NetQ2_1` (TL431 cathode via R3), Q2-2 `NetC1_1` (+24 V in), Q2-3 `NetC1_2` (Q1 gate). |
| Q4 MMBT3904 SOT-23 | PASS | 1 B (R19/R20), 2 E GND, 3 C (R18 → Q3 gate). |
| U2 = TJA1027TK/20,118 | PASS | C194379 confirmed NXP TJA1027TK/20,118 HVSON-8 3×3. |
| D10 ESD1LIN24DYFR | PASS | C20345389: bidirectional, 24 V standoff, 35.5 V breakdown — orientation moot. |
| J10/J11 FTSH-107-01-F-DH-A | PASS | C21934755: Samtec right-angle 14-pos, `-A` = alignment pins; footprint has 2 × 1.02 mm NPTH pegs each. |
| MD1 footprint vs DS14096 §LGA77 | PASS | DS: 10 × 6.5 mm, 0.45/0.50 pitch, pads 0.30 (×10), 0.35 × 0.30 (×54), 0.60 (×9), 0.35 (×4). PCB: 0.30 ×10, 0.35 × 0.30 ×33 + 0.25 × 0.35 ×22 (narrowed on the 0.45 columns), 0.60 ×9, 0.35 ×3. Pins 6/7 shorted (`NetMD1_6`). Pour cut-out 4.4 × 4.2 mm on all 8 layers at the antenna end; module end 1.1 mm from the board edge. |
| J1 ↔ Riverdi P8 | PASS (electrical) | DS Rev 1.4 table: 9 PE4, 10 PD11, 11 PK1, 12 PB10, 13 PD12, 14 PC7, 15 PD13, 16 PA3, 25 PA9, 27 PA10, 33 PJ10, 34 PB11, 36 PH4; GND 6/7/17/18/28. Netlist matches pin for pin. |
| J3 ↔ P10, J2 ↔ P11, J4 ↔ P2, J5 ↔ P6 | PASS | P10: 1 VCC_USB, 2 D−, 3 D+, 4 ID (GND = host), 5 GND. P11: 1 GND, 2 CAN_L, 3 CAN_H. P2: 1–3 VDD_IN, 4 EN open (pulled up), 5–6 GND. P6: 2 CLK, 3 GND, 4 DIO, 5 RST, 6 SWO. |

### Section 6 — data consistency

| Check | Result |
|---|---|
| PcbDoc ↔ Protel netlist | 324/324 pins, 56 connected nets, 99/99 components, 0 footprint-name differences. Only difference: C4-1/D7-1 (G1). |
| BOM ↔ netlist | 64 BOM rows; every row has an LCSC number. Excluded: J9, MTG1–7 (deliberate). Included but marked DNP on the sheet: R11, R12, D8 (Q3). |
| DNP marking | Still no Altium `Fitted` attribute anywhere; no-BOM component type is the working method. |

### Section 9 — fabrication (JLCPCB, 8 layers, 1 oz)

| Item | Design | JLCPCB limit (read 2026-10-28) | Verdict |
|---|---|---|---|
| Layer count / thickness | 8 copper, Altium stack ≈ 1.47 mm | 8-layer offered at 1.6 mm | PASS (order 1.6 mm; no impedance control) |
| Min trace width | 0.127 mm (40 top + 22 bottom segments), inner ≥ 0.254 | 0.09 mm | PASS |
| Min clearance rule | 0.127 mm (All), poly 0.165, poly-poly 0.254 | 0.09 mm | PASS (rule above fab limit, so DRC 0 is meaningful) |
| Via hole / diameter | 139 × 0.30/0.60 (ring 0.15), 9 × 0.50/1.00 | 0.15 hole / 0.25 dia; ring ≥ 0.15 | PASS |
| Aspect ratio | 1.6 / 0.30 = 5.3:1 | ≈ 10:1 | PASS |
| Hole-to-hole | min 0.30 mm (GND via pairs), 0.427 mm different nets (CAN_P/CAN_N) | 0.2 mm via-via | PASS |
| Via-in-pad | 0 via centres inside any SMD pad | — | PASS, but see G3 (barrels at pad edges) |
| THT annular ring | J9 1.25 mm hole / 2.3 pad (0.525), J7 slots 0.6 × 1.6 in 1.0 × 2.0 (0.2), J6 slots 1.0 × 3.0 and 1.0 × 3.5 (0.625) | PTH ring ≥ 0.15 (0.2 rec.) | PASS |
| Plated slot width | 0.6 mm (J7), 1.0 mm (J6) | ≥ 0.35 mm | PASS |
| Mounting holes | 7 × 3.45 mm PTH, no copper ring, ≥ 2 mm from every pour on every layer | — | PASS (not grounded; screws isolated) |
| NPTH | 2 × 0.8 (J7), 4 × 1.02 (J10/J11), 2 × 1.19 (J8) | — | PASS |
| Copper to board edge | pours pulled back 1.009 mm (rule 1 mm); nearest pad J10/J11 pegs 0.747, J6-2 0.94 | ≥ 0.2 mm | PASS |
| Solder mask dam | 0402 0.198, HVSON-8 0.148; MD1 rows 0.048, MD1 columns 0.098; USB-C 0.06 | ≥ 0.10 mm (green) | N9 — ganged openings expected on MD1 and J7 |
| Mask sliver rule | disabled (0.102) | — | DRC blind, measured above |
| Silk width / height | ≥ 0.127 except 8 segments at 0.08–0.10; text ≥ 1.4 mm | 0.15 mm / 1.0 mm | N10 cosmetic |
| Board outline layer | only `.GM` (Profile) | GKO or GM1 | **G2 FAIL** |
| Gerber set | 8 copper, GTS/GBS, GTP/GBP, GTO/GBO, GKO, GM, GM1–GM29, GD1/GG1, NC drill (186 holes, 9 tools) | — | strip mech layers (G2) |
| DRC vs PcbDoc | DRC 4 s older than the save | — | re-run |

## Order-time checklist (JLCPCB)

1. Fix **G1**, ECO, DRC, re-export netlist / BOM / PDF / Gerbers + NC drill; diff the new `Gerber.zip` against the folder.
2. **G2**: outline on GM1 (or rename `.GM` → `.GM1`), delete `.GKO` and `GM2…GM29` from the zip. Keep GD1/GG1 out as well.
3. Order options: 8 layers, 1.6 mm, 1 oz outer, green mask (0.10 mm dam), **Via Covering = Epoxy Filled & Capped** (G3), no impedance control, no castellations.
4. Expect an engineering query on ganged mask over MD1 rows and the USB-C pins — accept.
5. Assembly: both sides SMT; THT J6 (barrel) on the bottom; J9 OBD-II and MTG hardware hand-installed. Decide **Q3** (R11/R12/D8) before uploading the BOM.
6. Confirm **Q1** (8 layers) and **Q2** (J1 socket orientation / stack height in the SolidWorks assembly) before paying.
7. Commit the PcbDoc, schematics, OutJob and this record together.

## Re-check 12:17 (after Jared's fixes; PcbDoc 12:16:02, outputs 12:16–12:17, `Gerber.zip` 12:17:18)

| Item | Status | Evidence |
|---|---|---|
| G1 `VBUS` orphan | **CLOSED** | Netlist 12:16: `USB_VBUS` = {C4-1, D7-1, J3-1, J7-A4_B9, J7-B4_A9, R6-1, R7-1}; no `VBUS` net. PcbDoc ↔ netlist 324/324 pins, 56/56 nets, 99/99 components, 0 footprint differences. |
| G2 outline | **CLOSED, one leftover** | `.GM1` now carries the 52-segment profile (±84.0 × ±53.5 mm, R5) with a 0.0254 mm stroke; `.GM` (Profile) duplicates it. **`.GKO` is still in the zip with only the 13 keep-out circles** — delete it from the upload so JLCPCB cannot pick it as the outline. The 20 `GM2…GM29` layers are also still in the zip (harmless, but fewer questions if removed). |
| G3 vias | **CLOSED by order option** | ViaStructures: 148/148 vias assigned to the one via structure (Jared: IPC-4761 Type VII). GTS/GBS have **0 mask openings at via centres** (fully tented), so the only exposed barrel copper is where a pad's opening overlaps a via (R14-1, MD1 ring). The Altium structure never reaches a Gerber-only fab — select **Via Covering = Epoxy Filled & Capped** on the order form. |
| Q1 8 layers | **DECIDED: keep** | — |
| Q2 J1 socket | **DECIDED: direct mate onto P8** | Mirroring/stack height still to be eyeballed in the SLDASM (not re-checked here). |
| Q3 R11/R12/D8 | **CLOSED** | ECO 12:14:28 set D8, R11, R12 to Standard (No BOM); BOM 12:16 has 63 rows, no R11/R12, "D1, D9" qty 2. Netlist still lists them (correct — fitted/unfitted is a BOM property). |
| DRC | re-run before upload | DRC 12:05:37 is older than the 12:16:02 PcbDoc save. The only change between them is the 12:14 component-kind ECO plus the GM1 outline primitives (tracks 1743 → 1795, all non-copper); copper counts are identical (vias 148, pads 434, per-layer track histograms unchanged). Low risk, but a 2-second re-run makes the record clean. |
| Drill | PASS | DRR 12:16: 186 holes, 9 tools, unchanged. |
| N11 `.ioc` | open (firmware) | Jared: not yet started. |

**Remaining before upload:** delete `.GKO` (and optionally `GM2…GM29`) from `Gerber.zip`; re-run DRC; commit.

## Not checked

| Item | Why | What it would take to close |
|------|-----|------------------------------|
| Component 3D clearance (N8) | not in the batch DRC; only bounding boxes measured here | run DRC with Component Clearance enabled, or check 3D at J6/D3, J7/R6/R7/D5/D6/D7, L3/C22/C23, R18/Q4 |
| J1 pin-1 mirroring and stack height against the Riverdi (Q2) | mechanical, needs the SLDASM | open `Vehicle_Interface_Tester.SLDASM` with the Riverdi STEP; confirm P8 pin 1 lands on J1 pin 1 |
| Antenna keep-out size vs ST AN5165 | AN not on disk; cut-out measured at 4.4 × 4.2 mm | compare with the AN's keep-out figure for the WB1MMC |
| TPS5450 / TPS54302 abs max, EN current, TL431 | datasheets absent (carried from 10-21) | add to a `Datasheets/` folder here |
| Inrush into the DUT when Q3 closes | DUT +VBAT bulk unknown (carried) | read the DUT's C11 |
| MD1 77-pin walk | not repeated; netlist pin numbers identical to the 10-21 walk after today's symbol replacement | — |
