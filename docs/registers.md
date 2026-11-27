# Pages and registers

Status words used here:

- **verified**: matched against the pump display, or the display value was changed one step, the register was seen to follow, and the value was changed back (settings), or the value followed a real event (live values)
- **likely**: one consistent reading, not fully proven
- **candidate**: plausible, not confirmed
- **unknown**: not decoded

Word numbers (`word 0`, `ord0`) count from 0 within a page. Temperatures are in 0.1 degrees unless noted. Configuration word index = address - 1000 (decimal).

## Runtime pages (live values)

The ten runtime pages are sent in a ring, a full cycle about every 21 s. See [protocol-dcm.md](protocol-dcm.md).

### `07D0`: system temperatures

| Word | Meaning | Status |
|---|---|---|
| 0 | Supply temperature (framledning) | verified |
| 2 | Return temperature (returledning) | verified |
| 3 | Hot water, top | verified |
| 4 | Hot water, middle (value 48 seen) | candidate |
| 9 | Tank | verified |
| 10 | Outdoor temperature | verified |
| 11 | Unknown (divided by 10) | unknown |
| 13, 14 | Immersion heater status (`07DD` mask 0x80, `07DE` masks 0x10 and 0x08), from a forum report | candidate (never seen active) |

### `080C`: room

| Word | Meaning | Status |
|---|---|---|
| 0 | Room temperature (/10) | verified |

### `0820`: outdoor unit (INFO UTEDEL)

| Word | Meaning | Status |
|---|---|---|
| 0 | Condenser in | verified |
| 1 | Condenser out (/10) | verified |
| 2 | Refrigerant 1 (KÖLDM.1) | verified |
| 3 | Discharge pipe (TRYCKRÖR) | verified |
| 4 | Refrigerant 2 (KÖLDM.2) | verified |
| 5 | Outdoor unit ambient temperature | verified |
| 6 | Compressor temperature | verified |
| 7 | BEKRÄFTAD TEMP (/10) | verified |
| 8 | Compressor frequency (Hz) | verified |
| 9 | MAX FREK. RATIO | verified |
| 10 | Current (/10, A) | verified |
| 11 | Fan speed (rpm) | verified |
| 13 | Expansion valve steps (2000 = 100 %); percent = steps / 20 | verified (standby 2000 steps = 100 %; running 189 steps = 9 %) |
| 15 | Contains `082F`; **bit 2 = defrost** | verified (three defrosts, see [defrost.md](defrost.md)) |
| 16 | High pressure (/10, bar) | verified (standby 7.1 bar; running 17.4 bar) |

**SATT TEMP** (shown on the display next to BEKRÄFTAD TEMP) has no word of its own. It equals word 7 while the compressor runs and is 0 otherwise. It is derived in the firmware and was checked against the display at rest (0.0) and running (for example 27.5 and 27.5, 51.5 and 51.5 at 36 Hz). The display can be up to one 21 s sample ahead of Home Assistant.

### `0834`

| Word | Meaning | Status |
|---|---|---|
| 10, high byte | seconds counter (seen during the defrost) | candidate |

### `0848`: heat demand and clock

| Word | Meaning | Status |
|---|---|---|
| 0 | Heat demand in degree-minutes (sign inverted) | verified |
| 1, 2 | Setpoint in brackets after TANK on DRIFTDATA (for example "TANK 26 (29)"); both 29 at the same time | likely (one reading) |
| 3, 4 | Unknown | unknown |
| 16 to 22 | Pump clock | verified |

### `0864`: status bits

Word 0: **3** (bits 0 and 1) during the first captured defrost, 0 otherwise. It was 15 to 22 s before word 1 bit 4 in the first defrost and at the same second as bit 4 in the second (26 Nov 18:12) and 22 s before it in the third (27 Nov 07:38), so its timing is not constant. Meaning of the individual bits not known. Candidate, see [defrost.md](defrost.md).

Word 1 (status bits):

| Bit | Meaning | Status |
|---|---|---|
| 0 | Heating active (building) | verified (live) |
| 1 | Hot water active | verified (live) |
| 2 | Hot water top-up (legionella) | never seen set |
| 3 | Pool heating | never seen set |
| 4 | **Defrost using building heat** | verified (three defrosts) |
| 5 | Defrost using hot water heat | never seen set |
| 6 | Outdoor stop (utestopp) | never seen set |
| 7 | Cooling | never seen set |
| 9 | Concrete drying | never seen set |

Word 2:

| Bit | Meaning | Status |
|---|---|---|
| 0 | Compressor start phase (64 s and 127 s in two observed runs) | verified (live) |
| 3 | Seen once, in the sample at the start of the first defrost (word 2 = 8) | candidate |

Word 3: 16 bits, all unknown. In every sample so far the value is `0x00F0` (bits 4 to 7 set, constant). Some of the bits may be the flow switch ("F" on the display), not proven.

### `0870`: operating hours and defrost counters

| Word | Meaning | Status |
|---|---|---|
| 11 | Defrost count (857, then 858 after the first captured defrost) | verified |
| 12 | Minutes between the last two defrosts. At a defrost the last value of word 13 is copied here (438 before the first captured defrost, **2377** after it, as 2377 was the last value of word 13) | verified (copy behaviour), candidate (meaning) |
| 13 | Minutes since the last defrost. Reset to 0 at a defrost, then about 1 per minute. It counts only while the compressor runs (it stood still at 143 after a pump restart) | verified (behaviour) |
| others | Operating hours, outdoor unit (7074 / 6575 / 486 read on 24 November) and others | candidate |

The display's AVFROSTNING menu shows the same counters (`857`, `7.3 h`, `38.3 h`, `7H18M`, `38H15M` in the morning of 25 November before the first defrost).

### `07E4`: communication kit and compressor flags

| Item | Meaning | Status |
|---|---|---|
| COMM.KIT and MAIN MICOM versions (words 4-7) | firmware versions | verified |
| Word 16 bit 2 / bit 1 | compressor start / running | verified (live) |
| Word 16 during the first defrost | 6 -> 10 (bit 2 off, bit 3 on) -> 2 -> 6 | candidate |
| Word 0 flag | 1 for a short time after the defrost (12:02:01 to 12:02:43), 0 otherwise in the samples | candidate |

### `07F8`, `0884`

`0884` holds the alarm history (see [alarms.md](alarms.md)). `07F8` was seen all zero in normal operation.

## Settings (configuration pages)

"Changed" means the value was changed one step on the display, the register was seen to follow, and the value was changed back.

| Display menu / value | Register | How verified |
|---|---|---|
| Heating curve (KURVA), MIN BÖR, MAX BÖR, KURVA +5 / 0 / -5, VÄRMESTOPP, SÄNKNINGSTEMP, RUM FAKTOR | `03E8` to `03F0`, in menu order | matched to display values |
| Room setpoint | `03F4` (whole degrees C) | changed |
| HÖG EFFEKT (on/off) | `03F3` (0/1) | changed |
| Operating mode (off, auto, compressor, additional heat, hot water) | `0553` | matched |
| VARMVATTEN: START, STOPP | `041D`, `0424` | matched / changed |
| VARMVATTEN: VARMVATTENTID, TOPPV.INTERVALL, TOPPV.STOPP | `041E`, `041F`, `0421` | changed |
| VARMVATTEN: INFL. STARTGIV. (stored /5) | `0422` | changed |
| VÄRME (service): START / STOPP in degree-minutes (displayed = -5 x stored) | `0431`, `0432` | changed |
| VÄRME (service): STARTTID, VÄRMETID | `0433`, `0434` | changed |
| TILLSATS: MAX STEG | `0456` | changed |
| TILLSATS: START (displayed = -15 x stored, so -75 is stored as 5) | `0457` | changed (three points: -75, -60, -90) |
| TILLSATS: STARTFÖRDRÖJNING | `0459` | changed |
| TILLSATS: FRYSSKYDD (0/1) | `0461` | changed |
| OPTIMUM flows, charge temperatures, circulation | `0492` to `049C` | matched |
| AVFROSTNING (service): KOMPR. STOPP, ENERGIPRIO.VV | `047E`, `047F` | read only; both 0 on my pump, the display shows an on/off symbol for both; not changed |

Unknown settings words: `03F1` = 40, `03F2` = 30, `03F5` = 2 on the `03E8` page; `0458` = 20 and `045B` = 6 on the TILLSATS page; a second block starting at `03FC` looks like another heating circuit; EQ / OPTIMUM / ZONER flags are in `0546`.

## Alarms, calendar and concrete drying (read-only)

**Alarm history** is in runtime page `0884` (60 words): 8 records of 7 words, `[word0, 0, minute, hour, day, month, year-2000]`, newest first, and the last four words are the start of a ninth record (its hour only). Display entry *n* in the `LARM` menu is data record *n*; date and hour match (a new record also holds the minute, which is lost from older records). Entries 9 and 10 on the display are not fully in the 60 words. Word 0 of a record is not the event type and its meaning is unknown. The active alarm page `085F` (5 words) is all zero when there is no alarm. Details, the comparison with the display and the open questions are in [alarms.md](alarms.md).

**Calendar** is in configuration pages `055A` to `06C5` (396 words): four calendars of 98 words (VARMV. BLOCK., EVU, LJUDLÄGE, TEMP. SÄNKNING), each with 8 period slots of 12 words and 2 status words. The display shows only the first slot. Period layout, verified by changing values on the display:

| Word | Meaning |
|---|---|
| 0 | TIDSFUNKTION: 0 = DATUM, 1 = DAGAR/VECKA |
| 1, 2 | start minute, start hour |
| 3, 4, 5 | start day, month, year - 2000 |
| 6, 7 | stop minute, stop hour |
| 8, 9, 10 | stop day, month, year - 2000 |
| 11 | weekday bitmask for DAGAR/VECKA (Monday = bit 0, Wednesday = bit 2 verified; the other days are assumed) |

The default is 31-DEC-2000 00:00 (stored as day 31, month 12, year 0). The second status word of each calendar (word 97 of the 98) is the "active now" flag, shown as a star on the display. It was seen to switch on during an active DAGAR/VECKA period and off again when the period ended.

**Concrete drying (BETONGTORKNING)** is in page `04D8` (27 words): start day, month, year - 2000, minute, hour, hysteresis, number of points, then 10 pairs of (last day, temperature). The display derives "DAG 2 - 3" from the previous pair's last day plus one. The on/off switch of the function has not been located.

## Installation data

Serial number is in `06F2/06F3`; COMM.KIT and MAIN MICOM versions are in `07E4`.
