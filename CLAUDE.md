# CLAUDE.md — Denso RC8 EtherCAT ROS2 Driver Project

## Project Overview
This project replaces the bCAP-based Denso robot ROS2 driver with an EtherCAT-based driver using the IgH EtherCAT Master and the `ethercat_driver_ros2` framework from ICube-Robotics. The project also integrates an ATI F/T (Force/Torque) sensor on the same EtherCAT bus.

**Robot:** DENSO RC8 Controller (RC8 ECS MOTION)
**Protocol:** EtherCAT, CiA 402 (IEC 61800-7-204)
**ROS2 Distribution:** Humble
**EtherCAT Master:** IgH EtherCAT Master (stable-1.6)
**Framework:** ethercat_driver_ros2 (ICube-Robotics)

## Repository Structure
```
denso_robot_ros2/                   # Top-level workspace src
├── CLAUDE.md                       # This file
├── ROADMAP.md                      # Development roadmap with phases
├── docs/
│   ├── ESI_ANALYSIS.md             # ESI XML analysis and PDO mapping reference
│   ├── CIA402_STATE_MACHINE.md     # CiA 402 state machine reference
│   └── TESTING_CHECKLIST.md        # Per-phase verification checklists
│
├── denso_ethercat_driver/          # [NEW] EtherCAT slave plugin for Denso RC8
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── config/
│   │   ├── denso_rc8_slave.yaml    # GenericEcSlave config OR custom plugin params
│   │   └── ati_ft_slave.yaml       # ATI F/T sensor slave config (Phase 7+)
│   ├── include/denso_ethercat_driver/
│   │   ├── denso_rc8_slave.hpp     # Custom EcSlave plugin for RC8
│   │   └── cia402_state_machine.hpp
│   ├── src/
│   │   ├── denso_rc8_slave.cpp
│   │   └── cia402_state_machine.cpp
│   └── denso_ethercat_driver.xml   # Plugin description (pluginlib)
│
├── denso_robot_hw/                 # [MODIFIED] ros2_control hardware interface
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── include/denso_robot_hw/
│   │   └── denso_robot_hw.hpp      # Now wraps EtherCAT instead of bCAP
│   └── src/
│       └── denso_robot_hw.cpp
│
├── denso_robot_description/        # [KEEP] URDF/xacro (add ros2_control tags)
│   └── urdf/
│       ├── denso_robot.urdf.xacro
│       └── denso_robot.ros2_control.xacro  # [NEW] EtherCAT hw interface config
│
├── denso_robot_bringup/            # [MODIFY] Launch files for EtherCAT
│   ├── launch/
│   │   └── denso_robot_bringup.launch.py
│   └── config/
│       ├── controllers.yaml
│       └── ethercat_config.yaml
│
├── denso_robot_moveit_config/      # [KEEP] MoveIt2 config
│
├── ati_ft_ethercat/                # [NEW, Phase 7+] ATI F/T sensor EtherCAT plugin
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── config/
│   │   └── ati_ft_slave.yaml
│   ├── include/ati_ft_ethercat/
│   │   └── ati_ft_slave.hpp
│   └── src/
│       └── ati_ft_slave.cpp
│
└── bcap_core/                      # [REMOVE eventually] Legacy bCAP packages
    bcap_service/
    denso_robot_core/
```

## ESI Device Summary (from RC8_ECS_MOTION_V1_0.xml)

| Property | Value |
|---|---|
| Vendor ID | 0x0000078A (DENSO WAVE) |
| Product Code | 0x00000001 |
| Revision | 0x00000001 |
| Profile | CiA 402, 8 channels |
| Supported Mode (ESI default) | CSP (Cyclic Synchronous Position, mode 0x08) |
| Supported Mode (vendor-unlocked) | CSV (Cyclic Synchronous Velocity, mode 0x09) |
| Min Cycle Time | 250,000 ns (250 µs → max 4 kHz) |
| Axes | 8 (J1–J8), index offsets: 0x60xx, 0x68xx, 0x70xx, 0x78xx, 0x80xx, 0x88xx, 0x90xx, 0x98xx |
| Default Modes of Operation | 0x08 (CSP) for all axes |

### CSV Mode — Vendor-Unlocked (IMPORTANT)

The stock ESI only declares CSP (0x08). The vendor (DENSO WAVE) has provided custom firmware
that additionally supports **CSV (Cyclic Synchronous Velocity, mode 0x09)**.

**Development strategy:**
1. Phases 1–5: Develop and stabilize using **CSP only**
2. Phase 5b (new): Add CSV command_interface alongside CSP
3. The active mode per axis is selected via SDO 0x6060 (Modes of Operation) BEFORE enabling

**Open question — PDO mapping for CSV:**
The current fixed RxPDO only contains `Controlword + Target Position` per axis.
CSV mode typically requires `Target Velocity` (0x60FF) in the cyclic PDO instead.
Possible scenarios (verify with DENSO):
- (A) Vendor firmware added an **alternate RxPDO** with Target Velocity — need updated ESI
- (B) Vendor firmware allows **reconfigurable PDO mapping** (PDO Assign/Config enabled)
- (C) In CSV mode, the existing Target Position field is **reinterpreted** as Target Velocity
- (D) Target Velocity is set via SDO (not cyclic) — functional but not ideal for real-time

→ **ACTION ITEM:** Ask DENSO which scenario applies. If (A) or (B), request updated ESI XML.

**Target Velocity SDO objects (already in ESI, marked PDO-mappable):**
| Axis | Target Velocity Index | Velocity Offset Index |
|------|----------------------|----------------------|
| J1 | 0x60FF | 0x60B1 |
| J2 | 0x68FF | 0x68B1 |
| J3 | 0x70FF | 0x70B1 |
| J4 | 0x78FF | 0x78B1 |
| J5 | 0x80FF | 0x80B1 |
| J6 | 0x88FF | 0x88B1 |
| J7 | 0x90FF | 0x90B1 |
| J8 | 0x98FF | 0x98B1 |

### CiA 402 Mode Codes
| Mode | Value | Description | Status |
|------|-------|-------------|--------|
| CSP | 0x08 | Cyclic Synchronous Position | Default, primary |
| CSV | 0x09 | Cyclic Synchronous Velocity | Vendor-unlocked, secondary |

### PDO Mapping Summary

**RxPDO (Master → Slave, SM2, 51 bytes total):**
- Per axis (×8): Controlword (UINT16) + Target Position (DINT32) = 6 bytes
- I/O: Mini IO (UINT16) + Hand IO (USINT8) = 3 bytes
- Total: 8×6 + 3 = 51 bytes

**TxPDO (Slave → Master, SM3, 84 bytes total):**
- Per axis (×8): Statusword (UINT16) + Position Actual (DINT32) + Current Actual (INT16) + Torque Ref (INT16) = 10 bytes
- I/O: Mini IO (UINT16) + Hand IO (USINT8) + Status IO (USINT8) = 4 bytes
- Total: 8×10 + 4 = 84 bytes

### Axis Index Mapping
| Axis | Controlword | Target Pos | Statusword | Pos Actual | Current | Torque Ref |
|------|-------------|------------|------------|------------|---------|------------|
| J1 | 0x6040 | 0x607A | 0x6041 | 0x6064 | 0x6078 | 0x6074 |
| J2 | 0x6840 | 0x687A | 0x6841 | 0x6864 | 0x6878 | 0x6874 |
| J3 | 0x7040 | 0x707A | 0x7041 | 0x7064 | 0x7078 | 0x7074 |
| J4 | 0x7840 | 0x787A | 0x7841 | 0x7864 | 0x7878 | 0x7874 |
| J5 | 0x8040 | 0x807A | 0x8041 | 0x8064 | 0x8078 | 0x8074 |
| J6 | 0x8840 | 0x887A | 0x8841 | 0x8864 | 0x8878 | 0x8874 |
| J7 | 0x9040 | 0x907A | 0x9041 | 0x9064 | 0x9078 | 0x9074 |
| J8 | 0x9840 | 0x987A | 0x9841 | 0x9864 | 0x9878 | 0x9874 |

## Key Technical Decisions

1. **EtherCAT Master:** IgH (not SOEM) — kernel-space, real-time capable, required by ethercat_driver_ros2.
2. **Mode:** CSP (Cyclic Synchronous Position, mode 0x08) is the PRIMARY mode — develop and stabilize this first. CSV (Cyclic Synchronous Velocity, mode 0x09) is vendor-unlocked and available as SECONDARY mode — integrate after CSP is proven stable.
3. **Mode switching:** The active mode is set via SDO write to `Modes of Operation` (0x6060 etc.) BEFORE entering OPERATION_ENABLED. Mode cannot be changed while in OP_ENABLED. To switch: disable → change mode → re-enable.
4. **Framework:** `ethercat_driver_ros2` provides the EthercatDriver SystemInterface that manages the IgH master. We write a custom EcSlave plugin for the RC8.
5. **Why custom plugin (not GenericEcSlave):** The RC8 is a multi-axis device (8 axes in one slave). The generic plugin maps one slave = one joint. We need a custom plugin that exposes 8 joints from a single slave device.
6. **Safety first:** Development proceeds axis-by-axis. Never enable all motors before single-axis verification passes.

## CiA 402 State Machine Reference
```
                    ┌─────────────────────┐
                    │  NOT READY TO        │
                    │  SWITCH ON           │
                    └──────────┬──────────┘
                               │ (automatic)
                    ┌──────────▼──────────┐
                    │  SWITCH ON           │
                    │  DISABLED            │
                    └──────────┬──────────┘
                               │ Controlword: 0x0006 (Shutdown)
                    ┌──────────▼──────────┐
                    │  READY TO            │
                    │  SWITCH ON           │
                    └──────────┬──────────┘
                               │ Controlword: 0x0007 (Switch On)
                    ┌──────────▼──────────┐
                    │  SWITCHED ON         │
                    └──────────┬──────────┘
                               │ Controlword: 0x000F (Enable Operation)
                    ┌──────────▼──────────┐
                    │  OPERATION           │
                    │  ENABLED             │
                    └─────────────────────┘
```

**Key Controlword bits:**
| Transition | Controlword | Mask |
|---|---|---|
| Shutdown | xxxx.xxxx.x0xx.0110 | 0x0006 |
| Switch On | xxxx.xxxx.x0xx.0111 | 0x0007 |
| Enable Operation | xxxx.xxxx.x0xx.1111 | 0x000F |
| Disable Voltage | xxxx.xxxx.x0xx.0000 | 0x0000 |
| Quick Stop | xxxx.xxxx.x0xx.0010 | 0x0002 |
| Fault Reset | xxxx.xxxx.1xxx.xxxx | 0x0080 |

**Key Statusword bits:**
| State | Statusword Mask | Statusword Value |
|---|---|---|
| Not ready to switch on | 0x004F | 0x0000 |
| Switch on disabled | 0x004F | 0x0040 |
| Ready to switch on | 0x006F | 0x0021 |
| Switched on | 0x006F | 0x0023 |
| Operation enabled | 0x006F | 0x0027 |
| Quick stop active | 0x006F | 0x0007 |
| Fault reaction active | 0x004F | 0x000F |
| Fault | 0x004F | 0x0008 |

## Development Rules

1. **Never skip a phase.** Each phase must be verified on real hardware before proceeding.
2. **Safety is paramount.** Always start with motors disabled. Use Quick Stop as emergency.
3. **One joint at a time.** When first testing motor control, start with J6 only (smallest inertia, lowest risk).
4. **Commit per phase.** Tag each completed phase: `phase-1-comms`, `phase-2-state-read`, etc.
5. **All code comments in English.** All documentation sections in English.
6. **Test on real hardware.** No simulation shortcuts for EtherCAT communication.

## Dependencies
```
# ROS2 packages
ros2_control
ros2_controllers
ethercat_driver_ros2      # https://github.com/ICube-Robotics/ethercat_driver_ros2
moveit2

# System
IgH EtherCAT Master (stable-1.6)
Linux kernel with PREEMPT_RT (recommended)
```

## Useful Commands
```bash
# Check EtherCAT slaves
ethercat slaves
ethercat pdos
ethercat upload --type uint16 -p 0 0x6041 0  # Read Statusword J1
ethercat download --type uint16 -p 0 0x6040 0 0x0006  # Write Controlword J1

# Build
cd ~/denso_ws && colcon build --symlink-install

# Launch (after Phase 5)
ros2 launch denso_robot_bringup denso_robot_bringup.launch.py
```

## Current Phase
> **Phase 0: Project Setup** — Setting up repository structure, ESI analysis, CLAUDE.md
