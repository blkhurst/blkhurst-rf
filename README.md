# BlkhurstRF

A handheld 433 MHz radio development board built around an ESP32-C3 and CC1101, with an integrated screen, buttons, and rechargeable battery.

This repository contains the KiCad files, enclosure CAD design, and firmware.

## Features

- ESP32-C3FH4 microcontroller
- CC1101 RF Transceiver
- USB-C for programming, power, and battery charging.
- Protected lithium battery with detachable connector (JST).
- TFT screen, connected with an FPC connector.
- Three buttons and a charging LED indicator.
- External case-mounted antenna connected by coax to an SMA connector.
- CNC-machined aluminium enclosure.
- PCB-mounted on/off switch when not using deep sleep.

## Selected Components

- ESP32 - [ESP32-C3FH4X](https://www.mouser.co.uk/en/ProductDetail/Espressif-Systems/ESP32-C3FH4X?qs=i8QVZAFTkqTBHNQ1W71daA%3D%3D)
  - 40 MHz Crystal - [ABRACON ABM8-40.000MHZ-10-D1G-T](https://www.mouser.co.uk/en/ProductDetail/ABRACON/ABM8-40.000MHZ-10-D1G-T?qs=vLWxofP3U2ziONa1hNg0FA%3D%3D)
- Display - [MIKROE-6587 - Digikey](https://www.digikey.co.uk/en/products/detail/mikroelektronika/MIKROE-6587/26741734)
  - FPC connector - [Hirose Connector FH12-8S-0.5SH(55)](https://www.mouser.co.uk/en/ProductDetail/Hirose-Connector/FH12-8S-0.5SH55?qs=c1kC0VY36qtDQq5g6A7Q7w%3D%3D)
- Battery - [Jauch LP561836JU - CPC Farnell](https://cpc.farnell.com/jauch/lp561836ju/rechargeable-battery-li-ion-0/dp/BT07609)
  - JST connector - [JST Commercial S2B-PH-K-S(LF)(SN)](https://www.mouser.co.uk/en/ProductDetail/JST-Commercial/S2B-PH-K-SLFSN?qs=sGAEpiMZZMvlX3nhDDO4AFC8wiFPNfng%252BEsGCJj4wus%3D)
  - Leads - [Adafruit 261](https://www.digikey.co.uk/en/products/detail/adafruit-industries-llc/261/5353586)
- Push Button On/Off Controller [LTC2954 CDDB-1#TRPBF](https://www.mouser.co.uk/en/ProductDetail/Analog-Devices/LTC2954CDDB-1TRPBF?qs=hVkxg5c3xu8r5dNS7kQExw%3D%3D)
  - CDDB is the surface mount DDB smaller package vs the CTS8 TS8 package with larger legs.
