# The first captured defrost

Captured on **25 November 2026**, local time (Swedish time, UTC+1), between 11:59:44 and 12:01:52. It lasted about two minutes. Before this, logging had run for days without a defrost, so the flags were only candidates.

## What changed on the bus

| Time | What |
|---|---|
| 11:59:44 | Start of the event in the log |
| 11:59:47 | `0870` defrost count goes from **857 to 858** |
| 11:59:59 to 12:01:46 | `082F` (word 15 of `0820`) **bit 2** set |
| 12:00:06 to 12:01:52 | `0864` word 1 **bit 4** set (defrost using building heat) |
| about 12:01:52 | Defrost ends; values return to the running state |

During the defrost, in the `0820` page:

- Expansion valve steps went to 1000, then to 2000 (100 %).
- Fan speed fell from 550 rpm to 0.
- High pressure fell from about 21 bar to about 10 bar.

In page `0834`, the high byte of word 10 counted seconds during the event.

The defrost counter changed at 11:59:47, `082F` bit 2 was first seen at 11:59:59 and `0864` bit 4 at 12:00:06. The pages arrive in a ring about every 21 s, and each page is seen once per cycle, so the start and stop times taken from the flags are only accurate to about one cycle.

## What this means

- `0864` word 1 bit 4 = **defrost using building heat**: verified.
- `0820` word 15 (`082F`) bit 2 = defrost flag in the outdoor unit data: verified.
- `0870` word 11 = defrost count: verified.
- `0864` word 1 bit 5 (defrost using hot water heat) has not been seen. It may need a hot water cycle that was not running.
- The "minutes since defrost" counter (`0870` word 13) appears to count compressor running minutes rather than clock minutes; this is a candidate, not proven.
- The service menu AVFROSTNING items KOMPR. STOPP and ENERGIPRIO.VV (`047E`, `047F`) are both 0 on this pump.

## What is still wanted

A photo of the pump's INFO UTEDEL display during a defrost, to compare the display values with the words above.
