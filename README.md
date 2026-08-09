# Tanuki Badge IR Touch Protocol

Badge #266, sitting at 21 pts. Two of these DC NextGen badges can bump each other and both get a point over infrared. We want to know exactly what gets sent when that happens, well enough to fake it.

Done in collaboration with GitHub user [lebowitz](https://github.com/lebowitz).

Disclosure: Claude AI was used heavily throughout this, running the serial scripts, digging up parts, writing most of this doc. It didn't do everything on its own though, we were steering it the whole way.

![Badge back, PCB visible through the case](media/badge-back-pcb.jpg)
![Badge front, display showing #266, 21 pts, star row, and a hand-drawn egg icon](media/badge-front-display.jpg)
![Bare board with the display flipped open on its ribbon cable](media/badge-board-mcu.jpg)

Back, front, and the board with the case popped and the screen flipped out on its ribbon cable.

## Why

If we know what a touch says over IR, we can have a Flipper Zero say it back and rack up points without other badges around. The fake touch has to pass whatever checks the badge does, not just look close, which took a few tries to confirm.

## Where things stand

The badge talks over infrared, same kind of invisible light a TV remote uses. A Flipper Zero should read that no problem. It didn't, and figuring out why became the story here.

The Flipper's IR receiver only listens for one blink speed, ~38 kHz, baked into the hardware. This badge blinks at some other speed. Every recording came back as junk, and every guess fired back at the badge got ignored, pts sat at 21 no matter what.

The fix is a wideband sensor plus a recorder fast enough to actually see the signal. Both are ordered; this doc is the notebook until they land.

Quick terms: IR is light your eyes can't see but a sensor can. It's blinked on and off fast rather than sent as one steady beam, and both sides have to agree on the speed, called the carrier frequency, measured in kHz. A demodulator chip only understands one of those speeds; feed it the wrong one and it outputs garbage. A checksum is extra math tacked onto a message so the receiver can tell if it got scrambled, like a receipt total needing to match its line items, probably why blasting random signals at the badge did nothing. A logic analyzer records a wire's on/off state many times a second. MCU is the chip running the show.

## Tools

Flipper Zero (updated to 1.4.3 partway through this), and a laptop scripting it over USB.

## Try 1: hit Learn, like teaching a universal remote

Flipper's Learn mode saves whatever IR it sees, without checking whether it made sense. Aimed the badge at it, did a touch, hit Learn. Twice:

```
$ storage read /ext/infrared/Remote.ir
data: 123 125186 135 4012 88
```

```
$ storage read /ext/infrared/Remote2.ir
data: 493 3943 187 1925 240 216 125 428 126 227 125
```

(Full files saved in this repo as `Remote.ir`, `Remote2.ir`.)

Neither is a real signal. A real IR message is dozens of tightly packed blinks with one consistent rhythm. `Remote.ir` has a 125,186 microsecond gap between two blips that are only a few hundred microseconds long themselves, not a rhythm, noise with a long silence in it. `Remote2.ir`'s gaps are all over the place too.

Both files also claim `frequency: 38000` — that's not a measurement, it's the Flipper's default for every raw capture, since the receiver chip throws away the real blink speed before the software sees it. Don't trust that field.

The receiver wasn't hearing the badge's real signal, it was twitching at a pitch it doesn't understand. Same as tuning a radio to a station that doesn't exist: static, not a garbled broadcast.

## Try 2: listen longer, hold the button down

Watched the Flipper's live raw feed for 25-40 second stretches across repeated touches, holding the badge's contact button the whole time in case transmission is gated on that (it is).

Mostly nothing. One run caught a 3-blip fragment. A full 25 seconds at point-blank range caught zero.

That kills the bad-aim theory. A receiver that can hear this badge at all would catch something consistent over 25 seconds of contact, not silence once and three random blips the next. It can't hear this pitch.

## Try 3: throw random signals at it

Worth checking whether the badge validates anything at all, or just counts any IR blast nearby. Fired standard remote-style signals at it across six pitches (30, 33, 36, 38, 40, 56 kHz), several patterns each, 48 combinations, button held throughout.

pts stayed at 21 every time.

It's checking something before counting a touch, probably a format match plus a checksum. Can't brute-force past that; need to see a real message first.

## Side note: same badge twice doesn't score twice

Touching the same two badges together a second time does nothing, only the first meeting between two specific badges scores.

Each badge is remembering who it's already touched, by ID, not just counting events. That matters for farming points solo: replaying one captured message over and over will likely only ever score once, since the target already has that ID marked off. Climbing the counter probably means the replay needs a different sender ID each time.

## Why the Flipper couldn't do this

Its IR side is two pieces. Sending is a plain LED, controllable at nearly any pitch, which is how the carrier sweep in Try 3 worked. Receiving is a sealed chip hard-wired to ~38 kHz, no setting to change it; the blinks-to-data step happens inside the chip before the firmware gets a say. Wrong pitch in, garbage out.

The Flipper's Logic Analyzer mode, which watches a raw pin directly, samples ~100,000 times a second. Catching something blinking 30,000-60,000 times a second cleanly needs several times that rate, or you get a distorted picture instead of a missing one, called aliasing. Same effect as a spinning wheel looking like it's turning backward on camera because the frame rate can't keep up.

![Diagram of a fast signal sampled fast enough versus sampled too slow, producing a fake slower wave](media/aliasing-explained.png)

The bottom half of that chart is the trap: every orange dot is a real measurement of the real gray wave, but spaced too far apart, so connecting them draws a slower red wave that never happened. That's aliasing, confidently wrong data, not missing data. The top half is what sampling fast enough looks like: dense dots tracing the real thing. That's why the new 24-million-sample/second analyzer matters here.

## What's getting wired up

From Adafruit ($25.70 total): a [TSMP96000](https://www.adafruit.com/product/5970), a sensor built to hear any IR pitch from 20-60 kHz and pass the raw blinking through instead of assuming one fixed speed, plus an adapter cable, jumper wires, and a breadboard.

From Amazon (~$18.50): a USB logic analyzer sampling 24 million times a second, hundreds of times faster than needed.

The sensor turns the light into an electrical signal; the analyzer records exactly what that signal did.

## Plan once it all shows up

Short version: wire the sensor to the analyzer, find the real pitch, catch a real message, decode it, replay it through the Flipper, then automate. Step by step is at the bottom of this doc.

Open risks: only having one badge, which makes catching a real handshake harder. A checksum that changes per touch, which would need understanding the math, not just copying a capture. And the chance there's no carrier at all, just a flicker with nothing fast underneath, which the new sensor would still catch but replay through the Flipper (which assumes a carrier) might need a different approach.

## Popped the case open too

![Bare board with the display flipped open on its ribbon cable](media/badge-board-mcu.jpg)

Coin battery (CR2032), a large black part marked `KLJ-1230` that's probably a coil for the display driver, worth a datasheet search. Two chip packages, one is almost certainly the MCU, but the markings are too small and blurry here to read, need sharper close-ups of both. A row of unpopulated holes labeled something like `W R T U V G` looks like a debug or programming header, worth checking with a multimeter once the chip's identified. The screen is almost certainly e-paper, drawing power only to change what it shows and holding the image otherwise, worth remembering when testing since a slow redraw can look like nothing happened.

If that chip gets identified from a better photo, its datasheet or a firmware dump may already exist, handing over the carrier and packet format directly.

## Step by step: wiring up the sensor and analyzer

**1. Power and wire the TSMP96000.** The breakout has three pins: power, ground, and signal, labeled on the board itself, use those labels rather than guessing wire colors. Power it off a clean 3-5V source, the Flipper's own GPIO header has a spare 3V3 pin and a GND pin that work fine for this and don't touch anything the Logic Analyzer app reads. Run the sensor's signal pin to channel 0 on the Xicoolee analyzer, and tie the analyzer's ground to the same ground as the sensor and the Flipper.

![Breadboard layout showing the TSMP96000 wired through the breadboard to the Flipper's GPIO header for power and ground, and to the Xicoolee analyzer's CH0 for signal](media/breadboard-layout.png)

Power and ground from the sensor land in two breadboard columns and jump to the rails, the rails feed the Flipper's 3V3 and GND pins, and the signal column runs straight across to CH0 on the analyzer, with the analyzer's own ground tied back into the same rail. Three wires in, three wires out, nothing else on the board.

**2. Get PulseView talking to the analyzer.** Install sigrok/PulseView on the laptop, plug in the analyzer, and select the "fx2lafw (Cypress FX2 generic)" driver, since that's the chip inside it. If PulseView doesn't auto-detect the device, pick that driver manually from the connection dialog.

**3. First capture: just find the real pitch.** Set channel 0 active, sample rate as high as PulseView will allow for a short capture (start high since the window's only a few milliseconds). Arm the capture, hold the badge's contact button, hold the badge right up against the sensor, wait a second, stop. Zoom into the burst and measure the time between two blinks, then take 1 divided by that time to get the actual frequency. That number replaces every guess made so far.

**4. Capture a real message.** Once the pitch is known, set the sample rate to comfortably more than double it, five to ten times over is better, and capture several touches. If a second badge is around, capture an actual two-badge handshake instead of one badge beaconing by itself, that's the real exchange, not half of one. Save each capture as a `.sr` file in this repo so the actual data is here, not just notes about it.

**5. Read out the bits.** Use PulseView's cursors to measure each mark and space in a captured burst by hand, or export the channel and go through it in a script. Look for a repeating pattern, most remote-control style protocols use pulse length to mean 0 or 1. Once that's sorted, look for the badge's own ID (`#266`, or `0x10A` in hex) somewhere in the bits, and note which part of the message changes between different captures, that's probably the checksum, or possibly tied to the already-touched tracking noticed earlier.

**6. Replay it through the Flipper.** Convert the decoded pattern into the Flipper's raw format: `ir tx RAW F:<measured carrier> DC:<duty cycle> <mark> <space> <mark> <space> ...`. Fire it at the badge with the contact button held and watch pts. If it doesn't score on the first try, double check the duty cycle guess (33% is a reasonable starting point) and whether the bit order or polarity got flipped somewhere in the decode.

**7. If it scores, chase the repeat-scoring problem.** Since the badge only counts one score per sender ID, try cycling that field in the payload before each replay and see whether that's enough to keep the counter climbing on its own.

## Files in here

`Remote.ir` and `Remote2.ir`: the junk captures from Try 1, kept as proof of what doesn't work. `flipper-backup-2026-08-08.tar.gz`: a Flipper settings backup from before the firmware update. `media/`: badge photos plus the aliasing diagram. `FlipperHIDecoder/`: unrelated tool grabbed early on while poking at what the Flipper can do, not part of this project, kept for reference.

