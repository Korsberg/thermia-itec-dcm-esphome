# Changelog

Firmware history (the YAML will be published under `esphome/`).

## v40b (25 November 2026)

- Removed the hard-coded stop caps: 20 000 runtime ACKs and 30 000 mailbox replies no longer end the session. Both counters are 32-bit diagnostic counters. A real communication failure is now the only thing that ends a session.

## v40 (25 November 2026)

- ESP32-S3 system status: heap, largest free block, fragmentation, minimum free heap, PSRAM, loop stack, loop time, CPU frequency, chip temperature, uptime, reset reason, ESPHome version, Wi-Fi signal, IP, SSID, BSSID, MAC.
- Gateway health: bus OK, gateway armed, time since last frame, frames seen, resync share.

## v39 (25 November 2026)

- Read-only text sensors for alarm history 1 to 8, the active alarm, the four calendars (first period) and concrete drying.

## v38

- Candidate sensors for `082F` bit 2, the three immersion heater bits, raw words `03F1`, `03F2`, `03F5`, and the AVFROSTNING menu words `047E` (KOMPR. STOPP) and `047F` (ENERGIPRIO.VV).
- Settings keep their last read value and show their age separately.

## v37

- A configuration page re-sent by the pump while armed (for example `03E8` after a setting change on the display) is ACKed transport-only, stored and logged. Settings follow the display.

## v36

- A failed isolation gate no longer ends the session: that ACK is skipped and the pump re-sends the page. The session stops only after 5 failures in a row.
- Read-only timers: hot water run minutes, last hot water duration, hot water count, compressor run minutes.

## Earlier

- Automatic arming about 30 s after boot, no time limit, no button.
- Watchdog: a session lost for 90 s reboots the ESP.
