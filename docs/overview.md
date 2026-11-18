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

<img src="./002 - ABM8.png" height=300  />

- Verified the symbol's pin mapping. Moved each footprint pad 0.05 mm towards the centre to match Abracons [recommended land pattern](https://abracon.com/Resonators/abm8.pdf#page=2).

### USB4215-03-A

<img src="./003 - USB-C.png" height=300  />

- Symbol and footprint unavailable, created from manufacturers datasheet.

### LTC2954 (CDDB-1)

## Circuit Design

### USB-C

<img src="circuit_002 - USBC.png" height=300  />

- Used Espressif's recommended 33 Ω series resistors on USB D− and D+.
- ESD protection on data lines.
- CC1 and CC2 grounded via 5.1k resistors request 5v from the power source.

### LiPo battery - Jauch LP561836JU

Jauch [datasheet](https://www.jauch.com/downloadfile/5bf529acf2902e0e6585a358571c3a4c8/350mah_-_lp561836ju_1s1p_2_wire_50mm.pdf) specifies maximum charging currents of **74 mA from 0-15°C** and **370 mA from 15-45°C**.

Small LiPo packs often have only two power wires, with no thermistor connection. For this prototype, I do not use temperature monitoring and will only charge while attended, with the battery above 15C and below 45C.

<img src="circuit_004 - battery.png" height=300 />

- Used a keyed JST connector, so omitted reverse-polarity protection.
- A P-channel MOSFET connects VBAT to the divider when its gate is pulled low. A pull-up to VBAT keeps it off otherwise.
- An N-channel MOSFET pulls that gate low when the switched 3.3V supply is on. Its gate pull-down keeps it off when the supply is absent.
- Added a **0.1µF capacitor** at the ADC input to reduce noise, following [Espressif’s guidance](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-reference/peripherals/adc/adc_calibration.html#minimize-noise).

### Charging - BQ24074

The BQ24074 supports the power-path management I desired. The device can run from USB-C alone, battery alone, or USB-C while charging ([TI overview](https://www.ti.com/lit/pdf/SLVU274#page=2)). The charger prioritises powering the device over charging the battery. If USB power is insufficient, the battery can supplement it. The circuit is based on [Figure 10-12, page 38](https://www.ti.com/lit/ds/symlink/bq24074.pdf#page=38), using pages **34–35** for component calculation and **12–14** for electrical limits and tolerances.

<img src="circuit_003 - charging.png" height=300  />

- Selected **USB500 mode** using EN1 high and EN2 low. This limits total input current to **500mA**.
- Set charging current to **100mA** using $R_{ISET}$. Slower charging is acceptable for my occasional use.
- Set the required $R_{\mathrm{ILIM}}$ resistor to keep the limit below **500 mA**. USB500 mode overrides this setting.
- Set termination to approximately **10 mA** by leaving ITERM open, selecting the default **10% of charging current**.
- Set the fast-charge safety timer to approximately **nine hours** using $R_{\mathrm{TMR}}$.
- Disabled temperature monitoring as described above.

$$
\begin{aligned}
R_{\mathrm{ISET}}
&= \frac{890 \ \mathrm{A\Omega}}{0.100 \ \mathrm{A}}
= 8.90 \ \mathrm{k\Omega}
\quad\rightarrow\quad 8.87 \ \mathrm{k\Omega}
\\[12pt]
R_{\mathrm{TMR}}
&= \frac{9 \times 3600}{10 \times 48} \ \mathrm{k\Omega}
= 67.5 \ \mathrm{k\Omega}
\quad\rightarrow\quad 68.1 \ \mathrm{k\Omega}
\\[12pt]
R_{\mathrm{ILIM}}
&\geq \frac{1720 \ \mathrm{A\Omega}}{0.500 \ \mathrm{A} \times 0.99}
\approx 3.475 \ \mathrm{k\Omega}
\quad\rightarrow\quad 3.57 \ \mathrm{k\Omega}
\end{aligned}
$$

> **Prototype limitation:** The charger uses fixed USB500 mode and assumes a supply capable of **500 mA**. Before enumeration, USB 2.0 devices may draw up to **100mA** and USB 3.0 **150mA**. The device is responsible for observing these limits.
>
> Espressif's USB Serial/JTAG controller handles USB enumeration automatically and requests 500mA, but requires the ESP32 to be powered. The charger operates independently, so consumption may exceed the permitted limit before enumeration or while the ESP32 is powered off.
>
> Automatically selecting the input limit would require USB-state handling and separate supply detection while the ESP32 is off, so I omitted this complexity for a personal prototype used with a known suitable supply.

### On/Off Controller - LTC2954 (CDDB-1)

<img src="./circuit_005 - power controller.png" height=300 />

The LTC2954 allows one of my existing push buttons to double as power control _and_ ESP32 input via the debounced INT. Based on the [page 1 reference circuit](https://www.mouser.co.uk/datasheet/3/1014/1/2954fb.pdf), using the `-1` variant to enable the regulator.

- ONT controls the on-time, defaulting to **32 ms**.
- PDT controls the power-down time, approximately 6.5 seconds.
- Firmware can detect INT and pull KILL low to shut down sooner, or automatically after inactivity.
