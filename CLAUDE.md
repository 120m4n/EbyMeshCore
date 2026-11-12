# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

EbyMeshCore: a MeshCore LoRa node PCB — Pro Micro nRF52840 (nice!nano clone) + Ebyte E22-900M30S or E22-900M33S (SX1262) LoRa module, fed from an external 5 V (max 5.5 V) supply through an AMS1117-3.3 LDO. It is a variant of https://github.com/Sobollion/Ebytastic (itself derived from faketec v4); upstream holds the original photos, Altium sources, Meshtastic firmware and Russian README. README.md holds the description and BOM.

Hardware constraint worth remembering: the board must not be powered from USB and the external 5 V input at the same time — doing so damages the LDO.

Power topology (from PCB nets): `+5v` is fed only by J2 (`5V_IN`) and supplies the E22 VCC (U2 pins 9/10) and the AMS1117 (U3) input. The Pro Micro's 3V3 pin (U1 pin 22) is tied directly to the AMS1117 output (`+3.3v`); no Pro Micro pin reaches `+5v`, and VBUS is not on the header. Consequences:
- USB only: the nRF52840 and the `+3.3v` loads (I2C J3, SW_OUT J4, pull-ups) run from the Pro Micro's regulator, but the E22 is unpowered → radio init fails. Good for flashing/config only. The AMS1117 is back-fed from its output (input at 0 V) — the same path that damages it when USB and 5 V are both connected.
- USB power can't drive LoRa without a hardware change (e.g. a Schottky from VBUS to `+5v`), and USB's 500 mA is below the E22's 600–670 mA TX current at 30 dBm (more at 33 dBm).

## Layout

- `EbyMeshCore.kicad_pcb` / `.kicad_pro` — the KiCad 10 project. The board was imported from Altium; the PCB is the source of truth.
- `EbyMeshCore.kicad_sch` — schematic reconstructed in KiCad 10 after the import (also exported as `Docs/EbyMeshCore_schematic.pdf`). Only 11 of 22 PCB footprints carry a schematic `path` link, so check the sync before running "Update PCB from Schematic". When changing a footprint, update both the PCB and the symbol's `Footprint` field.
- `EbyMeshCore.kicad_sym`, `EbyMeshCore.pretty/` — project-local symbol and footprint libraries (registered in `sym-lib-table` / `fp-lib-table` via `${KIPRJMOD}`). Identical parts must share one footprint name, otherwise the BOM splits them and JLCPCB warns "Multiple lines in the BOM have been matched to the same part".
- `production/` — Fabrication Toolkit output for JLCPCB (Gerber zip, `bom.csv`, `positions.csv`, `designators.csv`, `netlist.ipc`); regenerate after PCB changes.
- `.history/` — KiCad automatic local-history snapshots, not source.
- `Docs/` — E22 module datasheets and the Pro Micro nRF52840 schematic (the reference for pin mapping).

## MeshCore firmware (EbyteMesh)

MeshCore variant `variants/ebytemesh/` lives in https://github.com/120m4n/MeshCore-HJ7RMN, branch `feature/ebytemesh` (firmware only, no PCB files). It reuses `variants/promicro` (faketec pinout = this PCB) and adds `EbyteMeshBoard` (battery reports 0 mV: P0.31 floats) and `EbyteMeshRadioWrapper` (clamps every `setTxPower()` to `MAX_LORA_TX_POWER`). Module is chosen in `[ebytemesh_module]`: M30S `LORA_TX_POWER=22`; M33S `LORA_TX_POWER=8 MAX_LORA_TX_POWER=8` (33S PA input must stay <= 9 dBm). Envs: `EbyteMesh_repeater`, `EbyteMesh_companion_radio_usb`, `EbyteMesh_companion_radio_ble`.

## TODO: E22P-915M30S support (not implemented)

The E22P-915M30S fits the same footprint and PCB wiring, but pins 6/7 have different semantics (source: Ebyte manual https://www.cdebyte.com/pdf-down.aspx?id=4333):

| Pin | E22-900M30S/33S | E22P-915M30S | PCB net |
|---|---|---|---|
| 6 | RXEN (high only in RX) | EN: RF FEM enable, must be HIGH in **both** TX and RX | P0.17 |
| 7 | TXEN | T/R CTRL: HIGH = TX, LOW = RX | tied to DIO2 |

Truth table (EN, T/R): 1,1 TX · 1,0 RX · 0,X off. Hardware needs no change — this is Ebyte's recommended circuit.

**Problem:** current EbyteMesh firmware sets `SX126X_RXEN = P0.17`, so RadioLib drives P0.17 LOW during TX → module disabled while transmitting (RX still works, which hides the bug).

**Planned fix (EbyteMesh):** add `-D EBYTE_E22P`; in `variants/ebytemesh/target.cpp` after `radio.std_init()` call `radio.setRfSwitchTable(...)` with pin P0.17 (Arduino pin 2): STBY LOW, RX HIGH, TX HIGH (pattern: `variants/rak3x72/target.cpp`). Keep `SX126X_DIO2_AS_RF_SWITCH`.

Other E22P facts:
- TX power: input up to 22 dBm allowed (no 9 dBm limit); 22 dBm in → ~31 dBm out @915 MHz, 10 → ~27.5, 5 → ~22.5. Use `LORA_TX_POWER=22`, no `MAX_LORA_TX_POWER=8`.
- TCXO via DIO3 at 1.8 V (matches current firmware).
- VCC 3.3–5.25 V (lower max than E22's 5.5 V) — feed 5.0 V; README "max 5.5 V" does not hold for E22P. TX current 600–670 mA @30 dBm.
