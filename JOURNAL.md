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

* [This](https://github.com/user-attachments/assets/96dde608-219d-472a-9682-96a95f16b071) is the power and decopuling.
* [This](https://github.com/user-attachments/assets/9e6f4866-57e1-4350-a7fc-b4eadb88d3cd) is the USB-C and step down.
* [This](https://github.com/user-attachments/assets/4e26f85b-4016-400b-a56b-7d17701c3016) is the crystal ossilator.
* [This](https://github.com/user-attachments/assets/0c8a8f67-9cef-4537-8e4d-47fdbf4ebf10) is the I/O pins broken out.

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
- **Header pins:** PinHeader_1x20_P2.54mm_Vertical for the two side headers and PinHeader_1x03_P2.54mm_Vertical for the extra 3-pin header.
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

Then came the part I was most careful with: the USB-C differential pair. I routed them randomly without realising the mistake I was making. I fixed my mistake on Day 4 by routing two D+ pins and two D- pins on the connector together first, then routed them from the USB-C connector, through the 27Ω termination resistors, and into the RP2040's USB_DP/USB_DM pins, leaving a gap inside the pair for the decoupling caps to sit in later.

I then went through and routed the rest of the RP2040's decoupling caps, the crystal's load caps and damping resistor, the USB-C CC pulldowns, and the button pull-up resistor, switching between the front copper layer for vertical-ish runs and the back copper layer (with vias where I needed to hop layers) for the rest, roughly following the guide's suggestion of routing front signals vertically and back signals horizontally where it made sense.

Back on my own board, I spent the rest of the night wiring up the header pin nets one by one, working through the GPIO breakout, and got a good chunk of the power distribution (VBUS/GND/+3V3) routed to the main cluster of components too. I left the boot button and its associated traces for later, since like the guide says, there's no one specific spot it needs to be, so it's an easy thing to slot in wherever there's room left in the devboard.

By the end of the day I had roughly half the board routed, with the fast USB signals, the flash memory, the crystal, and most of the decoupling done, and the header pins and remaining power distribution still to go.

Here's a glimpse of what I had achieved by the end of the day:

<img width="1365" height="708" alt="image" src="https://github.com/user-attachments/assets/65aa217f-e758-4a21-a872-f5569a3b56e7" />

Here're the lapses of today's session: [HackroDevX-LPS-2-D2-1](https://lapse.hackclub.com/timelapse/aFMjs7GqHVZj) and [HackroDevX-LPS-2-D2-2](https://lapse.hackclub.com/timelapse/7HHVShKjE44z)

**Total time spent: 2h 20m**

---

# Day 3 — 29.06.2026: Completing the Routing & Exporting PCB


Today I finished off the rest of the routing, working through the remaining header pin nets and finishing power distribution out to the pins further from the main cluster, until there were zero unrouted nets left on the board. It was really exhausting and also ironically fun at the same time. The feeling: with every net, I was a step closer to `0` unconnected pins; is just undefinable.

With routing done, I added the ground fill: drew a filled zone across both copper layers, set the net to `GND`, and used thermal reliefs for the pad connections with a minimum spoke count of `2`, same as the guide walks through. Running DRC after that threw a handful of `Thermal relief connection to zone incomplete` errors on a few isolated pads, on the crystal, a couple of the header pins, and one of the load caps, exactly the kind of ground-pour headache the guide warns you about. I went into `Board Setup → Design Rules → Constraints` to see what my clearance and thermal relief settings actually were, and also asked Google's `AI Mode` how to fix a DRC clearance error, which laid out two general approaches:

- **Change the layout** — spread the offending traces apart with the push/shove router, manual dragging, or a full re-route.
- **Loosen the design rule constraints** — if your manufacturer can handle tighter tolerances.

I ended up re-filling the zones and adding a few extra vias onto the isolated ground islands to get everything properly connected to the pour.

Next was silkscreen. I searched up how to batch-convert all the `F.Fab` layer text to silkscreen, since I had reference designators and values sitting on the fab layer that needed to move, and found that KiCad's footprint text field batch update lets you filter items by layer (set to `F.Fab`), tell it to apply to functional text items, and then change the target layer to `F.Silkscreen` in one go. After that I went through and manually cleaned things up further, deleting the less useful `C#`/`R#` labels for the tiny passives and keeping the important ones, and double-checked the bottom-side GPIO labels were mirrored and oriented properly so they'd actually read correctly from that side of the board.

Then it was time for a bit of art. I grabbed the Hack Club "Orpheus" flag logo from [here](https://hackclub.com/brand), ran it through KiCad's built-in Image Converter to threshold it down to black and white, and copied it to clipboard and exported it on the `F.Silkscreen` layer. I placed that near the crystal along with a `"HackroDevX"` text label and my name, so the board has a bit of personality on it.

I ran DRC one more time to make sure everything was clean, then went to `File → Fabrication Outputs` and exported everything (gerbers, drill files, footprint position files) with KiCad `9.0.7` into a `HackroDevX_Gerber` folder, keeping a `HackroDevX-backups` folder alongside it from earlier in the day. Following the guide, I opened the CPL (`top-pos`) file in Google Sheets and renamed the headers:

- `Ref` → `Designator`
- `PosX` → `Mid X`
- `PosY` → `Mid Y`
- `Rot` → `Rotation`
- `Side` → `Layer`

since that's what JLCPCB expects, and did the same for the BOM file, renaming `Designation` to `Comment`. I zipped the whole gerber set into a production zip and uploaded it to JLCPCB's quote page, added PCBA to the order for a batch of `5` assembled boards, and uploaded the renamed BOM and CPL files.

Going through JLCPCB's part matching, most parts auto-matched fine, the `RP2040` QFN-56, the `MCP1700x-330xxTT`, the USB-C receptacle, the crystal, and the resistors and caps. The flash memory needed a manual nudge, JLCPCB had it in stock as `W25Q16JVUXQTR` rather than the exact `W25Q16JVZPIQ TR` from my design, which the guide actually mentions can happen since they're functionally the same part with minor differences. I made sure to leave the pin headers off the assembled parts list, since those are cheap enough and easy enough to hand-solder myself instead of paying JLCPCB's per-part assembly fee for them.

With the BOM fully matched, I've landed at the JLCPCB cart, everything uploaded and matched, but not checked out yet.

Here's the lapse of today's session: [HackroDevX-LPS-3-D3](https://lapse.hackclub.com/timelapse/XPbqHUrSpcms)

**Total time spent: 6h 10m**

---

# Day 4 — 19.07.2026: Journaling & Fixing D-/D- Net Lengths

I mostly spent my time journaling days #1, #2, #3, and #4.

Day #1

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/b9ea5bd4-51e1-4089-b8d9-33b83add5f64" />

Day #2

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/ac897c66-ad11-4717-8b91-cbc877174f25" />

Day #3

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/c8560e19-21fa-4fb1-9d32-f70ed8702798" />

**There are two imporatant things to be noted:**

1. I also fixed a major bottleneck of the project: the crazy length difference between the D+ and D- length. It was a whopping 6.5 mm+ in different and the maximum I could afford was 0.2 mm. At the time, I did not realize that it would have major drawback with the fast incoming data. Anyways, I fixed it by moving the switch a little to the right and unconnecting some nets. Then, using differential pair, I properly routed them and made sure that the difference between their length is minimal. With that ensured, I connected the things that I had moved and unrouted.

2. After the lapse [HackroDevX-LPS-4-D4-1](https://lapse.hackclub.com/timelapse/-wHSsLC3JRmy), I started another lapse where I logged Day 3 full. But unfortunately, the second lapse failed and didn't compile. The time I took for the failed session was roughly a little over one hour. Like, 1 hour 6 minutes or something.

Here're the lapses of today's session: [HackroDevX-LPS-4-D4-1](https://lapse.hackclub.com/timelapse/-wHSsLC3JRmy) and [HackroDevX-LPS-4-D4-2]()

**Total time spent: 5h 00m**

---

# Day 5 — 20.07.2026: Completing the Repository



Here's the lapse of today's session: [HackroDevX-LPS-5-D5]()

**Total time spent: 0h 00m**

---
