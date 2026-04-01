# Team Deimos - ARM Control Track

Low-level joint control implementation for a 6-DOF robotic arm. This embedded system handles multi-axis motor control with advanced sensor integration, including I2C multiplexing for address collision resolution, absolute encoder calibration, and timer-interrupt-driven motion control.

## Table of Contents
- [Overview](#overview)
- [Technical Features](#technical-features)
- [Hardware Components](#hardware-components)
- [Architecture](#architecture)
- [I2C Multiplexer Integration](#i2c-multiplexer-integration)
- [Building and Deployment](#building-and-deployment)

## Overview

This project implements comprehensive low-level control for a 6-DOF robotic arm using STM32F4 microcontroller architecture. The system prioritizes real-time responsiveness, sensor accuracy, and reliable communication through multiple I2C buses via multiplexing.

## Technical Features

- **I2C Address Collision Resolution**: TCA9548A multiplexer for managing up to 8 independent I2C channels
- **Absolute Encoder Integration**: AS5600 encoder calibration and reading with DMA-optimized data retrieval
- **Timer-Interrupt Motor Control**: Precise PWM generation and timing for synchronized joint actuation
- **DMA-Based Communication**: Non-blocking I2C transfers for improved system responsiveness

## Hardware Components

- **Microcontroller**: STM32F407xx (ARM Cortex-M4)
- **I2C Multiplexer**: TCA9548A (0x70 base address)
- **Encoders**: AS5600 magnetic encoders (per-joint absolute position sensing)
- **Communication**: DMA-enabled I2C with USB host support

## Architecture

```
Core/
├── Inc/          - Header files
└── Src/          - Core implementation

Drivers/
├── CMSIS/        - ARM Cortex Microcontroller Software Interface Standard
└── STM32F4xx_HAL_Driver/  - STM32 Hardware Abstraction Layer

USB_HOST/        - USB host stack integration

Middlewares/     - Middleware components
```

## I2C Multiplexer Integration

The system uses the TCA9548A I2C multiplexer to address multiple sensors on separate channels. The AS5600 encoder is read through channel-specific addressing.

**Implementation** (located in [Core/Src/main.c](Core/Src/main.c#L68)):

```c
uint16_t Read_AS5600(uint8_t mux_channel) {
    HAL_I2C_Master_Transmit_DMA(&hi2c1, (0x70 << 1), &mux_channel, 1);
    uint16_t encoder_data[2];
    
    #define AS5600_ADDR (0x36 << 1)
    #define RAW_ANGLE_REG 0x0C
    
    HAL_I2C_Mem_Read_DMA(&hi2c1, AS5600_ADDR, RAW_ANGLE_REG, 
                         I2C_MEMADD_SIZE_8BIT, encoder_data, 2);
    return encoder_data;
}
```

This implementation:
- Addresses the TCA9548A multiplexer at 0x70
- Selects the target I2C channel via DMA transmission
- Reads raw angle data from the AS5600 at register 0x0C
- Uses DMA to avoid blocking I2C operations

## Building and Deployment

The project is configured for IAR Embedded Workbench (EWARM). Build files and project configuration are located in the EWARM/ directory.