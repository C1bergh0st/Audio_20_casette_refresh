# Mercedes W211 Audio 20 Cassette AUX / MOST Retrofit Investigation

## 1. Vehicle and original audio system

- Vehicle: Mercedes-Benz W211 E-Class.
- Model year: 2002.
- Original head unit: Audio 20 cassette / Audio 20 CC.
- The original system is an early W211 MOST-fiber audio architecture.
- The dashboard Audio 20 is not simply a conventional self-contained radio with direct speaker outputs.
- The car uses a rear Audio Gateway (AGW) / amplifier section connected over the MOST optical network.
- Because of this architecture, replacing the head unit with a generic aftermarket radio does not automatically provide sound through the factory speakers.
- Harman Kardon / upgraded sound-system status is unlikely, because the speaker grills are not marked as such.

## 2. Aftermarket head-unit attempt and no-sound problem

An aftermarket head unit was installed together with a blue MOST interface box.
The label on the blue box reads approximately:

- "Head Unit Replacement"
- Benz ML, GL, R, CLS CLASS
- BMW E8X, E9X, F7X, F0X
- Audi 2G, 3G Host
- Porsche 2004-2015 Bose
- Designed in Germany
- Assembled in China
- HW: V3.0
- SW: V3.1
- LZ26030001 / UC8949

Important observations:

- The label does not explicitly list W211 / E-Class compatibility.
- With the aftermarket head unit installed, there is no sound.
- A likely cause is that the factory W211 Audio Gateway / amplifier is not receiving the correct MOST audio stream and/or wake/control commands.
- Another possibility is simply that this blue interface is the wrong MOST decoder for the early 2002 W211 Audio 20 CC system.
- The original radio can be used as a diagnostic:
  - If the original Audio 20 is reinstalled and sound works, the factory AGW, MOST ring, speakers, and most factory wiring are probably healthy.
  - If the original radio also has no sound, the factory MOST ring / AGW / power supply would need investigation.

## 3. Decision to keep the original Audio 20

Instead of continuing to rely on the aftermarket MOST adapter, the preferred plan is now to keep the original Audio 20 and add a direct AUX input.

Reasoning:

- This preserves the original MOST communication with the rear Audio Gateway.
- It avoids having to reverse-engineer or replace the complete Mercedes MOST head-unit interface.
- The original Audio 20 can continue to perform all expected network/control functions.
- Only the analog cassette audio source needs to be replaced or intercepted.

## 4. Why a normal cassette adapter is not desired

- A magnetic cassette-to-AUX adapter has been tried / considered problematic.
- The Audio 20 cassette mechanism itself is having issues with the cassette adapter.
- The preference is therefore a true internal electrical AUX injection, even if this requires:
  - cutting traces,
  - lifting components,
  - soldering wires,
  - adding coupling capacitors / resistors,
  - or building a small replacement daughterboard.

## 5. Main Audio 20 PCB findings

The Audio 20 was opened and the main PCB was photographed.

### 5.1 Main analog-to-digital converter

A Burr-Brown / TI PCM1801U stereo ADC was identified on the main board.
This means the cassette audio eventually reaches this ADC before becoming part of the digital/MOST audio path.

### 5.2 Main-board injection concept

A possible direct injection method at the PCM1801 would be:

- isolate the existing cassette signals feeding VINL/VINR,
- AC-couple external AUX L/R into VINL/VINR,
- connect AUX ground to local analog ground.
- The preferred plan became to modify the cassette module instead, because it is more replaceable and keeps the main Audio 20 PCB untouched.

## 6. Cassette module PCB

The complete cassette mechanism / PCB has now been removed and photographed.

### 6.1 Ribbon cable

- The cassette module connects to the main radio through a wide flat-flex / ribbon cable.
- The ribbon terminates on the cassette PCB at a row of through-hole connections.
- The conductors then fan out into the cassette PCB circuitry.
- The physical ribbon format may be a standard FFC/FPC size/pitch, but the electrical pinout is proprietary and must not be assumed.
- Measured (see `pictures/ribbon_cable_*.jpg`): **21 conductors, 1.0 mm pitch, ≈ 22 mm wide**, generic "AWM 2896 80°C VW-1" FFC.
- Main-board end plugs into SMD connector CB201; cassette end is soldered into two staggered rows (odd pins 1–21 top, even pins 2–20 bottom).
- Pin mapping plan and interception board: see [w211_audio20_modification_strategy.md](w211_audio20_modification_strategy.md).

The user mentoined that the PCB of the casette contained the following components on the non-photographed back side:
- resistors,
- adjustable / tunable resistors,
- one switch,
- one capacitor.

This suggests that the cassette PCB is acting as a complete electromechanical/audio interface and that the ribbon likely carries both analog audio and mechanism/control/status signals.

## 8. Sony cassette audio IC

The small Sony chip on the cassette PCB was read as approximately:

- Sony A25600
- Dolby logo
- 135 B11G

The device was identified as most likely a Sony CXA2560Q cassette playback processor.

Functions associated with this IC include:

- tape-head preamplification,
- playback equalization,
- Dolby B processing,
- muting,
- forward/reverse audio handling,
- other cassette playback-related analog processing.

Relevant pin information already discussed:

- pin 7 = LINEOUT1
- pin 24 = LINEOUT2
- pin 26 = GND

The two line outputs are the most important signals for the AUX project.

Important electrical point:

- The CXA2560 line outputs are internally DC-biased, reportedly around 4 V.
- Therefore, a phone / Bluetooth module should not simply be hard-connected directly to the CXA2560 output pins.
- The existing circuit likely uses coupling components between the CXA2560 outputs and the next stage / ribbon.
- The preferred injection point is after the cassette audio source has been isolated, while preserving the correct DC conditions seen by the rest of the radio.

## 9. Cassette motor/control IC

A larger IC on the right side of the cassette PCB is marked BA6285A.

This appears to belong to a reversible motor-driver family.

Implication:

- The cassette PCB is not just an analog audio board.
- It also controls at least part of the tape transport / motor system.
- Therefore, completely deleting the cassette electronics may require emulating cassette-state and transport-related signals.

## 10. Best current modification strategy

The detailed two-stage plan has been moved to [w211_audio20_modification_strategy.md](w211_audio20_modification_strategy.md).

In short:

- **Stage 1:** keep the original cassette PCB and ribbon connected, and replace only the cassette L/R audio path with AUX / Bluetooth audio.
- **Stage 2 (optional):** once the ribbon pinout is mapped, replace the whole cassette mechanism with a daughterboard that emulates the required cassette status/control signals.

## 11. What must be mapped next

The most useful immediate task is to identify which ribbon conductors correspond to:

- left audio,
- right audio,
- analog/audio ground,
- cassette-present status,
- play / transport state,
- motor/control signals,
- power rails.

### Recommended unpowered continuity measurements

With the cassette PCB completely unpowered:

1. Set a multimeter to continuity / low-ohms mode.
2. Probe CXA2560 pin 7 (LINEOUT1).
3. Check each ribbon conductor / through-hole termination for continuity.
4. Probe CXA2560 pin 24 (LINEOUT2).
5. Check each ribbon conductor / through-hole termination.
6. Probe CXA2560 pin 26 (GND).
7. Check which ribbon conductor(s) connect to the same ground.
8. If pin 7 or pin 24 does not connect directly to a ribbon conductor:
   - trace to the first series resistor/capacitor,
   - then test from the far side of that component toward the ribbon.

The goal is to create a table such as:

```text
Ribbon pin  1 = ...
Ribbon pin  2 = ...
Ribbon pin  3 = ...
...
Ribbon pin ?? = Left audio
Ribbon pin ?? = Right audio
Ribbon pin ?? = Audio ground
Ribbon pin ?? = Cassette present
Ribbon pin ?? = Motor / control
```

## 12. Important uncertainties

The following are not yet confirmed:

- Whether the blue MOST adapter is fundamentally incompatible or simply wired incorrectly.
- Exact AGW part number / version in the trunk.
- FFC contact orientation (Type A / B) and free ribbon length.
- Exact left/right/ground ribbon pins.
- Exact cassette-present and play-state logic.
- Whether the Audio 20 requires the cassette transport to appear active continuously in order to stay in TAPE mode.
- Whether the CXA2560 audio outputs are sent directly to the main board or pass through additional muting / filtering / switching on the cassette PCB.
- Whether a complete cassette-module emulator can be passive, or whether some active logic will be required.

## 13. Important cautions

- Do not assume the ribbon pinout from connector shape or pitch.
- Do not connect external audio directly to a DC-biased analog node without coupling/isolation.
- Do not cut main-board traces until the cassette-board option has been fully explored.
- Do not probe or work around the "Warning of high voltage" section while the radio is powered.
- Do not short adjacent FFC conductors while measuring.
- When probing fine-pitch IC pins, use a sharp insulated probe and avoid slipping between pins.
- The radio should be fully unpowered for continuity testing.

## 14. Current physical state

At the present point in the project:

- The original Audio 20 has been removed from the car.
- The radio has been disassembled.
- The cassette module has been extracted.
- The cassette PCB is accessible.
- The cassette module remains connected to the rest of the system only through its ribbon cable when assembled.
- Good photographs now exist of:
  - the original radio rear,
  - the main Audio 20 PCB,
  - the PCM1801 area,
  - the board-to-board front-panel connector,
  - the complete cassette PCB,
  - the Sony CXA2560 area,
  - the ribbon termination area.
- No traces have yet been intentionally cut as part of the AUX modification.
- No final AUX injection point has yet been soldered.
- The next logical step is electrical mapping of the cassette module ribbon, beginning with the two CXA2560 line outputs and ground.