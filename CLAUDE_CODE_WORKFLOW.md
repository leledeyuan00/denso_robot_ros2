# CLAUDE_CODE_WORKFLOW.md — How to use Claude Code for this project

## Setup

### 1. Place CLAUDE.md at workspace root
```bash
cp CLAUDE.md ~/denso_ws/src/denso_robot_ros2/CLAUDE.md
```
Claude Code automatically reads CLAUDE.md from the project root on every session.

### 2. Recommended .claude directory structure
```
~/denso_ws/src/denso_robot_ros2/
├── CLAUDE.md               # Main context file (auto-read)
├── ROADMAP.md              # Development roadmap
├── .claude/
│   └── settings.json       # Claude Code project settings
└── docs/
    ├── ESI_ANALYSIS.md     # Detailed ESI reference
    └── TESTING_CHECKLIST.md
```

### 3. .claude/settings.json
```json
{
  "permissions": {
    "allow": [
      "Read(*)",
      "Write(~/denso_ws/src/denso_robot_ros2/**)",
      "Bash(colcon build*)",
      "Bash(ros2 *)",
      "Bash(ethercat *)",
      "Bash(cat *)",
      "Bash(grep *)",
      "Bash(find *)",
      "Bash(ls *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ]
  }
}
```

## Per-Phase Prompts

### Phase 0 — Kick-off
```
Read CLAUDE.md and ROADMAP.md. We are starting Phase 0.
Create the skeleton package structure for `denso_ethercat_driver`:
- CMakeLists.txt with dependencies on ethercat_driver_ros2, pluginlib, rclcpp
- package.xml
- Empty header: include/denso_ethercat_driver/denso_rc8_slave.hpp
- Empty source: src/denso_rc8_slave.cpp
- Plugin XML: denso_ethercat_driver.xml
The plugin class will be DensoRc8Slave inheriting from ethercat_driver::EcSlave.
Just the build skeleton — no implementation yet.
Verify it compiles with: colcon build --packages-select denso_ethercat_driver
```

### Phase 1 — EtherCAT Communication
```
Read CLAUDE.md (especially the PDO mapping tables) and ROADMAP.md Phase 1.
Implement the DensoRc8Slave plugin:
1. In configure(): register all PDO entries from ESI_ANALYSIS.md byte offset maps.
   - 9 RxPDOs (8 axes + 1 I/O) mapped to SM2
   - 9 TxPDOs (8 axes + 1 I/O) mapped to SM3
2. In processData():
   - Read all 8 statuswords and 8 position actual values from TxPDO
   - Write controlword = 0x0000 for all 8 axes (keep disabled)
   - Write target_position = position_actual for all 8 axes (echo back)
   - Read I/O states
3. Log statuswords every 1 second at INFO level.
4. Create a test launch file that brings up EthercatDriver with this plugin.
Safety: No motor enable. All controlwords are 0x0000.
```

### Phase 2 — Joint State Reading
```
Read CLAUDE.md and ROADMAP.md Phase 2.
Extend DensoRc8Slave to expose ros2_control state_interfaces:
- joint_1/position through joint_6/position (DINT32 encoder counts → double)
- joint_1/velocity through joint_6/velocity (computed from position delta)
- joint_1/effort through joint_6/effort (from torque reference, INT16 → double)
- gpio/mini_io_in, gpio/hand_io_in, gpio/status_io (from TxPDO I/O)
Add parameter: encoder_counts_per_radian (array of 6 doubles, default all 1.0)
— We will calibrate this experimentally.
Create launch file with joint_state_broadcaster.
Motors stay DISABLED.
```

### Phase 3 — J6 Single Axis Control
```
Read CLAUDE.md (CiA 402 state machine section) and ROADMAP.md Phase 3.
Implement CiA 402 state machine management:
1. Create cia402_state_machine.hpp/cpp with:
   - Enum: Cia402State {NOT_READY, SWITCH_ON_DISABLED, READY_TO_SWITCH_ON,
     SWITCHED_ON, OPERATION_ENABLED, QUICK_STOP_ACTIVE, FAULT_REACTION, FAULT}
   - Method: decode_statusword(uint16_t sw) → Cia402State
   - Method: get_transition_controlword(Cia402State current, Cia402State target) → uint16_t
   - Safety: max_position_step_per_cycle parameter
2. In DensoRc8Slave:
   - Add parameter: enabled_axes (default: [5]) — only J6 enabled
   - For enabled axes: run state machine to transition to OPERATION_ENABLED
   - On first OPERATION_ENABLED: latch target_position = actual_position
   - Accept position commands only from command_interface
   - Clamp position step to max_position_step_per_cycle
   - For disabled axes: controlword=0x0000, target=actual (unchanged)
3. Add fault handling: if any axis enters FAULT, attempt one reset, then error.
IMPORTANT: Set target = actual BEFORE enabling. Never leave target at 0.
```

### Phase 4 — All Joints
```
Read CLAUDE.md and ROADMAP.md Phase 4.
Extend to support all 6 joints (J1–J6).
1. Change enabled_axes parameter default to [0,1,2,3,4,5]
2. Enable axes SEQUENTIALLY: J1 first, then J2, etc. Wait for each to reach
   OPERATION_ENABLED before enabling the next.
3. If any axis fails → disable all and report error.
4. Add coordinated disable on shutdown.
5. Create a stability_test_node that:
   - Reads current positions
   - Sends small sinusoidal offsets (configurable amplitude, default ±0.01 rad)
   - Monitors for faults
   - Runs for configurable duration (default 60s)
   - Reports: max following error, fault count, timing jitter
```

### Phase 5 — Full ros2_control
```
Read CLAUDE.md and ROADMAP.md Phase 5.
Create the full ros2_control integration:
1. denso_robot.ros2_control.xacro — EtherCAT hardware interface definition
2. controllers.yaml — joint_state_broadcaster, forward_position_controller,
   joint_trajectory_controller
3. Update denso_robot_bringup.launch.py:
   - Replace bCAP-related nodes with EtherCAT driver
   - Spawn controllers
   - Add parameter for controller selection
4. Implement lifecycle: on_activate enables motors, on_deactivate disables them.
5. Handle controller switching: if active controller changes, hold position during switch.
```

### Phase 5b — CSV Velocity Mode
```
Read CLAUDE.md (especially the CSV mode section and open question about PDO mapping)
and ROADMAP.md Phase 5b. CSP mode is already stable from Phase 5.
Add CSV (Cyclic Synchronous Velocity, mode 0x09) support:
1. Add parameter: control_mode (string: "csp" or "csv", default "csp")
2. If "csv": before enabling, write 0x09 to Modes of Operation via SDO for each axis.
   Read back Modes of Operation Display to confirm.
3. [DEPENDS ON VENDOR ANSWER about PDO mapping — fill in after confirmation]
4. Add command_interface: velocity for each joint.
5. In CSV processData(): read velocity command, apply max_velocity clamp, write to PDO/SDO.
6. Velocity watchdog: if no command for 2 cycles → ramp velocity to 0 linearly.
7. Add forward_velocity_controller to controllers.yaml.
8. Test J6 only first, then all axes.
SAFETY: velocity=0 is always the safe default. On ANY error, all velocities go to 0.
```

### Phase 7 — ATI F/T Sensor
```
Read CLAUDE.md and ROADMAP.md Phase 7.
We need to add an ATI F/T sensor as a second EtherCAT slave.
The ATI sensor ESI file is: [provide path]
1. Analyze ATI ESI to determine PDO mapping for Fx,Fy,Fz,Tx,Ty,Tz.
2. Decide: GenericEcSlave with YAML config, or custom AtiFtSlave plugin?
3. Add to ros2_control URDF as a <sensor> element.
4. Add force_torque_sensor_broadcaster to controllers.yaml.
5. The sensor is at EtherCAT bus position 1 (RC8 is position 0).
Reference: the existing ATI_ROS2 repo (SOEM-based) for PDO structure.
```

## Communication Protocol with Claude Code

### Reporting Hardware Test Results
After each hardware test, report to Claude Code like this:
```
Phase 1 hardware test results:
- ethercat slaves output: [paste]
- Slave reached OP: yes/no
- Statusword readings: [paste hex values for each axis]
- Duration stable: X minutes
- Errors encountered: [describe]
- Ready to proceed to Phase 2: yes/no
```

### Reporting Compilation Issues
```
Build failed. Error output:
[paste colcon build error]
The file causing the issue is: [filename]
Please fix the compilation error.
```

### Requesting Code Review
```
Phase N implementation is complete and compiles.
Before hardware testing, please review:
1. All controlword writes are safe (no accidental motor enable)
2. All target positions are initialized from actual positions
3. Error handling covers all fault states
4. No race conditions in state machine transitions
```

## Safety Review Checklist (for every code change)

Before ANY hardware test, verify:
- [ ] Default controlword for all axes is 0x0000 (Disable Voltage)
- [ ] Target position is initialized from actual position, never from 0
- [ ] Maximum position step per cycle is enforced (CSP mode)
- [ ] Maximum velocity per cycle is enforced (CSV mode)
- [ ] Velocity watchdog: no command for N cycles → ramp to 0 (CSV mode)
- [ ] Fault detection checks every statusword every cycle
- [ ] Quick Stop path is implemented and tested
- [ ] No axis is enabled without explicit configuration parameter
- [ ] Watchdog timeout is configured (what happens if ROS2 node crashes?)
- [ ] On communication loss: CSP holds last position, CSV sends velocity=0
- [ ] Mode of Operation is written BEFORE enabling, not during OPERATION_ENABLED
