---
layout: post
title: "How to make Simulink and an ESP32 speak the same binary language"
date: 2026-05-26 12:00:00 +0200
description: Designing a binary serial protocol between a Simulink plant model and ESP32 FreeRTOS firmware — matching C structs to Byte Pack/Unpack blocks and validating everything in MATLAB before the hardware is even connected.
tags: embedded simulink esp32 serial-protocol freertos matlab
categories: engineering
giscus_comments: false
related_posts: false
thumbnail: assets/img/projects/sdp-minihvac-hero.png
toc:
  beginning: true
---

{% include figure.liquid loading="eager" path="assets/img/projects/sdp-minihvac-hero.png" class="img-fluid rounded z-depth-1" zoomable=true %}

If you've ever tried to exchange structured data between Simulink and a microcontroller, you've probably hit the same wall I did: **Simulink wants to send and receive tidy signal vectors, while your firmware thinks in C structs and raw bytes.** ASCII-based protocols like printing comma-separated values over serial are tempting, but they're slow, fragile, and a nightmare to parse at 115200 baud when your control loop needs deterministic timing.

This post walks through the binary serial protocol I built for [Mini-HVAC](https://github.com/dcamacho16/sdp-minihvac), an IoT Hardware-in-the-Loop platform that pairs a five-subsystem Simulink model with an ESP32-WROVER running custom FreeRTOS firmware. The protocol is simple, but getting it right required thinking carefully about struct layout, byte ordering, and how to prove correctness _before_ plugging in a single wire.

## The problem

The Mini-HVAC system needs two-way real-time communication between a PC running Simulink and an ESP32 microcontroller:

- **PC to ESP32**: simulated plant state (temperature, humidity, pressure) plus actuator commands (fan duty, heater duty, alarm flag).
- **ESP32 to PC**: real sensor readings, user-selected setpoints, operating mode (AUTO/MANUAL), and HVAC enable flag.

An ASCII protocol — something like `"25.0,50.0,0.5,0.3,0.1,1\n"` — would work for a demo, but it has real problems in a control loop:

1. **Variable-length messages** make framing unreliable. A dropped byte shifts every field.
2. **Float-to-string-to-float round trips** lose precision and burn CPU cycles on both ends.
3. **Parsing overhead** on the ESP32 side (`sscanf`, `strtok`, etc.) is surprisingly expensive when you're also running FreeRTOS tasks for sensors, actuators, and an LCD.

Binary structs solve all three. Fixed-length packets, zero conversion loss, and reception is just `Serial.readBytes` into a struct.

## The solution: matched C structs and Byte Pack/Unpack

The protocol is built around two packed C structs — one for each direction:

```c
// PC (Simulink) → ESP32: 24 bytes
struct PcToEsp {
  float T_sim;         // simulated temperature
  float RH_sim;        // simulated humidity
  float P_sim;         // simulated pressure/flow
  float u_fan_sim;     // fan duty cycle [0, 1]
  float u_heater_sim;  // heater duty cycle [0, 1]
  float alarm_flag;    // alarm active (0.0 or 1.0)
};  // 6 × 4 = 24 bytes

// ESP32 → PC (Simulink): 32 bytes
struct EspToPc {
  float T_real;            // measured temperature
  float RH_real;           // measured humidity
  float u_fan_real;        // actual fan state
  float u_heater_real;     // actual heater state
  float T_set;             // temperature setpoint
  float RH_set;            // humidity setpoint
  float mode_auto_manual;  // 0.0 = MANUAL, 1.0 = AUTO
  float hvac_enable;       // 0.0 = OFF, 1.0 = ON
};  // 8 × 4 = 32 bytes
```

Every field is a `float` (IEEE 754 single-precision, 4 bytes). This is a deliberate choice: Simulink's `single` type and C's `float` use the same representation, so there's no conversion step — just a `memcpy`-equivalent in both directions.

### On the ESP32 side

Sending is a one-liner. The FreeRTOS `TaskTX` task reads sensors, updates the state struct, and writes it directly to the UART:

```cpp
Serial.write((uint8_t *)&toSend, sizeof(toSend));  // 32 raw bytes
```

Receiving is equally simple. `TaskRX` waits for exactly 24 bytes, then casts them straight into the command struct:

```cpp
if (Serial.available() >= (int)sizeof(PcToEsp)) {
  Serial.readBytes((uint8_t *)&localCmd, sizeof(PcToEsp));
  safeUpdatePcCmd(localCmd);
  applyPCCommands(localCmd);
}
```

No parsing. No delimiters. No state machine. The struct _is_ the wire format.

### On the Simulink side

Simulink mirrors this with its **Byte Pack** and **Byte Unpack** blocks (from the Simulink Real-Time / Utility Blocks library):

- **`TX_To_ESP32` subsystem**: takes 6 `single` signals, feeds them into a Byte Pack block configured for 6 inputs of type `single`, and outputs a `uint8[24]` vector. The signal connection order must _exactly_ match the field order of `PcToEsp`.

- **`RX_From_ESP32` subsystem**: takes a `uint8[32]` vector from the Serial Receive block, feeds it into a Byte Unpack block configured for 8 outputs of type `single`, and produces the 8 signals that map back to `EspToPc` fields.

The critical configuration on the Simulink side:

- **Data type**: `single` (not `double` — Simulink defaults to `double` internally, so you need explicit Data Type Conversion blocks).
- **Byte order**: `little-endian` (matches the ESP32's Xtensa architecture).
- **Alignment**: 1 byte (no padding between fields).

## The gotchas

### Endianness

Both the ESP32 (Xtensa LX6) and x86/ARM PCs are **little-endian**, so the bytes land in the same order on both sides. If you're targeting a big-endian platform, you'd need to byte-swap each float — but for the ESP32 + modern PC combination, this is a non-issue. Still, it's worth verifying explicitly: the Serial Configuration block in Simulink has a byte-order setting, and it _must_ be set to `little-endian`.

### Struct padding

C compilers are free to insert padding bytes between struct fields for alignment. A struct with a `uint8_t` followed by a `float` might silently grow from 5 to 8 bytes. By using _only_ `float` fields (all 4 bytes, naturally aligned), padding never kicks in. The struct size is always `N × 4` bytes, exactly matching what Byte Pack/Unpack expects.

If your protocol needs mixed types (say, a `uint8_t` flag alongside floats), you'd need `__attribute__((packed))` on the C side — but then you pay a performance penalty on some architectures. The all-float approach avoids this entirely.

### Compile-time size checks with `static_assert`

Even with an all-float layout, a typo (an extra field, a wrong type) could silently change the struct size and break the protocol. The firmware guards against this with compile-time assertions:

```cpp
static_assert(sizeof(PcToEsp) == 24, "PcToEsp must be 24 bytes");
static_assert(sizeof(EspToPc) == 32, "EspToPc must be 32 bytes");
```

If someone adds a field and forgets to update the Simulink side, the build fails immediately instead of producing a subtle runtime bug where fields shift by 4 bytes and the fan duty cycle ends up in the temperature slot.

### Field ordering

The most insidious bug in binary protocols is a field-order mismatch. The bytes are _correct_ — the same floats, the same endianness, the same total size — but field 3 on one side is field 4 on the other. Nothing will catch this at compile time.

The defense is a single source of truth. In this project, the C struct definition in the firmware _is_ the spec. The Simulink subsystem wires its signals in the same order, and the MATLAB validation scripts (next section) verify byte-for-byte equivalence against that order.

## The validation: proving correctness without hardware

Here's the part I found most valuable in practice. Before the ESP32 was even connected, two MATLAB scripts validated that Simulink and the firmware would agree on every byte.

### `TEST_TX_To_ESP32_bytes.m` — validating the transmit path

This script builds the expected `uint8[24]` byte vector for a known set of `PcToEsp` values:

```matlab
% Define test values matching the C struct field order
T_sim = 25.0; RH_sim = 50.0; P_sim = 0.5;
u_fan_sim = 0.3; u_heater_sim = 0.1; alarm_flag = 1.0;

% Pack as single-precision floats, then reinterpret as bytes
vals = single([T_sim, RH_sim, P_sim, u_fan_sim, u_heater_sim, alarm_flag]);
expected_bytes = typecast(vals, 'uint8');  % uint8[24]
```

After running the Simulink model, the `tx_bytes` output from the `TX_To_ESP32` subsystem is compared against this reference:

```matlab
isequal(expected_bytes, out.tx_bytes)  % must return 1 (true)
```

If there's a mismatch, the script reports _which byte_ differs — making it trivial to trace back to a wiring or type-conversion error in the Simulink model.

### `TEST_RX_From_ESP32_bytes.m` — validating the receive path

The reverse direction: generate the 32 bytes that an ESP32 _would_ send, inject them into the `RX_From_ESP32` subsystem via a Constant block, and verify that the unpacked signals match the original values.

```matlab
% Build the EspToPc byte vector
vals = single([T_real, RH_real, u_fan_real, u_heater_real, ...
               T_set, RH_set, mode_auto_manual, hvac_enable]);
esp_bytes = typecast(vals, 'uint8');  % uint8[32]

% Verify round-trip: bytes → singles → check against originals
vals_recovered = typecast(esp_bytes, 'single');
```

Both scripts also perform a **round-trip check**: bytes back to floats, compared against the originals. This catches subtle issues like accidentally using `double` instead of `single` somewhere in the chain (which would produce 48 or 64 bytes instead of 24 or 32).

### Why this matters

This validation strategy means that by the time the ESP32 is physically connected, the _only_ things that can go wrong are hardware-level issues: a loose wire, a wrong baud rate, a task-scheduling conflict. The data packing and unpacking layer has already been proven correct. In practice, this saved significant debugging time — the first hardware test worked on the first try.

## Putting it all together

The full data flow looks like this:

```
┌─────────────────────────────────────────────────────┐
│                  PC (Simulink)                       │
│                                                      │
│  MiniHVAC_Zone ──► TX_To_ESP32 ──► Serial Send      │
│  (plant model)     (Byte Pack)     (uint8[24])       │
│                                        │             │
│  Scopes/Control ◄── RX_From_ESP32 ◄── Serial Recv   │
│                     (Byte Unpack)     (uint8[32])    │
│                                        │             │
└────────────────────────────────────────┼─────────────┘
                                    USB Serial
                                    115200 baud
┌────────────────────────────────────────┼─────────────┐
│                  ESP32-WROVER          │             │
│                                        │             │
│  TaskRX ──────► applyPCCommands()      │             │
│  (24 bytes in)   (fan PWM, heater,     │             │
│                   alarm LED)           │             │
│                                        │             │
│  TaskTX ◄────── updateSensorsAndInputs()            │
│  (32 bytes out)  (DHT22, pots, buttons)             │
│                                                      │
│  TaskLCD ──────► 16×2 I²C display                   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

Three FreeRTOS tasks share the `PcToEsp` and `EspToPc` structs through mutex-protected accessors (`safeUpdatePcCmd`, `safeReadEspState`, etc.), keeping synchronization simple and deterministic.

## Key takeaways

1. **Use all-float structs** to avoid padding issues. If you need boolean flags, encode them as `0.0`/`1.0` floats — it's 3 extra bytes per flag, but it eliminates an entire class of alignment bugs.

2. **`static_assert` the struct sizes** in firmware. It's one line of code that catches protocol-breaking changes at compile time.

3. **Validate byte-for-byte in MATLAB** before touching hardware. MATLAB's `typecast(single(...), 'uint8')` gives you the exact wire representation. Compare it against Simulink's output and the firmware's expected input.

4. **Set Simulink's byte order explicitly** to `little-endian`. Don't rely on defaults — the Serial Configuration block's byte-order setting is easy to miss.

5. **Keep one source of truth** for field ordering. The C struct definition is the spec; everything else (Simulink wiring, MATLAB test vectors) derives from it.

The full project — firmware, Simulink models, MATLAB validation scripts, schematics, and 11 chapters of documentation — is on GitHub: [dcamacho16/sdp-minihvac](https://github.com/dcamacho16/sdp-minihvac).
