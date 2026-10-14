# Klondike Connect 4 — Electronics

<img width="2160" height="932" alt="Front" src="https://github.com/user-attachments/assets/50a1a8ac-5ce3-4922-87dc-dc529ef0245e" />

Electronics and firmware for the semi-automatic giant Connect 4 board built by EPICS at Purdue (Fall 2026) for Klondike Elementary School. A motorized rack-and-pinion slider releases the game pieces at the end of a round, and addressable LEDs plus a Wi-Fi web interface add interactive control.

## Features

- **Automatic reset:** a stepper-driven slider under the board lines up its holes with the columns so the pieces drop out
- **Wireless control:** the ESP32 hosts a local Wi-Fi access point and a web control page
- **Addressable RGBW LEDs:** two independent LED channels driven in real time by the STM32
- **Safety:** endstop and motor-stall hardware interrupts stop the slider immediately

## System Overview

```
AC/DC (Mean Well) ──+24V──► TMC2209 ──4-wire──► NEMA17 stepper (rack & pinion)
        │
        └──+5V──► LED strips (A, B)
                └─► Buck converter ──+3V3──► STM32G431 ◄──UART──► ESP32-C3 ──Wi-Fi──► Web control app
                                                 ▲
                                    Endstops A/B, TMC DIAG (interrupts)
```

## Hardware

| Block | Part | Role |
|---|---|---|
| Main controller | STM32G431CBT6 (LQFP48) | LED rendering, motor control, interrupts |
| Wireless link | ESP32-C3 module | Wi-Fi access point, web UI, remote updates of the STM32 firmware |
| Motor driver | Trinamic TMC2209 | Quiet stepper drive (STEP/DIR + UART), stall detection |
| Motor | NEMA17 stepper | Drives the slider through a rack and pinion |
| Logic power | Buck converter, 5V → 3V3 | Powers STM32 and ESP32 |
| Power supplies | Mean Well AC/DC (+5V LED/logic, +24V motor) | Pre-certified, protected mains conversion |
| Homing | 2× microswitch endstops | Double-endstop slider limits |
| LEDs | Adafruit NeoPixel RGBW strips | Board lighting |

### Connectors

| Connector | Description |
|---|---|
| J1 — +5V input | Main power for LEDs and logic; fused (6A) with TVS protection |
| J2 — +24V input | Motor power only |
| J3 / J4 — EndA / EndB | Slider endstops |
| J5 / J6 — LEDA / LEDB | 3-pin LED outputs (Data, +5V, GND), each fused at 2A |
| J7 — MOTOR | 4-wire stepper (two coil pairs) |
| USB-C | Flash the ESP32; ESD protected |
| TC2030 | SWD programming for the STM32 |

### STM32 Pin Map

> Taken from the CubeMX pinout; verify against the KiCad schematic before writing firmware.

| Signal | Pin |
|---|---|
| LEDA data | PA8 |
| LEDB data | PA9 |
| EndA / EndB | PB13 / PB14 |
| InterA / InterB | PB11 / PB12 |
| TMC_STEP / TMC_DIR | PA5 / PA6 |
| TMC_EN | PA4 |
| TMC_DIAG | PA3 |
| TMC_PDN (UART) | PA2 |
| ESP32 UART | PB6 / PB7 |
| SWDIO / SWCLK | PA13 / PA14 |

## Firmware

| Component | Choice | Why |
|---|---|---|
| MCU | STM32 + ESP32 | Best score for documentation, performance, reliability |
| RTOS | Zephyr | Documentation and maintainability |
| Homing | Double endstop | Reliability |

### LED Protocol

- 800 kHz, 1.25 µs per bit
- `0` = 0.3 µs high / 0.9 µs low; `1` = 0.6 µs high / 0.6 µs low
- 32 bits per LED (GRBW, MSB first) = 40 µs per LED
- Hold low ≥ 80 µs after the last LED to latch
- Each LED takes its own packet and passes the rest down the chain

### Interrupts

- **Endstops:** fire when the slider reaches either end; the controller halts and runs the handler
- **TMC2209 DIAG:** fires on abnormal load (stall), stopping the motor

### Web Interface (ESP32)

The ESP32 starts a soft access point and serves a control page on port 80. Connect to the `KlondikeESP32` network and open the IP printed on the serial monitor (115200 baud). The current prototype toggles an LED via `GET /H` and `GET /L`.

## Repository Layout

```
hardware/    KiCad schematic, PCB layout, BOM
firmware/
  stm32/     Zephyr application and device tree drivers
  esp32/     Access point and web control page
docs/        Wiring diagrams and user manual
```

## Getting Started

1. Wire the Mean Well supplies to J1 (+5V) and J2 (+24V).
2. Flash the ESP32 over USB-C.
3. Flash the STM32 over SWD (TC2030), or remotely through the ESP32.
4. Connect to `KlondikeESP32` Wi-Fi and open the control page.
5. Home the slider against the endstops before the first game.

## Status

- [x] Electronics specification, schematic, PCB layout
- [x] ESP32 web control prototype
- [x] PCB fabrication (ordered)
- [ ] Firmware (device tree drivers in progress)
- [ ] Mechanical + electrical integration
- [ ] Validation testing and delivery (early December 2026)

## Team

**Electrical:** William Lin (Design Lead), Ram Vadhadia (Project Archivist), Alani Martinez (Financial Officer)

**Mechanical:** Robert Thomas (Design Lead), Max Siesky (Project Manager), Adler Torchia (Financial Officer), Sarah Doerich (Communications Officer), Chris Ferranti

EPICS at Purdue · Project partner: Klondike Elementary School
