# Single Phase Semi-Controlled Bridge Rectifier (AC to Variable DC) 

## Description

This is a phase controlled rectifier I built to convert a fixed 12V AC supply into a smooth, adjustable DC voltage. When we turn the potentiometer and the DC output changes anywhere from 0V up to close to the max Value of the DC we can achieve from the AC supply . Instead of burning off the extra voltage as heat (like a simple rheostat/resistor ), it controls power by deciding when in the AC cycle the current is allowed to start flowing. A microcontroller (ATMega328p) handles all the timing.

It started out as a simple half-wave version and I kept upgrading it irst to full-wave for more power range, then made the whole thing more reliable and easier to wire.

## Work Done

- Built the first version: a **half-wave controlled rectifier** using a single thyristor (SCR) and a PC817 optocoupler for zero-crossing detection. This only used the positive half of the AC wave, so it could only give 0–50% power control.
- Upgraded it to a **full-wave controlled rectifier** by switching to a half-controlled bridge added a second SCR and two diodes so both halves of the AC wave get controlled. This doubled the output and have a full 0–100% range.
- Reworked the circuit again to make it more **robust and simpler**:
  - Swapped the old zero-crossing detector for one built around an H11AA1 optocoupler + LM393 comparator, which gives a clean, noise-free square wave instead of a shaky signal.
  - Switched to sensitive-gate SCRs (MCR100-6 / 2N5064) that the Arduino can trigger directly through a single resistor and no separate driver transistor stage was needed.
- Verified everything on an oscilloscope (input sine wave vs the chopped output) and cross-checked the DC output against the theoretical value.

## Goals

- Get a DC output that can be smoothly adjusted with a knob (potentiometer), instead of a fixed voltage.
- Control the power efficiently .
- Cover the entire 0% to 100% power range .
- Make the zero-crossing detection accurate and immune to noise, since the whole timing depends on it.
- Keep the circuit reasonably simple to build and troubleshoot on a breadboard.

## Control Technique

The technique used here is **phase angle (firing angle) control**, done through zero-crossing detection + a timed delay:

1. Every time the AC wave crosses zero, a pulse is generated (through the optocoupler + comparator circuit) and fed into an Arduino interrupt pin.
2. The Arduino doesn't fire the SCR right away . It waits for a calculated delay first. This delay is what actually decides the output voltage:
   - Short delay → SCR fires early in the cycle → more of the wave gets through → higher DC output.
   - Long delay → SCR fires late → less of the wave gets through → lower DC output.
3. That delay is calculated from the potentiometer's position, mapped from a 0°–180° firing angle range into a microsecond delay.
4. After the delay, the Arduino sends a short ~50 microsecond pulse to the SCR's gate to trigger it.
5. Once triggered, the SCR stays "latched" on by itself (that's just how SCRs behave) until the AC crosses zero again, at which point it turns off naturally. Then the whole cycle repeats for the next half-wave.

For the full-wave version, both SCRs get the same trigger pulse at the same time, but only one of them is actually forward-biased at any given moment (depending on which half of the AC cycle it is), so only that one turns on. The two diodes handle the other half so current always has a path back, which is what makes it "half-controlled."

The DC output voltage follows this relationship (α = firing angle in radians):

```
Half-wave: V_avg = (V_peak / (2π)) × (1 + cos(α))
Full-wave: V_avg = (V_peak / π) × (1 + cos(α))
```

At α = 0° we get maximum output, at α = 180° we get 0V, and everywhere in between it scales smoothly, which is exactly what was observed on the multimeter and scope while testing.

## How It Was Achieved

**Hardware:**
A 230V-to-12V transformer steps the mains AC down to a safer 12V AC. This feeds both the power stage (the SCR + diode bridge) and the zero-crossing detector circuit. The ZCD circuit (optocoupler + comparator) watches the AC and sends a clean digital pulse to the Arduino every time the wave crosses zero. A potentiometer connected to an analog pin lets you set the firing angle on the fly. The SCR gates are wired straight to Arduino digital pins through small resistors (in the improved version), and the rectified/chopped output goes through a smoothing capacitor before reaching the load.

**Software:**
The zero-crossing pin is set up with `attachInterrupt()`. For the half-wave version it triggers on `RISING` only (once per full AC cycle), and for the full-wave version it's set to `CHANGE` so it catches both the rising and falling edges — meaning it fires twice per cycle, once for each half of the wave. The interrupt itself does almost nothing except set a flag (`zcd_Flag = true`) — this is intentional, since interrupt routines are supposed to be as short as possible. All the real work happens in the main `loop()`: it checks the flag, resets it, reads the potentiometer, converts that reading into a delay using `map()`, waits that long using `delayMicroseconds()`, and then fires a short pulse on the gate pin to trigger the SCR(s).

