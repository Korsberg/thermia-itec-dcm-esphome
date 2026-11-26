# Troubleshooting

Log lines quoted here come from the ESPHome log of the firmware.

## Nothing happens after flashing

The pump sends its gateway challenge only when the pump itself starts. After flashing or rebooting the ESP, wait about 30 s until the log shows `ARMED`, then restart the pump. A pump restart before the ESP is armed is lost. Restarting the pump resets the degree-minutes and stops the compressor, so do it when no heating or hot water cycle is running.

When it works, the log shows `CONFIG_EXPORT_COMPLETE pages=31` within about 75 s of the challenge, then `RUNTIME_PAGE_ACK_SENT` lines for the ten runtime pages about every 21 s, and the DCM symbol appears on the pump's main screen.

## `RESYNC header byte=00 next=02 pending=28`

The receiver resynchronised on a frame header. Over long runs this is about 6-10 % of frames and is normal here: there were 0 dropped frames and 0 CRC errors in the long runs. Changing the cable to doubled conductors made no measurable difference.

## `RX_GATE_FAIL` and `GATE_FAIL_SKIP`

Before ACKing a runtime page the firmware checks that the page arrived alone. If something else is pending in the UART, that one ACK is skipped (`GATE_FAIL_SKIP streak=N`), the session is kept and the pump re-sends the page. After 5 failures in a row the session stops (`GATE_FAIL_LIMIT`).

This happened once in a long run: on 25 November at 10:04 the pump sent page `0820` five times in eight seconds, each time with 5 bytes pending in the UART (`uart=5`, `pending=0`, `drops=0`). The watchdog rebooted the ESP at 10:06:03 and the session was back after a pump restart, with all 31 pages exported at 10:14. It happened again on 25 November at 20:15 (five failures on page `0820`, `uart=5`, session stopped; single failures at 19:45 and 20:13 recovered on the pump's retry). Since v40c the ESP is not rebooted automatically, so it stayed stopped until the power was cycled and the pump restarted. The content of the 5 bytes is not known; v40e logs them (`GATE_EXTRA_BYTES`).

## `SESSION_WAITING` and `SESSION_WAITING_LONG` (v40c and later)

No valid traffic for 30 s (90 s for the long form). The ESP keeps the session and resumes when clean traffic returns (`SESSION_RESUMED`). If the pump was restarted while the ESP was running, the new challenge is answered after 30 s of waiting (`RECHALLENGE_ACCEPTED`), at most three times per boot; if it is not answered, `CHALLENGE_NOT_ANSWERED` says why. This was added in v40c and v40d and is still being tested.

## `SESSION_LOST 90s / REBOOT TO REARM`

In firmware before v40c the session was silent for 90 s and the watchdog rebooted the ESP. The pump will not talk to the ESP again until the pump itself is restarted. Typical causes are a power loss on the ESP or the bus, or a run of failed gate checks. Wi-Fi is not needed for the session itself, only for Home Assistant.

## `SESSION_ACTIVITY_TIMEOUT`

No valid frame for 30 s after the export was complete. The ESP stops answering. Check the wiring and the bus.

## Sensors show `unknown` or `unavailable` after a restart

After the ESP reboots the sensors stay unavailable until the pump has started a new session and the pages have come in again (about 75 s for the settings, about 21 s for the runtime values). Settings keep their last value and show their age in a separate sensor.

## The ESPHome API accepts one client

Stop any log script before flashing, or the upload may be refused.

## Home Assistant dashboard shows `Loading...` or `N/A`

After the dashboard has been changed or Home Assistant restarted, graphs may show `Loading...`. A hard reload of the browser page fixes it.

## Alarm status in Home Assistant (v41d and later)

The summary sensor starts with `LARM|`, `INGET|` or `OKÄNT|`, followed by a status. `OKÄNT|Buss tyst` means no valid frame for 60 s: check the bus and the ESP. `OKÄNT|Session stoppad` or `Session väntar` means the bus is alive but the gateway session is not running (see the sections above). `Ingen läsning` or `Svar uteblir` means no alarm page has been read yet or the last request timed out; the next request follows after 300 s (900 s after three timeouts). `Gammal` means the last page is older than 11 minutes. Only `INGET|Aktuell` means that the alarm page was read recently while the session is up. The meaning of the bits is in [alarms.md](alarms.md).
