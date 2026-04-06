# Team Deimos - ARM Control Track

Low-level joint control implementation for a 6-DOF robotic arm. This embedded system handles multi-axis motor control with advanced sensor integration, including I2C multiplexing for address collision resolution, absolute encoder calibration, and timer-interrupt-driven motion control.

## Table of Contents
- [Overview](#overview)
- [Technical Features](#technical-features)
- [Hardware Components](#hardware-components)
- [Project Structure](#project-structure)
- [I2C Multiplexer Integration](#i2c-multiplexer-integration)
- [Encoder Rollover & Kinematics](#encoder-rollover--kinematics)
- [Motor Control Implementation](#motor-control-implementation)
- [Live-Tuning Command Interface](#live-tuning-command-interface)
- [Getting Started](#getting-started)
- [Building and Deployment](#building-and-deployment)

## Overview

This project implements comprehensive low-level control for a 6-DOF robotic arm using STM32F4 microcontroller architecture. The system prioritizes real-time responsiveness, sensor accuracy, and reliable communication through multiple I2C buses via multiplexing.

## Technical Features

- **I2C Address Collision Resolution**: TCA9548A multiplexer for managing up to 8 independent I2C channels
- **Absolute Encoder Integration**: AS5600 encoder calibration and reading with DMA-optimized data retrieval
- **Timer-Interrupt Motor Control**: Precise PWM generation and timing for synchronized joint actuation
- **Blocking I2C Communication**: Blocking mode ensures synchronous I2C operations, guaranteeing data validity before function return. This approach eliminates race conditions in time-critical joint control scenarios.
- **Live CLI**: THis i will update later on
## Hardware Components

- **Microcontroller**: STM32F407xx (ARM Cortex-M4)
- **I2C Multiplexer**: TCA9548A (0x70 base address)
- **Encoders**: AS5600 magnetic encoders (per-joint absolute position sensing)
- **Communication**: DMA-enabled I2C with USB host support

## Project Structure

```
Core/
├── Inc/          - Core header files (main.h, HAL config, interrupt handlers)
└── Src/          - Implementation (motor control, encoder reading, system init)

Drivers/
├── CMSIS/        - ARM Cortex Microcontroller Software Interface Standard
└── STM32F4xx_HAL_Driver/  - STM32 Hardware Abstraction Layer

USB_HOST/        - USB Host library and configuration
├── App/          - Application layer
└── Target/       - Platform-specific configuration

Middlewares/     - ST middleware libraries

EWARM/           - IAR Embedded Workbench project files
```

## I2C Multiplexer Integration

The system uses the TCA9548A I2C multiplexer to address multiple sensors on separate channels, enabling up to 8 independent I2C buses from a single master interface. Each AS5600 encoder is accessed through channel-specific addressing, resolving address collision issues inherent in multi-sensor systems.

### Implementation Details

**Reference**: [Core/Src/main.c](Core/Src/main.c#L68) - `Read_AS5600()` function

```c
uint16_t Read_AS5600(uint8_t mux_channel) {
    uint16_t angle = 0;
    uint8_t data[2];
    
    // Step 1: Select multiplexer channel
    uint8_t mux_address = 0x70;
    uint8_t mux_channel_select = 1 << mux_channel;
    HAL_I2C_Master_Transmit(&hi2c1, mux_address << 1, &mux_channel_select, 1, HAL_MAX_DELAY);
    
    // Step 2: Request angle data from AS5600
    uint8_t as5600_address = 0x36;
    uint8_t angle_register = 0x0E;  // Angle register address
    HAL_I2C_Master_Transmit(&hi2c1, as5600_address << 1, &angle_register, 1, HAL_MAX_DELAY);
    
    // Step 3: Read angle value (16-bit: MSB, LSB)
    HAL_I2C_Master_Receive(&hi2c1, as5600_address << 1, data, 2, HAL_MAX_DELAY);
    
    // Step 4: Combine bytes into 16-bit value
    angle = (data[0] << 8) | data[1];
    return angle;
}
```

**Operation Flow**:
1. **Multiplexer Selection**: Address the TCA9548A at 0x70 with channel mask (1 << channel)
2. **Channel Activation**: Transmit channel selection via blocking I2C (guarantees immediate effect)
3. **Encoder Read**: Access AS5600 at address 0x36, read angle register 0x0E
4. **Data Assembly**: Combine MSB and LSB into 16-bit angle value
5. **Synchronization**: Blocking operations ensure data validity before function return

**Key Design Decision**: Blocking I2C mode prevents race conditions where the function would return before data acquisition completed, eliminating garbage values in the control feedback loop.


## Encoder Rollover & Kinematics

The AS5600 encoder provides 12-bit absolute position data (0-4095 representing 0-360°). For a 2:1 reduction joint configuration, this raw data must be converted to true joint angles through calibration and rollover compensation.

### Angle Conversion Algorithm

The `GetTrueJointAngle()` function performs the following transformations:

```c
float GetTrueJointAngle(uint16_t raw_encoder_val) {

    int32_t relative_value = int32_t(raw_encoder_val) - int32_t(3500);

    while (relative_value > 2048)  relative_value -= 4096;
    while (relative_value < -2048) relative_value += 4096;

    float joint_angle = (float)relative_value * (180.0f / 4096.0f);
    
    if (joint_angle > 90.0f)  joint_angle = 90.0f;
    if (joint_angle < -90.0f) joint_angle = -90.0f;
    
    return joint_angle;
}
```

### Algorithm Explanation

| Step | Operation | Purpose |
|------|-----------|---------|
| **Calibration** | Subtract 3500 counts | Establish zero reference point at known mechanical position |
| **Rollover Handling** | Phase wrapping ±4096 counts | Resolve discontinuity when encoder crosses 0/4096 boundary |
| **Scale Conversion** | Multiply by 180/4096 | Transform encoder counts to degrees |
| **Limit Enforcement** | Clamp to ±90° | Respect mechanical joint constraints |

## Motor Control Implementation

The motor control subsystem implements proportional feedback control with configurable gain tuning and directional control for each joint actuator. PWM output drives H-bridge motor drivers for bidirectional rotation.

### PWM Output and Direction Control

The `Update_Motor_Drive()` function computes control efforts from position error and applies them to the motor driver through Timer 3:

```c
void Update_Motor_Drive(float target, float current) {
    // Compute position error for feedback loop
    float error = target - current;
    
    // Apply proportional gain scaling
    float output = Kp * error;

    // Determine motor rotation direction based on control output polarity
    if (output * motor_direction_multiplier > 0) {
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);   
    } else {
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
    }

    // Saturate output to valid 16-bit PWM range (0-65535)
    float abs_output = fabs(output);
    if (abs_output > 65535.0f) abs_output = 65535.0f;
    
    // Apply PWM duty cycle to Timer 3, Channel 1 (motor drive)
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, (uint32_t)abs_output);
}
```

**Control Architecture**:
- **Feedback Loop**: Computes position error between target and current joint angles in degrees
- **Proportional Gain**: Scales error by tunable coefficient $K_p$ to generate control effort
- **Direction Multiplier**: Software-based inversion flag enables motor polarity control without wiring changes
- **PWM Saturation**: Constrains output to 16-bit timer register range (0–65535 counts)
- **Bidirectional Motion**: Asymmetric PWM duty cycle enables precise forward and reverse actuation

## Live-Tuning Command Interface

The system provides real-time parameter adjustment through a UART-based command-line interface, enabling dynamic control tuning without recompilation or power cycling. This in-progress implementation supports interactive system calibration and gain modification during operation.

### Command Processing and UART Reception

The command interface accepts structured ASCII commands through interrupt-driven UART reception on USART2:

```c
void Parse_CLI_Command(char* cmd) {
    // SET_P <gain> - Update proportional feedback coefficient
    if (strncmp(cmd, "SET_P", 5) == 0) {
        Kp = atof(&cmd[6]);
    }
    // SET_HOME <angle> - Adjust target joint angle set-point
    else if (strncmp(cmd, "SET_HOME", 8) == 0) {
        target_angle = atof(&cmd[11]); 
    }
    // INV_DIR - Toggle motor rotation direction
    else if (strncmp(cmd, "INV_DIR", 7) == 0) {
        motor_direction_multiplier *= -1;
    }
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART2) {
        // Detect command terminator (CR or LF)
        if (rx_data == '\r' || rx_data == '\n') {
            rx_buffer[rx_index] = '\0';
            Parse_CLI_Command(rx_buffer);
            rx_index = 0;
        } else {
            // Accumulate characters into command buffer
            if (rx_index < UART_BUF_SIZE - 1) {
                rx_buffer[rx_index++] = (char)rx_data;
            }
        }
        // Re-enable interrupt for next character reception
        HAL_UART_Receive_IT(&huart2, &rx_data, 1);
    }
}
```

**Supported Commands**:

| Command | Format | Purpose |
|---------|--------|---------|
| `SET_P` | `SET_P <gain>` | Update proportional feedback gain coefficient |
| `SET_HOME` | `SET_HOME <angle>` | Set target joint angle set-point |
| `INV_DIR` | `INV_DIR` | Toggle motor direction polarity |

### Design Rationale for Motor Control

**Proportional Feedback Control**:
- Proportional gain eliminates steady-state error in position tracking
- Tunable $K_p$ coefficient enables system commissioning without firmware recompilation
- Symmetric positive/negative output range maintains linear feedback characteristics

**Directional Control Architecture**:
- Software-based direction multiplier enables rapid motor polarity inversion during commissioning
- Decouples mechanical motor wiring from control logic, improving modularity
- Eliminates hardware rewiring delays during prototype iteration

**Angle Resolution and Constraints**:
- 12-bit encoder resolution (4096 counts) provides ±0.088° granularity per count
- Symmetric ±90° mechanical limits simplify motion planning and prevent joint over-extension
- Single calibration offset (3500 counts) minimizes encoder drift compensation overhead

## Getting Started

### Prerequisites
- IAR Embedded Workbench for ARM (EWARM)
- STM32F4 HAL library and CMSIS headers
- USB host library support
- STM32CubeMX (optional, for regenerating configuration files)

### Quick Reference
- **Main entry point**: [Core/Src/main.c](Core/Src/main.c)
- **HAL configuration**: [Core/Inc/stm32f4xx_hal_conf.h](Core/Inc/stm32f4xx_hal_conf.h)
- **Interrupt handlers**: [Core/Src/stm32f4xx_it.c](Core/Src/stm32f4xx_it.c)

## Building and Deployment

The project is configured for **IAR Embedded Workbench (EWARM)**. Build and project configuration files are located in the [EWARM/](EWARM/) directory:

- **Project file**: `team-deimos-arm-control-track-c.ewp`
- **Workspace**: `team-deimos-arm-control-track-c.ewd`
- **Flash configuration**: `stm32f407xx_flash.icf`
- **SRAM configuration**: `stm32f407xx_sram.icf`

### Build Steps
1. Open the workspace in IAR Embedded Workbench
2. Configure project settings for STM32F407xx target
3. Build the project
4. Deploy to STM32F4 microcontroller via debugger or bootloader


Made by Reman Dey