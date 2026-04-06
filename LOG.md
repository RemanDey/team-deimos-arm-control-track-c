# Development Log

## March 29, 2026 | Duration: 1 hour

**Environment Setup & Initial Development**

- Installed STM32CubeMX VSCode Extension with CMake integration
- Installed STM32CubeMX IDE
- Completed basic LED blink proof-of-concept (firmware implementation; hardware testing pending)

## April 1, 2026 | Duration: 1 hour

**I2C Communication Implementation**

- Reviewed STM32 I2C protocol documentation from official STM32 reference materials (30 minutes)
- Implemented Part 1: I2C multiplexer and AS5600 encoder reading functionality (30 minutes)
- Code location: [Core/Src/main.c](Core/Src/main.c#L68) - `Read_AS5600()` function

## April 2, 2026 | Duration: 1.5 hours

**Joint Angle Computation & Encoder Rollover Handling**

- Developed comprehensive angle conversion algorithm to handle 12-bit AS5600 encoder data
- Implemented phase wrapping logic for encoder rollover compensation (resolves 0/4096 discontinuity)
- Applied calibration offset (3500 counts) for mechanical reference point establishment
- Integrated degree conversion and mechanical joint limits (±90° range)
- Code location: [Core/Src/main.c](Core/Src/main.c) - `GetTrueJointAngle()` function
- Documentation: Enhanced README.md with detailed algorithm explanation and design rationale

## April 4, 2026 | Duration: 1 hour

**Motor Control Implementation**

- Designed and implemented proportional feedback control law for joint position tracking
- Developed `Update_Motor_Drive()` function with configurable proportional gain tuning
- Integrated PWM output generation through STM32 Timer 3 with 16-bit resolution
- Implemented bidirectional motor direction control via GPIO (GPIOD Pin 12)
- Applied output saturation to valid timer compare register range (0–65535 counts)
- Code location: [Core/Src/main.c](Core/Src/main.c) - `Update_Motor_Drive()` function
- Documentation: Added motor control section to README.md with technical architecture details

## April 6, 2026 | Duration: 2 hours

**Live-Tuning Command Interface Development**

- Designed and implemented UART-based command-line interface for real-time parameter adjustment
- Developed `Parse_CLI_Command()` function supporting three command types:
  - `SET_P`: Dynamic proportional gain coefficient update
  - `SET_HOME`: Target angle set-point adjustment
  - `INV_DIR`: Motor rotation direction polarity toggle
- Implemented interrupt-driven UART reception callback (`HAL_UART_RxCpltCallback()`)
- Integrated character-buffered command accumulation with CR/LF termination detection
- Code location: [Core/Src/main.c](Core/Src/main.c) - `Parse_CLI_Command()` and UART callback functions
- Status: In-progress implementation; command parsing logic functional, serial communication validated
- Documentation: Added comprehensive CLI section to README.md with command reference table 