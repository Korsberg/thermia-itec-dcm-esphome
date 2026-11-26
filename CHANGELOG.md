# Changelog

Firmware history (the YAML will be published under `esphome/`).

## v41d (26 November 2026, candidate, not published)

- v41c plus a single bit policy for filter and text: `2143.9` and `2144.4` are treated as OEM condition inputs, never notify (token `INGET`) but are always shown as "OEM-ingång … (ej larm)". `2145.8` (HUB-KORT) and `2145.11` (ONLINE/LINK) are shown as candidates and do not notify. The text can say "inga aktiva bitar" only when all five words are zero. Every published alarm string is cut to 255 bytes on a UTF-8 boundary.
- Running on the pump since 26 November 2026, 11:03: the first seven alarm requests were answered within 5 to 15 s, no timeouts, no failed acknowledgements.

## v41c (26 November 2026, candidate, not published)

- First freshness limit is 11 minutes. `Session stoppad` and `Session väntar` are shown even when other traffic is on the bus. `Buss tyst` is kept when no valid frame has been heard for 60 s. A separate Home Assistant warning for alarm monitoring that stays unknown.
- No cap on `085F` acknowledgements: every valid, isolated page is acknowledged. Only the logging of identical repeats is limited (one line per 60 s, with a count), and the counters for changed, repeated and suppressed pages are kept.

## v41b (26 November 2026, candidate, not published)

- The alarm status is published by its own 5 s timer instead of only from the 30 s report. A page counts as observed only after its acknowledgement succeeded. The page kinds are named after timing only (`IN_REQUEST_WINDOW`, `AFTER_TIMEOUT_WINDOW`, `UNSOLICITED`). Repeated `0x0080` replies are counted. One atomic summary sensor `TOKEN|status|words|text`.

## v41 (26 November 2026, candidate, not published)

- Only alarm handling changed from v40e: a request state machine (30 s timeout, 300 s interval, 900 s after three consecutive timeouts, no early retry), logging of request, answer, timeout and the age of the observation, and new alarm status sensors. The hex sensor for `085F` is kept.
- Tested on the host with a simulated UART (unit tests, nine integration scenarios, Home Assistant template tests). Not tested on the pump until v41d.

## v40e (25 November 2026)

- v40d plus one diagnostic: at a failed isolation gate the stray UART bytes are logged in hex (`GATE_EXTRA_BYTES`, at most 24 bytes, read out so they can be shown) together with the last 6 bytes of the frame. No TX change; the ACK is still skipped. Added after the gate stopped the session on page `0820` a second time (uart=5), now at 20:15.

## v40d (25 November 2026)

- v40c plus one diagnostic log line: `CHALLENGE_NOT_ANSWERED`, which states why a clean challenge on an established session was not answered (waiting, idle time, rechallenge count, peer, drops, ambiguous, resync delta, frames, pending, UART). No protocol or TX change.

## v40c (25 November 2026)

- A 1 s timer detects 30 s of inactivity, including an incomplete configuration export. The session is retained (`SESSION_WAITING` after 30 s, `SESSION_WAITING_LONG` after 90 s, `SESSION_RESUMED` when traffic returns). The session-loss reboot watchdog was removed.
- Candidate: a new clean challenge is accepted after the retained session has been idle for 30 s, at most three times per boot (`RECHALLENGE_ACCEPTED`). The resync guard is re-based when the wait starts. Tested as software only; restarting the pump with the ESP running still has to be tested on the pump.

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
