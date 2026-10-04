# Autonomous Warehouse Inventory Rover: Embedded Systems

Firmware and electronics for an autonomous rover that navigates a warehouse and scans QR codes across multi-height racks. Built for Inter IIT Tech Meet 2025.

![MCU](https://img.shields.io/badge/MCU-STM32F446RE-03234B?style=flat-square&logo=stmicroelectronics&logoColor=white)
![RTOS](https://img.shields.io/badge/RTOS-FreeRTOS-5D9C3F?style=flat-square)
![Middleware](https://img.shields.io/badge/Middleware-micro--ROS-22314E?style=flat-square&logo=ros&logoColor=white)
![ROS 2](https://img.shields.io/badge/ROS_2-Jazzy-22314E?style=flat-square&logo=ros&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square)

## Scope

This repository covers the embedded layer of the rover: real-time motor control, Z-axis actuation, encoder feedback, power electronics, PCB design, and the micro-ROS link between the STM32 and a Raspberry Pi 5 running ROS 2. SLAM, Nav2, and the QR vision pipeline run on the Pi and were developed separately by other team members.

## My Contributions

- **PCB design:** power management board, H-bridge motor driver, quadrature encoder interface, and protection circuitry, with attention to grounding and EMI
- **Firmware:** FreeRTOS application on the STM32F446RE running as a micro-ROS node
- **Drive control:** PID velocity loop per wheel with interrupt-driven quadrature encoder feedback
- **Z-axis control:** non-blocking stepper state machine driving a NEMA 23 lead-screw axis through a DM556 driver
- **Safety:** independent watchdog (IWDG) and command timeout that stop the motors on MCU hang or loss of ROS communication
- **Bring-up and validation:** oscilloscope and logic analyzer debugging; UART, I2C, and SPI testing against the Raspberry Pi

## System Architecture

```
Raspberry Pi 5 (ROS 2 Jazzy: Nav2, SLAM, vision)
        │  UART 115200, micro-ROS agent
        ▼
STM32F446RE (FreeRTOS + micro-ROS)
  ├── Drive task     /cmd_vel → PID (50 ms) → PWM → H-bridge → DC motors
  ├── Odometry task  encoder ISR → tick count → /odom
  ├── Stepper task   /cmd_stepper → step/dir state machine → DM556 → NEMA 23
  └── Safety         IWDG + command timeout → motor cutoff
```

## ROS Interface

| Direction | Topic | Type | Purpose |
|---|---|---|---|
| Subscribe | `/cmd_vel` | `geometry_msgs/TwistStamped` | Velocity commands from Nav2 |
| Subscribe | `/cmd_stepper` | `std_msgs/Int32` | Z-axis position commands |
| Publish | `/odom` | `nav_msgs/Odometry` | Wheel odometry |

## Design Rationale

Low-level control runs on the MCU rather than the Pi for three reasons. The control loop needs a fixed period that the Linux scheduler cannot guarantee under SLAM and vision load. Encoder edges need hardware interrupts, since polling GPIO from Linux misses ticks at speed. And a dedicated MCU with a hardware watchdog can stop the motors independently if the Pi or the link fails.

## Hardware

| Subsystem | Component |
|---|---|
| MCU | STM32 Nucleo-F446RE (Cortex-M4F, 180 MHz, hardware FPU) |
| Drive | Differential drive, brushed DC motors, custom H-bridge, PWM |
| Feedback | Quadrature rotary encoders, interrupt-driven |
| Z-axis motor | NEMA 23 stepper (PR57HS51-2804-R) |
| Z-axis driver | DM556 microstepping driver, 22.2 V supply |
| Z-axis mechanics | Lead screw, 8.23 mm travel per revolution, 1.4 m stroke (170 rev) |
| Power | Custom power management PCB, power distribution board, boost converter, UBEC for the Pi |
| Protection | Fusing, reverse-polarity protection |

## Performance

| Metric | Value |
|---|---|
| Control loop period | 50 ms |
| Encoder tracking | No missed ticks up to 0.5 m/s |
| Z-axis stroke | 1.4 m |
| Watchdog timeout | 500 ms (configurable) |
| Link | UART, 115200 baud |

## Build and Flash

Requirements: STM32CubeIDE, ST-LINK (onboard Nucleo), ROS 2 Jazzy with the micro-ROS agent on the Pi or host.

```bash
git clone https://github.com/YuvanIII/autonomous-rover-warehouse-embedded.git
```

1. In STM32CubeIDE: File → Open Projects from File System → select `firmware/`
2. Project → Build All
3. Run → Debug with the Nucleo connected over USB

Start the agent on the Pi:

```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyACM0 -b 115200
```

Once the agent connects, the MCU starts publishing `/odom` and accepting `/cmd_vel` and `/cmd_stepper`.

## Repository Structure

```
firmware/
  Core/Src/
    main.c              FreeRTOS and peripheral init
    motor_control.c     PID and PWM drive
    encoder.c           Quadrature encoder ISR
    stepper_control.c   Z-axis state machine
    microros_node.c     Publishers and subscribers
  Core/Inc/             Headers
pcb/
  power_management/
  motor_driver/
  encoder_interface/
docs/
  schematics/
  component_selection.md   BOM and selection rationale
```

## License

MIT. See `LICENSE`.
