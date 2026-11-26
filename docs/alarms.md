# Alarms

Everything here is read-only. The ESP never writes a value to the pump and never acknowledges or clears an alarm. What is described below is what was observed on one installation (iTec XT10, display firmware CM930 2.4.0), plus clearly marked hypotheses. Status words: **verified** = checked against the pump display, **observed** = seen in logs, **candidate** = a plausible reading that is not proven, **unknown**.

The display menu is `LARM`. Entries there read `KOMM.FEL HUB-KORT`, `KOMM.FEL ONLINE/LINK` and `ELTILLSATSFEL`, each with a time and date.

## Two sources

| Source | Page | Holds |
|---|---|---|
| Active alarm | `085F` (5 words, registers 2143 to 2147) | which alarm conditions are on right now |
| Alarm history | `0884` (60 words, part of the runtime ring) | the time of past alarms, newest first |

All five words of `085F` are `0000` when no alarm is active (observed for hours at a time).

## Reading the active alarm (firmware v41d, candidate)

The pump sends `085F` as a function 16 page, like the other pages. The ESP asks for it through its stateful reply to the mailbox poll (`0708`): bit `0x0080` in word 4 of the reply means "alarm page only".

- A request is sent every 300 s, measured from the start of the previous request. After three consecutive timeouts the interval becomes 900 s.
- A request waits at most 30 s for a page. A timeout is logged and there is no early retry.
- Every valid, isolated `085F` page is acknowledged (transport only). There is no cap on these acknowledgements. Only the logging of identical repeated pages is limited (one line per 60 s, with a count of suppressed lines).
- A page that arrives at any time is counted as one of three kinds, named after **timing only**: `IN_REQUEST_WINDOW`, `AFTER_TIMEOUT_WINDOW`, `UNSOLICITED`. The names say when the page came, not what caused it.

Observed after the v41d flash on 26 November 2026, 11:03 (the pump was restarted by hand, which is what makes it send its challenge, see [troubleshooting.md](troubleshooting.md)): the first seven requests were each followed by a page within 5 to 15 s, and there were no timeouts, no failed acknowledgements and no gate failures. The pump also sent a few pages on its own. **It is therefore not proven that the request causes the page**, only that a page follows it every time so far.

## Bits in `085F` (candidates)

| Word, bit | Meaning | Status | Notifies |
|---|---|---|---|
| 2143 bit 2 | additional heat fault (ELTILLSATSFEL on the display) | candidate | yes |
| 2145 bit 8 | communication error, HUB-KORT | verified against the display (26 Nov 2026, 13:14:54: `085F` word 2145 = `0100` for about 47 s, display `KOMM.FEL HUB-KORT` 13:15) | no (shown only) |
| 2145 bit 11 | communication error, internet gateway (ONLINE/LINK) | verified against the display as a history record (26 Nov 2026, `LARM 1` = ONLINE/LINK 21:00 after a gateway flash); `0800` was seen in `085F` for the first minute after an ESP restart on 25 Nov 21:39 | no (shown only) |
| 2143 bit 9 and 2144 bit 4 | treated as OEM condition inputs, not alarms | assumption | no (shown as "OEM-ingång … (ej larm)") |
| all other bits | unknown | unknown | yes |

The two communication errors do not notify because, on this installation, the alarms in the history have matched ESP restarts and lost sessions. The bit names come from a read-only comparison with another user's published file and from the display texts. They have not been checked by watching the display and a live `085F` page at the same moment, so they stay candidates.

## What Home Assistant shows

Firmware v41d publishes one summary sensor, updated all at once so that the parts always belong together:

`TOKEN|status|<five words in hex>|<text>`

| Token | Meaning |
|---|---|
| `LARM` | at least one bit that should notify is set |
| `INGET` | no such bit |
| `OKÄNT` | the state cannot be judged |

| Status | When |
|---|---|
| Buss tyst | no valid frame on the bus for 60 s |
| Session stoppad | the gateway session is not armed or not established |
| Session väntar | the session is waiting for traffic |
| Ingen läsning / Svar uteblir | no `085F` page yet, or the last request timed out |
| Gammal | the last page is older than 11 minutes |
| Aktuell | a page is fresh |

`Aktuell` means that `085F` was read recently while the session is up, not that the pump has no problem. Every published text is cut to 255 bytes (the Home Assistant state limit) on a UTF-8 boundary; a cut text ends in `+N fler (se råord)`. The raw words are always published in the separate hex sensor.

The notification automations and the status sensors are published together with the firmware.

## Alarm history (`0884`)

Observed on 522 distinct captures:

- 60 words are 8 records of 7 words: `[word0, 0, minute, hour, day, month, year-2000]`, newest first. All 522 captures fit. The minute is stored for a new record (13:12 and 13:15 on 26 Nov, 15:56 and 21:16 on 25 Nov) and was seen as `00` in older records. When it is zeroed is not known: some records kept their minute for hours, and all minutes disappeared together at some points (for example at 20:15 on 25 Nov). Home Assistant shows only the hour for every record in v41d (the text is built with a fixed `:00`), so a new alarm at 13:12 shows as 13:00 there.
- A 10 x 6 layout fits only 522 of 5220 records, so it is not the layout.
- The last four words (56 to 59) are always `0000 0000 0000 00hh`. When a new alarm pushes the list one step, `hh` becomes the hour of the record that left position 8. This held for every shift that could be followed (13 shifts across the captures). So the tail is the start of a **ninth** record, with the hour only.
- Display entry *n* matches data record *n* (see the table). This replaces an earlier note that said *n* - 1.
- No further `FC16` page follows `0884` in the logs, about 6000 ring cycles, and the reference file has nothing after `0884/60`.

Comparison with the display, photos from 26 November 2026 (local 08:55 to 09:01). The data was identical before and after (22:19 on 25 November and 11:0x on 26 November), apart from the seconds field.

| Display entry | Text | Display time | Data record | Data hour |
|---|---|---|---|---|
| 1 | ONLINE/LINK | not photographed | 1 | 21 |
| 2 | ONLINE/LINK | 21:00 | 2 | 21 |
| 3 | HUB-KORT | 21:00 | 3 | 21 |
| 4 | HUB-KORT | 21:00 | 4 | 21 |
| 5 | ONLINE/LINK | not photographed | 5 | 20 |
| 6 | ELTILLSATSFEL | 20:00 | 6 | 20 |
| 7 | ONLINE/LINK | not photographed | 7 | 17 |
| 8 | HUB-KORT | 16:00 | 8 | 16 |
| 9 | HUB-KORT | not photographed | 9 (hour only, tail) | 16 |
| 10 | HUB-KORT | 16:00 | not in the 60 words | none |

All five photographed times (entries 2, 3, 4, 6 and 8) match. Entry 10 cannot be checked.

### What is not known

- **Word 0 of a record is not the event type.** Its value is `0x2C` for entries 1, 3 and 4, which read ONLINE/LINK, HUB-KORT and HUB-KORT, so the same value covers both texts. The value `0x29` occurs for entries 2 and 5 (both ONLINE/LINK), too few to say anything. One record changed from `0x29` to `0x2C` within two minutes. In records 6, 7 and 8 the first word was the same in all 14 list states captured (`0x01A9` and `0x001E` in 7 and 8; a seconds-like counter in 6), whatever record sat there. Its meaning is unknown.
- **Where entries 9 and 10 are stored.** Hypotheses, none proven:
  1. The history is 10 records of 7 words (70 words), and the page carries only the first 60.
  2. The type of an event is in the missing words or on another page.
  3. The remainder is sent only if requested.
  Nothing passive can tell these apart. No new bus request has been made for this, and none is planned until a new alarm can be compared with the display.
- The history sensors show the date, the time and the raw first word as `(kod N)`, for example `kod 44`. Up to V41d the time was the hour with a fixed `:00`, even when the data holds the minute; V41e shows the stored minute. They do not name an event type, because that word is not the type.
- **`KOMM.FEL ONLINE/LINK` records are stamped on the full hour (`hh:00`).** Photographed on 26 Nov 2026: display `LARM 1` = `KOMM.FEL ONLINE/LINK 21:00`, data `002C 0000 0000 0015 001A 000B 001A` (word 0 `0x2C`, minute 0, hour 21). It appeared in `0884` after the gateway was flashed at 21:34 and the pump restarted at about 21:41, with the alarm page `085F` all zero before it. The same happened with the ESP restart on 25 Nov (ONLINE/LINK 21:00) and the module swap on 26 Nov (15:00). So ONLINE/LINK is what the pump logs when the gateway disappears (restart, flash, bus reconnect), and its time has no minute. HUB-KORT records, in contrast, carry the real minute (13:12, 13:15).

## Observed alarm episodes

Alarm episodes seen so far, all `KOMM.FEL HUB-KORT` or `KOMM.FEL ONLINE/LINK`. Times are the minute stored in a new history record.

- 25 Nov 2026: 15:56, 16:39, 16:42, 16:50, 16:53, 20:10, 21:16, 21:18 (plus ONLINE/LINK at the ESP restart 21:39).
- 26 Nov 2026: 13:12 and 13:15.
- Records caused by us, not by the bus: ONLINE/LINK 15:00 on 26 Nov (RS485 module swap) and ONLINE/LINK 21:00 on 26 Nov (gateway flash and pump restart).
- After the 120 ohm terminator was taken off the RS485 module (26 Nov 15:32), there was no spontaneous episode for 6 hours until the flash at 21:34 (all `085F` pages were zero). Too short to be a result on its own.
- None in the first five hours of 25 Nov (08:54 to 13:39), none overnight from 21:39 to 11:01, none 11:02 to 13:11.

What the log shows around an episode (26 Nov 13:14 to 13:15):

- `080C` word 15 bit 5 (`0020`) is high for 25 to 70 s. It was seen at every episode we could follow (candidate: an "alarm active" flag on a page that arrives about every 21 s).
- The pump sends an extra `085F` page, in its normal cycle after `0848`. At 13:14:54 word 2145 was `0100`; at 13:15:41 all words were `0000` again. At 13:12 only the all-zero page was caught, so that episode was shorter than one cycle.
- The new record appears in `0884` about 6 s after the first `085F` page.
- The gateway side looked normal: mailbox replies within about 25 ms, no timeouts, no gate failures, no gap in the bus longer than 3.7 s. The requests we send do not line up with the episodes.

The cause is unknown. The episodes happened with firmware v37 to v41d, so no single firmware version explains them. Whether they also happen without the ESP connected is not known; the pump's list holds only entries from 25 Nov 16:00 onward.

## Next time an alarm appears

Photograph the `LARM` list on the display and write down the time, then compare with a new `0884` capture. That shows which words change for a new record and whether the type can be found among them.
