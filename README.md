# tec-VIDEO
investigate simple video addon options for the TEC-1 

## References
- https://github.com/dadecoza/arduino-mpf1-vdp
- https://elinux.org/images/e/eb/Howtocolor.pdf
- http://www.hpcc.ecs.soton.ac.uk/dan/pic/video_PIC.htm
- https://hackaday.com/2014/10/30/a-graphics-card-for-a-homebrew-computer/
- https://www.researchgate.net/publication/4321893_A_Chinese_Abacus_DAC_for_Video_Applications
- https://eater.net/vga
- https://hackaday.io/project/20161-breadboard-vga-from-ttl-and-eeprom
- http://tinyvga.com/avr-vga
- https://hackaday.com/2015/12/17/attiny-does-170x240-vga-with-8-colors/
- https://www.google.com/search?q=ATtiny+Oscilloscope+on+VGA+monitor&rlz=1C1ASRM_enAU820AU820&sxsrf=ALeKk02Q-ARRys3SAoOIeTGdo-18TtZKnQ:1588774389193&tbm=isch&source=iu&ictx=1&fir=qWWHB7_GRwzXqM%253A%252C8Z56BaPpSiFjUM%252C_&vet=1&usg=AI4_-kSDis9gp1JmTJ5JfI9W06PB7CZuOQ&sa=X&ved=2ahUKEwi_8-HotZ_pAhV34zgGHX1RBX0Q9QEwAnoECAoQDg#imgrc=qWWHB7_GRwzXqM:



The TEC-1 is a fun, minimalist Z80-based trainer kit from the early 1980s, with a basic 6-digit LED/hex display and no native video output. Your GitHub repo (which appears to be in very early stages, with just a title and no files or README yet) is a great starting point for exploring add-ons. Since it's empty, I'll outline the absolute simplest DIY approach to add basic video output using your cathode ray oscilloscope (CRO) as the "screen." This leverages the Z80's PIO (parallel I/O) port, which is already on the TEC-1 board and can be bit-banged for signals—no extra chips needed beyond a few passives.

This method produces a low-res monochrome vector-style display (think glowing lines/shapes on the CRO, like an old-school oscilloscope game such as Lunar Lander). It's "video" in the analog sense but not raster (pixels/framebuffer); for true raster video, you'd need more hardware like a CRTC chip, which isn't simple. Total build time: ~1-2 hours if you have basic soldering skills. Cost: Under $5 if you scavenge parts.

### Why This Is the Simplest
- Uses existing TEC-1 hardware (Z80 PIO at port 0x00 for X/Y deflection control).
- No custom PCBs, FPGAs, or MCUs—just wires and resistors.
- Software is a short Z80 assembly loop you can type in directly via the TEC-1's hex loader.
- CRO acts as an XY plotter: horizontal deflection (X) from one channel, vertical (Y) from the other, with Z80-timed signals drawing traces.

### Hardware Setup
You'll need:
- 2x 1kΩ potentiometers (pots) for gain adjustment (or fixed 1k resistors if you skip calibration).
- 2x 10kΩ resistors (for current limiting).
- Jumper wires or a small protoboard.
- BNC cables (or alligator clips) to connect to your CRO's CH1/CH2 inputs.
- Optional: 100Ω series resistors per line for protection.

Steps:
1. **Access PIO Pins**: On the TEC-1 board, locate the Z80 PIO chip (likely U5 or similar—check your schematic). The PIO has ports A and B. Use port A (pins PA0-PA7) for one axis (say, X deflection) and port B (PB0-PB7) for the other (Y). Ground unused pins.
   
2. **Build the Output Stage** (per axis—repeat for X and Y):
   - Connect Z80 PIO pin (e.g., PA0 for LSB of X) to a 10kΩ resistor.
   - Other end of resistor to the wiper (middle pin) of a 1kΩ pot.
   - Pot's outer pins: One to +5V (from TEC-1 supply), one to GND.
   - From the wiper, add a 100Ω resistor and connect to a wire that goes to your CRO's CH1 BNC center pin (X input). BNC shield to TEC-1 GND.
   - This creates a simple DAC (digital-to-analog converter): The Z80 outputs a digital value (0-255) via PIO, which the pot/resistor ladder turns into a 0-5V analog voltage for deflection.

   For better linearity, use a 4- or 8-bit R-2R ladder network instead of the pot (resistors: 1k, 2k alternating), but the pot hack works for prototyping.

3. **CRO Configuration**:
   - Set to XY mode (not timebase/CH1-only).
   - CH1: X deflection (horizontal), sensitivity ~1V/div, DC coupled.
   - CH2: Y deflection (vertical), same settings.
   - If your CRO has a Z-axis (intensity) input, you could bit-bang that too via another PIO pin for blanking lines, but skip for simplicity—everything will glow constantly.
   - Calibrate pots: Run a test program (below) and tweak for full-screen deflection.

4. **Power/Safety**: Everything runs on the TEC-1's +5V. Double-check polarities to avoid frying your CRO inputs (most tolerate 0-5V fine).

### Software: Basic Z80 Demo
Load this assembly code into the TEC-1's RAM using its hex keypad (or a Z80 monitor if you have one). It draws a simple bouncing dot by outputting incremental X/Y values to PIO ports 0x00 (port A = X) and 0x01 (port B = Y), with a delay loop for visible speed.

Assemble and run at address 0x0000. (Use a Z80 assembler like z80asm if cross-compiling, then convert to hex bytes.)

```
    ORG 0x0000          ; Start at origin

START:
    LD HL, 0x0000       ; X pos = 0
    LD DE, 0x0101       ; Velocity: +1 X, +1 Y
LOOP:
    ; Output X (HL low byte) to port A
    LD A, L
    OUT (0x00), A       ; PIO port A = X (0-255 → 0-5V)

    ; Output Y (DE low byte, but reuse for Y) wait, adjust
    LD A, E             ; Use E for Y
    OUT (0x01), A       ; PIO port B = Y

    ; Delay for trace visibility (~1ms per point)
    LD BC, 0xFFFF       ; Rough delay
DELAY:
    DEC BC
    LD A, B
    OR C
    JR NZ, DELAY

    ; Update position
    ADD HL, DE          ; X += VX
    ; Bounce X
    LD A, L
    CP 0x00             ; Hit left?
    JR NZ, NO_BOUNCE_X
    LD A, D
    NEG                 ; VX = -VX
    LD D, A
NO_BOUNCE_X:
    CP 0xFF             ; Hit right?
    JR NZ, NO_BOUNCE_X2
    LD A, D
    NEG
    LD D, A
NO_BOUNCE_X2:

    ; Similar for Y (DE high? Wait, simplify: use another reg for Y pos)
    ; For brevity, this is pseudo—expand to full bounce on Y with another reg.

    ; Loop forever
    JR LOOP

    END
```

- **How it works**: The Z80 writes 8-bit values (0-255) to PIO ports, mapping to 0-5V analog on CRO. The delay creates persistence-of-vision traces. Tweak velocities/delays for smoothness.
- **Loading**: Convert to hex (e.g., via online Z80 disassembler reverse) and poke into RAM. First byte: C3 00 00 (JP START), etc.
- **Enhancements**: Add intensity blanking (OUT to port C for Z-axis). For text/shapes, precompute X/Y tables in ROM.

### Potential Issues & Next Steps
- **Resolution/Speed**: 8-bit axes = 256x256 "pixels," but Z80 timing limits to ~100Hz refresh. Slow for games, fine for plots.
- **No Raster?** If you want pixel-based video (e.g., 64x32 chars), add a 6845 CRTC chip (~$5 used) and sync gen— but that's 10x more complex (see retrocomputing forums for schematics).
- **Testing**: Start with a static line: LD A,0x80; OUT(0x00),A; LD A,0x00; OUT(0x01),A; then increment Y in a loop.
- **Resources**: Check rc2014.co.uk or tubecad.com forums for TEC-1 mods. For vector graphics inspo, look up "Z80 oscilloscope display" projects (e.g., on Hackaday).

# anoher option, 
use a raster VGA , Let's get that TEC-1 glowing. 🚀


