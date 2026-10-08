# VCAT Strain Gauge Board — Aarav Artham

A dual-channel strain gauge acquisition board for **Longhorn Racing Solar** (UT Austin's solar car team). It reads suspension strain from 350 Ω gauges and puts the data on the car's CAN telemetry bus. Designed in KiCad 10.

- Full KiCad source: https://github.com/lhr-solar/VCAT-StrainGaugeBoard
- Schematic PDF: [`StrainGaugeBoard_schematic.pdf`](StrainGaugeBoard_schematic.pdf)
- Resume: [`Aarav_Artham_Resume.pdf`](Aarav_Artham_Resume.pdf)

## The feature I'm proudest of: one-wire shunt calibration with a "~1000 µε" resistor

Gauges on a race car get mounted with epoxy resin and routed through a harness. So the most common question during bring-up and at the track is *"Is this channel actually reading correctly, or is something open or miswired?"* Answering that by loading the suspension with a known force is slow.

Instead, each bridge has a **174 kΩ shunt resistor** that a **TS5A3166 analog switch** can place across one arm. A single MCU GPIO line (`SHUNT_CAL`) drives the switches on both channels at once.

The value is chosen so that the shunt simulates a round number of strain:

```
ΔR/R = R_g / (R_g + R_shunt) = 350 / (350 + 174 000) ≈ 0.2007 %
ε_sim = (ΔR/R) / GF = 0.2007 % / 2.0 ≈ 1 004 µε   (≈ 1000 µε)
```

With one firmware command, both channels should jump by about 1000 µε (micro strain). That one step-response check confirms the whole signal chain: gauge wiring, bridge completion, input filter, ADC, gain setting, SPI, firmware scaling, and CAN. It also gives a per-channel gain correction in the field, with no calibrated load needed. Using an analog switch rather than a jumper or relay keeps the check fast enough to run at every power-up.

## Other design choices

- **Bridge completion on the PCB, with selectable quarter, half, or full bridge.** Each gauge arm has its own pair of connector pins, and the bridge is built on the board. You populate only the completion resistors for the arms that have no gauge:
  - Quarter bridge: R25, R26, R27
  - Half bridge: R25, R27
  - Full bridge: none

  The same board then works for single-gauge bending sensing and for temperature-compensated half or full bridges.
- **One 24-bit ADC for two bridges.** A TI **ADS1220** (with PGA) samples both bridges differentially (AIN0/1 and AIN2/3). SPI is shared, with per-ADC CS and DRDY lines, so the channel sheet can be repeated without rerouting the bus.
- **Analog hygiene.** Each input has an RC anti-alias filter, and there's ESD protection (ESDA5V3SC5) at every gauge connector. Cable shields go to digital ground. The analog ground (**GNDA**: ADC AVSS, bridge, filter caps) is separate from digital ground and joins it at a single net tie near the ADC.
- **Power entry built for a car harness.** The power path has reverse-polarity and ideal-diode protection (LM74502 with back-to-back MOSFETs) plus TVS clamping. A 12 V → 5 V buck (LMR51635) feeds a 3.3 V LDO. A **TPS2117** power mux switches over to USB-C automatically, so the board can be brought up on a bench with just a laptop.
- **MCU and bus.** An STM32G473 handles processing, and a TCAN3413 with a jumper-selectable 120 Ω split termination connects to CAN. Test points are labeled on every rail and bus for bring-up.
