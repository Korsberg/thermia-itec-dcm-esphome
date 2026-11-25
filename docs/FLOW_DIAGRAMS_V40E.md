# Thermia iTec DCM gateway – firmware flows in V40e

This guide explains how the V40e ESPHome gateway receives information from a Thermia heat pump and exposes it in Home Assistant. The diagrams are simplified descriptions of this firmware, not an official Thermia protocol specification.

## From heat pump to Home Assistant

```mermaid
flowchart LR
    A["Thermia heat pump<br/>DCM bus"] --> B["RS485 module"]
    B --> C["ESP32<br/>UART receiver"]
    C --> D["Frame parser<br/>and CRC validation"]
    D --> E["Session and page handling"]
    E --> F["Cached raw values"]
    F --> G["ESPHome sensors<br/>interpret and scale"]
    G --> H["ESPHome API<br/>Wi-Fi"]
    H --> I["Home Assistant<br/>entities and dashboard"]
    E -->|"responses and ACKs"| B
```

The ESP32 receives pump data and sends protocol responses on the bus. V40e does not write heat pump settings. An ACK is a transport acknowledgement; it does not change a setpoint.

## Startup and configuration

```mermaid
flowchart TD
    A["ESP32 starts and listens"] --> B{"Automatic arming<br/>conditions met?"}
    B -->|No| A
    B -->|Yes| C["Armed: wait for Thermia challenge"]
    C --> D{"Valid challenge and<br/>safety checks pass?"}
    D -->|No| C
    D -->|Yes| E["Send challenge response"]
    E --> F["Pump sends configuration pages"]
    F --> G{"Expected page and<br/>receive checks pass?"}
    G -->|Yes| H["Cache words and send transport ACK"]
    H --> I{"31 pages complete?"}
    I -->|No| F
    I -->|Yes| J["Configuration complete:<br/>runtime monitoring"]
    G -->|No| K["Skip ACK or stop session,<br/>depending on the cause"]
```

Automatic arming requires sufficient observed bus traffic and no blocking condition in the gateway's safety checks. The pump must then send a challenge. If the ESP32 restarts while the pump is already running, the pump may need a restart to issue a new challenge.

The 31 expected configuration pages contain heating and hot water settings, advanced menus and calendar data. During a session the pump can also resend a known configuration page; when the checks pass, the cached values are updated.

## Follow the DCM connection in the log

```mermaid
flowchart TD
    A["CHALLENGE_RESPONSE_SENT<br/>ESP32 sends the challenge response"] --> B["CONFIG_PAGE_ACK_SENT<br/>configuration pages received and acknowledged"]
    B --> C["CONFIG_EXPORT_COMPLETE<br/>31 pages complete"]
    C --> D["MAILBOX_0708_STATEFUL_REPLY<br/>gateway responds to the mailbox"]
    D --> E["RUNTIME_PAGE_ACK_SENT<br/>runtime pages received and acknowledged"]
    E --> F["Sensor values published<br/>and displayed in Home Assistant"]
```

`CHALLENGE_RESPONSE_SENT` shows that the ESP32 sent its response. Completion of the configuration export followed by ongoing runtime data provides stronger evidence that the DCM session is working. This diagram shows the main sequence; other log entries and pages can appear between the steps.

## Runtime pages and sensors

```mermaid
flowchart TD
    A["Pump sends a page"] --> B{"Known address, expected length<br/>and valid CRC?"}
    B -->|No| C["Observe or stop session,<br/>depending on context"]
    B -->|Yes| D{"Receive isolation<br/>checks pass?"}
    D -->|No| E["Skip ACK and log gate failure"]
    D -->|Yes| F["Send transport ACK<br/>and cache page words"]
    F --> G["Sensor reads a word or bit"]
    G --> H["Apply sign, scaling and unit"]
    H --> I["Publish through ESPHome<br/>to Home Assistant"]
```

Examples include `07D0` for hydraulic temperatures, `080C` for room temperature, `0820` for outdoor unit data, `0848` for values including heat demand, `0870` for operating hours and defrost information, and `0884` for alarm history. Some fields remain candidates and need comparison with the pump display.

The `0708` mailbox is used to respond to controller polls. The gateway also requests page `0820` for selected sensor monitoring approximately 60 seconds after the preceding request cycle completes. Other known runtime pages are received and acknowledged when the pump sends them.

Many runtime sensors become unavailable if their page has not been updated for five minutes. Configuration values can remain visible after their initial read; the configuration data age sensor shows how old the cached page is.

## Inactivity and a new challenge

```mermaid
flowchart TD
    A["Established session"] --> B{"No successful session<br/>activity for 30 seconds?"}
    B -->|No| A
    B -->|Yes| C["SESSION_WAITING<br/>session retained"]
    C --> D{"New communication<br/>or challenge?"}
    D -->|"Successful page or mailbox exchange"| E["SESSION_RESUMED"]
    E --> A
    D -->|"Clean new challenge after waiting"| F{"Checks pass and fewer<br/>than three rechallenges?"}
    F -->|Yes| G["Reset session data<br/>and send challenge response"]
    G --> H["Wait for a new configuration export"]
    F -->|No| C
    D -->|"Still inactive"| I["After 90 seconds:<br/>SESSION_WAITING_LONG"]
    I --> C
```

During inactivity V40e retains the session. If a new challenge passes the checks, several session fields are reset and cached sensor data is marked for a fresh read. Up to three rechallenges are allowed per ESP32 boot. Other protection conditions can stop the session, in which case this recovery path does not automatically rearm it.

The rechallenge path has been exercised in a host test with simulated traffic. Recovery after a real heat pump restart still needs verification on the installation. A challenge arriving before the waiting state is active is not answered by the rechallenge rule.

## Receive gate and diagnostics

```mermaid
flowchart TD
    A["Known FC16 page received"] --> B{"Receive isolation<br/>checks pass?"}
    B -->|Yes| C["Reset gate failure streak<br/>page can be acknowledged"]
    B -->|No| D["RX_GATE_FAIL<br/>skip ACK"]
    D --> E["V40e logs up to 24 extra UART bytes"]
    E --> F{"Five consecutive<br/>gate failures?"}
    F -->|No| G["GATE_FAIL_SKIP<br/>retain session"]
    F -->|Yes| H["GATE_FAIL_LIMIT<br/>stop session"]
```

Before an FC16 response, the gateway checks for conditions including additional frames, incomplete parser data and bytes waiting in UART. V40e reads the extra UART bytes that it logs. This can consume bytes that might otherwise form part of a later frame and should be considered when interpreting the log. The mailbox has separate gate handling.

| Log or sensor | Meaning |
| --- | --- |
| `CONFIG_EXPORT_COMPLETE` | The 31 configuration pages have been processed. |
| `RUNTIME_PAGE_ACK_SENT` | A transport ACK was sent for a known runtime page. |
| `STATUS` | Session state, page counts and pending requests. |
| `RX_GATE_FAIL` / `GATE_EXTRA_BYTES` | Why an FC16 ACK was skipped and any extra captured UART bytes. |
| `SESSION_WAITING` / `SESSION_WAITING_LONG` | No successful session activity for approximately 30 or 90 seconds. |
| `RECHALLENGE_RESPONSE_SENT` | A response to a new challenge was sent; check that configuration export follows. |
| `DCM RX dropped frames` | The parser's counter for certain buffer overflows; it does not count all possible missed messages. |
| `Resync Share` | Parser resynchronization steps divided by valid frames. It is not a percentage of missed readings. |

## Alarms and current limits

Page `085F` still has a limit of 16 alarm ACKs per ESP32 boot, plus a one-time exception for an identical repeat. After the limit is reached, repeated offers of the page can be left without an ACK. Check the age of the last alarm observation: an old value does not prove the current alarm state.

Alarm history comes from `0884`. Calendar text displays the first period of each calendar. Some fields and time formats remain under verification against the pump menus.

The Wi-Fi configuration can restart the ESP32 after 30 minutes without a connection. After an ESP32 restart, the gateway may need a new challenge from the pump before receiving data again.
