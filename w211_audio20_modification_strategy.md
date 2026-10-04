# Mercedes W211 Audio 20 AUX Retrofit: Modification Strategy

Split out from [w211_audio20_aux_investigation.md](w211_audio20_aux_investigation.md) (section 10).

The current preferred approach is:

## Stage 1: keep the cassette PCB connected

Keep:

- the original cassette PCB,
- the original ribbon cable,
- all cassette-control / presence / mechanism-state electronics.

Then replace only the cassette audio path.

Conceptually:

```text
Audio 20 main unit
        |
        | original ribbon
        v
Cassette PCB
   |          |
   |          +--> control / motor / cassette-state signals remain original
   |
   +--> original L/R tape audio is isolated
             |
             +--> AUX / Bluetooth L/R is injected instead
```

Advantages:

- preserves all proprietary cassette-control signaling,
- avoids disturbing the main Audio 20 PCB,
- allows the radio to continue believing the cassette module is present,
- makes experimentation much safer and more reversible.

With an interception board (see section 4), Stage 1 can be done **without touching either PCB**.
The L/R audio is switched on the interceptor between the cassette module and the main board.

## Stage 2: optional complete cassette-module replacement

After the ribbon pinout is understood, it may be possible to design a small daughterboard that replaces the entire cassette mechanism.

Possible daughterboard functions:

- FFC connector matching the original ribbon,
- AUX left/right input,
- optional Bluetooth audio receiver,
- appropriate audio coupling / attenuation,
- simulated cassette-present / play-state signals,
- any required resistor or switch emulation.

This is considered possible, but not yet ready to implement because the ribbon pinout has not been mapped.

---

## 1. Ribbon cable: physical facts

From `pictures/ribbon_cable_1.jpg`, `pictures/ribbon_cable_2.jpg`, `IMG_6188.jpg`, and `IMG_6189.jpg`:

| Property | Value | Source / confidence |
|---|---|---|
| Conductor count | **21** | counted; matches the silkscreen "21" on both ends |
| Pitch | **1.0 mm** | 21 conductors span approximately 20 mm (measured 19.94 mm across the contacts). 0.8 mm pitch would give about 16 mm and 1.25 mm about 25 mm. |
| Overall width | **≈ 22.0 mm** (measured 22.02 mm) | matches the standard FFC width formula (n + 1) × pitch = 22 mm |
| Cable marking | "AWM 2896 80°C VW-1" (repeated print) | generic UL-style FFC, not a Mercedes-specific part |
| Main-board end | plugs into **CB201**, an SMD FFC connector on the main board | silkscreen "1" on the left and "21" on the right. "42" and "22" are also printed beneath it; this looks like a dual-footprint and should be checked. |
| Cassette end | **soldered** into two staggered rows of through-holes | top row = odd pins 1, 3 … 21 (11 holes, "21" printed at the right end near J129), bottom row = even pins 2 … 20 (10 holes, "2" printed at the left end) |
| Contact orientation (Type A / Type B) | **unknown, check it** | see 4.3. This matters for any jumper cable. |
| Free length / slack | **unknown, measure it** | decides whether an interceptor can sit inside the closed radio |

Conclusion: the ribbon is a **standard 21-pin, 1.0 mm pitch FFC**. This means off-the-shelf connectors, jumper cables, and breakout boards fit it.
The cassette end is soldered, so any interception happens at the **CB201 (main board) end**.

### 1.1 Pin numbering convention used in this document

- Pin numbers follow the PCB silkscreen: **pin 1 = CB201 pin 1 = the cassette-PCB hole next to the J130/J131 marking** (top-left of the odd row).
- Before recording any measurement, confirm that CB201 pin 1 and the cassette-side pin 1 hole are the same conductor (one continuity beep).

### 1.2 First-order clue from the main board

On `IMG_6189.jpg` the traces leaving CB201 split into two groups:

- **left / downward** toward the analog area (an SOIC IC, TP201–TP204 and TP220, and many electrolytic capacitors E201–E208). This group is probably **audio, audio ground, and analog supply**.
- **right** toward **IC501** (the large QFP, which is probably the system / mechanism microcontroller). This group is probably **logic control and sense lines**.

Photograph the traces between CB201 and those two areas closely. This photo alone may sort most of the 21 pins into "audio" and "logic" before any measurement.

---

## 2. What the 21 conductors are expected to carry (hypothesis)

The cassette PCB has three active blocks. Every ribbon pin should belong to one of them:

1. **Sony CXA2560Q** (IC101): playback EQ, Dolby B, head FWD/REV switching, mute, and music sensor. These are mode pins driven by logic levels from the main board.
2. **Rohm BA6285A** (IC501 on the cassette PCB): reversible motor driver with inputs FIN, RIN, and VREF, and outputs OUT1 and OUT2. This is probably the reel or mode/loading motor.
3. **Mechanism sensors and switches**: these go through the bottom 12-pad row near TP515/TP506 and through Q101, Q103, Q502, and Q503. They include cassette-in, mode/cam position, the reel-rotation pulse, and possibly the capstan motor on/off.

Expected inventory, which should add up to about 21:

| Group | Likely signals | Expected count | How it will look |
|---|---|---|---|
| Power | audio Vcc (CXA2560, probably +8 to +9 V), motor supply (VM), possibly separate logic Vcc | 2–3 | low resistance to large electrolytic capacitors; steady DC when powered |
| Ground | audio GND, power/motor GND (possibly several pins) | 2–4 | 0 Ω to CXA2560 pin 26 and/or the BA6285A GND |
| Audio | **Tape L out, Tape R out** | 2 | continuity (through a cap or resistor) to CXA2560 pin 7 / pin 24; AC audio during playback |
| CXA2560 control | FWD/REV head select, MUTE, Dolby NR on/off, EQ 120/70 µs (metal), music-sensor mode | 3–5 | high impedance logic inputs; change state when you press the matching button |
| CXA2560 status | music-sensor output (gap detect for track skip) | 1 | toggles during FF/REW across track gaps |
| Motor control | BA6285A FIN, RIN (and possibly VREF), capstan motor on/off | 2–4 | 0 V idle; pulses during insert, eject, FF, REW, and reverse |
| Sensors | cassette-in switch, mode/position switch(es), reel-rotation pulse | 2–4 | static levels that change on insert/eject; a square wave while tape is moving |

Getting these datasheets before measuring makes the work much faster:

- CXA2560Q: for its control-pin numbers (trace each one out to the ribbon).
- BA6285A: for its FIN, RIN, VREF, VCC, VM, and GND pins.

Links are in [datasheets/list.md](datasheets/list.md).

---

## 3. Mapping procedure

### Phase A: unpowered, cassette PCB only (no interceptor needed)

Work on the soldered-end holes. They are 2.54 mm-ish staggered pads and easy to probe.

1. **Ground set:** with one probe on CXA2560 pin 26, beep every ribbon pin. Repeat from the BA6285A GND pin and from the metal mechanism frame.
2. **Audio:** beep from CXA2560 pin 7 and pin 24. If nothing beeps, follow the trace to the first series R or C and beep from its far side, as described in investigation doc section 11.
3. **IC pins:** for each CXA2560 control pin and each BA6285A pin (FIN, RIN, VREF, VCC, VM), find the ribbon pin it reaches. A resistor in between is normal, so use the Ω range rather than just the beeper.
4. **Transistors:** for each of Q101, Q103, Q502, and Q503, find which ribbon pin (if any) reaches its base. These are probably open-collector sense outputs or motor/solenoid switches.
5. **Mechanism row:** for each pad in the bottom 12-pad row, find which ribbon pin it reaches. A direct connection is probably a sensor switch passed straight through.
6. **Signature of every pin:** with the meter in diode mode, measure each pin with **black on GND** and then **red on GND**, and record both readings. Supply pins show capacitor charging. Logic inputs show ESD-diode drops (about 0.5–0.7 V). Pins with no connection read OL.

### Phase B: powered, with the interceptor in place

1. Fit the interceptor between the ribbon and CB201 (section 4). All 21 lines must pass straight through.
2. Record the **DC voltage on every pin** in each state:
   radio off (ignition / standby) → radio on, no tape → tape inserted → play side A → play side B (reverse) → FF → REW → Dolby on/off → metal tape (if you have one) → eject.
3. Put a cheap 8-channel logic analyzer (a "24 MHz 8ch" Saleae-clone with sigrok/PulseView, about €10) on the pins that change state. Capture insert, play, FF/REW, auto-reverse, and eject. This shows the motor sequencing and the reel-pulse frequency.
4. Use a scope, or the PC sound card through a 10:1 divider and a cap, to confirm the L/R audio pins. Also measure the **DC bias** on the audio pins. This decides how AUX must be injected (section 5).

**Bench-power caution:** the W211 Audio 20 is a MOST node. On a bench it may not wake up, or may not enter TAPE mode, without the MOST ring closed (fiber loop-back) and a CAN or terminal-15 wake. If bench start-up fails, do Phase B in the car with the radio open on an extended harness. The interceptor's external breakout header is meant for this case.

### Result table (fill in)

| Pin | Side (odd/even row) | Main-board direction (analog / IC501) | Diode red→GND | Diode black→GND | Connects to on cassette PCB | V: radio on, no tape | V: tape playing | Activity seen | **Function** | Confidence |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | odd | | | | | | | | | |
| 2 | even | | | | | | | | | |
| 3 | odd | | | | | | | | | |
| 4 | even | | | | | | | | | |
| 5 | odd | | | | | | | | | |
| 6 | even | | | | | | | | | |
| 7 | odd | | | | | | | | | |
| 8 | even | | | | | | | | | |
| 9 | odd | | | | | | | | | |
| 10 | even | | | | | | | | | |
| 11 | odd | | | | | | | | | |
| 12 | even | | | | | | | | | |
| 13 | odd | | | | | | | | | |
| 14 | even | | | | | | | | | |
| 15 | odd | | | | | | | | | |
| 16 | even | | | | | | | | | |
| 17 | odd | | | | | | | | | |
| 18 | even | | | | | | | | | |
| 19 | odd | | | | | | | | | |
| 20 | even | | | | | | | | | |
| 21 | odd | | | | | | | | | |

---

## 4. Interception board

### 4.1 What it must do

```text
                    original ribbon (soldered to cassette PCB)
                                |
                       [ J1: 21p 1.0 mm FFC socket ]
                                |
   21 lines straight through, each one with:  test pad + cut-able link
                                |                       |
                                |               2.54 mm breakout header
                                |              (logic analyzer / meter)
                                |
              L/R lines only: tape <-> AUX selector + coupling network
                                |
                       [ J2: 21p 1.0 mm FFC socket ]
                                |
                    new short 21p 1.0 mm FFC jumper
                                |
                          CB201 on main board
```

- **Phase B mapping:** every line is pass-through with a probe point.
- **Stage 1:** the L/R lines are switched to AUX. Everything else stays original.
- **Stage 2 prototyping:** any single line can be opened (cut-able link or 0 Ω resistor). It can then be driven or pulled from the header to emulate a sensor before a dedicated replacement board is designed.

### 4.2 Option A: buy ready-made parts (fastest, about €10–15, good enough for Phase B)

There is no off-the-shelf "21-pin pass-through interceptor" that is worth hunting for. Generic parts do the job:

- **2× "FFC/FPC adapter board 1.0 mm pitch"**, the kind that converts to 2.54 mm DIP. Get a 21-pin (or larger universal) version that comes with or accepts a **21p 1.0 mm connector**. AliExpress, LCSC, and similar shops sell them, often universal boards with 0.5 mm and 1.0 mm footprints on opposite sides.
- **21p 1.0 mm FFC jumper cables, 10–20 cm long, Type A *and* Type B.** They cost a few euro, so buy both (see 4.3).
- Connect the two breakout boards with 21 Dupont wires or a 2.54 mm ribbon. That is your interceptor. Each wire can be pulled to break or swap a line.

Search terms: `FFC FPC adapter 1.0mm 21P`, `FFC to DIP 2.54 1.0mm pitch`, `FFC cable 21 pin 1.0mm pitch type A` / `type B`.

Downside: it is bulky and loose, and it won't fit inside the radio. It is fine for the bench and for mapping.

### 4.3 FFC orientation trap

- A **Type A** (same-side contacts) jumper keeps pin 1 → pin 1 when both connectors face the same way. A **Type B** (opposite-side contacts) jumper reverses the contact side.
- Whether a pin gets mirrored (1 ↔ 21) depends on the jumper type *and* on whether each socket is top-contact or bottom-contact.
- **Before connecting to the radio, beep pin 1 to pin 1 and pin 21 to pin 21 through the whole interceptor chain.** A mirrored ribbon can put the supply onto a logic or audio pin.
- Also check which side of the original ribbon's end has exposed contacts, and whether CB201 contacts the top or the bottom. Write it down in section 1.

### 4.4 Option B: custom PCB (recommended once Phase A is done)

A small 2-layer board. Design in KiCad and order from JLCPCB or PCBWay. 5 boards cost a few euro plus shipping. Both offer PCBA if you don't want to hand-solder the 1.0 mm connectors, though 1.0 mm pitch is easy to hand-solder.

Suggested content:

- **J1, J2:** 21p 1.0 mm FFC SMD connectors (flip-lock or slide-lock), picked from LCSC so PCBA is possible.
  Choose top-contact or bottom-contact deliberately so that a **Type A** jumper gives pin 1 → pin 1 (see 4.3). Optionally add a footprint for the other contact style as a fallback.
- **Per line:** an SMD 0 Ω / solder-jumper link between J1 and J2, plus a test pad. Label each pad with its pin number and (after mapping) its function in silkscreen.
- **2× 1×11 or 1×21 2.54 mm header** carrying all 21 lines, plus a GND header pin for probe clips.
- **Audio section on the two L/R lines** (exact values depend on the Phase B measurements, see section 5):
  - a DPDT slide switch or small signal relay (e.g. a 5 V DPDT relay driven from a transistor) to select TAPE or AUX,
  - AUX input: a 3.5 mm TRS jack footprint and a 3-pin header for a Bluetooth module,
  - series coupling caps (footprint for film or electrolytic caps, about 1–10 µF),
  - an attenuator divider footprint (phone line level ≫ tape line level),
  - a bias resistor footprint to reproduce the DC bias the main board expects on those pins,
  - an optional footprint for a ground-loop isolation transformer, which helps a lot with a Bluetooth receiver powered from the car.
- **Optional power tap:** jumper-selectable from the ribbon's supply pin (once known) to a small LDO for a Bluetooth module. Check the available current first.
- **Board size:** keep it narrow (about 30 × 40 mm) so it may fit inside the radio housing later. Measure the free space first.

Order a **custom-length 21p 1.0 mm FFC** if the stock lengths don't fit inside the radio. FFC sellers make these cheaply.

### 4.5 Recommendation

1. Do **Phase A** with just a multimeter. You can start now, and it may already identify L/R, GND, supplies, and the motor pins.
2. Order the **Option A** parts in parallel. They are cheap and arrive while Phase A is in progress, and they make Phase B possible.
3. Design the **Option B** board once the audio pins and their DC bias are known. Then the audio section can be laid out with real values, and the same board covers Stage 1 as well as Stage 2 experiments.

---

## 5. Stage 1 audio injection via the interceptor

Once the L/R pins are known, Phase B decides the injection circuit:

| Measured on the L/R ribbon pins | Meaning | Injection circuit |
|---|---|---|
| ~0 V DC, AC audio | cassette PCB already AC-couples, and the main board biases its own input | AUX → series cap → (attenuator) → J2 L/R. Break the J1 side with the switch. |
| ~4 V DC (CXA2560 bias) | DC-coupled to the main board, whose input may *rely* on that DC | Same as above, plus a **bias resistor** (about 47–100 kΩ) from a matching DC source so the main board still sees about 4 V. Verify that the main-board side doesn't shift once the cassette side is disconnected. |

Level: car tape line out is probably about 0.3–0.6 V rms, while a phone at full volume gives up to about 1 V rms. Start with a 1:2 divider and adjust by ear and on the scope to avoid clipping the PCM1801.

Keeping the radio in TAPE mode during AUX playback (open questions):

- Does the radio leave TAPE mode or auto-reverse/eject when it sees **no reel-rotation pulses**? This is likely with no tape or a stalled tape. If so, Stage 1 needs either a cassette that turns freely, or emulation of the reel pulse and cassette-in on the interceptor. That would be a small piece of Stage 2 pulled forward.
- Should the **MUTE line** be overridden? The CXA2560 mute is irrelevant once its output is disconnected, but the main board may also mute during FF/REW/MS.

---

## 6. Next actions checklist

- [ ] Confirm pin 1 ↔ pin 1 between CB201 and the cassette-side hole by J130/J131.
- [ ] Note which side has the exposed contacts and whether CB201 contacts the top or bottom. Measure the free ribbon length.
- [ ] Download the CXA2560Q and BA6285A datasheets (links in `datasheets/list.md`).
- [ ] Phase A: run the ground, audio, IC-pin, transistor, and mechanism-row continuity checks and take diode signatures. Fill in the table.
- [ ] Take a close-up photo of the CB201 traces toward the analog area and toward IC501.
- [ ] Order Option A parts: 2× 1.0 mm FFC→DIP breakout boards with 21p sockets, and 21p 1.0 mm FFC jumpers in Type A and Type B.
- [ ] Phase B: record DC voltages per state and take logic-analyzer captures. Confirm the DC bias on the audio pins.
- [ ] Design the Option B interceptor in KiCad and order it from JLCPCB or PCBWay.
