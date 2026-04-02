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
