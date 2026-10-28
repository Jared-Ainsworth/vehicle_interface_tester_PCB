# Vehicle Interface Tester vs UpRev Bluetooth Interface: test-coverage cross-check

Date: 2026-10-27
Inputs:
- Tester: `Project Outputs for Vehicle_Interface_Tester/ProtelNetlist/Vehicle_Interface_Tester.NET` + BOM, both exported 2026-10-27 13:30 (sheets saved 13:28, so current).
- DUT: `UpRev_interface_PCB/Project Outputs for UpRev Bluetooth Interface/ProtelNetlist/UpRev_Bluetooth_Interface.NET` + BOM, exported 2026-10-15 13:56. **Stale**: `LINBUS.SchDoc` (2026-10-20) and `BLE_MODULE.SchDoc` (2026-10-21) are newer than the netlist. Re-export before trusting the DUT side of this table.
- Riverdi DS_RVT70HSSNWC00-B Rev 1.4 connector pinouts (P2, P6, P8, P10, P11).
- Riverdi firmware `.ioc` for current pin assignments.

Cable map (from Jared): J1 -> P8 expansion, J2 -> P11 CAN1, J3 -> P10 USB, J4 -> P2 power, J5 -> P6 SWD.

## 1. DUT external interfaces and what the tester provides

| DUT interface | DUT pins / nets | Tester path | Riverdi resource | Status |
|---|---|---|---|---|
| USB-C data | J1 D+/D- (USB_CONN_P/N via L2) | J7 -> L1 -> J3-3/J3-2 -> P10-3/P10-2 | OTG_HS FS PHY PB14/PB15, host mode | PASS (no host stack in firmware yet) |
| USB VBUS on/off | J1 VBUS -> Q2 power path, VBUS_DET PC4 | J7 VBUS = J3-1 = P10-1 VCC_USB | PF10 `USB1_EN` load switch 500 mA, PC15 overcurrent | PASS |
| USB-C CC | R3/R4 5.1K Rd | R6/R7 56K Rp to VBUS | - | PASS (G4 closed) |
| USB OTG ID | - | **J3-4 unconnected** | P10-4 must be GND for host mode | **FAIL** |
| OBD CAN | J2-6 CAN2_P, J2-14 CAN2_N, TCAN3404, no termination | J9-6/14 -> R1 120R -> J2-3/J2-2 -> P11-3 CAN_H / P11-2 CAN_L | On-board CAN FD transceiver, "CAN1 TERM" solder bridge | PASS (check CAN1 TERM state: want exactly two 120R on the bus) |
| OBD K-line | J2-7, TJA1027 U5, R34/R35 2K pull-ups | J9-7 -> U2 TJA1027, R11\|\|R12 1K pull-up from switched VBAT via D8/D9 | PC6 USART6_TX (BTN1, remove R58), PC7 USART6_RX, PE4 SLP_N | PASS after Riverdi R58 removal + .ioc changes |
| OBD +VBAT | J2-16 | U3 TPS5450 14.04 V -> Q3 P-FET -> 14V_SWITCHED -> J9-16 | PK1 via Q4 | PASS |
| OBD grounds | J2-4 CHS_GND, J2-5 GND | J9-4 CHS_GND (10M + 1nF to GND), J9-5 GND | - | PASS |
| MCU SWD (J3 TC2030: 3V3, SWDIO, RST, SWCLK, GND, SWO) | PA13/PA14/NRST/PB3 | J10-1 NC, J10-2 PD12, J10-3 PD13, J10-4 PD11, J10-5 GND, J10-6 PA3 | Bit-bang SWD from Riverdi GPIO | OPEN: connector/cable convention (see 2.2); firmware is a large item |
| BLE module SWD (J4 TC2030: 3V3_D, SWDIO, RESET, SWCLK, GND) | WB1MMC PA13/PA14/NRST | Same J10, move Tag-Connect to DUT J4 (identical pin order) | same | OPEN as above |
| Bluetooth RF | STM32WB1MMC MD1 | Tester MD1 STM32WB1MMC, UART to Riverdi | PA9 USART1_TX -> BLE_RX (MD1-28 PA3 LPUART RX); PA10 USART1_RX <- BLE_TX (MD1-26 PB5 LPUART TX) | PASS (direction now correct, G5 closed) |
| Bind switch SW1 | MD1-13 PA2 | none | - | Operator step, DUT reports over USB |
| RGB LED D9 | MCU GPIO | none | - | Operator visual |
| SD card, QSPI flash, RAM | internal | none | - | DUT self-test, reported over USB; SD card must be fitted |
| Current draw (USB / battery) | - | none | PC15 overcurrent flag only | Not measured. Optional: INA/shunt on VBUS and 14V_SWITCHED |

## 2. Findings

### 2.1 FAIL: J3-4 open (USB OTG ID)
Riverdi P10 pin 4 must be grounded for host mode and 5 V output. Tie tester J3-4 to GND. No TPS2553 needed (review item N2 closes).

### 2.2 OPEN: J10 / J11 pin convention matches no standard 14-pin debug cable
J8 (tester's own WB1MMC programming header) is wired as STDC14: 3 VCC, 4 SWDIO, 5/7 GND, 6 SWCLK, 12 NRST. Correct for an ST-Link V3 cable.

J10 and J11 are the same 14-pin 1.27 mm footprint but use pins 1-6 in Tag-Connect TC2030 order (1 VCC, 2 SWDIO, 3 RST, 4 SWCLK, 5 GND, 6 SWO):
- J11: 1 = tester 3V3, 2 = Riverdi SW_DIO (J5-4 -> P6-4), 3 = RST (J5-5 -> P6-5), 4 = SW_CLK (J5-2 -> P6-2), 5 GND, 6 SWO (J5-6 -> P6-6).
- J10: 2 = PD12, 3 = PD13, 4 = PD11, 6 = PA3, 5 GND, 1 open.

An ST-Link V3 STDC14 plug on J11 would see RST on its VTref pin, SW_CLK on SWDIO and SWO on SWCLK. A TC2030-CTX-NL cable ends in a 10-pin ARM plug (1 VTref, 2 SWDIO, 3 GND, 4 SWCLK, 5 GND, 6 SWO, 10 nRESET), a TC2030-IDC-NL in a 6-pin 0.1" IDC; neither mates with FTSH-107.
Fix options: J11 -> J8's STDC14 map (and feed VTref from J5-1 = Riverdi VCC_+3.3V rather than tester 3V3); J10 -> 10-pin FTSH-105 wired ARM-style so a TC2030-CTX-NL plugs straight in, or keep 14-pin and wire STDC14 with a custom cable. Decide which cable will physically be used first.

### 2.3 FAIL: tester MD1 BOOT0 sits on a 10K/10K divider
MD1-22 (WB1MMC BOOT0) has R9 10K to 3V3 and R10 10K to GND = 1.65 V, undefined. Make it a plain pull-down (or drive it from a Riverdi GPIO if UART-bootloader entry is wanted). DUT has the identical pattern: MD1-22 with R25/R26 1K/1K (both in BOM) — fix there too.

### 2.4 Riverdi board mods and .ioc changes needed
| Tester net | J1 pin | Riverdi pin | Riverdi mod | .ioc change |
|---|---|---|---|---|
| LIN_TXD | 37 | PC6 | remove R58 (BTN1) | USR_BTN_1 -> USART6_TX |
| LIN_RXD | 14 | PC7 | - | TIM3_CH2 -> USART6_RX |
| LIN_SLP_N | 9 | PE4 | - | TIM15_CH1N -> GPIO out |
| BLE_WKUP | 33 | PJ10 | remove R60 if the USR LED load matters (optional) | USR_LED_1 -> GPIO out |
| BLE_RESET | 36 | PH4 | - | I2C2_SCL -> GPIO out (I2C2 is only on P8, nothing on-board uses it) |
| 14V switch | 11 | PK1 | - | TIM1_CH1 -> GPIO out |
| SWD to DUT | 10/13/15/16 | PD11/PD12/PD13/PA3 | - | LPTIM2_IN2 / I2C4 / TIM5_CH4 -> GPIO (PA3 = USART2_RX if SWO is wanted; RS232 USART2 is unused on the tester) |
| BLE UART | 25/27 | PA9/PA10 | - | already USART1 |

### 2.5 Other checks (PASS)
- J4 -> P2: pins 1-3 24V_SW (VDD_IN 6-48 V), pin 4 ENABLE left floating (= enabled), 5-6 GND.
- J2 -> P11: 1 GND, 2 CAN_L, 3 CAN_H, 4 (VDD out) left open. Correct.
- J5 -> P6 signal mapping correct (CLK/GND/DIO/RST/SWO on 2/3/4/5/6); P6-1 unused.
- J1 GND pins 6, 7, 17, 18, 28 match P8 GND pins. VCC_+5V / VCC_+3.3V on P8 1-4 not used; tester has its own 3V3 buck (U4) from 24V_SW.
- 14 V rail: R23 21K / R24 2K -> 14.04 V (TPS5450 Vref 1.221 V).
- DUT BLE UART: MD1 PB5 (LPUART1_TX) -> H573 PB11 (USART3_RX); MD1 PA3 (LPUART1_RX) <- H573 PB10 (USART3_TX). Correct.
- DUT K-line UART: TJA1027 TXD <- H573 PA0 (UART4_TX), RXD -> PA1 (UART4_RX). Correct (old PA0_C/PA1_C finding was for the H753).

## 3. USB-link-survives-VBUS requirement
Test plan reads DUT state over USB while VBUS is cycled. Current DUT firmware (`UPREV_Prog/USB_DEVICE/Target/usbd_conf.c`) has `vbus_sensing_enable = ENABLE`, so the DUT detaches when VBUS drops. For the link to survive, the DUT firmware must disable VBUS sensing (hardware already OK: DUT stays powered from +VBAT, D+/D- unswitched on the tester). Deferred by Jared.

## 4. Re-check 2026-10-27 14:23 netlist (after Jared's fixes)
- 2.1 J3-4 now GND. CLOSED.
- 2.2 J10 and J11 now wired STDC14 like J8 (3 VTref, 4 SWDIO, 5/7/11 GND, 6 SWCLK, 8 SWO, 12 NRST). CLOSED.
  J11: 4 -> P6-4 SW_DIO, 6 -> P6-2 SW_CLK, 8 -> P6-6 SWO, 12 -> P6-5 RST, 3 = tester 3V3 (J5-1 / P6-1 still unused).
  J10: 4 = PD12 (SWDIO), 6 = PD11 (SWCLK), 8 = PA3 (SWO in), 12 = PD13 (NRST), 3 open. Riverdi is the SWD master
  (bit-bang), cable to the DUT = Tag-Connect TC2030-CTX-NL-STDC14 (14-pin ribbon end, 6-pin no-legs TC2030 end), one cable
  serves DUT J3 (MCU) and J4 (BLE) since both use the same TC2030 pin order.
- 2.3 Tester BOOT0: R9 removed, R10 10K pull-down only. CLOSED. DUT R25/R26 1K/1K still open on the DUT project.
- K-line alternative to avoid removing Riverdi R58: USART3 on PB10 (P8 pin 12, TX) / PB11 (P8 pin 34, RX). Both unused on
  the tester; in the .ioc they are LPTIM2_IN1 / I2C2_SDA (expansion-only functions, on-board I2C devices are on I2C1 PB6/PB7).
  Move LIN_TXD J1-37 -> J1-12 and LIN_RXD J1-14 -> J1-12/34 accordingly.
- Still outstanding: DUT netlist re-export (LINBUS/BLE_MODULE sheets newer than 10-15 netlist); DUT BOOT0 divider;
  CAN1 TERM bridge state on the Riverdi; R60 on PJ10 optional; no current sensing (optional); firmware: USB host stack,
  SWD bit-bang + H573/WB flash loaders, .ioc reassignments (PE4, PH4, PK1, PD11-13, PA3, PB10/PB11).

## 5. Re-check 14:44 netlist
- LIN_TXD moved to J1-12 (PB10 USART3_TX) -> TJA1027 TXD; LIN_RXD to J1-34 (PB11 USART3_RX) <- RXD. J1-14/J1-37 free, PC6/PC7 buttons untouched. CLOSED.
- J10 confirmed as the Riverdi bit-bang SWD output to the product; J11 is the external ST-Link input for the Riverdi.
- Tester PCB: nothing outstanding from this cross-check. Remaining items are on the DUT project (netlist re-export, BOOT0 R25/R26), the Riverdi (CAN1 TERM bridge) and firmware.
