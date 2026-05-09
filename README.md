# Autonomous Rover for Warehouse Inventory — Embedded Systems 🏭⚙️

> Real-time embedded firmware and electronics for an autonomous warehouse inventory scanning rover — built for Inter IIT Tech Meet 2025.

[![STM32](https://img.shields.io/badge/MCU-STM32%20Nucleo%20F446RE-blue.svg)](https://www.st.com/en/microcontrollers-microprocessors/stm32f446re.html)
[![Micro-ROS](https://img.shields.io/badge/Middleware-Micro--ROS-brightgreen.svg)](https://micro.ros.org/)
[![FreeRTOS](https://img.shields.io/badge/RTOS-FreeRTOS-orange.svg)](https://www.freertos.org/)
[![C/C++](https://img.shields.io/badge/Language-C%2FC%2B%2B-blue.svg)](https://isocpp.org/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 🧠 About This Repository

This repository contains the **embedded systems and electronics** work for the autonomous warehouse inventory rover built at **Inter IIT Tech Meet 2025**.

The rover autonomously navigates a warehouse and scans QR codes across multi-height racks. This repo covers the hardware brain of the system — real-time motor control, PCB design, power electronics, Z-axis actuation, encoder feedback, and Micro-ROS communication bridging the STM32 to the Raspberry Pi 5 running ROS 2.

> **Note:** The full system also includes SLAM, Navigation (Nav2), and a QR vision pipeline running on the Raspberry Pi 5 — handled separately by other team members.

---

## ✨ My Contributions

- **Custom PCB Design** — Power management board, motor driver circuits (H-bridge), quadrature encoder interfaces, and safety circuits with proper grounding and EMI considerations
- **STM32 Embedded Firmware** — Real-time control on STM32 Nucleo F446RE using FreeRTOS and Micro-ROS
- **DC Motor Control** — PID velocity control loop with interrupt-based quadrature encoder feedback
- **Z-Axis Stepper Control** — Non-blocking stepper state machine using `micros()` timers for NEMA 23 lead-screw actuation
- **Micro-ROS Integration** — STM32 publishes encoder odometry and subscribes to `/cmd_vel` and `/cmd_stepper` topics via micro-ROS agent on the Pi
- **Safety Systems** — Independent IWDG (watchdog) timer that cuts motors automatically on MCU hang or ROS communication loss
- **Validation & Debugging** — Hardware bring-up and signal validation using oscilloscopes and logic analyzers; UART/I2C/SPI interface testing with Raspberry Pi

---

## 🔧 Hardware

### Microcontroller
| Parameter | Value |
|---|---|
| MCU | STM32 Nucleo F446RE |
| Clock Speed | 180 MHz |
| FPU | Yes (for PID computation) |
| Communication | UART (micro-ROS), I2C, SPI |

### Motor System
| Component | Specification |
|---|---|
| DC Drive Motors | Differential drive, PWM-controlled via H-bridge |
| Wheel Encoders | Quadrature rotary encoders (interrupt-driven) |
| Z-Axis Motor | NEMA 23 PR57HS51-2804-R Stepper |
| Stepper Driver | DM556 Microstepper Driver |
| Z-Axis Power | 22.2V |
| Lead Screw Pitch | 2mm (8.23mm/rotation) |
| Z Travel | 1.4m (170 rotations) |

### Power & PCB
- Custom power management PCB with proper grounding planes and EMI shielding
- H-bridge motor driver circuits for DC wheel motors
- Quadrature encoder interface circuits
- Safety circuits with fusing and reverse-polarity protection
- Boost converter for voltage step-up
- UBEC voltage regulator for Raspberry Pi power supply
- Power Distribution Board (PDB)

---

## 💻 Firmware Architecture

```
FreeRTOS Tasks
├── Task: cmd_vel Subscriber        ← Receives velocity from ROS 2 Nav2
│     └── PID Loop (50ms)          ← Compares target vs actual velocity
│           └── PWM Output         ← Drives H-bridge / DC motors
│
├── Task: Encoder Publisher         ← Reads quadrature ticks via interrupt
│     └── /odom topic              ← Sends real-time odometry to Pi
│
├── Task: Stepper Controller        ← Receives /cmd_stepper from Pi
│     └── State Machine            ← Non-blocking micros() pulse generation
│           └── NEMA 23 Z-axis     ← Up/Down actuation (170 rotations)
│
└── Watchdog Timer (IWDG)          ← Auto motor cutoff on hang/ROS loss
```

---

## 🔌 Micro-ROS Communication

The STM32 runs as a **Micro-ROS node** communicating with the Raspberry Pi 5 over UART via the micro-ROS agent.

| Direction | Topic | Message Type | Description |
|---|---|---|---|
| Subscribe | `/cmd_vel` | `TwistStamped` | Velocity commands from Nav2 |
| Subscribe | `/cmd_stepper` | `Int32` | Z-axis stepper commands |
| Publish | `/odom` | `Odometry` | Wheel encoder odometry |

---

## ⚡ Why Micro-ROS on STM32?

| Requirement | Standard ROS 2 on Linux | Micro-ROS on STM32 |
|---|---|---|
| Real-Time Determinism | 10–100ms jitter (Linux scheduler) | Hard 50ms guarantee |
| Hardware Interrupts | GPIO latency → missed encoder ticks | Zero-latency interrupts |
| Safety & Redundancy | Motors may keep running on crash | IWDG watchdog auto-cutoff |
| CPU Overhead | High I/O usage steals from Vision/SLAM | Distributed — offloads Pi completely |

---

## 🚀 Getting Started

### Prerequisites

- STM32CubeIDE (for building and flashing firmware)
- micro-ROS agent running on Raspberry Pi 5 (or host PC)
- ROS 2 Jazzy on the companion computer

### Flashing the Firmware

1. **Clone the repository:**

```bash
git clone https://github.com/YuvanIII/autonomous-rover-warehouse-embedded.git
cd autonomous-rover-warehouse-embedded
```

2. **Open in STM32CubeIDE:**

```
File → Open Projects from File System → select /firmware folder
```

3. **Build and flash:**

```
Project → Build All
Run → Debug (with STLink connected to Nucleo F446RE)
```

### Starting Micro-ROS Agent (on Raspberry Pi)

```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyACM0 -b 115200
```

Once connected, the STM32 will automatically start publishing `/odom` and listening to `/cmd_vel` and `/cmd_stepper`.

---

## 📂 Project Structure

```
autonomous-rover-warehouse-embedded/
├── firmware/
│   ├── Core/
│   │   ├── Src/
│   │   │   ├── main.c                  # FreeRTOS task init
│   │   │   ├── motor_control.c         # PID + PWM motor driver
│   │   │   ├── encoder.c               # Quadrature encoder ISR
│   │   │   ├── stepper_control.c       # Z-axis state machine
│   │   │   └── microros_node.c         # Micro-ROS publishers/subscribers
│   │   └── Inc/
│   │       ├── motor_control.h
│   │       ├── encoder.h
│   │       └── stepper_control.h
│   └── CMakeLists.txt
├── pcb/
│   ├── power_management/               # Power PCB design files
│   ├── motor_driver/                   # H-bridge PCB design files
│   └── encoder_interface/             # Encoder interface circuits
├── docs/
│   ├── schematics/                     # Circuit schematics
│   └── component_selection.md         # BOM and selection rationale
└── README.md
```

---

## 📊 Performance

| Metric | Value |
|---|---|
| Control Loop Period | 50ms (hard real-time) |
| Encoder Resolution | Zero missed ticks up to 0.5 m/s |
| Z-Axis Travel | 1.4m (170 stepper rotations) |
| Z-Axis Resolution | 8.23mm/rotation (2mm pitch lead screw) |
| Watchdog Timeout | Configurable (default 500ms) |
| UART Baud Rate | 115200 bps (micro-ROS) |

---

## 🤝 Contributing

Contributions are welcome and greatly appreciated.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📜 License

Distributed under the MIT License. See `LICENSE` for more information.
