# UNIDRIVE V1

A compact brushed DC motor driver board with encoder feedback, shunt current sensing and CAN bus, controlled by an **STM32G431KBU6**.

> ⚠️ **Status: hardware V1 done, firmware under development.**
> The firmware has not been checked on real hardware yet. Nothing in the firmware sections below should be read as tested or working.

## Overview

**UNIDRIVE** is a single-channel brushed DC motor driver built around the **TI DRV8701P** gate driver and an H-bridge made of 4 discrete N-channel MOSFETs. An **STM32G431KBU6** (Cortex-M4F, 170 MHz) handles the control. Feedback comes from:

- a **quadrature encoder** for speed and position
- a **5 mΩ low-side shunt** read through the DRV8701 current-sense amplifier

The board can be commanded over **CAN (FDCAN1)** or **UART**, and four status LEDs show its state.

The goal is a small, general-purpose driver for robotics and hobby projects, with duty, current, velocity and position control modes.

## Main Components

### Hardware

| Part | Function |
|------|----------|
| STM32G431KBU6 (UFQFPN-32) + 24 MHz crystal | Main MCU (HSE → PLL 170 MHz) |
| DRV8701P | Brushed DC H-bridge gate driver (PWM IN1/IN2 variant) |
| 4 × N-channel MOSFET | Full H-bridge power stage |
| 5 mΩ shunt (R8) | Current sensing, read through the DRV8701 SO amplifier (gain 20 V/V) |
| Quadrature encoder (external) | Speed / position feedback (x4 decoding on TIM2) |
| TJA1051T/3 | CAN transceiver, with optional 120 Ω termination (JP2) |
| AP2112K-3.3 | 3.3 V LDO for the MCU |
| 4 LEDs | RUN / CAL / FAULT / COMM status |
| SWD header (J6) | Programming and debugging |

### Pin Map

| Signal | Pin | Peripheral | Notes |
|--------|-----|------------|-------|
| HSE OSC_IN / OSC_OUT | PF0 / PF1 | RCC | 24 MHz crystal |
| CSENSE (DRV8701 SO) | PA0 | ADC1_IN1 | Current-sense amplifier output |
| nFAULT | PA1 | GPIO / EXTI1 | Pull-up to DVDD, active low |
| IN1 / IN2 | PA2 / PA3 | TIM15_CH1 / CH2 | 20 kHz PWM |
| nSLEEP | PA4 | GPIO | Driver enable |
| UART TX / RX | PA9 / PA10 | USART1 | On header J5 |
| CAN RX / TX | PA11 / PA12 | FDCAN1 | To TJA1051T/3 |
| SWDIO / SWCLK | PA13 / PA14 | SWD | Header J6 |
| Encoder A / B | PA15 / PB3 | TIM2_CH1 / CH2 | 32-bit encoder counter |
| LED RUN / CAL / FAULT / COMM | PB4 / PB5 / PB6 / PB7 | GPIO | Active low |
| BOOT0 | PB8 | — | 10k to GND (boot from flash) |
| Expansion | PA5, PA6, PA7, PA8, PB0 | — | Free, on header J5 |

### Connectors

| Connector | Pinout |
|-----------|--------|
| **J1** (1×12) | 1/3 CANL, 2/4 CANH, 5 VENC, 6 ENCA, 7 ENCB, 8 GND, 9 M−, 10 M+, 11 GND, 12 VCC (power input) |
| **J2** (1×3) | Encoder supply select: +5 V (1-2) or +3.3 V (2-3) |
| **J5** (2×4) | PA9 (TX), PA10 (RX), GND, PA8, PB0, PA7, PA6, PA5 |
| **J6** (1×3) | SWDIO, SWCLK, GND |

> **JP1** connects the TJA1051 VIO pin to +3.3 V. It is an **open** solder jumper by default and **must be bridged** for CAN to work.
> **JP2** enables the 120 Ω CAN termination. Bridge it only on the boards at the ends of the bus.

### Driver Logic (DRV8701P)

| IN1 | IN2 | Bridge state |
|-----|-----|--------------|
| 0 | 0 | Coast (Hi-Z) |
| 1 | 0 | Forward |
| 0 | 1 | Reverse |
| 1 | 1 | Brake (low-side slow decay) |

- Current scaling: `I ≈ (V_SO − 0.05 V) / (0.005 Ω × 20)` ≈ **10 A/V**
- The SO output is valid only while the bridge is driving, not during brake or coast.
- Hardware current chopping limit: VREF ≈ 2.09 V → about **20 A**

## Firmware

> 🚧 **Under development.** The firmware structure and features below are the design target. They have been written and partly simulated on a PC, but **not yet confirmed on the real board**.

The firmware is built with **STM32CubeMX + CMake + STM32 HAL (G4)**. CubeMX generates the pin setup and peripheral init. All application code is in `App/`, and `main.c` only calls `app_init()` and `app_update()`.

### Planned Software Modules

| Layer / File | Description | Status |
|--------------|-------------|--------|
| `App/config.h` | Single place for every tunable value (PWM frequency, gains, limits, motor model) | 🚧 In progress |
| `App/bsp/` | Low-level peripheral drivers: GPIO/LEDs, TIM15 PWM, ADC + DMA, TIM2 encoder, TIM6 tick, USART1 DMA, FDCAN | 🚧 Written, not verified on hardware |
| `App/sensor/current` | Shunt current measurement synced to the PWM, offset calibration, filtering, overcurrent detection | 🚧 Written, not verified on hardware |
| `App/sensor/encoder` | Position (64-bit) and velocity estimation (PLL observer) | 🚧 Written, not verified on hardware |
| `App/sensor/monitor` | VDDA and MCU temperature monitoring | 🚧 Written, not verified on hardware |
| `App/control/drv8701` | Driver state machine: sleep / wake / ready / fault handling | 🚧 Written, not verified on hardware |
| `App/control/hbridge` | Duty → DRV8701P switching pattern, ADC sample timing | 🚧 Written, not verified on hardware |
| `App/control/pid`, `profile` | PID controller (ported from VIXULY), trapezoidal motion profile | 🚧 Written, not verified on hardware |
| `App/control/control` | Control loops: duty, current, velocity and position modes, fault handling | 🚧 Written, not verified on hardware |
| `App/app/` | State machine, CLI, telemetry, CAN protocol, parameter storage in flash | ⏳ Not started |

### Planned Features

- [ ] Control modes: **OFF / BRAKE / DUTY / CURRENT / VELOCITY / POSITION**
- [ ] 20 kHz PWM with current sampling locked to the PWM period
- [ ] Velocity PID with feed-forward and setpoint ramp
- [ ] Position control with a trapezoidal motion profile
- [ ] Software overcurrent, stall, following-error and over-temperature protection
- [ ] DRV8701 fault handling (nFAULT)
- [ ] UART command line and telemetry
- [ ] CAN protocol with node addressing
- [ ] Parameter storage in flash

### Development Progress

- [x] Schematic and PCB (V1)
- [x] Pin map and peripheral plan
- [x] Firmware architecture
- [x] Low-level drivers and control code written
- [x] Control logic tested in a PC simulation (motor + bridge + encoder model)
- [ ] **Board bring-up on real hardware**
- [ ] Current-sense calibration on hardware
- [ ] PID tuning on a real motor
- [ ] Application layer (CLI, CAN protocol, flash parameters)

## Quick User Guide

> The steps below describe the intended workflow. They may change as the firmware develops.

### 1. Build and flash

1. Open the firmware project (CMake, generated by STM32CubeMX).
2. Review `App/config.h`, especially the motor model, encoder resolution and current limits.
3. Build and flash through the **SWD header (J6)** with an ST-Link or a similar probe.

### 2. Hardware setup

1. Bridge **JP1** if you want to use CAN.
2. Set **J2** to match your encoder supply voltage (5 V or 3.3 V).
3. Connect the motor to **M+ / M−**, the encoder to **ENCA / ENCB / VENC / GND**, and the supply to **VCC / GND** on J1.
4. Connect a USB-UART adapter to **J5** (PA9 TX, PA10 RX, GND) for the debug terminal.

### 3. Run

🚧 *Under development: the command interface and CAN protocol are not finished yet.*

## Known Issues (Hardware V1)

- **CAN:** JP1 is open by default, so CAN does not work until it is bridged.
- **nFAULT:** the pull-up goes to DVDD, which is off in sleep, so nFAULT reads low while the driver sleeps. The firmware has to ignore it while asleep and during wake-up.
- **PA0 input range:** the SO output can swing up to about 4.8 V during an overload, which is above the 3.6 V limit of PA0. A series resistor or clamp is recommended in the next revision.
- **Net names:** "IN1/EN" and "IN2/PH" are misleading. The DRV8701**P** only has the PWM (IN1/IN2) mode. Select DRV8701**E** variant for PH/EN mode.
- **Missing sensing:** there is no bus voltage or board temperature sense going to the MCU.
- **Current measurement limits:** current can't be measured below about 10 % duty, or during brake or coast (a limit of the low-side, unidirectional sense amplifier).

## Repository Structure

```text
UNIDRIVE-V1/
├── Hardware/
│   └── (KiCad schematic and PCB files)
├── Firmware/
│   ├── Core/              # CubeMX generated
│   ├── Drivers/           # STM32 HAL
│   ├── App/
│   │   ├── config.h
│   │   ├── bsp/
│   │   ├── sensor/
│   │   ├── control/
│   │   └── app/
│   ├── UNIDRIVE_FW.ioc
│   └── CMakeLists.txt
├── docs/
└── README.md
```

## Author
[@1m4n00b6292](https://github.com/1m4n00b6292)

## License

This project is licensed under the [MIT License](LICENSE).
