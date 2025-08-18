# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

moteus is an integrated brushless servo motor controller designed by mjbots. This is a complete hardware and firmware project that includes:
- Motor controller firmware (C++) for STM32G4 microcontrollers
- PCB designs (KiCad) for multiple hardware variants (r4.11, c1, n1, x1)
- Client libraries (Python and C++)
- Diagnostic and calibration tools

## Build System

This project uses **Bazel** as its primary build system. Key build files:
- `WORKSPACE` - Bazel workspace configuration
- `BUILD` files throughout the project - Bazel build targets
- `CMakeLists.txt` - Minimal CMake support for client libraries only

### Common Build Commands

```bash
# Build all host-side targets (tests, tools, libraries)
bazel build //...

# Build firmware targets for embedded hardware
bazel build //:target

# Run all host tests
bazel test //:host

# Build specific firmware variants
bazel build //fw:moteus
bazel build //fw:can_bootloader

# Build Python moteus tools
bazel build //lib/python:moteus

# Build C++ client library examples
bazel build //lib/cpp/examples:simple
```

### Python Tools Installation

For end-users, Python tools can be installed via pip:
```bash
pip3 install moteus_gui  # Installs tview and moteus_tool
```

## Architecture

### Firmware Architecture (`fw/`)
- **moteus.cc** - Main firmware entry point and hardware initialization
- **moteus_controller.h/.cc** - Top-level controller that orchestrates all subsystems
- **bldc_servo.h/.cc** - Core brushless motor control logic (FOC, PID loops)
- **fdcan.h/.cc** - CAN-FD communication interface
- **motor_position.h** - Encoder and position feedback systems
- **Hardware abstraction layers** - STM32G4-specific implementations

The firmware implements:
1. Field-Oriented Control (FOC) for 3-phase brushless motors
2. Cascaded position/velocity/current control loops running at 15-30kHz
3. Magnetic encoder feedback with calibration routines
4. CAN-FD communication protocol for commands and telemetry

### Client Libraries

**Python Library** (`lib/python/moteus/`):
- **moteus.py** - High-level controller interface
- **multiplex.py** - Multi-device communication protocol
- **transport.py** - Communication transport abstractions
- **fdcanusb.py** - USB-to-CAN adapter interface
- **moteus_tool.py** - Command-line diagnostic tool

**C++ Library** (`lib/cpp/mjbots/moteus/`):
- **moteus.h** - Primary controller interface with sync/async APIs
- **moteus_protocol.h** - Binary communication protocol
- **moteus_transport.h** - Transport layer abstractions

### Hardware Variants (`hw/`)
- **controller/** - Original r4.11 design (Eagle CAD)
- **c1/** - Compact variant (KiCad)
- **n1/** - Next-gen variant (KiCad)  
- **x1/** - High-power variant (KiCad)

## Testing

```bash
# Run all host tests
bazel test //:host

# Run specific test suites
bazel test //fw:host          # Firmware unit tests
bazel test //lib/python:host  # Python library tests
bazel test //lib/cpp:host     # C++ library tests
bazel test //utils:host       # Utility tests

# Python-specific testing
cd lib/python && python -m pytest test/
```

## Key Development Workflows

### Firmware Development
1. Modify firmware sources in `fw/`
2. Build: `bazel build //fw:moteus`
3. Flash to hardware using provided scripts in `fw/`
4. Use `tview` for debugging and parameter adjustment

### Client Library Development
1. Modify library sources in `lib/python/moteus/` or `lib/cpp/mjbots/moteus/`
2. Run tests: `bazel test //lib/python:host` or `bazel test //lib/cpp:host`
3. Test with example programs in respective `examples/` directories

### Calibration and Testing
- **Motor calibration**: `python3 -m moteus.moteus_tool --target 1 --calibrate`
- **Real-time monitoring**: `python3 -m moteus_gui.tview --devices=1`
- **Custom diagnostic tools**: Located in `utils/`

## Communication Protocol

moteus uses a custom binary protocol over CAN-FD for efficient, high-frequency communication. The protocol supports:
- Position/velocity/torque commands with feedforward
- Real-time telemetry streaming  
- Parameter configuration
- Firmware updates over CAN

## Important Notes

- This project combines low-level embedded firmware with high-level robotics control
- Hardware designs handle significant electrical power (up to 2kW) - safety critical
- Magnetic encoder calibration is required for each motor/controller combination
- CAN-FD communication requires compatible hardware (fdcanusb adapter recommended)
- Multiple hardware variants exist with different power ratings and form factors