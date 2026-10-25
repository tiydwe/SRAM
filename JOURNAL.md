---
title: "SRAM from scratch"
author: "tiydwe"
description: "256 bytes of SRAM from scratch! No ICs at all. Made from over 20 000 transistors!"
created_at: "2026-10-01"
---



# 2026-10-02: Research

**Total time spent: 2 hours**

![image.png](images/2026-10-2_sramcell.png)
Source: https://commons.wikimedia.org/wiki/File:SRAM_Cell_(6_Transistors).svg
## What I did
I spent some time researching how modern RAM works and DRAM vs. SRAM. I initially considered DRAM for this project but the main issue was that although the DRAM cell would take more transistors, it would probally actually be more complex because of the refresh stuff and timing would be even worse.
### How DRAM works
Basically in DRAM, you have a transistor connecting a single capacitor to a bit line. The issue here is that your capacitor will slowly leak charge out onto the bit line, meaning that every once in a while the RAM has to read the row and refresh it. Modern ram does this every like 30ms, but I did the math and I will only have to refresh every like minute because my capacitors would be HUGE compared to what is in a chip (btw, they literally just put a cylinder inside another one thats a few 100s of nm big to store a few thousand electrons in modern DRAM).

Although this can save lots of space when you are talking about gigabytes of ram, with 256 bytes of ram, it would be more work, complexity, component count, and space probally to get all that refresh logic working while AT THE SAME TIME making sure refresh didn't interfere with the normal operations.

### SRAM
SRAM uses a system where two inverters make a positive feedback loop. If the first inverter outputs HIGH, then the second outputs LOW, making the first output HIGH. This positive feedback loop means that it doesn't have to be refreshed making peripheral circuitry much easier. The main issue is that each cell requires 6 transistors (instead of 1 transistor and 1 capacitor), making the actual cell more space consuming.

I decided to go with SRAM because it is easier to make, less things can go wrong, and I don't have to mess with crazy timing.

## Lapse links
I didn't record this because it was really spread out but it was over about the week before I pitched this.


# 2026-10-03: Research body diode stuff

**Total time spent: 2.5 hours**

![alt text](images/2026-10-3_body-diode.png)

Mosfet schematic with body diode

## What I did
So I forgot that mosfets have an intrinsic body diode, which makes things a lot more annoying. The issue is that a MOSFET acts as a great switch for electricity in one direction, but if the voltage is the other direction, current will just flow because of the body diode. This is a major issue because if you notice in the SRAM cell in the previous devlog, M_5 and M_6 are supposed to be acting as bidirectional switches. If they had a body diode, regardless of what direction it was pointed, it would either cause one of the BL to get pulled LOW or one of the Q/~Q to get pulled low, corrupting the data.

I spent a lot of time trying to come up with a satisfactory solution and I think the only real solution here is to put two mosfets back to back to act as a true bidirectonal switch. The main disadvantage is that this will require 2 more mosfets PER cell, which will take more space and components.

While I was reasearching, I started wondering, how do modern SRAM chips use a 6T layout? It turns out that the body diode is not *truely* intrinsic between source and drain, it's really between the body (get it, body diode) and the drain, but in most mosfets the body is tied to the source. In chips, they just tie body to ground which I cannot do.


# 2026-10-04: Ran some simulations and started design for main board!

**Total time spent: 7 hours**
<figure>
 <img src="images/2026-10-4_sram-cell-simulation.png" alt="Final simulation results">
 <figcaption>Final simulation results! Explanation of this below.</figcaption>
</figure>
<figure>
 <img src="images/2026-10-4_initial-pcb-layout.png" alt="Dense 4-layer pcb with lots of sot-23 transistors">
 <figcaption>My PCB layout! Explanation below too.</figcaption>
</figure>

## What I did
### Simulating
I started simulating a single SRAM cell to make sure that the resistances were properly matched and I wouldn't get a read failure, write failure, or any crazy transients. Initially, I had some really bad transient current spikes on a read operation, but that was because I was improperly pulling up the bit lines. Then I had to debug a ton of write failures due to the bit line pull-downs being to weak. I fixed that by putting 3 transistors in parallel to pull the bit lines down hard, but I'll probably use like 5 to make sure that it works in the main design (I'll simulate it too).

### PCB
I designed out the main board PCB and the schematic. There are (as expected) a ton of DRC errors so I need to fix those. The main board PCB is actually the easiest part of this because its extremely repetitive. I used [Project Instances](https://github.com/OfficialDyray/ProjectInstances/) and hierarchical schematics in KiCad to repeat the same block over and over. As I was writing this I just realized there is a huge issue with the logic level inverter, so I'm going to fix that first.

## PCB explanation

The PCB has 3 main parts: The main SRAM bank, the 1:16 demultiplexer, and the logic level shifter/NOT gate. The SRAM bank is the huge grid in the bottom right corner along with the two columns in the middle of the left side. It's really just one cell repeated 128 times (if you zoom in you can see the single 8 transistor cell). The demultiplexer is the block of transistors on the left side. The demultiplexer is basically a few not gates (left side) and 16 5 way NAND gates (right side). It isn't like a "true" demultiplexer because it outputs NOT of whatever it should be. This is really useful because it can easily be fed into a logic level shifter / inverter to output the correct signal at 10v. The logic level shifter / inverter is the small column of transistors in the middle. Now right now, it won't work (just realized that), but I'm going to fix that soon.

## Explanation of that really cool image of the simulation results

The image of the simulation shows the current bit the cell is storing (Q) and the not bit line (~BL). Upon startup, Q is initially high (in real life, it will be random, I may or may not decide to change that).
### Writing HIGH
The first operation is to write HIGH. To do this, both bit lines are weakly pulled HIGH and the bit line that should be low is strongly pulled LOW. Here, ~BL is LOW because ~HIGH = LOW. Then WL is pulled HIGH, opening the access transistors. Since the cell is already in the same state, nothing happens.
### Reading
The second operation is to read. To read, both bit lines are again weakly pulled HIGH and the WL then pulled HIGH. Since the bit lines are weakly HIGH, the side of the cell that is strong LOW will pull the bitline almost all the way to LOW. Here, since the cell is HIGH, ~Q = LOW, and ~BL is pulled LOW, which you can see. This can easily be read and converted to a signal.
### Writing LOW
To write LOW, the same procedure in writing HIGH is followed. This time, ~BL is HIGH and BL is LOW. As you can see, the cell immediately flips as soon as WL is HIGH (WL not shown).

### A note about transient current events
Each write where the cell is flipped causes a very short transient current event. This is because current shoots through the pull up PMOS in the cell, through the access NMOS transistors, and through the bit line pull down NMOSs. Hopefully it won't be an issue because its only about 12 amps and only lasts until the cell flips, which in the simulation was 200-300 us.

## Lapse links:
The lapse videos are sram 1 through sram 7 here (yes i accidently put two sram 6s, they are different) https://lapse.hackclub.com/user/@tiydwe


# 2026-10-04: Fixed the level-shifter issue!

**Total time spent: 1.5 hours**

![image.png](images/2026-10-4_level-shifter-schematic.png)
<small>New schematic, notice the pmos has been replaced by simple pull-up resistors</small>
![image.png](images/2026-10-4_slower-level-shifter-simulation.png)
<small>Small time step of simulation, green is Q, blue is ~BL, red is WL. Notice that its now slower  because of the pull up resistor/parasitic capacitance.</small>

## What I did
I replaced the PMOS pull up mosfets in the level shifter with pull up resistors. Previously the level shifter would work fine on a LOW signal (PMOS gate at -10v so ON and NMOS gate at 0v so OFF) but fail on a 5v HIGH signal (PMOS gate at -5v so ON and NMOS gate at 5v so ON). On a 5v HIGH signal, there would be a huge short-circuit current through the PMOS and NMOS (upwards of 30A in the simulation). The fix here I choose is to forget complicated CMOS logic and just replace the PMOS pull-up with a 10k ohm resistor. The advantage is the incredible simplicity and smaller footprint (I don't know how I would fix another like 5 transistors per WL in that area). The main disadvantage is that it causes the WL to take one or two milliseconds longer to rise, but that's totally unimportant here because the max clock speed this is designed for will be 10Hz (100ms / clock cycle).

### Other PCB stuff
Here's the new part of the PCB with the pull-up resistors.
![image.png](images/2026-10-4_pcb-with-pullup.png)

I also added the 2x20 IDC header for connecting the boards together.
![image.png](images/2026-10-4_idc-header.png)

## Lapse link & time:

I spent some time working on it when I didn't realised it was paused, ~30mins. I did some more simulation (you can see the jump at around the 13 second mark in lapse)

https://lapse.hackclub.com/timelapse/vyfiN1gcHwnZ


# 2026-10-04: Did some more simulation and found parasitic resistance issues

**Total time spent: 1.5 hours**

![image.png](images/2026-10-4_parasitic-resistance-simulation.png)
Parasitic resistance issues are fun! /s

## What I did

I ran a bunch more simulations on the single SRAM cell and added in parasitic effects. Previously I ignored parasitic capacitance and inductance because my clock speed is so s l o w, 10Hz. This is correct, but for some reason I totally forgot about the parasitic resistance in long PCB traces and connectors (especially between the boards).

## I hate parasitic resistance

Parasitic resistance is the unavoidable resistance in PCB traces, connectors, and everything in between. I thought that a few tens of milliohms would not make a difference and just ignored it earlier, but it turns out that adding just ten milliohms of resistance to the bit lines can cause write failures and strong current transients (tens of amps for a whole clock cycle). This is horrible news because even if I made traces wider (which I'll do anyway cause its good to lower resistance), the resistance from controller board to end of BL will easily be over 10 mOhms.

The reason that resistances on the bit line causes write failures is because to flip the cell, one side must be pulled <b>HARD</b> to LOW, hard enough to overcome the PMOS pull-up. Now, because the current flows from VCC to PMOS though the two back to back NMOSs and then through the array of parallel NMOSs, the cell gate voltage is determined by a voltage divider, specifically `V_g = (R_access + R_blpulldown) / (R_pmos + R_access + R_blpulldown) * Vcc`. This `V_g` must be low enough that the cell flips, which is when the NMOS overpowers the PMOS. In my case, it was about 2v-3v. It just so happened that because of my specific resistances, the system was basically balanced on a knife's edge. `R_blpulldown` was about 10mOhms, `R_access` about 80mOhms, and `R_pmos` about 120mOhms. This gives a voltage of 2.1v which is right at the edge of what will produce a valid write. Remember, the transistors won't be perfect too so even without the parasitic resistance many of the cells would likely fail. With 10mOhms of extra `R_blpulldown` resistance, `V_g` would be about 2.3v, causing the write to fail.

## The fix 
I tinkered around with the circuit for a good hour or so and finally have come up with probably the best solution in my case. If you read the previous paragraph, you might have noticed that the only way to reduce the voltage is to reduce `R_blpulldown` and/or `R_access`, which is impossible OR increase `R_pmos`. I found adding a large resistance in series with the p mosfets completely solved the issue. This can cause some read instability so I tested it a lot but I really didn't find anything.

Additionally, I found that I could get away with only using one 47 ohm resistor per cell instead of two by having both PMOSs share the resistor. This can cause some issues with voltage sag, but I didn't see any. This really does matter because even 1 component per cell saves 2048 components overall. Still, I need to find a way to fit one more resistor on each cell which is already packed super full, so that will be a challenge. I don't want to make the cell bigger because then I'll have to basically re route the whole board again which will take like 4 hours.

## Lapse link & time
About 1.5 hours (I'm just going off of my hackatime total and subtracting how much I already have)
https://lapse.hackclub.com/timelapse/Vni9q50XzASu


# 2026-10-06: Added LEDs to the board!

**Total time spent: 3 hours**

![image.png](images/2026-10-6_main-pcb-with-led.png)
The new back of the PCB with the LEDs
![image.png](images/2026-10-6_sram-cell-schematic-with-led.png)
The new schematic.

## What I did

I added the resistors I talked about earlier. I stuffed them in the corner of the cell and miraculously, there weren't any important DRC errors!

I also added the LEDs to the cells. I would have really liked to be able to directly put LEDs on it, but that would draw too much current so I instead put another mosfet to control the LED which will not draw any extra current. This adds 3 extra components per cell (mosfet, resistor, and LED), but I'm not too concerned because it is totally optional and you can just not put LEDs at all and it will still work.

Of course, I simulated this in LTspice just to be sure it will still work and it looks fine. I also added some decoupling capacitors because you do not want the voltage supply sagging every time the transistors switch.

I also tried getting a quote for JLC pcb assembly, and it is **expensive**. It cost around $400, but $250 comes from the board and the components, so the assembly cost about $150. If I ever do try building this, I think the only option is assembly because I don't want to solder 28 000 components by hand (even with solder paste and hot plate, that will take like 50 hours). 

### Figured out what all those weird transients are from

Earlier, I noticed that my circuit was drawing large current transients on certain clock cycles. Well, I finally figured out why. So basically my bit lines are fed from a LTspice variable voltage source which I set to have a 1ms rise time. To ensure that the BL signals were sharp, I fed it through two inverters (also to test the characteristics of my inverters). Now, this was successful at producing very, very sharp edges (see image below) BUT I totally forgot since the actual signal fed into the first inverter was (comparatively) slow at rising AND the inverter was not fed by a line with a resistor, significant time was spent in the linear region of both transistors, causing a dead short.

I think this can be fixed by just putting a small resistor before the CMOS power, but I need to do more research/simulation because that can cause the CMOS inverter's voltage supply to sag, which can cause issues.

![image.png](https://cdn.hackclub.com/01a1141c-daab-7795-aa66-a5b7d069228c/image.png)

Stages of my two inverter setup. Green, blue and red are the input signal, output after 1 inverter, and output after 2 inverter signals respectively. Notice how the blue signal is sharper but still pretty distorted while the red signal is lightning fast.

## Lapse links

https://lapse.hackclub.com/timelapse/rngF5m_neMcB<br>
https://lapse.hackclub.com/timelapse/9JZD9-N2Nd2J<br>
https://lapse.hackclub.com/timelapse/2NOMcpykwXSq


# 2026-10-07: Designed the main demux board!

**Total time spent: 2 hours**

![image.png](images/2026-10-7_demux-pcb.png)
The completed PCB
![image.png](images/2026-10-7_demux-schematic.png)
The schematic

## What I did

Today, I finished designing the schematic for the main demux board (I started in a previous devlog but didn't talk about it). The main demux board basically converts the 16 ram boards into one large 256x8 array by demuxing the most significant 4 bits and enabling only the WL on the correct board. The board itself was really simple, just 16 connectors for the ram boards and 1 connector for the external connection to the controller board. The only logic on it is about 200 transistors for the 1:16 demux which I just copy/pasted from the main RAM board.

I didn't simulate this board because the logic is dead simple and copy/pasted from the previous board, so if that one works this one should work too (though I actually havent simulated the demux in that one yet so I'll do that probably next devlog, hopefully I don't find any issues).

I'm planning on manual assembly for this board because one, I only have to assemble one board of 200 transistors so that's only a few hours, and two, it basically doubles the price of the boards if I get them assembled. There really isn't much more to say, but now I have to start to work on the actual hard part of the project, the RAM controller board.

## Lapse links
https://lapse.hackclub.com/timelapse/sfn6wTg3hgO3<br>
https://lapse.hackclub.com/timelapse/YwO8DZddrAsB<br>
https://lapse.hackclub.com/timelapse/HZtLlioMd9gv<br>


# 2026-10-08: Started the main controller board!

**Total time spent: 2.5 hours**

![image.png](images/2026-10-8_main-schematic-progress.png)
The progress I've made on the main schematic (not very much)
![image.png](images/2026-10-8_dlatch-simulation.png)
Simulation I made of a D-latch

## What I did
Today, I began work on the main controller board. It's harder than I thought and I'm still struggling to wrap my head around everything that needs to be done. I've barely done the processing part of the input stage and not even the d latch yet. Here's what I've come up with

### What the main controller board does
The main controller board is given an 8-bit address, 8-bit data (optional), and a command. If the command is a write, it will write the data to the address. Regardless, the RAM will read whatever data is at the address (the new data if its after a write) and output that.
The hard part here is timing. Every signal needs to have a clearly defined time when it is valid. Because the RAM array doesn't have any built in timing, the RAM board needs to handle that. Here's how I decided the RAM controller will work:
### How I'm going to design the RAM board
The ram board will be composed of 3 separate timing stages: the input stage, the ram stage, and the output stage.
The ram board will take all input on a RISING clock edge and output on the next FALLING clock edge.
#### Input stage
The input stage processes the input and outputs at the correct time and in a correct state to the ram stage. The input stage takes the current input (address, data, and command) and process that into BL commands (pull up or down) and WL address commands. The BL and WL commands are fed into a D-latch. A rising edge detector detects the rising edge and sends a pulse to the EN input of the D-latch, which passes the bit line commands and the WL commands into the RAM stage. The input stage also sends a pulse to the RAM stage to signal it.
#### RAM stage
The ram stage only has 1 job: make sure the BL is correct before enabling the WL. The RAM stage always applies whatever BL commands it gets from the input stage irrespective of the clock because the input stage already guarantees BL commands are synchronized. Upon receiving the signal pulse, a circuit will delay the pulse then extend it to 20-30 ms. The extended pulse is the WL signal, which is delayed to ensure the BL commands have been applied before enabling the WL.
Finally, the RAM stage sends the WL pulse to the output stage.
#### Output stage
The job of the output stage is to read the RAM at the correct type and output clock-synchronized read. The raw bit lines from the ram array are fed into a circuit which processes the BLs into data. This data is fed into ANOTHER d-latch reads it after a delay from the rising edge of the WL pulse. This ensures that the BLs have time to settle AND the WL is still enabled when read. Finally, the output from the d-latch is fed into another d-latch that is controlled by the clock to clock-synchronize it.

---

Okay. That was a lot and I'm sure I made lots of mistakes which is why I'm working on simulating it. But hopefully that will work.

Today, all I did was start with the input stage processing. This board is by far the most complex for sure though.

## Lapse links
https://lapse.hackclub.com/timelapse/HK0Go3kwnsZ8
https://lapse.hackclub.com/timelapse/240UbDXo5Uvz
https://lapse.hackclub.com/timelapse/HlwZQAF2P8GN


# 2026-10-09: Finished the input stage schematic

**Total time spent: 1.5 hours**

![image.png](images/2026-10-9_finish-schematic-input.png)
The finished schematic for the input stage
![image.png](images/2026-10-9_dlatch-simulation.png)
Simulation of a single d-latch. Green is Q, blue is EN, and red is D
![image.png](images/2026-10-9_schmitt-trigger-simulation.png)
Simulation of a Schmitt trigger. Green is the messy, noisy input signal, blue is after the Schmitt trigger, and red is the edge detector 2ms pulse.
## What I did
I finished the schematic for just the input stage! The two things I made were the d-latches and the rising edge detector/Schmitt trigger. I simulated both because both of them can be kind of finicky. 

It's good I simulated the Schmitt trigger because all the designs online are for ICs and basically assume that all the mosfets have high resistance, but since I'm using low resistance discrete mosfets, the "standard" Schmitt trigger circuit would cause a 30 amp short between VCC and GND every switch. I fixed this by adding some resistors. I had to do a lot of tuning (mostly because I've never worked with Schmitt triggers before) but I'm now able to control the threshold voltages by modifying two resistors (the two 27 ohm ones in the picture) and I've removed the huge short circuits.

I also simulated the D-latch but there weren't any circuit issues (I do have to put two back-to-back mosfets again like in the main SRAM cell because my mosfets will have body diodes) and you can see the simulation in the picture above.

I wired everything up and I'm going to work on the schematic for the RAM stage next (I'll do all the PCBs last).

## Lapse link
https://lapse.hackclub.com/timelapse/8shvfVoj0mrP


# 2026-10-11: Finished the main schematic and power management

**Total time spent: 2.5 hours**

![image.png](images/2026-10-11_finished-schematic.png)
The finished schematic
![image.png](images/2026-10-11_power-management.png)
Power management
![image.png](images/2026-10-11_pulse-simulation.png)
Simulation of the pulses. Blue is the input stage pulse, green is the WL pulse, and red is the pulse for reading.

## What I did
I spent a ton of time on LTspice trying to figure out how in the world to delay and elongate a pulse using RC circuits. Here's what I came up with:
- First, the pulse from the edge detector is used to enable the dlatch input.
- The circuit then takes the NOT of the pulse and uses another rising edge detector to detect the falling edge of the dlatch input. This rising edge detector is tuned so the pulse takes a while.
- The circuit then uses an inverter to convert that into a cleaner wave and a capacitor to get rid of some high frequency ringing.
- Then the output of that is connected to yet another inverter with a resistor on the pmos to limit the speed which a capacitor can charge, creating a delay.
- Then, this signal is inverted twice more to make a nice clean signal.

I used this strategy twice to create the two signals for the WL and reading.

I also used a LM3478 to get the 10v supply. Although I originally planned to use a simple charge pump, I found that for one, that won't provide very much current, and two, no ICs only applies to the logic part of this in my opinion.

I still need to do the PCB which has a ton of stuff because there are like 30 sub schematics to layout and stuff.

## Lapse links
https://lapse.hackclub.com/timelapse/-xv_CujW-8NQ
https://lapse.hackclub.com/timelapse/67BOb2_xgU9D


# 2026-10-13: Changed back to a charge pump!

**Total time spent: 1.5 hours**

![image.png](images/2026-10-13_charge-pump-sim.png)
Simulation of oscillator and charge pump. Green is the output voltage (under 2A of load), red is the oscillator output, and blue is the voltage across one capacitor in the charge pump.

## What I did
I decided to not use a IC actually and use a charge pump instead. The charge pump runs at around 50kHz and uses a signal generated from an inverting Schmitt trigger fed back into itself with a rc circuit. I had to spend a good amount of time trying to figure out how to get the oscillation stable and to work. The signal is fed into two inverters to drive two parallel charge pumps. I tested the charge pumps in LTspice and it can drive around 2A without sagging too much. It outputs slightly above 10v, so I'll probably end up using some sort of zener diode to bring it down to 10v or 9v or just a bunch of diodes.

## Lapse links
https://lapse.hackclub.com/timelapse/0Xzy1dWXinuu


# 2026-10-14: Did more simulation and wrote the (current) BOM

**Total time spent: 2 hours**

![image.png](images/2026-10-14_bom.png)
The BOM I made
![image.png](images/2026-10-14_demux-sim.png)
Simulation of a 1 to 2 demux. Green is output 0, blue is output 1, and red is the select signal
## What I did
I simulated the 1 to 2 demux in LTspice and it seems to work fine. I'm kind of procrastinating to making the PCB so I just wrote up the BOM so I don't have to search up the components every time. That's basically all I did today.

## Lapse links
https://lapse.hackclub.com/timelapse/A2O3CRLZz1La


# 2026-10-15: Re routed the 1 to 16 demultiplexer

**Total time spent: 2.5 hours**

![image.png](images/2026-10-15_demuxFixed.png)
The new correctly routed 1 to 16 multiplexer
## What I did
So it turns out that the correct order is read datasheet THEN spend 20 hours routing... Pretty sad. Basically, my kicad model had Gate-Drain-Source on the sot-23 mosfet pins but the correct pinout is Gate-Source-Drain. That was pretty sad to figure out and I just spent 2 hours routing the 1:16 demux board (this time I checked BOTH datasheets for pmos and nmos to be sure). I think I'll probably have to take another like 15 hours to re-route everything. At least I can fix some annoying things from last time.

## Lapse links
https://lapse.hackclub.com/timelapse/dMLu61iej506


# 2026-10-16: Re-routed the rest of the main RAM board

**Total time spent: 2.5 hours**

![image.png](images/2026-10-16_ramBoard.png)
The re-routed RAM board
## What I did
I just finished re-routing the rest of the board to the correct GSD configuration. I also fixed the LED mosfets because I accidently put the n-mosfets high-side instead of low-side, which is not good, so I fixed that which was pretty easy. Other than that, not much more of note today.
## Lapse links
https://lapse.hackclub.com/timelapse/37X8GlZVRLlY


# 2026-10-17: Made some SMT to DIP boards for testing

**Total time spent: 1.25 hour**

![image.png](images/2026-10-17_smt-to-dip.png)
The panelized board (there will be v-cuts between each smt to dip board)
## What I did
I was feeling a bit burned out so I took a break and designed a small SMT to DIP board for testing my parts on a breadboard. Each board can support a sot-23, 1206, 0805, and 0603.

## Lapse links
https://lapse.hackclub.com/timelapse/5zcHNgXlIq7x


# 2026-10-18: Routed the main demultiplexer board and started the main controller board

**Total time spent: 2.75 hours**

![image.png](images/2026-10-18_demux.png)
The main demux board
![image.png](images/2026-10-18_charge-pump.png)
The 5v to 10v charge pump
## What I did
I finished fixing the demultiplexer board to use the correct G-S-D pinout. It was pretty easy because I copy/pasted the schematics I placed out earlier.

I also started the routing for the main controller board which was harder. The most interesting thing was the charge pump. One, because it switches at 50kHz its bound to have a lot of noise so I had to put a bunch of decoupling capacitors (i'm going to literally be using like 1000s of decoupling capacitors across this whole project). Two, it's designed to handle high current so I'm using super thick pcb traces. I'm also kind of lazy and don't want to get actual power mosfets so I'm just putting like 3 signal mosfets in parallel (simulation shows each should only).

The charge pump uses two parallel pumps operating 180 degrees apart to provide more current and smoother opreration. I'm using 1000uF capacitors with a ESR of 14mOhms to make sure this works well. Also the diodes are wayyyy over speced for this (they are rated for 15 amps) but that means that they will basically never overheat in normal conditions and I don't need to heatsink them. The diodes are very fast recovery too and have a good forward voltage. This is fine because here over designing stuff is really cheap while making a mistake is super expensive.

## Lapse links
https://lapse.hackclub.com/timelapse/JS_6fl9y1UiG
https://lapse.hackclub.com/timelapse/_ycI8WU505L9
https://lapse.hackclub.com/timelapse/ZLOBHP1drQfX


# 2026-10-18: Finished routing all sub-schematicss for the main controller board

**Total time spent: 1h**

![image.png](images/2026-10-18_sub-schematics.png)
The sub schematics all arranged neatly. No traces routed yet
## What I did
I finished routing the sub schematics for the main controller board. It wasn't that interesting and was pretty easy, but I arranged it on the main board and it looks like its going to be a huge pain to route. I'm going to try and use a 150mmx150mm main board to fit everything but I might have to make it bigger later. The issue is the d-latches, they take up a huge amount of space and I use 5 of them. The main controller board basically has to have 8 bytes of extra memory, and its a d latch so it's super space consuming.

I really want to order some parts for testing and I'm waiting for a different project to get rated in stardance so I can get a pcb and electronics grant from there. In the meantime, I'll try to route this board and maybe do some simulation work.


# 2026-10-21: Added mounting holes

**Total time spent: 1.5 hours**

![image.png](images/2026-10-21_sram-mountin.png)
The main sram board with the mounting holes

## What I did
I realized that I totally forgot mounting holes on my SRAM board, so I added them. I wasn't sure if I should ground my mounting holes so I just put a zero ohm resistor between the mounting hole and GND. Another issue was that my computer was low on battery and KiCad was being super slow with like 2000 components on the pcb.

I also fixed up the 8 bit dlatch in the main controller board because for some reason the order of the bits was all messed up.

## Lapse link
https://lapse.hackclub.com/timelapse/NCqCv2yI-q3Z


# 2026-10-21: Routed the data lines for the main controller board

**Total time spent: 1.5 hours**

![image.png](images/2026-10-21_data.png)
Doesn't it look so nice when all the data lines are nice and tidy?
## What I did
I routed all the data lines on the main controller board. Now that the dlatch ordering is all fixed up it was all nice and tidy.

It may seem like I am finishing this project incredibly quickly, but the main problem is NONE of the stuff here has been tested in real life. I really want to get some parts and I'm waiting on a project in Stardance to get reviewed so I can get some pcb/electornics grant money to buy a small amount of components to test them. I really should run more simulations but simulations are like mind-dulling and boring.

## Lapse links
I did some work while lapse wasn't recording, probably about 30 minutes so I added that to the time
https://lapse.hackclub.com/timelapse/39xw5WRfKN29


# 2026-10-24: Added the through holes for the buttons

**Total time spent: 2 hrs**

![image.png](images/2026-10-24_throuh-holes.png)
The new squeezed in thorough holes
# What I did
I started routing the main controller board but then I realized that I forgot the input buttons on the SRAM board. As it would be 100% impossible to fit the button and control circuitry onto the main board, the only option for me was to add some sort of through hole to attach another pcb ON TOP to manually write to it with a switch. This was pretty hard because through holes take ALL layers which made it almost impossible for me to fit the relatively GIANT 1x2 pin header. Even after reducing to a tiny 1mm pitch, 0.5mm diameter header (smallest on LCSC), I had to move the ~BL wire into a really weird position around it and fix some DRC errors. It took me an entire hour just to figure out how in the world to fit another part into a completely full PCB footprint without increasing the size.

Next time, I'll make that breakout board, adding yet ANOTHER separate board that I'll have to design, order, and assemble. Luckily, I don't need that many because only some of the SRAM cells need manual input.

# Lapse link
https://lapse.hackclub.com/timelapse/njcIv-Fqtl5y


# 2026-10-25: Created the input board schematic and started the pcb

**Total time spent: 1.5 hours**

![image.png](images/2026-10-25_input-board-pcb.png)

The pcb I worked on

## What I did
I worked on the input breakout board that I talked about earlier. The schematic portion was pretty easy, but they small 4.5mm buttons I was using weren't in the kicad footprint so I had to figure out how to make new ones based on the schematic.

I had to use really small buttons because otherwise it wouldn't properly fit into the 1 cell size. Other than that, it wasn't that hard to make. I really hope that I got the switch footprints correct. I'll check again later against the datasheet.

## Lapse link
https://lapse.hackclub.com/timelapse/jG3-Um5Zeg8E
