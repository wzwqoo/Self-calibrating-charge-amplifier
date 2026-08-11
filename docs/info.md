<!--
This file is used to generate your project datasheet.
-->

## How it works

This is a **charge amplifier that measures its own scale.** Most charge
amplifiers have a gain you must trust; this one carries the instrument that
calibrates it on the same die, and closes the loop on the demo board's MCU.
It is fully analog — there is no clocked logic anywhere on the tile.

**The core.** A single-ended OTA with its output integrating onto a 500 fF
MiM feedback cap, with a transmission-gate reset across it. The summing node
never leaves the die: it is fenced by grounded shields on both sides of the
routing bus, because its leakage floor *is* the measurement. The output
appears on `ua[0]` and the amplifier's reference on `ua[1]`, which must be
driven at 0.90 V.

**Knob 1 — a known charge (`qdac`).** A 6-bit split-capacitor DAC (a 3-bit
MSB array of 27 fF units, a 3-bit LSB array behind a 27 fF bridge cap) has
its top plates on the virtual ground. Each bottom plate switches between a
stepped node `st` and VSS, so a code drops `Q = C_sel · ΔV` onto the summing
node — electrically indistinguishable from a real sensor event.

The trick is that **ΔV is selectable**, which makes one array do two jobs:

| strobe | `st` goes to | ΔV | what it is |
|---|---|---|---|
| `TRIGBC` low | VDPWR | 1.8 V | **calibration** — large, known Q that pins the absolute charge scale |
| `TRIGBT` low | `vtrim` | ~180 mV | **offset trim** — same 64 codes, ~10× finer |
| `TRIGN` high | VSS | — | idle |

Nominal, from the drawn cap values: full-scale ≈ 212 fF, so the cal step
spans ≈ 380 fC ≈ 0.76 V out, LSB ≈ 12 mV; the trim step spans ≈ 76 mV out,
LSB ≈ 1.2 mV. Computed DNL over all 64 codes is ±0.123 LSB, INL ±0.07,
monotonic.

**Knob 2 — a known leakage (`leakdac`).** A subthreshold NMOS current source
whose gate is tapped off a resistor divider, giving three decades — 1 nA,
100 pA, 10 pA — selected by `S0`/`S1`/`S2`. The taps were placed from an
*extracted* n·V_T of 38.1 mV, not the model's (whose prefactor was 2.8× off).
Driven into the 500 fF cap, those are ramps of 2 V/ms, 200 mV/ms and
20 mV/ms.

**Why both.** `qdac` fixes the absolute charge scale of the amplifier;
`leakdac` then rides on that calibrated scale to give an absolute current.
Neither alone is enough — the injector without the current source calibrates
gain but says nothing about the leakage floor, and the current source without
the injector has nothing to check itself against.

**What is not on this die.** The comparator and the single-slope counter were
moved off chip. `ua[0]` carries the *whole ramp* to the demo board's 12-bit
ADC rather than one threshold crossing, which costs nothing in area and means
**the measurement protocol is changeable after tapeout** — on a one-shot
shuttle that is worth a great deal. There is likewise no chopper: chopping is
already proven by the earlier 4×2 tile and would have cost ~90 µm of row that
a 2×2 does not have.

## How to test

Everything needs only the chip, a 1.8 V supply, and an MCU with an ADC and
13 GPIOs (an RP2040 was the target).

**1. Power-up and DC check.** `VDPWR` = 1.8 V, `ua[1]` (VCM) = 0.90 V,
`ui[6]` (RST) high, and the step driver idle — `uio[1]` (TRIGN) **high**,
`ui[7]` (TRIGBC) and `uio[0]` (TRIGBT) **both high** (they are active low).
Leave `S0`/`S1`/`S2` all low. `ua[0]` should sit at 0.90 V.

**2. Charge-injection transfer curve — the main self-test.** With RST
released, set a code on `ui[5:0]`, then drive TRIGN low and TRIGBC low to
fire the calibration step. `ua[0]` should step down by `Q/Cf`. Return TRIGN
high and pulse RST between points. Sweep all 64 codes: they should fall on a
straight line whose slope gives the absolute fC-per-code, and hence `Cf`.
**The complements are generated on chip** — unlike the earlier 4×2 tile, you
do not need to drive them.

**3. Fine trim.** Repeat step 2 with TRIGBT instead of TRIGBC. Same codes,
~10× smaller steps — this is the offset knob.

**4. Leakage ramp.** Assert exactly one of `S0`/`S1`/`S2`, release RST, and
sample `ua[0]` as it ramps. `I = Cf · dV/dt`, with `Cf` from step 2, so the
current comes out absolute. **Assert only one at a time** — the three taps
are independent pass transistors on one divider and asserting two shorts two
nodes of it together.

**5. The loop.** Steps 2 and 4 together are the point of the chip: run 2 to
calibrate the scale, run 4 to measure a current against that scale, and
repeat over temperature or supply to watch the calibration track.

## External hardware

- 1.8 V supply on `VDPWR`.
- A clean 0.90 V reference driven into `ua[1]` (a buffered divider or a DAC
  output — it is the amplifier's reference and must be low impedance).
- An MCU with a 12-bit ADC on `ua[0]` and 13 GPIOs for `ui[7:0]` and
  `uio[4:0]`. The RP2040 on the TT demo board is sufficient and is where the
  digital half of the calibration loop lives.
