---
**Title:** HackroDevX  
**Author:** Sunrit Hazra  
**Description:** A development board (devboard) based on Raspberry Pi RP2040 SoC with USB-C connectivity.
**Created on:** 2026-06-30  
**Designing Progress (%): 90%**
**Building Progress (%): 0%**
---

# Day 1 — 27.06.2026: Drawing the Schematic, Assigning Footprints & Placing Components

For the development board I have followed the Starter Project Guide from Hack Club Stasis. Before I started, I was thinking about what I can name my project. I guessed I could name it "HackroDevX"

In this naming system, there are three parts:

* The first phrase is a either a random word or "Hackro".
* The second phrase is a short phrase taken out of the thing's real name like "Dev", "Pad", or "Split".
* The third part is just a letter like "X" or somtehing else.

Thus giving the name "HackroPadX".

With the name confirmed, I opened KiCad and started reading the guide at the same time.

Here's how I did the schematic:

- First I placed down the RP2040 symbol in the center of the sheet, since it's the fundamental of the whole board.
- Then it was power and decoupling. The RP2040 has a bunch of VDD pins, so I placed one 0.1uF decoupling capacitor per power pin, eight of them for the 3.3V IOVDD line and two for the 1.1V DVDD line, plus a larger 1uF bulk capacitor on each of the two lines to smooth out bigger ripples. All power labels face upward and grounds face downward, as good schematic practice goes.
- Next was USB-C. I added a USB-C receptacle, wired SHIELD and GND to ground, and gave the D+/D- pins global labels since I'm only using one sheet. The CC1/CC2 pins got pulled down to GND through 5.1KΩ resistors so the port actually knows to deliver power.
- For stepping 5V down to 3.3V, the guide has you start with an NCP1117-3.3_SOT223 LDO, so that's what I placed first, with a 10uF bulk capacitor on either side of it, VBUS labeled before it and +3V3 labeled on its output.
- The USB D+/D- lines go into the RP2040's USB_DP/USB_DM pins through a pair of 27Ω termination resistors to stop signal distortion at high speed, and I made sure to set the D+/D- global labels to bidirectional since they carry data both ways.
- Then the crystal oscillator: a 12MHz crystal (Y1) with the standard two 15pF load capacitors on its I/O pins, pins 2 and 4 tied to GND, and a 1KΩ damping resistor on XOUT to protect the crystal and keep the signal clean. XIN/XOUT got global labels going to the RP2040.
- Since the RP2040 doesn't have any onboard flash, I added a W25Q128JVS quad-SPI flash IC and wired up all the QSPI pins (CLK, CS, and the four IO lines) with global labels matching the RP2040's QSPI pins, gave it a 0.1uF decoupling cap on VCC, and added a 10KΩ pull-up on the CS line plus a BOOTSEL push button in series with a 1KΩ current-limiting resistor down to GND, so the board can be dropped into BOOTSEL mode on boot.
- Last was breaking out the I/O. TESTEN got tied straight to GND since that pin's only for factory testing, and then I labeled every remaining GPIO along with SWCLK/SWDIO with their own names. I placed two 1x20 header symbols and one 1x3 header symbol laid out to match the Pi Pico pinout, since it's a pinout people are already used to. I skipped VSYS, 3V3_EN and ADC_VREF since I'm not adding battery support, and used the freed-up pins for extra GPIOs instead, including GPIO29 for the extra ADC-capable pin, no-connecting GPIO25 in its place.

* This is the power and decopuling: <img width="553" height="139" alt="image" src="https://github.com/user-attachments/assets/96dde608-219d-472a-9682-96a95f16b071" />
* This is the USB-C and step down: <img width="564" height="333" alt="image" src="https://github.com/user-attachments/assets/9e6f4866-57e1-4350-a7fc-b4eadb88d3cd" />
* This is the crystal ossilator: <img width="393" height="170" alt="image" src="https://github.com/user-attachments/assets/4e26f85b-4016-400b-a56b-7d17701c3016" />
* This is the I/O pins broken out: <img width="452" height="456" alt="image" src="https://github.com/user-attachments/assets/0c8a8f67-9cef-4537-8e4d-47fdbf4ebf10" />

Here's the full schematic:

<img width="3507" height="2480" alt="image" src="https://github.com/user-attachments/assets/9f5b815b-f46e-4861-bd8f-c9ad13450b54" />

With the schematic done, I ran ERC and got these 4 violations:

<img width="672" height="411" alt="image" src="https://github.com/user-attachments/assets/b94dea23-1723-4678-9995-7fa41b519661" />

Three of them were the standard annoying "Input Power pin not driven by any Output Power pins" warning the guide already tells you to expect and ignore, since we know the LDO and the RP2040's own regulator are genuinely supplying those rails. The fourth flagged the RP2040's VREG_VOUT pin and the LDO's output pin as two power outputs sitting on the same net, which I made a mental note of to keep an eye on, since that's not one of the errors the guide tells to ignore. It happened I kind of messed up some of the wiring as the symbol I was using was a little different from what was shown in the guide.

This is my mistake:

<img width="1366" height="733" alt="image" src="https://github.com/user-attachments/assets/c08c151a-44a4-4494-9332-b908b86933df" />

And I fixed it carefully by understanding what the connections meant.

<img width="1366" height="733" alt="image" src="https://github.com/user-attachments/assets/6a7e96a0-3e53-44df-ba0f-44bcd3efd251" />

Next came footprints. For the most part I stuck close to what the guide uses, since JLCPCB's basic/extended part split makes a lot of these decisions for you anyway:

- **RP2040:** QFN-56-1EP_7x7mm_P0.4mm_EP3.2x3.2mm, straight off the datasheet.
- **Decoupling capacitors and resistors:** 0402 for all the small signal caps and resistors, since anything smaller gets hard to hand-place and 0402 is plenty stable for low current.
- **10uF bulk capacitors:** 0603, since they need a bit more room than the tiny 0402 footprint comfortably allows.
- **USB-C receptacle:** USB_C_Receptacle_HRO_TYPE-C-31-M-12, since JLCPCB doesn't have a true basic-library USB-C part.
- **Header pins:** PinHeader_1x20_P2.54mm_Vertical for the two side headers and PinHeader_1x03_P2.54mm_Vertical for the extra 3-pin header, all through-hole so they're strong enough to survive being plugged into repeatedly. These I'm buying separately and hand-soldering, not paying JLCPCB to assemble.
- **Boot button:** SW_Push_SPST_NO_Alps_SKRK, a compact SMD footprint sitting in JLCPCB's basic library, so it doesn't add anything to the assembly cost.
- **Crystal:** Crystal_SMD_3225-4Pin_3.2x2.5mm, also a JLCPCB basic part. I double-checked the footprint's pin 1/3 orientation against the datasheet since it's really easy to get a crystal's pinout backwards.
- **LDO:** switched from the NCP1117-3.3_SOT223 to the smaller MCP1700x-330xxTT in a SOT-23 package. It only handles 250mA instead of the NCP1117's much larger current, but that's more than enough headroom for the devboard.
- **Flash:** switched to the W25Q16JVZPIQ TR in the Winbond_USON-8-1EP_3x2mm_P0.5mm_EP0.2x1.6mm package, the same smaller flash chip the actual Pi Pico itself uses, instead of the bigger W25Q128JVS from the datasheet.

Because I swapped in a different crystal than the guide's reference part, its load capacitance was slightly different, so I bumped the two load capacitors from 15pF up to 33pF to match.

With footprints assigned, I switched over to the PCB editor and hit F8 to update the PCB from the schematic. I drew the board outline next: the guide uses a compact **21×51mm** Pico-shaped outline, but I gave myself more breathing room (length) unintentionally and sized mine at **21mm × 51.5mm** instead.

From there I placed the two big 1x20 headers using Positioning Tools against the board's corners, then placed the RP2040 dead center then moved it a little down, the USB-C connector up near the top edge, the LDO close to VBUS, the flash memory right next to the RP2040's QSPI pins, and the crystal tight against XIN/XOUT with its load caps and damping resistor sitting right beside it. Then I went through and grouped all the decoupling caps next to whichever pins they were actually decoupling, RP2040 caps together, LDO bulk caps together, and so on.

By the end of the day I had every component placed except some capacitors and resistors and the board outline drawn, but nothing routed yet.

Here's the lapse of today's session: [HackroDevX-LPS-1-D1](https://lapse.hackclub.com/timelapse/FK_UCOY82zEO)

**Total time spent: 3h 15m**

---

# Day 2 — 28.06.2026: Routing

Today was all about routing. I started, like the guide suggests, with the flash memory's QSPI signals, since those are simple, short, low-speed traces and a good warm-up, temporarily moving the nearby decoupling caps out of the way so I had room to work.

Then came the part I was most careful with: the USB-C differential pair. I wired the two D+ pins and two D- pins on the connector together first, then held down the route track button and switched to the differential pair routing tool to run the pair down from the USB-C connector, through the 27Ω termination resistors, and into the RP2040's USB_DP/USB_DM pins, leaving a gap inside the pair for the decoupling caps to sit in later. Once that was in, I used the "Tune length of a single track" tool to check the two resistor traces matched in length, and "Tune skew of a differential pair" to nudge the shorter USB trace until both legs of the pair matched up, so data actually arrives at both pins at the same time.

Here're the lapses of today's session: [HackroDevX-LPS-2-D2-1](https://lapse.hackclub.com/timelapse/aFMjs7GqHVZj) and [HackroDevX-LPS-2-D2-2](https://lapse.hackclub.com/timelapse/7HHVShKjE44z)

**Total time spent: 2h 20m**

---

# Day 3 — 29.06.2026: 

Here's the lapse of today's session: [HackroDevX-LPS-3-D3](https://lapse.hackclub.com/timelapse/XPbqHUrSpcms)

**Total time spent: 6h 10m**

---

# Day 4 — 19.07.2026: 

Here's the lapse of today's session: 

**Total time spent: 3h 50m**

---
