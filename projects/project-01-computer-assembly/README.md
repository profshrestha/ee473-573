# Project 1: Computer Assembly

- CPU: AMD Ryzen 9 9900X (AM5)
- Motherboard: MSI MAG X870 TOMAHAWK WIFI
- RAM: G.SKILL Trident Z5 Neo RGB 32 GB (2 × 16 GB) DDR5-6000 CL30-38-38-96 1.35 V, EXPO
- GPU: NVIDIA RTX 5080 (MSI SHADOW 3X OC)
- SSD: Crucial T705 PCIe Gen 5 NVMe M.2
- CPU Cooler: Thermalright Peerless Assassin 120 SE
- PSU: MSI MAG A1000GL PCIE5 (1000W)
- Case: LIAN LI LANCOOL 216 E-ATX

Build order matters here. The CPU, memory, SSD, and cooler all go onto the motherboard while
it is still on the bench, because the cooler overhangs the memory slots and the M.2 socket
once it is mounted. The case does not come out of its box until step 6.

## Instructions

1. **Prepare your workspace.** Work on a hard, non-carpeted surface. Wear an anti-static wrist strap, or regularly touch an unpainted metal surface to discharge static before handling components. Keep all component boxes and manuals nearby.

2. **Install the CPU.**
   - Unlock the motherboard's CPU socket lever and lift the retention arm.
   - Align the gold triangle/notch on the AMD Ryzen 9 9900X with the triangle on the socket.
   - Lower the CPU straight down with no force. It should drop in flush.
   - Close the retention arm and lever to lock the CPU in place.

   <img src="images/cpu-in-open-socket.jpg" alt="AMD Ryzen 9 9900X resting in the open AM5 socket, retention bracket swung clear" height="300"> <img src="images/cpu-socket-latched.jpg" alt="Load plate closed down over the seated Ryzen 9 9900X before the lever is locked" height="300">

   The CPU sits flat in the open socket on the left, and the load plate is closed over it on the right. Do not press down on the CPU itself at any point.

3. **Install the memory.**
   - The kit is two 16 GB DDR5-6000 modules rated CL30-38-38-96 at 1.35 V, with an EXPO profile. Check the label before you start.
   - Use **DIMMA2 and DIMMB2**, the 2nd and 4th slots counting from the CPU. The board silkscreens exactly these two as `FIRST` beside the slots.
   - One module per channel is what gives you dual-channel operation. Putting both in the same channel halves your memory bandwidth and nothing will warn you.
   - Using the far slot of each channel also matters electrically. The traces run from the CPU through the first slot to the second, so leaving the end slot empty creates an unterminated stub that reflects signal. That is often the difference between the EXPO profile training and failing.
   - Open the retaining clip on each slot, line up the off-centre notch in the module's contact edge, and press straight down on both ends until the clips snap shut. It takes more force than you expect.

   <img src="images/ram-kit.jpg" alt="Two G.SKILL Trident Z5 Neo RGB modules in packaging with the DDR5-6000 CL30 EXPO spec label visible" height="280">

   <img src="images/dimm-slots-empty.jpg" alt="The four empty DIMM slots with the DIMMA1 to DIMMB2 silkscreen and FIRST marking" width="780">

   <img src="images/ram-installed.jpg" alt="Both modules installed in DIMMA2 and DIMMB2, leaving DIMMA1 and DIMMB1 empty" width="780">

   Note in the last photo that the populated slots are the 2nd and 4th, with an empty slot between them and another at the end. That is correct.

4. **Install the M.2 NVMe SSD.**
   - The drive is a Crucial T705, a PCIe Gen 5 NVMe M.2 SSD.
   - Use `M2_1`, the CPU-connected Gen 5 slot beside the memory slots. It is the only slot that will run this drive at full Gen 5 speed. The board silkscreens each M.2 slot with its source (`CPU` or `CHIPSET`) and which interfaces it supports, so read the labels rather than guessing.
   - Remove the M.2 heatsink/shield covering that slot.
   - Peel the protective film off the thermal pad underneath. It is printed with `REMOVE` across its whole face. Leaving it on insulates the drive from the very heatsink meant to cool it.
   - Slide the drive into the connector at roughly a 30° angle, then press the far end down flat.
   - This board uses a toolless retention clip rather than a screw. With the drive pressed flat, the clip latches over the end notch. No screwdriver needed. If the clip will not catch, the drive is not fully seated in the connector.
   - Reattach the heatsink/shield.

   <img src="images/ssd-kit.jpg" alt="Crucial T705 PCIe Gen 5 NVMe M.2 SSD in its packaging" height="260"> <img src="images/m2-slot-empty.jpg" alt="Empty M.2 socket showing the toolless retention clip" height="260">

   <img src="images/m2-shields.jpg" alt="M.2 heatsink shields in place, with the M2_2 CPU and M2_3 CHIPSET slot labels silkscreened on the board" width="780">

   <img src="images/m2-thermal-pad-film.jpg" alt="Protective films printed with REMOVE covering the M.2 thermal pads" height="330"> <img src="images/ssd-installed.jpg" alt="Crucial T705 seated in the M2_1 slot and held by the retention clip" height="330">

5. **Install the CPU cooler.**
   - Lay the kit out first. The box also contains Intel LGA115x/1200 hardware you will not use on this build, so set it aside to avoid confusion.
   - The mounting bars are stamped **AM4**. That is correct. AM4 and AM5 share the same cooler mounting pattern, so one bracket covers both sockets.
   - Screw the four red AM5 standoffs onto the board's mounting posts at the corners of the socket.
   - Lay the two mounting bars across the socket and tighten them down onto the standoffs.
   - Apply thermal paste: a pea-sized dot or a short line across the middle of the heat spreader. Do not spread it by hand. Clamping pressure spreads it far more evenly than you can.
   - Lower the heatsink squarely onto the CPU with its bracket over the two bars, then tighten the two captive spring screws **alternately**, a few turns on each side at a time, until they bottom out. Tightening one side fully first will cock the cooler and give you uneven contact.

   <img src="images/cooler-kit.jpg" alt="Peerless Assassin 120 SE kit laid out: wire clips, mounting hardware, backplate, two fans, and the dual-tower heatsink" width="780">

   <img src="images/am5-standoffs.jpg" alt="Four red AM5 standoffs installed at the corners of the CPU socket" height="300"> <img src="images/am5-brackets.jpg" alt="The two AM4-stamped mounting bars installed across the socket" height="300">

   <img src="images/thermal-paste.jpg" alt="Thermal paste applied to the centre of the CPU heat spreader" height="300"> <img src="images/heatsink-tighten.jpg" alt="Tightening the captive spring screw between the two towers with a screwdriver" height="300">

   - Clip the wire fan brackets into the mounting holes on each fan, then hook them over the fin stack.
   - **Fan orientation matters.** Air enters the open blade side and leaves the side carrying the motor label and support struts, so the **label side faces the fin stack**. Both fans face the same way, moving air front to back toward the rear of the case. Two small arrows moulded into the thin edge of the fan frame show airflow and rotation if you want to confirm it.
   - If the front fan fouls the memory heatspreaders, slide it upward on its clips.
   - Connect the fans to `CPU_FAN1`, using the supplied Y-splitter so both run from the one header.

   <img src="images/fan-clips.jpg" alt="A TL-C12C fan with the wire mounting clips and PWM cable laid out beside it" height="290"> <img src="images/fans-mounted.jpg" alt="Both fans clipped onto the dual-tower heatsink mounted on the motherboard" height="290">

   <img src="images/fan-cable-connect.jpg" alt="Fan cables routed down to the CPU_FAN1 header beside PUMP_SYS1 and SYS_FAN1" height="360">

6. **Unbox the case and remove both side panels.** Set aside the screw packs and standoffs that came with it.

7. **Check the case standoffs.**
   - **There is no separate I/O shield to install.** On this board the shield is built into the rear I/O shroud and comes pre-attached from the factory. Older boards shipped a loose stamped-metal shield that had to be pressed into the case first; this one does not, so do not go looking for one in the box.
   - Confirm the standoffs in the case line up with the motherboard's screw holes, and add or relocate standoffs as needed.
   - Just as important, make sure there is **no** standoff anywhere the board has no matching hole. A stray standoff under the board can short traces on the underside when everything is screwed down.

8. **Mount the motherboard in the case.**
   - Lower the motherboard in at an angle so the rear ports and their attached shield line up with the case cutout, then lay it flat onto the standoffs.
   - Secure with screws in a crisscross pattern, snug but not overtightened.

9. **Install the power supply.**
   - Mount the MSI MAG A1000GL PCIE5 in the PSU bay (fan facing down toward a vent, or up if the case has no bottom vent) and secure with the four included screws.

10. **Connect motherboard power cables.**
    - Connect the 24-pin ATX cable from the PSU to the motherboard.
    - Connect the 8-pin (4+4) EPS CPU power cable from the PSU to the motherboard's CPU power connector near the top.

11. **Install the GPU.**
    - Remove the appropriate rear expansion slot covers on the case for a double/triple-slot card.
    - Open the primary PCIe x16 slot latch, align the RTX 5080's connector with the slot, and press down firmly until the latch clicks and the card is seated flush.
    - Screw the GPU bracket into the case.
    - Connect the required PCIe power cable(s) from the PSU to the GPU (check the card for the number of connectors required).
    - If the case includes a GPU support bracket, install it to prevent sag.

12. **Connect case fans and front-panel connectors.**
    - Connect any case fans (pre-installed in the LANCOOL 216) to available `SYS_FAN` headers on the motherboard, or to a fan hub if provided.
    - Connect the front-panel header cables (power switch, reset switch, power LED, HDD LED) to the motherboard's front-panel header, matching pin labels in the motherboard manual.
    - Connect front-panel USB and audio headers to their corresponding motherboard headers.

13. **Cable management.**
    - Route cables through the case's cable-routing channels and behind the motherboard tray where possible.
    - Use zip ties or the case's built-in cable straps to secure loose cables away from fans.

14. **Final check before power-on.**
    - Verify RAM, GPU, and CPU cooler are fully seated.
    - Verify 24-pin, EPS, and GPU power cables are all connected.
    - Verify no tools or loose screws are inside the case.

15. **First boot.**
    - Connect a monitor to the GPU's video output (not the motherboard's).
    - Connect keyboard, mouse, and power cable, then power on.
    - Confirm the system POSTs (displays the motherboard splash screen), then enter BIOS to confirm the CPU, RAM (correct speed/capacity), and storage are all detected.
    - **Enable the memory's EXPO profile in BIOS.** Until you do, DDR5 runs at its JEDEC default of 4800 MT/s rather than the 6000 the kit is rated for, no matter which slots you used.
    - If the board fails to POST, the `EZ Debug LED` block (BOOT / VGA / DRAM / CPU) tells you which subsystem it stalled on, and the two-digit POST code display narrows it further.

16. **Close up the case.**
    - Once the system boots successfully, reattach both side panels.
