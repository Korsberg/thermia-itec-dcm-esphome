# Troubleshooting

Log lines quoted here come from the ESPHome log of the firmware.

## Nothing happens after flashing

The pump sends its gateway challenge only when the pump itself starts. After flashing or rebooting the ESP, wait about 30 s until the log shows `ARMED`, then restart the pump. A pump restart before the ESP is armed is lost. Restarting the pump resets the degree-minutes and stops the compressor, so do it when no heating or hot water cycle is running.

When it works, the log shows `CONFIG_EXPORT_COMPLETE pages=31` within about 75 s of the challenge, then `RUNTIME_PAGE_ACK_SENT` lines for the ten runtime pages about every 21 s, and the DCM symbol appears on the pump's main screen.

## `RESYNC header byte=00 next=02 pending=28`

The receiver resynchronised on a frame header. Over long runs this is about 6-10 % of frames and is normal here: there were 0 dropped frames and 0 CRC errors in the long runs. Changing the cable to doubled conductors made no measurable difference. Removing the 120 ohm terminator from the RS485 module made no difference either: the first 24 resync samples after arming are byte for byte the same before and after (a 28-byte frame that repeats about once a second, and a stray `00` byte in front of the first configuration page). It is traffic on the bus that is not addressed to the gateway, stepped over by the parser, and not noise on the line. The resync share is therefore not a measure of signal quality.

## `RX_GATE_FAIL` and `GATE_FAIL_SKIP`

Before ACKing a runtime page the firmware checks that the page arrived alone. If something else is pending in the UART, that one ACK is skipped (`GATE_FAIL_SKIP streak=N`), the session is kept and the pump re-sends the page. Up to v41e the session stopped after 5 failures in a row (`GATE_FAIL_LIMIT`). From v41f it is kept (`GATE_FAIL_LIMIT_REACHED`, status `Gate-fel pågår`) and the pump keeps re-sending the page.

This happened once in a long run: on 25 November at 10:04 the pump sent page `0820` five times in eight seconds, each time with 5 bytes pending in the UART (`uart=5`, `pending=0`, `drops=0`). The watchdog rebooted the ESP at 10:06:03 and the session was back after a pump restart, with all 31 pages exported at 10:14. It happened again on 25 November at 20:15 (five failures on page `0820`, `uart=5`, session stopped; single failures at 19:45 and 20:13 recovered on the pump's retry). Since v40c the ESP is not rebooted automatically, so it stayed stopped until the power was cycled and the pump restarted. The content of the 5 bytes is not known; v40e logs them (`GATE_EXTRA_BYTES`).

### The 5 extra bytes are a frame: `0A 83 21 F0 EA` (27 November)

On 27 November at 02:04:15 to 02:04:24 the gate failed five times in a row on page `0820` (45 raw bytes), about 2 s apart, as the pump retried. The session had run 4 h 33 min with 7229 runtime ACKs and stopped at 02:04:24 (v41e). Each time 5 bytes were pending in the UART (`uart=5`, `pending=0`, `drops=0`). v40e logged them and they were the same every time: `0A 83 21 F0 EA`.

- The bytes are a **CRC-valid Modbus RTU frame**: slave `0x0A`, function `0x83` (the exception form of function 3), code `0x21`, CRC `F0 EA` (CRC16 of `0A 83 21`). So it is not line noise. Code `0x21` is not one of the standard Modbus exception codes.
- Slave `0x0A` is not the gateway (`0x0F`). Who sent the frame is not known. The ESP does not transmit as slave `0x0A`.
- The v40e diagnostic empties the UART at every failed gate, so the identical bytes arrived anew with each retry and were not left over from the previous one.
- The terminator on the RS485 module had been removed 10.5 hours earlier (26 November 15:32). The event was not noise, so it does not show that the terminator matters either way.
- On 25 November the same failure showed `uart=5` as well; the content was not logged then, so it is not proven to be the same frame.

**What v41f does with it:** the frame is only recognised and logged (`KNOWN_EXTRA_FRAME_SEEN`). The page is **not** acknowledged because of it. Modbus RTU needs a silent gap of at least 3.5 character times (about 4 ms at 9600 8E1) between frames, and from the UART data alone the gap *before* the extra frame cannot be proven, so an ACK would be a guess. The ACK is skipped, the session is kept, and the pump retries.

Whether the pump accepts an ACK delayed by several milliseconds, and whether a delay-free ACK would have let the pump carry on, is unknown. v41f has been on the pump since 27 November 06:46 and has not met the frame yet, so this behaviour is only tested on the host.

## Reconnecting the bus while the ESP is running (`MAILBOX_0708_GATE_FAIL`)

Disconnecting the A and B wires for a minute and connecting them again (for example when swapping the RS485 module) leaves a few stray bytes on the line. On 26 November 15:08, 77 s after the bus went quiet, the first mailbox poll arrived with 4 stale bytes in front of it: `MAILBOX_0708_GATE_FAIL ... STOP RX ONLY`. The ESP stopped answering (`armed=0`), and the pump kept re-sending page `080C` without getting an ACK.

Restarting only the ESP did not help: it armed itself, the pump sent nothing but mailbox polls, and no challenge came, twice over about 20 minutes. The session came up when the pump was restarted by hand, with the ESP running and armed (`CONFIG_EXPORT_COMPLETE` 15:31:47, about 80 s after the pump restart).

Safer order when changing the module: stop the ESP first, change the module, start the ESP, wait for `ARMED`, then restart the pump (see the first section).

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

The summary sensor starts with `LARM|`, `INGET|` or `OKÄNT|`, followed by a status. `OKÄNT|Buss tyst` means no valid frame for 60 s: check the bus and the ESP. `OKÄNT|Gate-fel pågår` (v41f) means five or more isolation gates in a row have failed; the session is kept and the pump retries. `OKÄNT|Session stoppad` or `Session väntar` means the bus is alive but the gateway session is not running (see the sections above). `Ingen läsning` or `Svar uteblir` means no alarm page has been read yet or the last request timed out; the next request follows after 300 s (900 s after three timeouts). `Gammal` means the last page is older than 11 minutes. Only `INGET|Aktuell` means that the alarm page was read recently while the session is up. The meaning of the bits is in [alarms.md](alarms.md).
