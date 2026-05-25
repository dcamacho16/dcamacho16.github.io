---
layout: page
title: Mini-HVAC IoT System
description: "ESP32 + Simulink Hardware-in-the-Loop Platform"
img: assets/img/projects/sdp-minihvac-hero.png
importance: 1
category: embedded
---

A hardware-in-the-loop mini-HVAC platform built as the capstone project for Sistemas Digitales Programables at UCAM (Universidad Católica de Murcia). The system pairs a five-subsystem Simulink plant model with an ESP32-WROVER running custom FreeRTOS firmware, enabling closed-loop thermal control where the simulated plant and the physical controller exchange data in real time over a binary serial protocol.

The serial interface is built around compact packed structs — a 24-byte `PcToEsp` command frame and a 32-byte `EspToPc` telemetry frame — validated byte-by-byte with MATLAB scripts and enforced at compile time via `static_assert` size checks, ensuring wire-format consistency between the C firmware and the MATLAB/Simulink host.

The FreeRTOS firmware is organized into three cooperative tasks: **TaskTX** transmits sensor readings and controller state at a fixed cadence, **TaskRX** parses incoming plant commands and updates setpoints, and **TaskLCD** drives the I²C display. All three tasks share state through a mutex-protected global struct, keeping synchronization simple and deterministic.

The physical platform is mounted on a custom wooden board and includes a DHT22 temperature/humidity sensor, a PWM-controlled fan, a 16×2 I²C LCD, potentiometers for manual setpoint adjustment, and debounced pushbuttons for mode selection.

The project is accompanied by 11-chapter ReadTheDocs-style documentation written entirely in English, covering everything from plant modeling and controller design to the serial protocol specification and macOS development environment setup.

<div class="row justify-content-sm-center">
    <div class="col-sm-10 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/sdp-minihvac-prototype.jpeg" title="Finished Mini-HVAC prototype mounted on custom board" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The finished prototype: ESP32-WROVER, DHT22 sensor, PWM fan, I²C LCD, potentiometers, and pushbuttons mounted on a custom wooden board.
</div>

<a href="https://github.com/dcamacho16/sdp-minihvac" target="_blank" rel="noopener noreferrer">View on GitHub →</a>
