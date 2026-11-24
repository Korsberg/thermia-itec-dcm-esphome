# thermia-itec-dcm-esphome
# Thermia iTec XT10: read-only DCM gateway emulation with ESPHome

A read-only emulation of the Thermia online gateway (DCM) for the **Thermia iTec XT10** air/water heat pump, running on an **ESP32-S3** with an RS485 transceiver and **ESPHome**. It lets Home Assistant read the heat pump's runtime values and menu settings locally, without the cloud and without writing anything to the pump.

> **Status: work in progress.** The ESP answers the pump's gateway handshake automatically after every power-up and then reads the pump for days without errors. Most sensors are verified against the pump display; the rest are marked as candidates (see below). The handshake code is not published yet.

## What it does

- Answers the pump's gateway challenge on slave address `0x0F` (RS485, 9600 baud, 8E1), automatically after each pump start, with no button press and no time limit.
- Acknowledges the 31 configuration pages the pump exports at start-up and the ten runtime pages that follow, and answers the `0708` mailbox. These acknowledgements are transport-level only.
- Decodes the received pages into Home Assistant sensors named like the pump's display menus (VARMVATTEN, VÄRME, TILLSATS, OPTIMUM, INFO UTEDEL, INSTALLATION, KALENDER and others).
- Reboots itself and re-arms if the pump restarts and the session is lost.

## What it does not do

- **No writes.** The ESP never changes a pump parameter. There are no setpoint sliders or room sensor writes, by design. Writing blind to a working heat pump can do real damage.
- No cloud, no account, no vendor app.

## Hardware

| Part | Notes |
|---|---|
| ESP32-S3 DevKitC-1 N16R8 | UART on GPIO17 (TX), GPIO18 (RX) |
| DollaTek 5 V MAX485 / RS485 module (TTL to RS-485) | DE/RE on GPIO21 |
| 3 x 4.7 kOhm resistors | wiring: to be added |
| Thermia iTec XT10 with Total EQ | display firmware CM930 2.4.0, COMM.KIT 2310 1801, MAIN MICOM 2207 601 |

### Wiring

| ESP32-S3 | RS485 module |
|---|---|
| GPIO17 (TX) | DI |
| GPIO18 (RX) | RO |
| GPIO21 | DE and RE (tied together) |

| Cable colour | RS485 module terminal |
|---|---|
| brown | A |
| green | B |
| blue | ground |

The three 4.7 kOhm resistors and the module supply voltage are still to be documented.

Connect to the pump's gateway (DCM) RS485 port. Do not connect while the pump is running a service operation, and use it at your own risk.

## Verified against the pump display

| Display menu / value | Where it comes from | Verified |
|---|---|---|
| Heating curve, min/max setpoint, heat stop, setback, room factor | page `03E8` | yes |
| Operating mode (off, auto, compressor, additional heat, hot water) | `0553` | yes |
| VARMVATTEN menu (start, stop, time, interval, top-up) | `041D` to `0424` | yes |
| VÄRME (service) start/stop in degree-minutes | `0431`, `0432`, `0433`, `0434` | yes |
| TILLSATS (max steps, start, delay) | `0456`, `0457`, `0459`, `0461` | yes |
| OPTIMUM flows, charge temperatures, circulation | `0492` to `049C` | yes |
| INFO UTEDEL: compressor Hz, max frequency ratio, current, fan rpm | page `0820` | yes |
| INFO UTEDEL: high pressure, expansion valve steps and percent | page `0820` words 16 and 13 | yes (standby values) |
| Heat demand in degree-minutes | page `0848` word 0 (sign inverted) | yes |
| Compressor start at -50, stop at +50 degree-minutes | observed over two runs | yes |
| Pump clock | page `0848` words 16 to 22 | yes |
| Serial number, COMM.KIT and MAIN MICOM versions | `06F2/06F3`, `07E4` | yes |
| Operating hours, defrost count and intervals | page `0870` | yes |

Items still marked as candidates: EQ / OPTIMUM / ZONER flags, several status bits (`0864`, `07E4 word 16`), the defrost flag, SATT TEMP and BEKRÄFTAD TEMP, and pressure and valve values while the compressor is running.

## Home Assistant

The repository will include a dashboard with one card per pump menu, a card with power, compressor frequency and electricity price, and a card for defrost tracking.

## Safety

This is an independent hobby project and is not affiliated with Thermia. Use it at your own risk. It only reads, but it does connect a custom device to your pump's service bus.

## Credits

- [ryckema/Thermia_itec](https://github.com/ryckema/Thermia_itec) for the iTec protocol work this builds on.
- [piotr-romanowski/thermia-bus-sniffer](https://github.com/piotr-romanowski/thermia-bus-sniffer) for the passive bus notes.
- The Home Assistant community thread on local Thermia iTec integration.

## License

To be decided before the code is published.
