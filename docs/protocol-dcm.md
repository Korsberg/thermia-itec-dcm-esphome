# The DCM gateway protocol (as observed)

Everything here was observed on one installation (iTec XT10, display firmware CM930 2.4.0). It is a description of what the pump does on the bus, not a vendor specification. Nothing in this project writes to the pump.

## Bus

| Item | Value |
|---|---|
| Physical layer | RS485 on the pump's gateway (DCM) RJ45 port |
| Framing | Modbus RTU, 9600 baud, 8E1 |
| Master | the pump |
| Gateway slave address | `0x0F` |

Addresses in this documentation are Modbus register addresses in hex. For configuration pages the word index in the exported data is `address - 1000` (decimal), for example the page starting at `03E8` (1000) begins at word 0.

## A passive reader sees very little

The pump exports its data pages only to a gateway that has answered its challenge. A passive reader that only listens on the bus never answers, so the pump never exports its pages to it. That is why this project acts as an active gateway.

## Challenge and handshake

The pump sends a challenge to gateway address `0x0F` **only when the pump itself starts**: a Modbus function 17 (report slave ID) frame carrying 16 challenge bytes.

1. The ESP calculates the 16-byte answer with the CM930 handshake calculation (a block-cipher style routine, see Credits in the README) and checks it against a known test vector at every boot.
2. It answers once per boot, and only if exactly one clean frame was received and nobody else is talking on the bus.
3. When the pump accepts the answer, the DCM symbol appears on the pump's main screen and the pump starts the export.

Consequences:

- After flashing or rebooting the ESP, the pump has to be restarted to start a new session. A pump restart while the ESP is off or not yet armed is lost.
- The ESP arms itself about 30 s after boot, so it is ready before a pump restart that follows boot.

## Export, runtime ring and mailbox

After the handshake the pump sends three kinds of traffic. All acknowledgements from the ESP are transport-level only (`NO PARAMETER WRITE`).

**1. Configuration export.** 31 configuration pages (function 16 frames). The ESP ACKs each page. The export takes about 75 s from the challenge. The last page is `06F4`; an optional alarm page `085F` may follow.

**2. Runtime ring.** Ten runtime pages are sent in a fixed order, one about every 2 s, so a full cycle takes about 21 s:

`07D0`, `07E4`, `07F8`, `080C`, `0820`, `0834`, `0848`, `0864`, `0870`, `0884`

Each is ACKed. These pages carry **all** live values. The mailbox and the configuration pages carry no live values.

**3. Mailbox poll.** Every few seconds the pump polls address `0708` (a read of 6 words). The ESP answers with a stateful reply that also requests the alarm page `085F` and the watched pages.

**Re-sent configuration pages.** When a setting is changed on the display, the pump re-sends the affected configuration page (for example `03E8`, 14 words) and expects an ACK. The ESP ACKs it, stores the new words and logs the raw page, so the sensors follow the display within seconds. Pages of this kind are recognised by address and length.

Only dynamic data (the ten runtime pages, plus occasional single reads such as `085F` and `04A6`) is live. Everything else is a snapshot of settings.

## Session handling

| Rule | Behaviour |
|---|---|
| Isolation gate | Before ACKing a runtime page the ESP checks that the page arrived alone: no stray bytes pending in the UART, no dropped or ambiguous frames. A failed gate skips that one ACK (the pump re-sends the page) and the session is kept. |
| Gate failure limit | After 5 failed gates in a row the session is stopped (`GATE_FAIL_LIMIT`). |
| Activity timeout | If no valid function 16 frame or `0708` poll arrives for 30 s after the export is complete, the session is stopped. |
| Watchdog | If the session is lost for 90 s the ESP reboots (`SESSION_LOST`). A new session needs a pump restart. |
| Caps | The old caps on runtime ACKs (20 000) and mailbox replies (30 000) were removed in v40b. Both counters are now 32-bit diagnostic counters. |

A real communication failure is therefore the only thing that ends a session.

## Resync

The receiver sometimes has to resynchronise on a frame header (`RESYNC header byte=00 next=02 pending=28` in the log). Over long runs this is about 6-10 % of frames. There were 0 dropped frames and 0 CRC errors in the long runs. Changing the cable from single wires to doubled conductors (A, B and ground) made no measurable difference: about 9-10 % before and after (10 416 frames over about 25 minutes after the change, 25 453 of 255 052 frames over a night before it).

## Open questions

- Which 5 bytes were pending in the UART when the gate failed on page `0820` (`uart=5`, `pending=0`, `drops=0`) on 25 November at 10:04. The log shows the count but not the content; later firmware will log the bytes.
- Whether and how the pump reports a change in the other direction (gateway to pump). Not researched; nothing is sent today.
