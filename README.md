# 12V → 5V Buck Converter (Power Stage)

A hand-calculated, simulated, and laid-out synchronous-less buck converter power stage. Designed from first principles rather than from a reference design, with every component value derived by hand and then verified in LTspice before layout.

**Scope: this is the power stage only.** It is open loop and requires an external floating gate drive to operate. See [Limitations](#limitations) before building it.

<img width="400"  alt="Screenshot 2026-09-14 152633" src="https://github.com/user-attachments/assets/175a31df-5013-4422-bec4-b890eba5406b" />
<img width="400" alt="Screenshot 2026-09-14 152742" src="https://github.com/user-attachments/assets/c0df2f15-0e95-4ee3-99ed-9a4208bfb56d" />


## Specifications

| Parameter | Value |
|---|---|
| Input voltage | 12 V |
| Output voltage | 5 V (nominal) |
| Output current | 2 A |
| Switching frequency | 300 kHz |
| Duty cycle | 0.4167 |
| Inductor ripple current | 0.6 A (30% of I_out) |
| Output voltage ripple | ~50 mV target |

---

## How it works

A buck converter steps voltage down by switching the input on and off rapidly and letting an LC filter average the result.

**Switch ON (41.67% of each cycle):**
Q1 conducts, connecting the 12 V input to the inductor. The voltage across L1 is `V_in − V_out = 7 V`, so inductor current ramps up. Energy is stored in the inductor's magnetic field and delivered to the load.

**Switch OFF (58.33% of each cycle):**
Q1 opens. The inductor opposes the change in current and forces it to keep flowing, pulling it up through D1 from ground. The voltage across L1 is now `−V_out`, so current ramps down as the stored energy discharges into the load.

The output capacitor C1 smooths what's left into a roughly DC 5 V, and the input capacitor C2 supplies the sharp current pulses Q1 demands during its on-time.

The governing relationship is **volt-second balance**: in steady state the current rise during ON must exactly equal the fall during OFF, or the inductor current would drift indefinitely. Setting those equal gives the duty cycle:

```
D = V_out / V_in = 5 / 12 = 0.4167
```

---

## Design calculations

All values were derived by hand before any simulation.

**Duty cycle**

```
D = V_out / V_in = 5 / 12 = 0.4167
```

**Inductor** — chosen for 30% ripple current (`ΔI_L = 0.3 × 2 A = 0.6 A`), which is the standard tradeoff between inductor size and RMS losses:

```
L = (V_in − V_out) × D / (f_sw × ΔI_L)
L = (12 − 5)(0.4167) / (300,000 × 0.6)
L = 16.2 µH
```

**Output capacitor** — sized for ~1% output ripple (`ΔV_out = 50 mV`):

```
C = ΔI_L / (8 × f_sw × ΔV_out)
C = 0.6 / (8 × 300,000 × 0.05)
C = 5 µF
```


Note that `ΔI_L` and `ΔV_out` are **design choices**, not calculated quantities. Everything else follows from them algebraically.

---

## Simulation results

Verified in LTspice using an ideal switch model (`Ron = 0.02Ω`) driven by a PULSE source at 300 kHz.

| Measurement | Calculated | Simulated |
|---|---|---|
| Output voltage | 5.00 V | 4.75 V |
| Output ripple | 50 mV | 54 mV |
| Inductor ripple current | 0.6 A | 0.6 A |

**The 250 mV output shortfall is expected and understood.** The ideal equations assume a lossless switch and diode. Accounting for the Schottky forward drop during the off-time:

```
V_out ≈ D·V_in − (1−D)·V_f
V_out ≈ 0.4167(12) − 0.583(0.4) ≈ 4.77 V
```

This matches the simulated 4.75 V. In a closed-loop design the feedback network would compensate automatically by raising duty cycle to roughly 0.44.

---

## Bill of materials

| Ref | Part | Value | Package |
|---|---|---|---|
| Q1 | IRLZ44N | Logic-level N-MOSFET | TO-220-3 |
| D1 | MBRS340 | Schottky, 40 V / 3 A | DO-201AD |
| L1 | — | 16.2 µH | Radial, 12 mm |
| C1 | — | 5 µF | Radial electrolytic, 8 mm |
| C2 | — | 100 µF | Disc, 5 mm |
| J1 | — | 5 V output | Screw terminal, 5 mm |
| J2 | — | 12 V input | Screw terminal, 5 mm |
| J3 | — | Gate drive input | Pin header, 2.54 mm |

**Component selection notes:**

- **Logic-level MOSFET** — the IRLZ44N fully enhances at `V_GS ≈ 5 V`, unlike standard power MOSFETs that need 10 V+. Its `R_DS(on) ≈ 22 mΩ` gives roughly 37 mW of conduction loss at 2 A, negligible.
- **Schottky, not a standard rectifier** — the diode conducts 58.3% of every cycle, so the ~0.4 V forward drop (vs 0.7–1 V for a PN diode) matters directly to efficiency. Diode loss here is around 0.5 W.
- **Inductor saturation rating** — peak inductor current is `2 + 0.3 = 2.3 A`. The part should be rated well above that, not at it.

---

## PCB layout

Two-layer board, through-hole, with a ground pour on the back layer.

<img width="400" alt="Screenshot 2026-09-14 152523" src="https://github.com/user-attachments/assets/5c7cdc3b-993a-47b3-9373-449b34c56b36" />

**Layout priorities, in order:**

1. **Hot loop minimization.** The path `C2(+) → Q1 drain → Q1 source → D1 cathode → D1 anode → C2(−)` carries the sharp switching current pulses. Every millimetre of that loop is parasitic inductance that produces voltage ringing and EMI on switch turn-off. C2, Q1, and D1 are placed as a tight cluster.
2. **Compact switch node.** The node joining Q1 source, D1 cathode, and L1 is the primary noise radiator, swinging 0–12 V at 300 kHz. Kept short without being a large copper area.
3. **Ground pour on the back layer.** The ground return carries the same current pulses as the switch node. A continuous plane keeps all ground pads at the same potential instead of relying on a daisy-chained trace with real impedance.
4. **Power trace width.** VIN, VOUT, and GND are widened for 2 A on 1 oz copper.

DRC passes clean with courtyard checking enabled.

---

## Limitations

These are deliberate scope choices, not oversights.

**Open loop — no regulation.** There is no feedback network, error amplifier, or compensation. The duty cycle is fixed. Change the load and the output voltage drifts with nothing to correct it. Adding closed-loop control is the intended next revision.

**Requires external floating gate drive.** Q1 is a high-side switch: its source is the switch node, which swings between 0 V and 12 V every cycle. Since `V_GS = V_gate − V_source`, the gate signal must float with the source.

J3 provides both pins for this:

- **Pin 1** → gate
- **Pin 2** → source (switch node), the drive reference

**A ground-referenced function generator will not work here** and may damage something by shorting the switch node through its ground connection. Driving this board requires an isolated gate driver, a bootstrap circuit, or a gate drive transformer.

**Footprints are typical packages, not verified parts.** Footprints were selected for standard package types rather than checked against specific ordered components. Lead pitch should be verified against real datasheets before fabrication.

---

## Repository contents

```
BuckConverter.kicad_sch    Schematic
BuckConverter.kicad_pcb    PCB layout
BuckConverter.kicad_pro    KiCad project file
```

---

## Next steps

- [ ] Add feedback loop: voltage divider, error amplifier, PWM comparator, type II compensation
- [ ] Add bootstrap gate drive circuit so the board is self-contained
- [ ] Build a synchronous version, replacing D1 with a low-side FET and dead-time control
- [ ] Verify footprints against real part datasheets

---

*Designed by Ali Cheema*
