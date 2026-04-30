# Bluetooth Speaker — BM83 + TPA3116D2 Class D

A stereo Bluetooth speaker designed in Altium Designer around the **Microchip BM83** Bluetooth audio module and the **TI TPA3116D2DAD** Class D amplifier. Single-board design with a hierarchical schematic, mixed-signal grounding, and RF-aware antenna placement.

<img width="1042" height="676" alt="PCB 3D Top View" src="https://github.com/user-attachments/assets/5d02ce84-9b29-44f8-a4f3-59439045990f" />

---

## Overview

This board is a fully integrated stereo Bluetooth speaker driver: it receives a Bluetooth audio stream, decodes it on the BM83 module, and drives a pair of speakers through a Class D amplifier — all on a single ~100 × 100 mm PCB.

The design is intentionally compact and inexpensive to fabricate (single-board, 2-layer, hand-solderable footprints throughout) while still respecting the discipline a mixed-signal RF audio board demands: clean separation of analog and high-current ground domains, careful output filtering on the Class D amplifier, and a defined antenna keep-out zone for the BM83's integrated chip antenna.

## Key specs

| Parameter | Value |
|---|---|
| Bluetooth module | Microchip BM83 (Bluetooth 5.0, A2DP / AVRCP) |
| Amplifier | TI TPA3116D2DAD — Class D, BTL stereo |
| Output power | Up to ~50 W/channel (supply-dependent) |
| Speaker configuration | Stereo, Bridge-Tied Load |
| Logic supply | Takes 12V PVCC input, 3.3 V regulator via LM1117-3.3 LDO + MCP111T-270E/TT reset supervisor for BM83|
| Board | 2-layer FR-4, ~100 × 100 mm |
| EDA | Altium Designer |

## Block diagram


  Power input ──┬──────────────────────► TPA3116D2 (Class D amp) ──► BTL LC filter ──► Speakers
                │
                └─► LM1117-3.3 LDO ──┬─► BM83 (Bluetooth audio) ──► (audio out to amp)
                                     │
                                     └─► MCP111T reset supervisor

## Design decisions

### Why the BM83
The BM83 is a fully-certified Bluetooth audio module — the alternative was building a Bluetooth stack on a bare BT chip, which is dramatically more work for a personal project and requires going through Bluetooth SIG certification. The BM83 ships with a complete stack, a stereo audio path, and an integrated chip antenna in a small footprint. The trade-off is less flexibility and a higher cost per unit, but it is the right choice for a single-board portfolio piece.

### Why the TPA3116D2DAD
Industry-standard Class D amplifier with strong documentation, stereo BTL configuration in a single chip, and well-understood thermal behavior. The DAD-suffix package (HTSSOP with PowerPad) is easier to hand-assemble than the alternative DAP package while still providing adequate thermal performance for the target output level.

### Power architecture
A single supply input feeds the TPA3116 directly and is regulated down to 3.3 V by an LM1117-3.3 LDO for the BM83. An MCP111T-270E/TT reset supervisor holds the BM83 in reset until the 3.3 V rail crosses its threshold, ensuring clean power-on-reset behavior. An LDO was chosen over a switcher for the BM83 supply specifically to avoid injecting switching noise into the Bluetooth radio's PLL.

### BTL output filter
The Class D switching outputs need an LC filter before reaching the speakers. Without one, you radiate the switching frequency directly through the speaker leads — which act as antennas — and you cause both EMI compliance problems and audible distortion. The filter is sized for the audio bandwidth and the target speaker impedance.

### Mixed-signal grounding
The board uses an **SGND / GND star ground** scheme: the analog return from the BM83 audio output and the high-current return from the speaker terminals are kept on separate copper pours and joined at a single point near the connector. Without this separation, speaker-current-induced ground bounce modulates the analog reference and shows up as audible noise in the amplified output.

### Antenna placement
The BM83's chip antenna is placed at the **board edge** with **copper pour exclusion** in the keep-out zone defined by the BM83 datasheet. Putting copper pour under or near the antenna detunes it and crushes the radiation pattern. This is one of those layout rules that's easy to miss but completely changes whether your Bluetooth actually works at usable range.

## Repository structure

```
.
├── README.md                       This file
├── LICENSE
├── altium/                         Raw Altium project source
├── outputs/
│   ├── gerbers/                    Manufacturing Gerbers + drill files (ZIP)
│   ├── bom/                        Bill of materials (CSV)
│   ├── pick-and-place/             Pick-and-place files
│   └── pdf/
│       ├── schematic.pdf           Browser-viewable schematic
│       └── fabrication-drawing.pdf Board outline and layer stack
├── docs/
│   ├── component-reference.md      Detailed component selection notes
│   └── design-notes.md             Extended design rationale
└── images/                         Renders and diagrams
```

## Viewing the design

Don't have Altium? You can still inspect everything:

- **[Schematic (PDF)](outputs/pdf/schematic.pdf)** — full hierarchical schematic
- **[Fabrication drawing (PDF)](outputs/pdf/fabrication-drawing.pdf)** — board outline, layer stack, mechanical dimensions
- **[Bill of materials](outputs/bom/)**
- **[3D renders](images/)**

To open the source files, clone the repo and open `altium/PCB1.PrjPcb` in Altium Designer (a free Altium Education license is available for students).

## Manufacturing

The Gerber set in `outputs/gerbers/` is fab-ready and tested-friendly with JLCPCB, PCBWay, and most North American fabs. Default specifications:

- 2-layer FR-4
- 1.6 mm thickness
- 1 oz copper
- HASL or ENIG finish

Most components are stocked at Digi-Key and Mouser. The BM83 has occasional supply constraints — check stock before placing an order.

## Status

- ✅ Design complete, schematic ERC clean
- ✅ Manufacturing outputs generated
- 🔲 Boards ordered / assembled
- 🔲 Bring-up and validation

## Lessons learned

A few things that came out of this design that I'd carry forward to the next revision:

- **Reset supervisor is non-negotiable.** An earlier iteration tied the BM83 reset pin to the 3.3 V rail through a simple pull-up. The MCP111T was added after observing the BM83 occasionally powering up into an undefined state on slow rail rise. Explicit POR handling is cheap and worth it.
- **Signal grounding paid off.** Iterations with a single solid GND plane showed audible noise on the speakers that correlated with bass-heavy content — high-current speaker returns were modulating the analog reference. Splitting SGND from GND and tying them at a single point cleared it.
- **Antenna keep-out matters.** The first layout had ground pour right up to the BM83 antenna. Pulling the pour back to the manufacturer-defined keep-out improved Bluetooth range noticeably.
- **Checking data sheets prior to designing.** There were several times where I would design haphazardly, or lazily adding a footprint, and then end up checking the data sheet and realizing that the pin count is wrong, or the module is a different component than what I needed, etc.
- **Power route widths are non-negotiable and must be accounted for in the design.** When finishing the basic routing I soon realized afterwards that the routes all needed to be readjusted anyway to account for the increased width of the power routes, such as the 12V power supply and 3.3V regulator power.
- **Always expect to use more than one layer** While this seem obvious now, I assumed when making the schematic that I would be able to mathematically fit all routes together without crossing. Needless to say, I soon realized after starting the PCB design that this was impossible, at least for a project of this complexity.

## License

Released under the MIT License — see [LICENSE](LICENSE). You're free to use, modify, and distribute the design.

The Altium source files in this repository were created using an Altium Education student license. The design itself is yours under MIT, but you'll need your own Altium license (commercial or student) to open and modify the source files.

**No warranty.** This is a hobby/educational design. It has not been certified to any safety, EMC, or Bluetooth SIG standard. Building, powering, or distributing it is at your own risk — switching amplifiers can dissipate significant heat and Bluetooth radios are subject to regional regulatory requirements.

## Author

**Connor Carlisle** — Electrical Engineering, University of Mississippi
[GitHub](https://github.com/[connor-carlisle]) · [LinkedIn](https://linkedin.com/in/[connor-carlisle])
