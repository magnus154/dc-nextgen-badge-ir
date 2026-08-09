# Tanuki Badge IR Touch Protocol

Badge #266, sitting at 21 pts. Two of these DC NextGen badges can bump each other and both get a point. They do this using infrared light. We want to know exactly what gets sent when that happens. Well enough to fake it.

Done in collaboration with GitHub user [lebowitz](https://github.com/lebowitz). Badge created by Relic, who also gave us some of the hints below in person.

Disclosure: Claude AI was used heavily throughout this, running the scripts, digging up parts, writing most of this doc. It didn't do everything on its own though. We were steering it the whole way.

![Badge back, PCB visible through the case](media/badge-back-pcb.jpg)
![Badge front, display showing #266, 21 pts, star row, and a hand-drawn egg icon](media/badge-front-display.jpg)
![Bare board with the display flipped open on its ribbon cable](media/badge-board-mcu.jpg)

Back, front, and the board with the case popped open and the screen flipped out.

## Why

If we know what a touch says, we can have a Flipper Zero say it back. Then we could rack up points without needing other badges around. But the fake touch has to actually fool the badge, not just look close. That took a few tries to figure out.

## Where things stand

The badge talks over infrared, the same invisible light a TV remote uses. We expected a Flipper Zero to read that easily. It didn't. Figuring out why became the whole story here.

The Flipper's IR receiver only listens for one blink speed, built into its hardware. This badge blinks at some other speed. Every recording we made came back as junk. Every guess we sent back got ignored. Points stayed at 21 no matter what.

The fix is better tools: a sensor that can hear any speed, and a recorder fast enough to see it clearly. Both are ordered. This doc is the notebook until they arrive.

## Glossary

Terms that came up along the way, in the order they show up below.

| Term | What it means |
|---|---|
| IR (infrared) | Light your eyes can't see, but a sensor can. |
| Carrier frequency | The speed a signal blinks at, underneath the actual data. Both sides have to agree on this speed to understand each other. |
| Baseband | No blinking at all, just a plain on/off signal. This is what the badge turned out to use. |
| Demodulator | A chip built to understand only one blink speed. The wrong speed comes out as garbage. |
| Mark / space | The "on" and "off" parts of an IR signal. Their lengths usually carry the real data. |
| Checksum | Extra math added to a message so the receiver can tell if it got scrambled. Like a receipt total that has to match its line items. |
| Aliasing | What happens when a fast signal is measured too slowly. A fake, slower signal appears that never really happened. |
| Logic analyzer | A tool that records a wire's on/off state, many times a second. |
| AC-coupled | A sensor that only reacts to a changing signal, and ignores anything steady. Good for filtering out sunlight and lamps. Bad for reading a plain on/off signal. |
| MCU (microcontroller) | The chip in charge, running everything. |
| GPIO | A basic pin on a chip that software can turn on or off, or read. |
| UART | A simple way for two chips to talk, one wire each direction. |
| UPDI | A single wire this chip uses to get reprogrammed, or to have its memory read back out. |
| SerialUPDI | Using a cheap USB adapter, plus one resistor, to do UPDI instead of buying a special programmer. |
| Fuse bits | Special settings stored on the chip that control low-level hardware behavior, like what a pin's job is. |
| Lock bit | One specific setting that can block reading the chip's memory. This badge doesn't have it set. |
| Duty cycle | How much of one on/off cycle is spent "on." 100% duty cycle means always on, no blinking. |
| Boost converter | A small circuit that turns a lower voltage into a higher one. |
| E-paper | A screen that only needs power to change what it shows, then holds the image with no power at all. |
| Flash / SRAM | Flash is a chip's permanent storage. SRAM is its working memory, which clears when the power goes off. |

## Tools

A Flipper Zero (updated to firmware 1.4.3 partway through this), and a laptop running small scripts to talk to it over USB.

## Try 1: hit Learn, like teaching a universal remote

The Flipper has a Learn mode for IR: point it at something, press the button, it saves whatever it saw. It doesn't check whether that made any sense. We aimed the badge at it, did a touch, hit Learn. Twice:

```
$ storage read /ext/infrared/Remote.ir
data: 123 125186 135 4012 88
```

```
$ storage read /ext/infrared/Remote2.ir
data: 493 3943 187 1925 240 216 125 428 126 227 125
```

(Both files are saved in this repo as `Remote.ir` and `Remote2.ir`.)

Neither of these is a real signal. A real IR message is dozens of tightly packed blinks, all following one steady rhythm. `Remote.ir` has a gap of 125,186 microseconds sitting between two blips that only last a few hundred microseconds each. That's not a rhythm. That's noise, with a long silence in the middle. `Remote2.ir` is the same story, its gaps jump all over the place.

One more trap: both files say `frequency: 38000`. That looks like a real measurement. It isn't. It's just the number the Flipper writes into every file by default, since the receiver throws away the real speed before the software even sees it. Don't trust that number.

So the receiver wasn't actually hearing the badge. It was just twitching at a speed it doesn't understand. Like tuning a radio to a station that doesn't exist: you get static, not a garbled version of something real.

## Try 2: listen longer, hold the button down

We watched the Flipper's live feed for 25 to 40 second stretches, doing several touches in a row. We also held the badge's little contact button down the whole time, in case it only sends anything while that's pressed. (It does.)

Mostly, nothing. One run caught a 3-blip fragment. A full 25 seconds with the badge held right against the sensor caught zero.

That rules out bad aim. If the receiver could hear this badge at all, holding them together for 25 seconds would catch something steady, not silence one time and three random blips the next. It just can't hear this badge's speed.

## Try 3: throw random signals at it

New question: maybe the badge doesn't check carefully, and any IR blast nearby bumps the counter. Worth testing before doing anything harder. We fired a bunch of standard remote-style signals at it, at six different speeds, a few patterns each, 48 tries total, button held the whole time.

Points stayed at 21. Every single time.

So the badge is checking something before it counts a touch. Probably matching a format, and a checksum. We can't guess our way past that. We need to see a real message first.

## Side note: touching the same badge twice doesn't score twice

Separate from the Flipper testing, we noticed something. Touch the same two badges together a second time, and nothing happens. Only the first meeting between two specific badges scores.

That means each badge remembers who it's already touched, probably by ID number, not just how many touches happened. This matters for our plan: once we can replay one captured message perfectly, replaying that exact same message again and again will likely only ever score once. The badge already has that ID marked off. To keep the score climbing, the replayed message probably needs a different sender ID each time.

## Then we talked to Relic, who built the thing

We met Relic in person, at the DC NextGen party at the Las Vegas Convention Center during DEF CON 34. Relic gave us a few hints that settle some of what was still a guess above:

- **It does not use a carrier.** This confirms what comparing our three noise captures already pointed to: not a carrier that's just far from the Flipper's speed, no carrier at all. The badge just switches its IR light on and off directly. No fast blinking underneath. This is exactly why the Flipper's receiver, and maybe the sensor we ordered too, never stood a chance. Both of them expect a carrier to be there.
- **The firmware is not locked.** This removes the one real risk in reading it out. No chance of hitting a locked chip and having to choose between giving up or erasing it. A read should just work.
- **We need a USB-to-UART adapter to read the firmware.** This confirms the plan: a cheap USB adapter can act as a programmer, over a header we found on the board (more on that below).

One problem this creates: the sensor we already ordered is built to catch a flickering signal, and ignore a steady one. A signal with no carrier looks steady, not flickering. So that sensor might not see this badge's signal properly. Worth adding a plain IR photodiode too, one with no filtering, wired straight into the logic analyzer, so we can see the raw on/off signal with nothing filtered out.

Replaying a no-carrier signal through the Flipper also needs a different setting than anything we tried in Try 3. All that speed-sweeping assumed a carrier existed. To fake "no carrier," the LED just needs to stay fully on during each "on" period, instead of flickering at any speed. Worth testing once we actually have a real capture.

## Why the Flipper couldn't do this

The Flipper's IR side is really two separate pieces. Sending is simple, just a plain LED that can be switched at nearly any speed, which is how the speed-sweep in Try 3 worked. Receiving is the hard part: a sealed chip built to only understand one speed, with no way to change it. Turning blinks into data happens inside that chip, before the Flipper's own software even gets involved. Wrong speed in, garbage out.

We also checked the Flipper's separate Logic Analyzer tool, which watches a raw pin directly. It only measures about 100,000 times a second. That sounds like a lot, but to properly catch something blinking 30,000 to 60,000 times a second, you need to measure several times faster than that, or you get a distorted picture instead of a missing one. That's called aliasing. Same reason a spinning wheel can look like it's spinning backward in a video, the camera just isn't fast enough to keep up.

![Diagram of a fast signal sampled fast enough versus sampled too slow, producing a fake slower wave](media/aliasing-explained.png)

Look at the bottom half of that picture. Every orange dot is a real, honest measurement. But they're spaced too far apart, so connecting them draws a slower red wave that never actually happened. That's aliasing: not missing data, confidently wrong data. The top half shows what measuring fast enough looks like instead, dense dots that trace the real thing. That's why we ordered a much faster recorder.

## What's getting wired up

From Adafruit ($25.70 total): a [TSMP96000](https://www.adafruit.com/product/5970), a sensor built to hear a wide range of speeds instead of just one, plus a cable, some jumper wires, and a breadboard.

From Amazon (~$18.50): a USB logic analyzer that measures 24 million times a second, hundreds of times faster than needed.

Still to add: a plain IR photodiode with no filtering, as a backup to the TSMP96000, now that we know the badge uses no carrier at all.

The sensor's job is to turn the invisible light into an electrical signal. The analyzer's job is to record exactly what that signal did, so we can actually look at it.

## Plan once it all shows up

Short version: wire the sensor to the analyzer, confirm the signal shows up cleanly, catch a real touch, figure out what it means, replay it through the Flipper, then automate it. Full steps are further down.

Things that could still go wrong: only having one badge, which makes catching a real two-badge exchange harder. A checksum that changes with every touch, which would mean understanding real math, not just copying one capture. And now that we know there's no carrier, our first sensor might not be the right tool, so a plain photodiode may be needed alongside it.

## Popped the case open too

![Bare board with the display flipped open on its ribbon cable](media/badge-board-mcu.jpg)

There's a coin battery, and what looks like a small power circuit: a small coil marked `L1`, a bigger one marked `KLJ-1230`, and a small transistor marked `Q2`. This is almost certainly what turns the coin cell's 3 volts into whatever the display needs to redraw. The screen itself is e-paper, meaning it only needs power to change what it shows, and holds the picture with no power the rest of the time. Worth remembering when testing, a slow-changing screen can look like nothing happened when something actually did.

We went back for closer, sharper photos of the two things that mattered most.

![Close-up of the MCU chip, marked AVR128DA28 with the Microchip logo](media/mcu-closeup.jpg)

The chip: a **Microchip AVR128DA28**. We could read the marking clearly, plus a date code. It's a small computer chip with 128KB of permanent storage and 16KB of working memory. It has no special hardware built in for infrared, so the badge's IR signal is handled entirely by its own software.

![Close-up of the six-pin header, labeled G V U T R W](media/header-closeup.jpg)

There's also an unused six-pin header on the board, labeled `G, V, U, T, R, W`. That reads as Ground, Power, a programming pin, and two more for basic serial communication. Relic confirmed the firmware isn't locked, so this header is our way in.

## Step by step: wiring up the sensor and analyzer

**1. Power and wire the TSMP96000.** It has three pins: power, ground, and signal, all labeled right on the board. Use those labels instead of guessing by wire color. Power it from a clean source between 3 and 5 volts, the Flipper's own header has a spare power pin and ground pin that work fine for this. Run the sensor's signal wire to the first channel on the logic analyzer. Connect the analyzer's ground to the same ground as everything else.

![Breadboard layout showing the TSMP96000 wired through the breadboard to the Flipper's GPIO header for power and ground, and to the Xicoolee analyzer's CH0 for signal](media/breadboard-layout.png)

Power and ground from the sensor land on the breadboard and jump up to its rails. Those rails feed the Flipper's power and ground pins. The signal wire runs straight across to the analyzer, and the analyzer's own ground ties into the same rail. Three wires in, three wires out, nothing else on the board.

**2. Get the software talking to the analyzer.** Install a free program called PulseView on the laptop, plug the analyzer in, and pick the right driver for it if it doesn't show up automatically.

**3. First capture: see what's actually there.** Turn the first channel on, set the recording speed as high as the software allows, since this only needs a few milliseconds. Start recording, hold the badge's button, hold the badge right against the sensor, wait a second, then stop. Since there's no carrier, this should look like a simple on/off pattern, not a fast flicker. If the sensor shows nothing useful, try the plain photodiode instead.

**4. Capture a real touch.** Once the sensor is working, record a few more touches. If a second badge is around, try to catch two real badges touching each other, that's the real exchange, not just one side talking to nobody. Save each recording as a file in this repo, so the actual data lives here, not just notes about it.

**5. Read out what it means.** Use the software's tools to measure how long each "on" and "off" stretch lasts, either by hand or by exporting the data and writing a small script. Look for a repeating pattern, most remote-style signals use pulse length to mean a 0 or a 1. Once that's worked out, look for the badge's own ID number somewhere in the pattern, and note which part changes between different recordings. That's probably the checksum, or maybe tied to the already-touched memory we found earlier.

**6. Replay it through the Flipper.** Since there's no carrier to worry about, the LED just needs to stay fully on during each "on" period. Send that pattern at the badge with its button held, and watch the score. If it doesn't work the first time, double-check whether anything got flipped or reversed while decoding it.

**7. If it scores, solve the repeat-scoring problem.** Since the badge only counts one score per sender ID, try changing that part of the message before each replay, and see if that's enough to keep the score climbing on its own.

## Dumping the firmware directly

There's a second path that doesn't need any of the hardware above. Since Relic confirmed the firmware isn't locked, and told us we'd need a USB-to-UART adapter, this should be a fairly direct read over the header we found on the board.

**What's needed:** a cheap USB-to-UART adapter. Connect its two data wires together through a resistor, and connect that joined point to the badge's `U` pin. Connect the adapter's ground to the badge's `G` pin. This trick is called SerialUPDI. Leave the badge running on its own battery rather than powering it from the adapter, so skip the `V` pin for now.

**Before wiring anything,** double-check which pin is which with a multimeter, rather than trusting our guess from the board's printed labels.

**Software:** a free tool called `pymcuprog`.

```
pip install pymcuprog
pymcuprog ping -d avr128da28 -t uart -u <serial port>
pymcuprog read -d avr128da28 -t uart -u <serial port> -m flash -o 0x0000 -b <size>
```

Only read, don't erase or change any low-level settings. That's the one way this could actually go wrong, one specific setting controls whether that pin can even do this at all, and undoing a mistake there needs special recovery equipment.

Worth trying either way. A clean firmware read would hand us the exact signal format directly, and skip most of the steps above.

## Files in here

`Remote.ir`, `Remote2.ir`, and `Remote3.ir`: the junk recordings from Try 1, kept as proof of what doesn't work. `flipper-backup-2026-08-08.tar.gz`: a backup of the Flipper's settings from before its firmware update. `media/`: badge photos, close-ups, and our two diagrams. `FlipperHIDecoder/`: an unrelated tool we grabbed early on while poking around at what the Flipper can do, not part of this project, kept for reference.
