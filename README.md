# EbyMeshCore

LoRa mesh node PCB: Pro Micro nRF52840 (nice!nano clone) + Ebyte E22-900M30S / E22-900M33S (SX1262), fed from an external 5 V (max 5.5 V) supply through an AMS1117-3.3 LDO. Targeted at [MeshCore](https://github.com/meshcore-dev/MeshCore) firmware.

> **This is a variant.** The hardware is [Ebytastic](https://github.com/Sobollion/Ebytastic) by [Sobollion](https://github.com/Sobollion), itself derived from [faketec v4](https://github.com/gargomoma/fakeTec_pcb). This repo ports the project to KiCad 10 and switches the firmware focus from Meshtastic to MeshCore. For the original photos, Meshtastic firmware images, Altium sources and the Russian README, see the upstream repo.

> **WARNING:** do not power the board from USB and the external 5 V input at the same time — it damages the LDO.

## Firmware

MeshCore variant `variants/ebytemesh/` in [120m4n/MeshCore-HJ7RMN](https://github.com/120m4n/MeshCore-HJ7RMN/tree/feature/ebytemesh), branch `feature/ebytemesh`.

- Reuses `variants/promicro` (faketec pinout = this PCB).
- `EbyteMeshBoard`: battery reports 0 mV (P0.31 floats, no battery).
- `EbyteMeshRadioWrapper`: clamps every `setTxPower()` to `MAX_LORA_TX_POWER`.
- Module selected in `[ebytemesh_module]`:
  - E22-900M30S: `LORA_TX_POWER=22`
  - E22-900M33S: `LORA_TX_POWER=8`, `MAX_LORA_TX_POWER=8` (33S PA input must stay ≤ 9 dBm)
- Envs: `EbyteMesh_repeater`, `EbyteMesh_companion_radio_usb`, `EbyteMesh_companion_radio_ble`.

Flash by copying the `.uf2` onto the Pro Micro's UF2 bootloader drive.

### Pinout (nRF52840)

| Function | Pin |
|---|---|
| SPI SCK / MOSI / MISO | P0.02 / P1.15 / P1.11 |
| LoRa CS | P1.13 |
| DIO1 | P0.10 |
| BUSY | P0.29 |
| RESET | P0.09 |
| RXEN | P0.17 |
| TXEN | driven by SX1262 DIO2 |
| TCXO | DIO3, 1.8 V |

## Power

- `+5v` comes only from J2 (`5V_IN`) and feeds the E22 VCC and the AMS1117 input.
- The Pro Micro 3V3 pin is tied directly to the AMS1117 output (`+3.3v`); VBUS is not on the header.
- **USB only:** the MCU and 3.3 V loads (I2C, SW_OUT, pull-ups) run, but the E22 is unpowered → radio init fails. Fine for flashing/config only.
- USB can't drive the radio without a hardware change (e.g. Schottky VBUS → `+5v`), and 500 mA is below the E22's 600–670 mA TX current at 30 dBm (more at 33 dBm).
- Supply: 5 V, 1 A (1.3 A peak for the 33S).

## Repository layout

| Path | Contents |
|---|---|
| `EbyMeshCore.kicad_pro` / `.kicad_pcb` | KiCad 10 project; the PCB (imported from Altium) is the source of truth |
| `EbyMeshCore.kicad_sch` | Schematic reconstructed after import (PDF: `Docs/EbyMeshCore_schematic.pdf`) |
| `EbyMeshCore.kicad_sym`, `EbyMeshCore.pretty/` | Project-local symbol and footprint libraries |
| `production/` | JLCPCB outputs (Fabrication Toolkit): Gerbers, BOM, positions |
| `Docs/` | E22 datasheets, Pro Micro nRF52840 schematic |

## BOM

- Pro Micro nRF52840 (nice!nano clone)
- Ebyte E22-900M30S or E22-900M33S
- AMS1117-3.3
- Electrolytic cap 100 µF 16 V (220 µF better)
- Ceramic caps: 2× 0.1 µF, 1 µF, 4.7 µF
- Push buttons
- SMA-F edge connector
- Optional: MOSFETs (SOT-23) for peripherals, 10 kΩ I2C pull-ups

Full BOM with JLCPCB parts: `production/bom.csv`.

## TODO: E22P-915M30S

Same footprint and wiring, but pin 6 is a FEM enable that must be HIGH in both TX and RX (current firmware drives it as RXEN → LOW during TX). Fix: `-D EBYTE_E22P` + `radio.setRfSwitchTable(...)` on P0.17 (STBY LOW, RX HIGH, TX HIGH). E22P allows 22 dBm input (`LORA_TX_POWER=22`) and has VCC max 5.25 V — feed 5.0 V. Not implemented yet.

## Credits

- [Sobollion](https://github.com/Sobollion) for [Ebytastic](https://github.com/Sobollion/Ebytastic) — [support the original author](https://pay.cloudtips.ru/p/c197b86d)
- [gargomoma](https://github.com/gargomoma) for [faketec](https://github.com/gargomoma/fakeTec_pcb)
- [lupusworax](https://github.com/lupusworax) for the [faketec v4](https://github.com/gargomoma/fakeTec_pcb/issues/16) sources

## License

See [LICENSE](LICENSE).
