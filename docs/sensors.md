# Sensors

Status of the sensors and flags exposed by the firmware (v40b). Where a register is described, see [registers.md](registers.md). Entity ids in Home Assistant are built from the area, the device's friendly name and the sensor name, and are fixed when the entity is first registered; names in the firmware will be shortened in a later version.

Status words: **verified** (matched against the pump display or a real event), **candidate** (plausible, not confirmed), **never seen** (the sensor exists but the pump has not set it yet), **unknown**.

## Binary flags

Observation window for "seen": 23 to 25 November 2026 (about two and a half days of history for this set of sensors).

### Verified

| Sensor | Source | Notes |
|---|---|---|
| Heating active (building) | `0864` word 1 bit 0 | follows the pump |
| Hot water active | `0864` word 1 bit 1 | seen on twice |
| **Defrost using building** | `0864` word 1 bit 4 | seen during the first captured defrost, see [defrost.md](defrost.md) |
| **Defrost, outdoor unit flag** | `0820` word 15 (`082F`) bit 2 | seen at the same time as the defrost |
| High power (HÖG EFFEKT) | `03F3` | follows the display |
| Hot water on (VARMVATTEN) | `042E` | follows the display |
| TILLSATS FRYSSKYDD | `0461` | follows the display |
| Calendar F1 to F4 active | `05BB`, `061D`, `067F`, `06E1` | F3 (quiet mode) seen on |
| Active alarm | `085F` | all zero without an alarm; see [alarms.md](alarms.md) |
| Alarm history 1 to 8 | `0884` | date and hour per record (minute not shown in v41d, always `:00`); the type of the event is not known (see [alarms.md](alarms.md)) |
| Bus OK, Gateway armed | firmware | derived from last frame age and session state |

### Candidates

| Sensor | Source | Notes |
|---|---|---|
| Compressor start phase | `0864` word 2 bit 0 | 64 s and 127 s in two runs; follows heating start |
| Compressor start / running | `07E4` word 16 bit 2 / bit 1 | follows the compressor |
| `07E4` word 0 flag | `07E4` word 0 | active also in standby |
| Defrost, earliest signal | `0864` word 0 (value 3) | seen once, 15 to 22 s before bit 4 of word 1; bits not decoded |
| `0864` word 2 bit 3, `07E4` word 16 change | `0864` word 2, `07E4` word 16 | changed around the first defrost |
| System EQ / OPTIMUM / ZONER | `0546` words 0 to 2 | EQ and OPTIMUM on, ZONER off |
| Immersion heater 1 / 2 / 3 | `07DD` bit 7, `07DE` bits 4 and 3 | taken from a forum report; never seen active |

### Never seen set

Hot water top-up (`0864` word 1 bit 2), pool heating (bit 3), defrost using hot water (bit 5), outdoor stop (bit 6), cooling (bit 7), concrete drying (bit 9), calendar F1, F2 and F4 active, KOMPR. STOPP and ENERGIPRIO.VV (service menu AVFROSTNING, `047E`, `047F`; both 0 and not changed), and all eight alarm slots.

### Unknown

The 16 bits of `0864` word 3 (always `0x00F0` so far). One of them may be the flow switch.

## Numeric sensors

### Verified against the display

| Group | Sensors |
|---|---|
| INFO UTEDEL (`0820`) | compressor frequency, BEKRÄFTAD TEMP, SATT TEMP (derived), MAX FREK. RATIO, current, fan speed, expansion valve steps and percent, high pressure |
| Outdoor unit temperatures (`0820`) | condenser in and out, refrigerant 1 and 2, discharge pipe, compressor temperature, outdoor unit ambient |
| System temperatures (`07D0`, `080C`) | supply, return, hot water top, tank, outdoor, room |
| Heat demand (`0848`) | degree-minutes (sign inverted so it matches the display) |
| Settings | heating curve and related (`03E8` page), room setpoint, hot water, heating and additional heat settings, OPTIMUM flows and temperatures |
| Counters | defrost count (`0870`), hot water run minutes, last hot water charge duration, number of charges, compressor run minutes |
| Installation | serial number, COMM.KIT and MAIN MICOM versions, pump clock |

### Candidates

Operating hours of the outdoor unit, heating, hot water and additional heat; minutes between defrosts and minutes since the last defrost (`0870`); setpoints in brackets on DRIFTDATA (`0848` words 1 and 2); hot water middle (`07D0` word 4); `0848` words 3 and 4; `07D0` word 11.

### Unknown

Settings words `03F1`, `03F2`, `03F5`, `0458`, `045B`; the first word of each alarm record (not the event type, see [alarms.md](alarms.md)).

## Gateway and ESP32 health sensors

These come from the firmware and the ESPHome debug component and are not read from the pump.

| Group | Sensors |
|---|---|
| Memory | heap free, largest free block, fragmentation, minimum free since boot, PSRAM free, minimum free loop stack |
| Chip and system | uptime, reset reason, chip temperature, CPU frequency, loop time, ESPHome version |
| Network | Wi-Fi signal, IP address, SSID, BSSID, MAC address |
| Bus and gateway | bus OK, gateway armed, time since last frame, frames seen since start, resync share, configuration pages ACKed, runtime ACKs, mailbox replies, alarm page reads, unknown pages, RX resynchronisations, dropped frames |
