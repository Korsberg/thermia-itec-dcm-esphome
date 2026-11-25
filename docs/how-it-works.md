# How it works, in plain words

This page explains what the project does without any technical terms. For the details, see [firmware.md](firmware.md) and [protocol-dcm.md](protocol-dcm.md).

## The short version

The ESP32 listens to the heat pump and passes what it hears on to Home Assistant. It never changes anything in the pump. It only reads.

## The story

Think of the pump as a guard at a door, and the ESP32 as a guest who wants to listen to what is said inside.

1. **The pump is the boss.** It decides everything and talks whenever it wants. The ESP32 never decides anything and never writes a setting to the pump.
2. **The pump wants to know who is listening.** When the pump starts, it asks once: "Who is there? Here is a code, show me that you can answer it." It asks only at start.
3. **The ESP32 works out the right answer.** It is like a password question. The ESP32 can calculate the answer and sends it. If the pump is happy, a small DCM symbol appears on the pump's main screen.
4. **The pump starts to talk.** First it sends all its settings, 31 "pages" like a booklet. After that it keeps sending the current values: temperatures, pressures, whether the compressor is running, and so on.
5. **The ESP32 says "received" after every page.** That is only a polite "OK, I heard that". It changes nothing.
6. **The ESP32 translates the numbers** into something you can read, such as "supply temperature 30 degrees", and sends them to Home Assistant. There you see them as sensors and graphs.

## Why does the pump have to be restarted?

The pump asks "who is there?" only when it starts. If the ESP32 was not ready then, or has restarted since, the pump never hears an answer and does not tell it anything. Restart the pump, and it asks again.

Two rules of thumb:

- After flashing or restarting the ESP32, wait about 30 seconds until the log shows `ARMED`, then restart the pump.
- Restarting the pump resets the degree-minutes and stops the compressor, so do it when no heating or hot water cycle is running.

## What you should see when it works

| Where | What |
|---|---|
| Pump's main screen | The DCM symbol |
| ESP32 log | `ARMED`, then `CONFIG_EXPORT_COMPLETE pages=31` about 30 to 75 seconds after the pump started |
| Home Assistant | Sensors fill with values within a minute or two and update about every 21 seconds |

## What can go wrong

Sometimes there is extra noise on the cable at the moment the ESP32 is about to say "received". It then waits and the pump asks again, which normally works. If it happens five times in a row, the ESP32 gives up to be safe. Then both the ESP32 and the pump have to be restarted. See [troubleshooting.md](troubleshooting.md).

## Words you may meet

| Word | Meaning |
|---|---|
| DCM | The gateway connection on the pump (an RJ45 port) that the ESP32 uses |
| Gateway | The thing the pump talks to over the DCM port. Here, the ESP32 |
| Challenge | The "who is there?" question the pump sends at start |
| Page | One block of data the pump sends, with an address such as `0820` |
| ACK | "Received". The ESP32's short reply after each page |
| Armed | The ESP32 is ready to answer a challenge |
| Resync | The ESP32 finds the start of a message again after noise. Normal and harmless |
| Candidate | A sensor that probably means what we think, but is not yet confirmed against the pump display |
