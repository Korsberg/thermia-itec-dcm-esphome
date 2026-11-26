# The first captured defrost

Captured on **25 November 2026**, local time (Swedish time, UTC+1), between 11:59:44 and 12:02:07. The defrost itself lasted about two minutes. Before this, logging had run for days without a defrost, so the flags were only candidates. No other defrost was seen that day (checked against all logs and the Home Assistant history).

Pump data arrives once per 21 s cycle, and each page is seen once per cycle, so all times below are accurate to about one cycle.

## Timeline

| Time | What |
|---|---|
| 11:59:44 | `0864` word 0 goes from 0 to **3**, word 2 gets bit 3 (value 8) |
| 11:59:47 | `0870` defrost count goes from **857 to 858** |
| 11:59:59 | `082F` (word 15 of `0820`) **bit 2** set. Compressor 31 to 20 Hz, expansion valve 176 to 1000 steps, high pressure 21.2 to 16.9 bar |
| 12:00:06 | `0864` word 1 **bit 4** set (defrost using building heat); word 1 = `0x11` |
| 12:00:21 to 12:00:42 | High pressure falls to 10 bar, expansion valve 2000 steps (100 %), fan 550 to 180 to 0 rpm |
| 12:01:03 to 12:01:25 | Fan stopped. Compressor up to 50 Hz, the outdoor unit temperature word (`0820` word 5) rises 3, 5, 11, 15 degrees C |
| 12:01:25 | `082F` bit 2 last seen set |
| 12:01:31 | `0864` bit 4 last seen set |
| 12:01:46 to 12:02:07 | Fan back (510, then 550 rpm), expansion valve 481 steps, normal running |
| 12:01:52 | `0864` word 0 back to 0 |

## Values during the defrost (`0820` unless noted)

| Value | Before | During | After |
|---|---|---|---|
| Compressor | 31 Hz | 20 Hz, 28 Hz, 50 Hz, 23 Hz | 50 Hz, then 20 to 25 Hz |
| Expansion valve | 176 steps | 1000, then 2000 (100 %) | 481, then about 175 |
| High pressure | 21.2 bar | 16.9, 14.9, 10.3, 10.1 bar | 17.3 to 19.3 bar |
| Fan | 550 rpm | 180, then 0 rpm | 510, then 550 rpm |
| Current | 2.1 A | 0.5 to 1.9 A | 0.9 to 3.0 A |
| Supply temperature (`07D0` word 0) | 30 | down to 23 | back to 30 |
| Return temperature (`07D0` word 2) | 27 | 27 | 27 |

The supply temperature fell by about 7 degrees while the pump defrosted, which fits "defrost using building heat": the heat for the outdoor unit is taken from the heating circuit. The return temperature did not change.

Heat demand (`0848` word 0) counted up by one per cycle before the defrost (33 to 38) and stood still at 38 from 12:00:04 on.

## Counters (`0870`)

| Time | Count (word 11) | Between (word 12) | Since (word 13) |
|---|---|---|---|
| 11:59:25 | 857 | 438 | 2377 |
| 11:59:47 | 858 | **2377** | **0** |
| 12:06:53 | 858 | 2377 | 6 |

At a defrost the last value of word 13 (2377 minutes, 39 h 37 min) is copied to word 12 and word 13 restarts from 0. Word 13 counts about one per minute, but only while the compressor runs: after a pump restart at 15:26 it stood still at 143.

## What this means

- `0864` word 1 bit 4 = **defrost using building heat**: verified.
- `0820` word 15 (`082F`) bit 2 = defrost flag in the outdoor unit data: verified. It was set 7 s before bit 4 and released 6 s before it, so it is the earlier signal of the two.
- `0870` word 11 = defrost count: verified. Words 12 and 13: behaviour verified as described above.
- `0864` word 0 (value 3, bits not decoded) changed together with bit 4 in the second defrost, and 15 to 22 s before it in the first. Candidate: it belongs to the defrost, but its timing against bit 4 is not constant.
- `0864` word 2 bit 3 and `07E4` word 16 (6 to 10 to 2 to 6) changed around the defrost. Candidates.
- `0864` word 1 bit 5 (defrost using hot water heat) has not been seen. It may need a hot water cycle that was not running.
- The service menu AVFROSTNING items KOMPR. STOPP and ENERGIPRIO.VV (`047E`, `047F`) are both 0 on this pump.
- The reversing valve cannot be read directly from the pages seen so far. That a valve operates follows from the pressure, temperature and fan changes above, and is not proven.

## Second defrost (26 November 2026, 18:12 to 18:17)

Seen in the log and in the flow card, which showed "avfrostar" (outdoor -4 °C, expansion valve 100 %, supply and return equal, 0.3 A).

| Signal | On | Off | Length |
|---|---|---|---|
| `0864` word 1 bit 4 | 18:12:53 | 18:17:35 | 4 min 42 s |
| `082F` bit 2 | 18:12:47 | 18:17:28 | 4 min 41 s |
| `0864` word 0 (`0000` to `0003`) | 18:12:53 | 18:17:35 | same as bit 4 |

`082F` bit 2 came 6 s before bit 4 both at the start and at the end, as in the first defrost, so it is confirmed by a second defrost. Word 0 of `0864` followed bit 4 to the second this time (see above).

For an animation or an automation, the safest trigger is bit 4 or `082F` bit 2, whichever comes first.

## What is still wanted

A photo of the pump's INFO UTEDEL display during a defrost, to compare the display values with the words above.
