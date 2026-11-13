# BlkhurstRF Overview

These notes capture the chronological decisions.

The goal was to create a rechargeable handheld device with a screen, buttons, LED, and CC1101 radio. It should support button-controlled power on/off, automatic shutdown after inactivity, and USB programming.

## KiCad Setup

Created project-specific libraries. `blkhurst-rf` for symbols and `blkhurst-rf.pretty` for footprints.

The workflow is to download parts from Mouser, check them against the datasheets, link each symbol to its footprint and 3D model, then save verified copies in the project libraries.

## Power flow

1. USB-C and the battery connect to the charging module.
2. Its system output supplies the regulator and LTC2954.
3. The LTC2954 controls the regulator's enable pin.
4. The regulator powers the ESP32, CC1101 and display.

## Symbols and Footprints

Create and understand the components with key bullets.

- [x] ESP32-C3FH4X
  - [ ] ABRACON ABM8-40.000MHZ-10-D1G-T
- [ ] CC1101
  - [ ] 26 MHz Crystal
  - [ ] SMA antenna
- [ ] USB-C
  - [ ] ESD protection
- [ ] Charging Module
  - [ ] JST connector
- [ ] On/Off Controller
- [ ] Regulator

### ESP32-C3FH4X

<img src="./001 - ESP32 Symbol.png" height=300 />

- Requires 40 MHz external crystal.
- This specific variant includes built-in 4 MB flash, and USB Serial/JTAG controller.
- For the `C3FH4X` variant, pins 19 through 24 are **not bonded** and are labeled as "not connected" [[1]](https://documentation.espressif.com/esp32-c3_datasheet_en.pdf).
- Strapping GPIOs (2, 8, 9) are used to configure boot. After reset, the strapping pins work as normal function pins [[2]](https://documentation.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32c3/schematic-checklist.html?title=ESP32-C3%20Hardware%20Design%20Guidelines#strapping-pins).

### ABRACON ABM8-40.000MHZ-10-D1G-T

### LTC2954 (CDDB-1)

## Circuit Design
