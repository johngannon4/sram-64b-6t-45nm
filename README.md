# 64-Bit 6T SRAM Array with Custom Peripheral Circuitry (45 nm)

**ESE 5700: Digital Integrated Circuits and VLSI** · University of Pennsylvania · Fall 2025
**Authors:** John Gannon, Fabian Sander
**Tools:** Cadence Virtuoso, 45 nm PDK

A complete, fully synchronous 64-bit (16 × 4) SRAM built from standard 6T cells and hand-designed peripheral circuits: non-overlapping clock generator, dynamic row decoder, precharge, write drivers, and read circuitry. Two versions were designed and compared:

1. **Baseline:** full-VDD bitline precharge, tri-state read buffers, and conservatively sized cells.
2. **Alternate:** VDD/2 bitline precharge, OTA-based differential sense amplifiers, and minimum-sized cells.

Both designs pass full functional verification across all 16 rows. The alternate design scores better on the figure of merit (FOM = Area × Power × Delay), but most of that difference comes from the smaller cell sizing rather than the new peripheral circuits. See [Limitations](#limitations).

| Design    | Array area (μm²) | Power (mW) | Min. clock period (ps) | FOM (μm²·mW·ps) |
| --------- | ---------------- | ---------- | ---------------------- | --------------- |
| Baseline  | 7.60             | 0.713      | 250                    | 1355            |
| Alternate | 2.07             | 0.805      | 255                    | 425             |
| Change    | −73%             | +13%       | +2%                    | −69%            |

*Area covers the cell array only; peripheral circuits are excluded.*

The full project report is in [`Gannon_Sander_Final_Project.pdf`](Gannon_Sander_Final_Project.pdf).

---

## Architecture

![Full 64-bit SRAM array with peripheral circuitry](images/sram-array-overview.png)
*Fully synchronous 64-bit 6T SRAM array with peripheral circuitry.*

- **Memory array:** 16 rows × 4 columns of 6T cells
- **Shared peripherals:** non-overlapping clock generator, 4-to-16 dynamic row decoder, clocked transmission-gate control logic
- **Per-column circuits:** bitline precharge, bitline equalization, write driver, read buffer / sense amplifier

### 6T cell

![6T SRAM cell schematic](images/6t-cell.png)
*Standard 6T cell: two cross-coupled inverters for storage, two access transistors gated by the word line.*

The core design tension is the classic read-stability vs. writeability tradeoff. Strong pull-downs protect the cell from read disturb but make it harder to overwrite; weak pull-ups make writes easier but cost noise margin. With 16 cells sharing each bitline pair, bitline capacitance also makes the precharge and sensing scheme a first-order performance factor.

---

## Baseline design

### Non-overlapping clock generator
Cross-coupled NOR gates with buffered outputs turn one input clock into two complementary, non-overlapping phases (CLK / CLKB). Every block in the array is sequenced off these phases, which prevents races and short-circuit current between precharge and access.

| Schematic | Waveform |
| --- | --- |
| ![Clock generator schematic](images/clock-gen-schematic.png) | ![Clock generator waveform](images/clock-gen-waveform.png) |

### 4-to-16 dynamic decoder
Precharges internal nodes while CLKB is high and evaluates the address while CLK is high, so exactly one word line fires per cycle.

| Schematic | Waveform |
| --- | --- |
| ![Decoder schematic](images/decoder-schematic.png) | ![Decoder waveform: address bits cycling and word line outputs](images/decoder-waveform.png) |

### Full-VDD bitline precharge
PMOS devices pull BL and BLB to VDD while CLK is low, giving symmetric starting conditions for every access at the cost of full-swing bitline power.

| Schematic | Waveform |
| --- | --- |
| ![Full-VDD precharge schematic](images/precharge-vdd-schematic.png) | ![Full-VDD precharge waveform](images/precharge-vdd-waveform.png) |

### Write driver
A differential driver gated by Write Enable: data drives BL through an NMOS pass gate while its complement drives BLB. Sized to overcome the bitline load and flip the cell.

![Write driver schematic](images/write-driver-schematic.png)

### Tri-state read buffer
One per column. Follows the bitline when enabled, high-impedance otherwise.

| Schematic | Waveform |
| --- | --- |
| ![Tri-state buffer schematic](images/tristate-buffer-schematic.png) | ![Tri-state buffer waveform](images/tristate-buffer-waveform.png) |

### Clocked transmission gate with pull-down (TG-PD)
Passes control signals (including word-line gating) during CLK high and actively pulls the output to ground during CLKB high, so no node floats or charge-shares into the array during precharge.

| Schematic | Waveform |
| --- | --- |
| ![TG-PD schematic](images/tg-pulldown-schematic.png) | ![TG-PD waveform](images/tg-pulldown-waveform.png) |

### Cell sizing

| Transistor    | Width            | Rationale                                              |
| ------------- | ---------------- | ------------------------------------------------------ |
| Pull-up PMOS  | 1× min (120 nm)  | Weak feedback during write improves writeability       |
| Access NMOS   | 2× min (240 nm)  | Enough drive for read/write without excessive loading  |
| Pull-down NMOS| 8× min (960 nm)  | Strong pull-down prevents read disturb, improves SNM   |

### Functional verification
Every row was exercised with the sequence
`W0 → R0 → R0 → W1 → R1 → R1 → W1 → R1 → R1 → W0 → R0 → R0`,
confirming that only one word line is active during CLK high, all word lines are grounded during CLKB high, bitlines fully precharge every cycle, and reads/writes occur only when their enables are asserted.

![Baseline array functional test waveform](images/baseline-array-waveform.png)
*Full-array test: accesses on WL0 for the first half of the run and WL15 for the second.*

---

## Alternate design

Three changes from the baseline:

1. **VDD/2 bitline precharge:** smaller bitline swing, intended to reduce read-disturb stress on the cell.
2. **OTA-based differential sense amplifier:** amplifies a small BL/BLB difference to a full logic level.
3. **Minimum-sized 6T cells:** all six transistors at minimum width.

![Alternate SRAM array](images/optimized-array-overview.png)
*Alternate array with VDD/2 precharge, sense amplifiers, and minimum-sized cells.*

### VDD/2 precharge
A diode-connected PMOS/NMOS stack generates a mid-rail reference (~0.685 V), buffered by an inverter and applied to the bitlines through NMOS pass devices on CLKB.

| Schematic | Waveform |
| --- | --- |
| ![VDD/2 precharge schematic](images/precharge-half-vdd-schematic.png) | ![VDD/2 reference waveform](images/precharge-half-vdd-waveform.png) |

![VDD/2 precharge connected to the bitlines](images/optimized-precharge-connection.png)
*Precharge module on each bitline pair, controlled by CLKB.*

The design intent: with the bitlines at mid-rail rather than VDD, the storage nodes see a smaller voltage difference when the word line opens, which should reduce read-disturb risk. The write driver still applies full differential swing for writes.

### OTA sense amplifier
An NMOS differential pair on BL/BLB with PMOS active loads gives a single-ended output; the tail current source is gated by Read Enable, so the amplifier draws no static current when not reading. Differential sensing also rejects common-mode noise on the bitlines.

![OTA sense amplifier schematic](images/sense-amp-schematic.png)

![Alternate read/write circuitry](images/optimized-read-write-circuitry.png)
*Per-column read/write circuitry: the sense amplifier replaces the tri-state buffers.*

### Minimum-sized cells
All six cell transistors reduced to minimum width (120 nm), taking array area from 7.60 μm² to 2.07 μm² (−73%). This sizing change accounts for the area reduction.

### Functional verification
Same test sequence as the baseline. Bitlines precharge to ~0.685 V, all reads and writes complete correctly with minimum-sized cells, and the sense amplifiers produce clean full-swing outputs.

![Alternate array functional test waveform](images/optimized-array-waveform.png)

---

## Performance comparison

### Methodology
- **Write access time (WA):** 50% of word-line swing → 50% of storage-node Q swing, measured separately for high-to-low (WAHL) and low-to-high (WALH).
- **Read access time (RA):** 50% of word-line swing → 50% of data-bus swing, with the bus forced to the opposite value beforehand to capture worst case (RA0, RA1).
- **Minimum clock period:** not simply 1 / (worst access time), because of setup/hold. Starting from the worst-case delay, clock period was swept parametrically; the minimum period is the shortest one where Q tracks the input on both edges *and* the data bus is valid by the end of the cycle when Read Enable is asserted.
- **Power:** average supply power while alternately writing all-1s and all-0s across the array (four full cycles each) at the maximum clock frequency, from integrated supply current.
- **Area:** 16 × 4 × [2·W<sub>PU</sub> + 2·W<sub>ACC</sub> + 2·W<sub>PD</sub>] × 45 nm (cell array only; peripheral area excluded).

### Results

| Design    | WAHL (ps) | WALH (ps) | RA0 (ps) | RA1 (ps) | Worst access (ps) |
| --------- | --------- | --------- | -------- | -------- | ----------------- |
| Baseline  | 78.4      | 148.0     | 89.5     | 54.7     | 148.0             |
| Alternate | 81.7      | 134.4     | 57.2     | 121.5    | 134.4             |

| Design    | Min. period (ps) | Max. frequency (GHz) | Power (mW) |
| --------- | ---------------- | -------------------- | ---------- |
| Baseline  | 250              | 4.00                 | 0.713      |
| Alternate | 255              | 3.92                 | 0.805      |

| Design    | Cell widths (nm)             | Array area (μm²) |
| --------- | ---------------------------- | ---------------- |
| Baseline  | PU 120 · ACC 240 · PD 960    | 7.60             |
| Alternate | PU 120 · ACC 120 · PD 120    | 2.07 (−73%)      |

The alternate design's worst-case access time is ~9% faster, maximum clock frequency is essentially unchanged (~4 GHz), and average power is 13% higher. The source of the power increase was not isolated.

<details>
<summary><b>Measurement waveforms</b></summary>

**Baseline write access (low→high, high→low)**

![Baseline write LH](images/baseline-write-lh.png)
![Baseline write HL](images/baseline-write-hl.png)

**Data-bus forcing circuits used for worst-case read measurement (baseline)**

| Force '1' | Force '0' |
| --- | --- |
| ![Force 1](images/baseline-force-1.png) | ![Force 0](images/baseline-force-0.png) |

**Alternate write access (low→high, high→low)**

![Optimized write LH](images/optimized-write-lh.png)
![Optimized write HL](images/optimized-write-hl.png)

**Data-bus forcing circuits (alternate)**

| Force '1' | Force '0' |
| --- | --- |
| ![Force 1](images/optimized-force-1.png) | ![Force 0](images/optimized-force-0.png) |

**Alternate read access ('0', '1')**

![Optimized read 0](images/optimized-read-0.png)
![Optimized read 1](images/optimized-read-1.png)

**Baseline minimum-period search:** 250 ps is the first swept period where Q follows the input data; the data bus is confirmed valid at the end of the cycle.

![Baseline period sweep](images/baseline-period-sweep.png)
![Baseline period verification](images/baseline-period-verify.png)

**Supply current during successive writes (baseline, alternate)**

![Baseline write current](images/baseline-write-current.png)
![Optimized write current](images/optimized-write-current.png)

</details>

---

## Limitations

- **The comparison changes several variables at once.** The area reduction comes from moving the baseline's conservative sizing (8× pull-down, 2× access) to minimum-sized cells. Minimum-sized cells were never tested with the baseline's full-VDD precharge and tri-state buffers, so the contribution of the VDD/2 precharge and sense amplifier to the FOM is not isolated. The baseline cells may have been larger than necessary.
- **Area covers the cell array only.** The sense amplifiers and VDD/2 reference generator add peripheral area that the FOM does not count.
- **Cell stability was checked functionally only.** Minimum-sized cells passed functional simulation, but no static noise margin (SNM) or read/write margin analysis was performed, and results are from a single operating condition.
- **The power increase was not broken down** by block.

## What this project demonstrates

- Transistor-level design of a complete synchronous SRAM: cell, dynamic decoder, non-overlapping clocking, precharge, write drivers, and sense amplifier
- Systematic verification of every block and of the full array
- Careful timing measurement: write/read access times, worst-case read setup, and minimum clock period found by parametric sweep rather than inferred from access delay

---

## Repository contents

| File | Description |
| --- | --- |
| `README.md` | This summary |
| `Gannon_Sander_Final_Project.pdf` | Full project report: all schematics, waveforms, and analysis |
| `images/` | Figures used in this README |

Schematics and testbenches were built in Cadence Virtuoso with a proprietary PDK and are not included.

## Contact

**John Gannon** · M.S.E. Electrical Engineering, University of Pennsylvania
[LinkedIn](https://www.linkedin.com/in/john-gannon4) · johntgannon4@gmail.com
