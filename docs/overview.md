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
- An N-channel MOSFET pulls that gate low when the switched 3.2V supply is on. Its gate pull-down keeps it off when the supply is absent.
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

### Regulator - TPS63030DSKT

<img src="./circuit_006 - regulator.png" height=300 />

EN connects to the LTC2954 for on/off control.

Based on [Figure 5](https://www.ti.com/lit/ds/symlink/tps63030.pdf#page=10). PS/SYNC is pulled high for tighter voltage regulation at the expense of light-load efficiency. The regulator adjusts its switching to keep FB (feedback) at approximately **0.5 V**. [calculation](https://www.ti.com/lit/ds/symlink/tps63030.pdf#page=11):

$$
V_{\mathrm{OUT}}=0.5\left(1+\frac{R18}{R19}\right)
=0.5\left(1+\frac{1.13\,\mathrm{M\Omega}}{210\,\mathrm{k\Omega}}\right)
\approx3.190\,\mathrm{V}
$$

### CC1101

<img src="./circuit_007 - cc1101.png" height=300 />

This is my first RF circuit, so I followed TI’s reference closely to reduce the guesswork. I kept the RF network and supply decoupling identical, replacing the crystal capacitors with the appropriate ones for my crystal.

- Checked the footprint against TI's [land pattern](https://www.ti.com/lit/ds/symlink/cc1101.pdf#page=107) and [solder-paste example](https://www.ti.com/lit/ds/symlink/cc1101.pdf#page=108) for RGP0020H. Four separate paste openings reduce coverage beneath the exposed ground pad.
- Followed the [433 MHz application circuit](https://www.ti.com/lit/ds/symlink/cc1101.pdf#page=25) and [component table](https://www.ti.com/lit/ds/symlink/cc1101.pdf#page=26). Copied the RF network and supply decoupling from the [complete reference design](https://www.ti.com/lit/zip/swrr046), which also includes PCB layout files.
- Used an ABRACON ABM8 26 MHz crystal with a 10 pF load rating. Selected **15 pF** for C28 and C29 using [Section 7.3](https://www.ti.com/lit/ds/symlink/cc1101.pdf#page=23) and TI's typical **2.5 pF** parasitic capacitance. This assumes the completed PCB has similar parasitics.

$$
C_L = \frac{C28 \times C29}{C28 + C29} + C_{\mathrm{parasitic}}
\quad\Rightarrow\quad
C28 = C29 = 2(10 - 2.5) = 15\,\mathrm{pF}
$$

<details>
<summary>TI reference screenshots</summary>

[Land pattern and solder-paste example, pages 107–108](https://www.ti.com/lit/ds/symlink/cc1101.pdf#page=107):

<img src="./cc1101-ti-land-pattern.png" height=300 />
<img src="./cc1101-ti-solder-paste.png" height=300 />

[433 MHz application circuit and component table, pages 25–26](https://www.ti.com/lit/ds/symlink/cc1101.pdf#page=25):

<img src="./cc1101-ti-433mhz-application.png" height=300 />
<img src="./cc1101-ti-component-table.png" height=300 />

Complete CC1101EM 434 MHz reference schematic, revision 2.0.0, including supply decoupling:

<img src="./cc1101-ti-complete-reference.png" height=300 />

</details>

### Display & FPC Connector

<img src="./circuit_008 - display.png" height=300 />
<img src="./circuit_008 - fpc-footprint.png" height=300 />

Choosing the connector took more checking than expected. I measured the ribbon’s width, thickness, exposed contacts and stiffener to check its fit with the **Molex 503480-0800**.

- Confirmed pin numbering using the [display image](https://download.mikroe.com/documents/datasheets/ER-TFT1.14-2_datasheet.pdf#page=4) and [pin table](https://download.mikroe.com/documents/datasheets/ER-TFT1.14-2_datasheet.pdf#page=7).
- Rotating the screen 90° clockwise puts pin 1 at the top and pin 8 at the bottom. Folding the ribbon underneath without twisting preserves this order, with the contacts facing the PCB.
- Checked the connector’s entry side and numbering against the [Molex drawing](https://www.molex.com/content/dam/molex/molex-dot-com/products/automated/en-us/salesdrawingpdf/503/503480/5034800800_sd.pdf). The connection remains **1-to-1**, with no reversed pins.
- Used the [IPS Display Click schematic](https://download.mikroe.com/documents/add-on-boards/click/ips-display-click/ips-display-click-schematic.pdf) as a circuit reference and Molex’s recommended land pattern for the footprint.
