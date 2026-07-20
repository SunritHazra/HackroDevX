# HackroDevX

An open-source RP2040 development board with USB-C power/programming, onboard QSPI flash, and a Raspberry Pi Pico-compatible pinout. Built by following Hack Club's ["Custom Devboard"](https://blueprint.hackclub.com/starter-projects/devboard) starter project guide, entirely in KiCad. This is my first custom PCB from schematic to fabrication.

> Full day-by-day design log, decisions, and mistakes: [Engineering Log.md](https://github.com/SunritHazra/HackroDevX/blob/main/Engineering%20Log.md) (see also [JOURNAL.md](https://github.com/SunritHazra/HackroDevX/blob/main/JOURNAL.md) and [Fabrication Log.md](https://github.com/SunritHazra/HackroDevX/blob/main/Fabrication%20Log.md))

**Project stats:** 5 build-log days · ~17h 00m total design time · 90% designing progress, 0% building progress (as of the latest journal entry — board is ordered for PCBA, not yet assembled)


**On the name:** Follows the author's personal naming convention for electronics projects — "Hackro" as the base word, a short form of the project's defining trait ("Dev", for devboard), and a single trailing letter — giving **HackroDevX**.

---

## Hardware Architecture

### 1. Electrical Schematic

The schematic follows the five functional blocks the guide breaks a devboard into: power/decoupling, USB-C, the crystal oscillator, QSPI flash storage, and the I/O breakout, all wired around a central RP2040.

Full Schematic

**Power & decoupling:** One 0.1uF decoupling capacitor per RP2040 VDD pin — eight on the 3.3V IOVDD rail, two on the 1.1V DVDD rail — plus a 1uF bulk capacitor on each rail to smooth larger ripples. Power labels face up, ground labels face down, per standard schematic convention.

**USB-C & step-down:** A USB-C receptacle with SHIELD/GND tied to ground, CC1/CC2 pulled down through 5.1KΩ resistors so the port negotiates power delivery, and the D+/D- lines routed through 27Ω termination resistors into the RP2040's USB_DP/USB_DM pins (set to bidirectional global labels). VBUS is stepped down from 5V to 3.3V using an **MCP1700x-330xxTT** LDO (swapped in from the guide's larger NCP1117-3.3_SOT223 for a smaller footprint, at the cost of a lower 250mA current limit), with 10uF bulk capacitors bracketing it.

**Crystal oscillator:** A 12MHz crystal (Y1) with two load capacitors on its I/O pins (bumped from the guide's 15pF to 33pF to match the load capacitance of the specific crystal part used), pins 2/4 tied to GND, and a 1KΩ damping resistor on XOUT to protect the crystal and keep the clock signal clean.

**QSPI flash:** Since the RP2040 has no onboard flash, a quad-SPI flash IC is wired up with global labels matching the RP2040's QSPI pins, a 0.1uF decoupling cap on VCC, a 10KΩ pull-up on CS, and a BOOTSEL push button in series with a 1KΩ current-limiting resistor to GND — dropping the board into BOOTSEL/mass-storage mode on boot.

**I/O breakout:** TESTEN tied to GND (factory-test-only pin), and every remaining GPIO plus SWCLK/SWDIO broken out across two 1x20 headers and one 1x3 header, laid out to match the Raspberry Pi Pico pinout. VSYS, 3V3_EN, and ADC_VREF were skipped since there's no battery support, freeing up pins for extra GPIOs — including GPIO29 (ADC-capable) in place of the unused GPIO25.

Running ERC initially surfaced 4 violations — three were the standard "Input Power pin not driven by any Output Power pins" warning the guide says to expect and ignore, but the fourth flagged a genuine wiring mistake where the RP2040's VREG_VOUT and the LDO's output ended up sharing a net. That was traced back to a symbol difference from the guide and fixed by re-checking the actual pin connections.

### 2. Footprints

- **RP2040:** `QFN-56-1EP_7x7mm_P0.4mm_EP3.2x3.2mm`, straight from the datasheet.
- **Decoupling capacitors:** `0402` for all small signal capacitors.
- **Decopuling resistors:** `0402` for all small resistors.
- **Bulk (10uF) capacitors:** `0603`, for the extra physical room they need.
- **USB-C receptacle:** `USB_C_Receptacle_HRO_TYPE-C-31-M-12` — JLCPCB has no true basic-library USB-C part.
- **Header pins:** `PinHeader_1x20_P2.54mm_Vertical` (×2) and `PinHeader_1x03_P2.54mm_Vertical`.
- **Boot button:** `SW_Push_SPST_NO_Alps_SKRK`, a compact SMD footprint in JLCPCB's basic library.
- **Crystal:** `Crystal_SMD_3225-4Pin_3.2x2.5mm`, also a JLCPCB basic part — pin 1/3 orientation double-checked against the datasheet.
- **LDO:** `MCP1700x-330xxTT` (SOT-23), swapped from the NCP1117 for size.
- **Flash:** `W25Q16JVZPIQ TR` in the `Winbond_USON-8-1EP_3x2mm_P0.5mm_EP0.2x1.6mm` package — the same flash chip used on the real Pi Pico, swapped from the datasheet's larger W25Q128JVS.

  The board outline follows the guide's Pico-shaped form factor, sized **21mm × 51.5mm** (slightly longer than the guide's reference 21×51mm outline). Components were placed by signal flow: the two 1x20 headers positioned off the board corners, the RP2040 centered and shifted slightly down, the USB-C connector near the top edge, the LDO close to VBUS, the flash memory tight against the RP2040's QSPI pins, and the crystal with its load caps and damping resistor right beside XIN/XOUT. Decoupling capacitors were grouped next to whichever pins they actually decouple.

Routing progress (mid-project)

Routing started with the flash memory's QSPI lines as a warm-up, followed by the USB-C differential pair (D+/D- routed together through the termination resistors into the RP2040), the crystal's load caps and damping resistor, and the rest of the decoupling and header nets — switching between front/back copper layers and using vias to hop layers where needed.

**Differential pair fix:** an early routing pass left a 6.5mm+ length mismatch between the USB D+ and D- traces against a 0.2mm tolerance budget — enough to distort the high-speed USB signal. This was fixed later by rerouting the pair properly as a matched differential pair after repositioning a nearby component.

Once every net was routed, a ground fill was added across both copper layers on the `GND` net, using thermal reliefs (2 minimum spokes) for pad connections. DRC then flagged a handful of "Thermal relief connection to zone incomplete" errors on a few isolated pads (crystal, some header pins, a load cap), resolved by re-filling the zones and adding extra vias to tie the isolated islands into the pour.

## Design Notes

**Crystal load capacitance:** swapping to a different crystal part than the guide's reference meant the load capacitors had to be bumped from 15pF to 33pF to match its datasheet load capacitance.

**ERC net conflict:** an early wiring mistake merged the LDO's output and the RP2040's VREG_VOUT onto the same net (both flagged as power outputs), caused by using a slightly different LDO symbol than the guide's. Fixed by re-tracing the actual intended connections.

**Differential pair skew:** the USB D+/D- traces were initially routed without matching their lengths, resulting in a 6.5mm+ mismatch against a ~0.2mm budget — fixed by rerouting them as a proper differential pair.

**Silkscreen:** reference designators and values were batch-moved from the `F.Fab` layer to `F.Silkscreen` using KiCad's footprint text field batch update (filtered by layer, applied to functional text items). Less-useful `C#`/`R#` labels on small passives were deleted for a cleaner look, and bottom-side GPIO labels were checked for correct mirroring/orientation. The Hack Club "Orpheus" flag logo was thresholded through KiCad's Image Converter and placed on the silkscreen next to a `HackroDevX` text label and the author's name.**Silkscreen:** reference designators and values were batch-moved from the `F.Fab` layer to `F.Silkscreen` using KiCad's footprint text field batch update (filtered by layer, applied to functional text items). Less-useful `C#`/`R#` labels on small passives were deleted for a cleaner look, and bottom-side GPIO labels were checked for correct mirroring/orientation. The Hack Club "Orpheus" flag logo was thresholded through KiCad's Image Converter and placed on the silkscreen next to a `HackroDevX` text label and the author's name.

---

## Fabrication

- **Electrical Rule Check (ERC)** run after the schematic was completed.
- **Design Rule Check (DRC)** run after routing and ground pour, with thermal-relief/clearance errors resolved before export.
- **Gerber export:** full fabrication output (gerbers, drill files, footprint position files) exported from KiCad `9.0.7`.
- **CPL/BOM formatting for JLCPCB:** the CPL (`top-pos`) file headers were renamed `Ref → Designator`, `PosX → Mid X`, `PosY → Mid Y`, `Rot → Rotation`, `Side → Layer`, and the BOM's `Designation` column renamed to `Comment`, per JLCPCB's expected format.
- **Ordered:** production zip uploaded to JLCPCB with PCBA added for a batch of 5 assembled boards. Most parts auto-matched (RP2040 QFN-56, MCP1700x-330xxTT, USB-C receptacle, crystal, resistors, caps); the flash memory needed a manual match to JLCPCB's in-stock `W25Q16JVUXQTR` (functionally equivalent to the designed `W25Q16JVZPIQ TR`). Pin headers were left off the assembled parts list to hand-solder separately instead of paying per-part assembly fees.

Design files: [`/PCBA`](https://github.com/SunritHazra/HackroDevX/tree/main/PCBA)

---

## Bill of Materials (BOM)

Full sourcing sheet: [`/Bill of Materials`](https://github.com/SunritHazra/HackroDevX/tree/main/Bill%20of%20Materials)

### Electronics & PCB (SMD)

- **1x** PCB (21mm × 51.5mm, Pico-shaped outline)
- **1x** Raspberry Pi RP2040 SoC (QFN-56)
- **1x** W25Q16JVZPIQ TR QSPI flash memory
- **1x** MCP1700x-330xxTT LDO (3.3V, SOT-23)
- **1x** USB-C receptacle (HRO TYPE-C-31-M-12)
- **1x** 12MHz crystal oscillator (SMD 3225-4Pin)
- **1x** SPST push button (BOOTSEL)
- Decoupling capacitors (0402: 0.1uF ×10, 1uF ×2), bulk capacitors (0603: 10uF ×2), termination resistors (27Ω ×2), pull-up/pull-down/damping resistors (5.1KΩ, 10KΩ, 1KΩ)

### Hardware (THT)

- **2x** `PinHeader_1x20_P2.54mm_Vertical`
- **1x** `PinHeader_1x03_P2.54mm_Vertical`

### Sourcing

| Component            | Vendor  |
| --------------------- | ------- |
| PCB Fabrication & PCBA | JLCPCB  |

--

## AI as a Research & Documentation Aid

The schematic, footprint selection, PCB layout, routing, and all fabrication decisions were done manually by following the Hack Club guide and the RP2040 datasheet. AI tools were used only in a supporting role — for example, researching general approaches to fixing a stubborn DRC thermal-relief/clearance error, and for helping structure and write up this documentation.

> **Project Integrity:** All schematic wiring, footprint selection, PCB routing, and fabrication file prep were done manually in KiCad by following the Hack Club Devboard guide and the RP2040 datasheet. AI was used only for research, troubleshooting suggestions, and documentation help.

---

## License

This project is licensed under the MIT License — see the [LICENSE](https://github.com/SunritHazra/HackroDevX/blob/main/LICENSE) file for details.
