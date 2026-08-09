# Cracking the DC NextGen "Tanuki" Badge IR "Touch" Protocol

*Badge #266, 21 pts and counting. Two of these badges can "touch" and give each other points using invisible light. This is the story of trying to figure out exactly how that trick works — mistakes included, because that's how real hacking actually goes.*

## The badge, up close

| | |
|---|---|
| ![Badge back, PCB visible through the case](media/badge-back-pcb.jpg) | ![Badge front, e-paper-style display showing #266, 21 pts, star row, and a hand-drawn egg icon](media/badge-front-display.jpg) |
|:--:|:--:|
| **Back** — the circuit board, the coin battery, and the badge's name | **Front** — the screen: badge `#266`, `21 pts`, a row of stars, and a little egg drawing |

| |
|:--:|
| ![Bare board with the display flipped open on its ribbon cable, showing the battery, chips, and a header](media/badge-board-mcu.jpg) |
| **Inside** — popped the case open. You can see the battery, the little computer chips, and the screen flipped out on its ribbon cable. |

---

## The goal

The [DC NextGen](https://dcnextgen.org) "Tanuki" badge earns points by "touching" other badges — bump two badges together and both of their `pts` counters go up. **The real goal here is to rack up points without needing a room full of people to bump badges with** — by figuring out exactly what message a "touch" sends over infrared, then having the Flipper Zero send that same message on demand. If it works, one hacker with a Flipper could rack up points as fast as they want, alone.

That's a bigger ask than just "peek at the signal" — it means the fake touch has to be good enough to actually fool the badge, not just look close. So this write-up isn't just curiosity, it's building toward: capture a real touch → understand it completely → replay it → watch the points go up without another badge in the room.

## The short version (read this first)

The badge sends its "touch" over **infrared light** — the same invisible light your TV remote uses. I wanted to see exactly what that light signal looks like and what it says, so I can copy it.

I tried to use a **Flipper Zero** (a little hacker gadget that can read and send infrared signals) to catch and copy the badge's signal. **It didn't work** — and figuring out *why* it didn't work turned out to be most of the interesting part. Short answer: the Flipper's infrared "ear" is built to only understand one specific "pitch" of blinking light, and this badge blinks at a different pitch. Every recording came back as garbage, and every guess I fired back at the badge got ignored — the points counter stayed at 21 no matter what.

So: I ordered the right tools for the job (a sensor that can hear *any* pitch, plus a proper recorder fast enough to write it all down). This file is the lab notebook — what I tried, why it failed, what I learned anyway, and what's next.

---

## Quick vocabulary (skip if you already know this stuff)

- **Infrared (IR) light** — light that's invisible to human eyes but small electronic sensors can see it. TV remotes, badge "touches," and old-school data cables all use it.
- **Carrier frequency** — infrared isn't sent as one steady beam, it's blinked on and off super fast, like Morse code but way faster — thousands of times per second. How fast it blinks is called the *frequency*, measured in kHz (thousands of blinks per second). Two devices only understand each other if they agree on this blink speed.
- **Demodulator** — a chip that's built to listen for ONE specific blink speed and translate it into 1s and 0s. If you point a different blink speed at it, it doesn't understand — it just spits out garbage.
- **Packet** — a structured little message, like a text: an intro (so the receiver knows a message is starting), the actual content, and often a "did this arrive correctly?" check at the end.
- **Checksum** — a bit of built-in math a message carries so the receiver can double check it wasn't scrambled in transit. Kind of like how a receipt total should match up with all the item prices added together — if it doesn't, something's wrong, and you throw the message out. This is probably why just firing random signals at the badge never worked: it checks its math before it believes you.
- **Logic analyzer** — a tool that watches a wire and writes down, many times per second, whether it's HIGH or LOW (on or off) at that instant. Basically a super-fast flip-book camera for electricity.
- **MCU (microcontroller)** — the tiny computer chip that's the "brain" of a gadget like this badge.

---

## Tools I'm using

- A **Flipper Zero**, updated to the newest firmware (version 1.4.3) partway through this project.
- A laptop, talking to the Flipper over a USB cable using little Python scripts, so I could run tests even when I wasn't standing right next to it.

---

## Attempt 1 — just try "Learn" mode, like teaching a universal remote

Flipper has a "Learn" button for infrared: point it at a remote, press a button on the remote, and Flipper saves whatever it saw. Important catch: **it saves whatever it saw even if that was nonsense** — it doesn't check whether the signal actually made sense.

I aimed the badge at the Flipper, did a "touch," and hit Learn. Twice. Here's exactly what got saved:

```
$ storage read /ext/infrared/Remote.ir
name: Magnus
type: raw
frequency: 38000
duty_cycle: 0.330000
data: 123 125186 135 4012 88
```

```
$ storage read /ext/infrared/Remote2.ir
name: RAW_11
type: raw
frequency: 38000
duty_cycle: 0.330000
data: 493 3943 187 1925 240 216 125 428 126 227 125
```

(Both of these are saved in this repo as `Remote.ir` and `Remote2.ir` — proof, not just my word for it.)

**Why I know this is garbage and not a real signal:** a real infrared message is dozens of blinks packed tightly together, all following one consistent rhythm — like a drum beat. Look at the numbers in `Remote.ir`: there's a **125,186 microsecond** gap (that's over a tenth of a second — enormous compared to everything else, which is a few *hundred* microseconds) sitting between two tiny blips. That's not a rhythm, that's three random noises with silence in between. Same story in `Remote2.ir` — the gaps between blinks jump all over the place with no shared pattern.

One more trap I nearly fell for: both files say `frequency: 38000`. That looks like a measurement, but it's not — **it's just the default number the Flipper writes into every file**, because its receiver chip throws away the actual blink speed before it ever tells the software anything. It physically can't report a number it never learned. Don't trust that field.

**What this means:** the Flipper's ear wasn't hearing the badge's real "voice" — it was just twitching randomly because it heard a pitch it doesn't understand. Like trying to tune an old radio to a frequency it doesn't cover: you get static, not a garbled version of the station.

## Attempt 2 — listen live, for longer, holding the button

Switched to watching the Flipper's raw feed live, for 25-40 second stretches, doing several "touches" in a row — including holding the badge's little contact button down the entire time (in case the badge only "talks" while that button is pressed — turns out, it does).

Result: **almost total silence.** One run caught a 3-blip fragment. A whole 25-second window, badge held right up against the Flipper's sensor the entire time, caught **zero** blips at all.

**What this means:** this ruled out "I'm just aiming it wrong." If the Flipper's ear could hear this badge at all, holding them together for 25 straight seconds would catch *something* consistent, not silence one time and 3 random blips the next. The receiver just plain can't hear this badge's pitch.

## Attempt 3 — okay, can I fool it by just yelling random stuff at it?

New question: maybe the badge doesn't check very carefully — maybe *any* infrared blast near it bumps the counter. Worth testing before doing anything harder. Fired a bunch of standard remote-control-style signals at the badge's sensor, at several different pitches (30, 33, 36, 38, 40, and 56 kHz), with different patterns, 48 combinations total, badge's button held the whole time.

`pts` stayed at 21. Every single time.

**What this means:** the badge is not a pushover. It's checking that an incoming message actually looks right — matches its expected format, probably including a checksum — before it'll count it as a real touch. Good design on their part, annoying for me: I can't brute-force this, I actually have to see a real message first.

---

## Why the Flipper hit a wall (and what tool actually works)

The Flipper's infrared parts are really two separate things:

- **The mouth (sending)** — a plain infrared LED, controllable at basically any pitch. This part is flexible and worked fine for my tests.
- **The ear (receiving)** — a sealed chip **hard-wired to only understand one pitch (~38 kHz)**. There's no setting to change this — the "translate blinks into data" part happens inside the chip itself, before the Flipper's software ever gets a say. If the badge blinks at a different speed, the Flipper's ear literally cannot hear it correctly, no matter how I aim it or how long I wait.

I also checked whether the Flipper's "Logic Analyzer" mode (a tool for watching raw wires) could work around this. It samples about 100,000 times per second. That sounds like a lot, but to cleanly capture something blinking 30,000–60,000 times per second, you need to sample several times *faster* than the blink — otherwise you get a blurry, misleading picture (this is called *aliasing*, like a spinning wheel looking like it's going backward in a movie because the camera isn't fast enough). So even hacking around the sealed ear, the Flipper's other tools aren't fast enough either.

![Diagram showing a fast-blinking signal sampled fast enough to see it correctly, versus the same signal sampled too slowly and appearing as a completely different, slower fake wave](media/aliasing-explained.png)

Look at the bottom half of that picture: the orange dots are all real measurements of the real (light gray) wave, no lying involved — but because they're too spread out in time, connecting them draws a completely different, much slower red wave that never actually happened. That fake red wave is what "aliasing" means: not missing data, but *confidently wrong* data. This is exactly the trap a too-slow recorder falls into with the badge's signal, and exactly why the new 24-million-samples-per-second recorder matters — the top half of the picture is what "recording fast enough" looks like instead: dense dots that trace the real wave with no fakery.

## The fix: get the right sensor and the right recorder

**Ordered from Adafruit ($25.70 total):**
- A [TSMP96000](https://www.adafruit.com/product/5970) — a sensor built specifically to hear *any* infrared pitch from 20-60 kHz and pass along the raw blinking, instead of assuming one fixed speed like the Flipper does.
- A little adapter cable, jumper wires, and a breadboard, to wire it all together.

**Ordered from Amazon (~$18.50):**
- A proper USB logic analyzer that samples 24 million times per second — hundreds of times faster than we need, so no more blurry aliasing problem.

Once both arrive, the sensor's job is to turn the invisible blinking into an electrical signal, and the logic analyzer's job is to write down that signal in perfect detail so I can look at it on my laptop.

---

## The plan once the parts show up

1. **Wire it up.** Sensor → breadboard → logic analyzer, all sharing a common ground wire.
2. **Find the real pitch.** Hold the badge's contact button, aim it at the sensor, and just *look* at the recording — finally see the actual blink speed instead of guessing.
3. **Catch a real message.** Do several touches. If I can borrow a second badge, catch an actual two-badge handshake — that's the real deal, not just one badge talking to itself.
4. **Decode it.** Figure out the rhythm, find the badge's ID (`#266`) inside the message, and find whatever part changes each time (probably the checksum).
5. **Prove it by replaying it.** Have the Flipper send the exact decoded message back at the right pitch, and watch `pts` finally move past 21. That's the finish line — one badge, one Flipper, no second person needed.
6. **If step 5 works, automate it.** Loop the replay and watch the points climb as fast as the badge will accept them.
7. **Write it all up properly**, with the real signal diagrams.

## Things that could still trip this up

- **Only having one badge** makes it hard to see a real two-way handshake, not just one badge talking to nobody.
- **The checksum might be tricky** — if it includes something that changes every time (to stop people from just replaying an old recording), just copying a message won't be enough; I'd need to understand the actual math behind it.
- **It's possible there's no "pitch" at all** — some infrared systems send data as a plain on/off flicker with no fast blinking underneath. The new sensor would still catch it, but replaying it through the Flipper (which assumes a pitch) might need a different trick.

## Bonus path: read the badge's own brain

While digging around, I also popped the case open to look at the actual circuit board:

![Bare board with the display flipped open on its ribbon cable](media/badge-board-mcu.jpg)

What I could make out:
- A **coin battery** (CR2032), the same kind used in some watches.
- A big black part labeled `KLJ-1230` — probably a coil that helps power the screen. Worth looking up.
- Two computer chip packages — one of them is almost certainly the badge's "brain" (the MCU), but the writing on them is too small and blurry in this photo to read for sure. **Next time: closer, sharper photos of each chip.**
- A row of unused connector holes labeled roughly `W R T U V G` — this smells like a hidden **programming port**, the kind engineers use to load new software onto the chip. Worth testing with a multimeter later.
- The screen is almost certainly **e-paper** (like a Kindle) rather than a normal screen — it only needs power to *change* what it shows, and holds the picture with no power at all otherwise. Good to know, because it also means the screen updates slowly — if I test something and the screen doesn't change right away, that might just be the screen being slow, not proof that nothing happened.

If I can identify that brain chip from a clearer photo, there's a chance someone's already published its manual (a "datasheet") or even dumped its software — which could just hand me the answer instead of me having to reverse-engineer it from scratch.

---

## What's in this folder

| File | What it is |
|---|---|
| `Remote.ir`, `Remote2.ir` | The two "recordings" from Attempt 1 — proven garbage, kept as proof of what *doesn't* work. |
| `flipper-backup-2026-08-08.tar.gz` | A backup of the Flipper's settings, taken before updating its software. |
| `media/badge-back-pcb.jpg`, `media/badge-front-display.jpg` | Photos of the outside of the badge. |
| `media/badge-board-mcu.jpg` | Photo of the badge with its case popped open. |
| `FlipperHIDecoder/` | An unrelated tool I grabbed early on while exploring what the Flipper can do — converts ID-card data into a format the Flipper understands. Not part of this badge project, kept for reference. |

---

Done in collaboration with GitHub user [lebowitz](https://github.com/lebowitz).
