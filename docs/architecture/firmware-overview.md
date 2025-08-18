# moteus Firmware Architecture Overview

This document provides a high-level overview of the moteus firmware architecture. It is intended for developers who want to understand the overall system design before diving into specific subsystems.

## System Overview

The moteus firmware implements a Field-Oriented Control (FOC) brushless servo controller running on an STM32G4 microcontroller. The system provides:

- **Real-time control loops** operating at 15-30kHz
- **Cascaded control architecture** with position, velocity, and current control
- **CAN-FD communication** for high-bandwidth command and telemetry
- **Multiple encoder support** including magnetic, quadrature, and hall sensors
- **Safety systems** with comprehensive fault detection and recovery

### Key Specifications
- **Target**: STM32G474 microcontroller with hardware floating-point and CORDIC acceleration
- **Control frequency**: 15-30kHz (configurable, PWM-synchronized)
- **Communication**: CAN-FD up to 5Mbps with binary protocol
- **Power handling**: Up to 2kW electrical power (depending on hardware variant)
- **Position resolution**: Sub-degree precision with magnetic encoders

## Main Architecture Components

### MoteusController (`fw/moteus_controller.h/.cc`)
The top-level orchestrator that initializes and coordinates all subsystems:
- **System integration**: Manages dependencies between BldcServo, communication, and hardware layers
- **Protocol handling**: Implements the register-based command/telemetry interface
- **Configuration management**: Handles persistent parameter storage and validation
- **Safety coordination**: Orchestrates fault detection and recovery across subsystems

```
MoteusController
├── BldcServo (motor control)
├── FDCan (communication)
├── MotorPosition (feedback)
├── Hardware drivers (ADC, PWM, etc.)
└── System services (telemetry, config, etc.)
```

### BldcServo (`fw/bldc_servo.h/.cc`)
The core motor control logic implementing field-oriented control:
- **Control loop orchestration**: Manages the cascaded position/velocity/current control loops
- **Operating mode management**: Implements 15+ operating modes from PWM to position control
- **Safety systems**: Current limiting, thermal protection, position bounds checking
- **Calibration routines**: Automatic motor parameter identification and encoder calibration

### MotorPosition (`fw/motor_position.h`)
Multi-source position feedback system supporting:
- **Magnetic encoders**: AS5047, MA732, CUI AMT21/22, IC-PZ, AKSiM-2
- **Incremental encoders**: Quadrature with optional index
- **Hall sensors**: 6-step commutation sensors
- **Sensor fusion**: Combining multiple position sources with PLL filtering
- **Calibration**: Automatic compensation for encoder nonlinearity

### FDCan (`fw/fdcan.h/.cc`)
High-speed CAN-FD communication interface:
- **Hardware abstraction**: STM32G4 FDCAN peripheral driver
- **Protocol implementation**: Binary protocol with efficient data packing
- **Real-time constraints**: Interrupt-driven transmission with deterministic timing
- **Error handling**: Comprehensive CAN error detection and recovery

### Hardware Abstraction Layer
Target-specific drivers and hardware interfaces:
- **STM32G4 drivers**: ADC, PWM, UART, I2C, SPI implementations
- **Motor driver interface**: DRV8323 gate driver integration with SPI control
- **Auxiliary ports**: Configurable GPIO/ADC/communication interfaces for sensors
- **System services**: Clock management, millisecond timing, debug interfaces

## Control Architecture

The moteus firmware implements a sophisticated cascaded control architecture optimized for servo applications.

### Control Loop Hierarchy

```
Command → [Trajectory] → [Position/Velocity PID] → [Current PI] → [FOC] → PWM → Motor
            Planner           Controller            Controller   Transform
```

#### 1. Trajectory Planner (Outer Loop)
- **Acceleration limiting**: Generates smooth trajectories with configurable acceleration limits
- **Velocity limiting**: Enforces maximum velocity constraints  
- **Position bounds**: Implements hard position limits for safety
- **Feedforward**: Supports velocity and acceleration feedforward terms

#### 2. Position/Velocity Controller (Middle Loop)
Integrated PID controller with flexible operation modes:
- **Position mode**: Standard PID position control with velocity feedforward
- **Velocity mode**: Pure velocity control (kp=0 or position=NaN)
- **Torque mode**: Direct torque control bypassing outer loops
- **Configurable gains**: Per-command kp/kd scaling for adaptive control

Control law implementation:
```
torque = position_integrator + 
         kp * kp_scale * position_error + 
         kd * kd_scale * velocity_error + 
         command_torque_feedforward
```

#### 3. Current Controller (Inner Loop)
High-bandwidth PI controller for D/Q axis currents:
- **Field-oriented control**: Separate control of flux (D) and torque (Q) producing currents
- **Current limiting**: Configurable maximum current limits for safety
- **Bandwidth**: Optimized for high-frequency operation (15-30kHz)

Control law:
```
voltage = current_integrator + kp * current_error
```

### Field-Oriented Control Implementation

#### Clarke/Park Transforms (`fw/foc.h`)
- **Hardware acceleration**: Uses STM32G4 CORDIC unit for sin/cos calculations
- **DQ transformation**: Converts 3-phase measurements to rotating reference frame
- **Inverse transforms**: Converts DQ voltage commands back to 3-phase PWM

#### Current Sampling and PWM Synchronization
- **Center-aligned PWM**: Ensures optimal current sampling timing
- **ADC synchronization**: Current sampling triggered at PWM center for accuracy
- **Deadtime compensation**: Automatic compensation for gate driver deadtime effects

## Operating Modes

The firmware supports a comprehensive set of operating modes for different applications:

### Core Control Modes
- **kStopped**: Motor disabled, safe state for transitions
- **kPosition**: Full cascaded position/velocity/torque control
- **kCurrent**: Direct D/Q current control bypassing outer loops
- **kVoltage/kVoltageFoc/kVoltageDq**: Direct voltage control modes
- **kPwm**: Direct PWM control for testing and low-level access

### Specialized Modes  
- **kZeroVelocity**: Derivative-only control for smooth stopping
- **kPositionTimeout**: Fallback mode when position commands timeout
- **kStayWithinBounds**: Position limiting mode for safety
- **kMeasureInductance**: Automated motor parameter identification
- **kBrake**: Emergency braking mode (all phases to ground)

### State Management
- **Fault handling**: Automatic transition to kFault mode on errors
- **Calibration sequences**: Automated progression through kEnabling → kCalibrating → kCalibrationComplete
- **Safety transitions**: Enforced safe state transitions with proper initialization

## Communication Architecture

### Binary Protocol over CAN-FD
- **Efficient encoding**: Compact binary representation of commands and telemetry
- **Scalable precision**: Variable precision encoding (int8/int16/int32/float) based on requirements
- **Register map**: Versioned register-based interface for commands and configuration
- **Streaming telemetry**: High-frequency telemetry streaming for real-time monitoring

### Multiplex Protocol Support
- **Multi-device coordination**: Synchronous control of multiple moteus controllers
- **Efficient bus utilization**: Batched commands to minimize CAN bus overhead
- **Deterministic timing**: Predictable communication timing for control applications

### Register Map and Scaling
- **Hierarchical organization**: Logical grouping of parameters (motor, servo, system, etc.)
- **Automatic scaling**: Transparent conversion between internal float and wire formats
- **Version management**: Protocol versioning for backward compatibility

## Data Flow and Timing

### High-Frequency Control Loop (15-30kHz)
```
PWM Interrupt → Current Sampling → FOC Transform → Control Loops → PWM Update
     ↓
CAN TX Queue ← Telemetry Formatting ← Status Update
```

#### Critical Timing Constraints
- **PWM synchronization**: Current sampling must occur during FET low time
- **Control loop execution**: Complete position/velocity/current control within PWM period
- **CAN transmission**: Non-blocking queued transmission to avoid control loop delays
- **Deterministic execution**: All high-frequency code runs from CCM (Core Coupled Memory)

### Medium-Frequency Tasks (1kHz)
- **Parameter validation**: Configuration change processing and validation
- **Thermal monitoring**: Temperature sensor reading and thermal protection
- **System health**: Watchdog updates, system status monitoring
- **Auxiliary sensors**: Non-critical sensor reading (temperature, auxiliary ADC)

### Low-Frequency Tasks (Background)
- **Configuration management**: Persistent storage and retrieval of parameters
- **Diagnostic protocols**: Debug console and diagnostic command processing
- **CAN protocol processing**: Command parsing and response generation
- **System maintenance**: Memory management, error logging

## Hardware Interface Layer

### ADC Management (`fw/stm32g4_adc.h/.cc`)
- **Multi-ADC coordination**: Simultaneous sampling across ADC1/2/3/4/5 
- **Synchronized conversion**: Hardware-triggered sampling synchronized with PWM
- **Current measurement**: Dedicated ADC channels for 3-phase current sensing
- **Auxiliary measurements**: Voltage, temperature, and auxiliary sensor inputs

### Motor Driver Interface (`fw/drv8323.h/.cc`)
- **SPI configuration**: Real-time configuration of DRV8323 gate driver parameters
- **Fault monitoring**: Hardware fault detection and reporting from gate driver
- **Current shunt amplification**: Configurable current measurement gain settings
- **Protection features**: Gate driver integrated protection (overcurrent, thermal, etc.)

### Auxiliary Port System (`fw/aux_port.h`)
Configurable hardware interfaces for sensors and external devices:
- **Flexible configuration**: GPIO, ADC, I2C, SPI, UART modes per port
- **Encoder interfaces**: Direct support for multiple encoder protocols
- **Sensor integration**: Temperature sensors, auxiliary feedback devices
- **Debug interfaces**: Logic analyzer outputs, debugging signals

### Peripheral Management
- **Clock configuration**: Optimized clock tree for maximum performance
- **DMA coordination**: Efficient data transfer without CPU intervention
- **Interrupt prioritization**: Hierarchical interrupt priorities for real-time operation
- **Power management**: Efficient power usage and thermal management

## Key Design Principles

### Real-Time Operation
- **Deterministic execution**: All critical paths have bounded execution time
- **Priority inversion avoidance**: Careful interrupt and task priority design
- **Memory management**: Static allocation in critical paths, no dynamic allocation
- **Hardware optimization**: Leverages STM32G4 specialized peripherals (CORDIC, high-speed ADC)

### Safety and Reliability  
- **Comprehensive fault detection**: Current, thermal, position, communication fault detection
- **Graceful degradation**: Safe fallback modes when faults occur
- **Watchdog protection**: Multiple layers of watchdog protection
- **Parameter validation**: Extensive validation of configuration parameters

### Modularity and Testability
- **Clear interfaces**: Well-defined interfaces between major components
- **Hardware abstraction**: Clean separation between algorithm and hardware layers  
- **Testable design**: Unit testable components with dependency injection
- **Configuration management**: Flexible parameter system for different applications

## Asynchronous Command Handling with Real-Time Guarantees

One of the most critical aspects of the moteus firmware is how it handles asynchronous interface calls and commands while guaranteeing deterministic real-time control loop timing.

### Interrupt Priority Architecture

The firmware implements a **hierarchical interrupt system** with carefully managed priorities to separate time-critical operations from communication processing:

#### **Priority Level 0 (Highest) - PWM Timer Interrupt**
- **Critical timing**: Handles PWM center-aligned interrupt for current sampling
- **Ultra-low latency**: Samples motor currents at precise PWM timing (≤1.03μs window)
- **Minimal processing**: Only ADC reads and triggers lower-priority processing
- **Cannot be interrupted**: Guarantees deterministic current sampling timing

#### **Priority Level 6 (Lowest) - PendSV for Control Processing**
- **Control loop execution**: Position/velocity/current PID processing 
- **FOC calculations**: Clarke/Park transforms using CORDIC
- **PWM updates**: Calculate and apply new PWM values
- **Preemptible**: Can be interrupted by GPIO/encoder interrupts

#### **Priority Level 1 - GPIO/Encoder Interrupts**
- **Encoder feedback**: Quadrature, hall sensor, and index pulse processing
- **Can preempt control loop**: Ensures encoder timing accuracy

#### **Priority Level 2 - UART/Communication**
- **Serial communication**: Debug console and diagnostic interfaces
- **Non-critical**: Cannot disrupt control timing

### Two-Phase Control Loop Design

The control loop uses a **split-phase approach** to minimize critical section timing:

#### **Phase 1: High-Priority Timer ISR**
1. **ADC triggering**: Samples 3-phase currents at PWM center
2. **Minimal processing**: Only critical-timing operations  
3. **PendSV trigger**: Schedules lower-priority processing
4. **Fixed execution time**: Deterministic, sub-microsecond duration

#### **Phase 2: Lower-Priority PendSV ISR**
1. **Control calculations**: PID loops, FOC transforms, trajectory planning
2. **PWM updates**: Calculate new duty cycles
3. **Telemetry preparation**: Queue status data for transmission
4. **Interruptible**: GPIO/encoder interrupts can preempt

### Asynchronous Command Processing

Commands from CAN-FD are handled **completely outside** the control loop timing:

#### **Polled Communication Architecture**
```
Main Loop:
  rs485->Poll()              // Serial interface
  fdcan_micro_server.Poll()  // CAN-FD command processing  
  moteus_controller.Poll()   // Parameter updates
  multiplex_protocol.Poll()  // Protocol handling
```

- **No CAN interrupts**: CAN reception uses polling to avoid timing disruption
- **Hardware buffering**: STM32G4 FDCAN peripheral buffers incoming messages
- **Cooperative processing**: Commands processed during non-critical time slots

#### **Thread-Safe Parameter Updates**
- **Atomic operations**: Configuration changes use atomic reads/writes where possible
- **Interrupt disabling**: Critical parameter updates disable interrupts briefly
- **Deferred validation**: Parameter validation occurs outside control loop
- **Safe state transitions**: Mode changes only allowed at specific control phases

### Telemetry Streaming

High-frequency telemetry is handled through **queued, non-blocking transmission**:

#### **Queue Management**
- **Control loop writes**: Status data written to queue during PendSV ISR
- **Main loop transmission**: CAN frames transmitted during polling cycles  
- **Overflow protection**: Old telemetry discarded if transmission can't keep up
- **Rate limiting**: Telemetry rate never exceeds control loop frequency

#### **Bandwidth Optimization**
- **Efficient encoding**: Binary protocol with configurable precision (int8/16/32/float)
- **Selective streaming**: Only requested telemetry items transmitted
- **Batched updates**: Multiple parameters packed into single CAN frames

### Key Design Principles

#### **Separation of Concerns**
- **Time-critical**: Current sampling and basic safety checks (Priority 0)
- **Important but flexible**: Full control loop processing (Priority 6, interruptible)  
- **Background**: Command processing, parameter validation, telemetry (Main loop)

#### **Deterministic Timing**
- **Fixed priority levels**: Carefully chosen interrupt priorities ensure proper preemption
- **Bounded execution times**: Critical sections have maximum time guarantees
- **No dynamic allocation**: All memory statically allocated in critical paths
- **Watchdog protection**: Multiple layers detect and recover from timing violations

#### **Fault Tolerance**
- **Communication timeouts**: Position mode times out to safety mode if commands stop
- **Safe fallbacks**: System transitions to safe states on timing faults
- **Parameter validation**: Extensive checking prevents invalid configurations

This architecture ensures **sub-millisecond control loop timing** (15-30kHz) while supporting **high-bandwidth communication** (up to 5Mbps CAN-FD) with zero timing interference between control and communication domains.

This architecture provides a robust foundation for high-performance servo control while maintaining safety, reliability, and flexibility for diverse robotic applications.