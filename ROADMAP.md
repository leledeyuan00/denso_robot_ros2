# ROADMAP.md — Denso RC8 EtherCAT ROS2 Driver

## Phase Overview

| Phase | Goal | Risk Level | Deliverable |
|-------|------|------------|-------------|
| 0 | Project setup & ESI analysis | None | Repo structure, docs, CLAUDE.md |
| 1 | EtherCAT communication (no motor) | Low | Stable PREOP/SAFEOP/OP transition |
| 2 | Read all joint states (motors disabled) | Low | Joint positions published to ROS2 |
| 3 | Single joint control (J6 only, CSP) | Medium | J6 CSP position control verified |
| 4 | All joints control (CSP) | High | 6-axis CSP control stable |
| 5 | ros2_control hardware interface (CSP) | Medium | Full ros2_control integration |
| 5b | CSV velocity mode integration | Medium | Dual-mode CSP+CSV, velocity control verified |
| 6 | MoveIt2 integration | Low | Cartesian planning and execution |
| 7 | ATI F/T sensor integration | Medium | Force/torque data on ROS2 topic |
| 8 | Combined multi-device EtherCAT | Medium | Robot + F/T on same bus, synchronized |

---

## Phase 0: Project Setup & ESI Analysis
**Goal:** Repository structure, documentation, dependency verification.

### Tasks
- [ ] Analyze ESI XML file completely (PDO mapping, object dictionary, data types)
- [ ] Create project repository structure
- [ ] Write CLAUDE.md with all ESI-derived constants and conventions
- [ ] Verify IgH EtherCAT Master is installed and can detect RC8 slave
- [ ] Verify `ethercat_driver_ros2` compiles in workspace
- [ ] Create skeleton ROS2 packages (CMakeLists.txt, package.xml)

### Verification
```bash
# IgH master loaded
lsmod | grep ec_master

# Slave detected
ethercat slaves
# Expected: 0  0:0  PREOP  +  RC8 ECS MOTION

# ethercat_driver_ros2 builds
cd ~/denso_ws && colcon build
```

### Claude Instructions (Phase 0)
```
Create the skeleton ROS2 package structure for denso_ethercat_driver.
This package will be an ethercat_driver_ros2 plugin.
Include: CMakeLists.txt, package.xml, plugin XML, empty header/source stubs.
Dependencies: ethercat_driver_ros2, pluginlib, rclcpp.
Do NOT implement any logic yet — only the build infrastructure.
```

---

## Phase 1: EtherCAT Communication (No Motor Control)
**Goal:** Establish stable EtherCAT communication. Transition slave to OP state. Verify cyclic PDO exchange without enabling any motor.

### Tasks
- [ ] Create custom EcSlave plugin class `DensoRc8Slave` inheriting from `EcSlave`
- [ ] Implement `configure()` — register all 8 axes of RxPDO/TxPDO domains
- [ ] Implement `processData()` — read TxPDO (statuswords, positions), write RxPDO (controlword=0x0000 for all axes, target_pos=current_pos)
- [ ] Create ros2_control URDF snippet that loads EthercatDriver with our plugin
- [ ] Create launch file for communication-only test
- [ ] Verify slave reaches OP state and stays there stably for >10 minutes

### Key Design Decisions
- All controlwords set to 0x0000 (Disable Voltage) — motors stay in "Switch On Disabled" state
- Target position echoes back current position (safe: even if accidentally enabled, no movement)
- Log all 8 statuswords every second for monitoring

### Verification
```bash
# Launch communication test
ros2 launch denso_robot_bringup ethercat_test.launch.py

# Check slave in OP
ethercat slaves
# Expected: 0  0:0  OP  +  RC8 ECS MOTION

# Monitor statuswords (should all show 0x0040 = Switch On Disabled, or 0x0050)
ros2 topic echo /denso_debug/statuswords
```

### Claude Instructions (Phase 1)
```
Implement the DensoRc8Slave EcSlave plugin for ethercat_driver_ros2.
Key requirements:
1. The RC8 is a SINGLE EtherCAT slave with 8 CiA 402 axes + I/O PDOs.
2. Register PDO entries matching the ESI exactly (see CLAUDE.md for addresses).
3. In processData(), write Controlword=0x0000 for ALL axes (keep disabled).
4. In processData(), copy current Position Actual Value to Target Position.
5. Read and store all Statuswords, Position Actual Values, Currents, Torques.
6. Expose debug state_interfaces for all statuswords.
7. Add a periodic log (every 1s) printing all 8 statuswords in hex.
Reference: ethercat_driver_ros2 EcSlave base class API.
Reference: ESI analysis in CLAUDE.md.
```

---

## Phase 2: Read All Joint States (Motors Disabled)
**Goal:** Publish joint states (position, velocity=0, effort=torque) to ROS2 while keeping all motors disabled. Verify sensor data makes physical sense.

### Tasks
- [ ] Expose `state_interface` for each joint: position, velocity (from 0x606C etc. if available, or compute from position delta), effort (torque reference)
- [ ] Wire into ros2_control `joint_state_broadcaster`
- [ ] Manually move robot arm (in teach mode / motor-off) and verify position readings change
- [ ] Verify position values make sense (check units: typically encoder counts, need conversion factor)
- [ ] Document position-to-radian conversion factor per joint
- [ ] Expose I/O state interfaces (Mini IO, Hand IO, Status IO)

### Important Notes
- Position values from ESI are DINT (32-bit signed integer) — likely encoder counts
- Need to determine encoder resolution per joint (counts per revolution)
- This info may come from the RC8 controller manual or by experiment
- Velocity is not directly in the default TxPDO; compute as `(pos[n] - pos[n-1]) / dt`

### Verification
```bash
ros2 launch denso_robot_bringup ethercat_test.launch.py

# Check joint states
ros2 topic echo /joint_states
# Move robot by hand → positions should change
# All velocities ≈ 0 when stationary
# All efforts ≈ 0 when motors off
```

### Claude Instructions (Phase 2)
```
Extend the DensoRc8Slave plugin to expose ros2_control state interfaces.
For each joint (joint_1 through joint_6, optionally joint_7, joint_8):
- state_interface: position (convert encoder counts to radians using a configurable factor)
- state_interface: velocity (compute from position difference / cycle time)
- state_interface: effort (from torque reference PDO value)
Also expose GPIO state interfaces for Mini IO, Hand IO, Status IO.
Create a launch file that starts:
- EthercatDriver hardware interface
- joint_state_broadcaster controller
Keep all motors DISABLED (controlword = 0x0000).
```

---

## Phase 3: Single Joint Control — J6
**Goal:** Enable and control J6 (last joint, typically smallest inertia) in CSP mode. Verify CiA 402 state machine transitions and basic position control.

### Tasks
- [ ] Implement CiA 402 state machine handler (per-axis)
- [ ] Add a `command_interface` for J6 position only
- [ ] Implement controlled state transitions: Disabled → Ready → Switched On → Operation Enabled
- [ ] On first enable: set target position = current position (no jump!)
- [ ] Test small position steps (±100 encoder counts)
- [ ] Implement fault detection and recovery (Fault → Fault Reset → re-enable)
- [ ] Add emergency stop: any anomaly → set all controlwords to Quick Stop (0x0002)
- [ ] Keep J1–J5 (and J7–J8) in Disabled state with target_pos = actual_pos

### Safety Measures
- **CRITICAL:** Before enabling J6, set its target_position = its current actual_position
- Hard-code a maximum position step per cycle (e.g., max 100 counts/cycle)
- If following error detected in Statusword → immediate Quick Stop all axes
- Timeout: if no valid command for 100ms → disable

### Verification
```bash
ros2 launch denso_robot_bringup ethercat_j6_test.launch.py

# Check J6 statusword transitions
# 0x0040 → 0x0021 → 0x0023 → 0x0027 (Operation Enabled)

# Send small position command
ros2 topic pub --once /forward_position_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0,0,0,0,0,0.01]}"
# J6 should move slightly

# Test fault recovery
# Manually trigger fault → verify auto-recovery works
```

### Claude Instructions (Phase 3)
```
Implement CiA 402 state machine management for the DensoRc8Slave.
Requirements:
1. Create a Cia402StateMachine class that tracks per-axis state.
2. It reads Statusword, determines current state, and outputs the correct Controlword.
3. State transitions: NOT_READY → SWITCH_ON_DISABLED → READY_TO_SWITCH_ON →
   SWITCHED_ON → OPERATION_ENABLED
4. On transition to OPERATION_ENABLED, latch current position as target.
5. Add a configurable parameter: enabled_axes (default: only J6, index 5).
6. Non-enabled axes always get Controlword=0x0000 and target=actual.
7. Implement fault detection: if statusword indicates FAULT, attempt reset once,
   then escalate to error.
8. Add maximum position step limit per cycle as a safety parameter.
9. If step exceeds limit, refuse the command and hold current position.
Reference: CiA 402 state machine table in CLAUDE.md.
```

---

## Phase 4: All Joints Control
**Goal:** Enable all robot joints (typically J1–J6 for a 6-axis Denso). Verify multi-axis CSP control is stable.

### Tasks
- [ ] Enable J1–J6 sequentially (one at a time, verify each)
- [ ] Verify all axes reach Operation Enabled
- [ ] Test coordinated motion: move all joints simultaneously
- [ ] Verify torque/current readings during motion
- [ ] Stress test: continuous small motions for 30+ minutes without fault
- [ ] Determine if J7/J8 are physical axes or reserved (depends on robot model)
- [ ] Tune position step limits per joint (different joints may need different limits)
- [ ] Implement coordinated enable/disable: all axes enable/disable together

### Verification
```bash
ros2 launch denso_robot_bringup ethercat_full.launch.py

# All 6 joints in OPERATION_ENABLED
ros2 topic echo /denso_debug/statuswords

# Move to a known position
ros2 topic pub --once /forward_position_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0,0,0,0,0,0]}"

# Run stability test (custom node that sends small sine waves)
ros2 run denso_ethercat_driver stability_test_node
```

### Claude Instructions (Phase 4)
```
Extend the DensoRc8Slave to support enabling all 6 (or 8) axes.
1. Add parameter: num_active_axes (default: 6).
2. Enable axes J1 through J<num_active_axes> in sequence.
3. All axes must reach OPERATION_ENABLED before accepting commands.
4. If any axis fails to reach OP_ENABLED within timeout → disable all, report error.
5. Add coordinated shutdown: on deactivate, transition all axes through
   OPERATION_ENABLED → SWITCHED_ON → READY_TO_SWITCH_ON → SWITCH_ON_DISABLED.
6. Create a stability test node that sends small sinusoidal position commands
   and monitors for faults over a configurable duration.
```

---

## Phase 5: ros2_control Hardware Interface Integration
**Goal:** Full integration with ros2_control. Support `forward_position_controller` and `joint_trajectory_controller`.

### Tasks
- [ ] Clean up hardware interface to expose standard joint interfaces
- [ ] Create proper ros2_control URDF xacro with EtherCAT parameters
- [ ] Configure and test `forward_position_controller`
- [ ] Configure and test `joint_trajectory_controller`
- [ ] Add proper lifecycle management (on_activate, on_deactivate, on_error)
- [ ] Handle controller switching gracefully
- [ ] Add I/O GPIO controller for Mini IO / Hand IO

### Verification
```bash
ros2 launch denso_robot_bringup denso_robot_bringup.launch.py

# List controllers
ros2 control list_controllers

# Forward position control
ros2 topic pub --once /forward_position_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0,0,1.57,0,0,0]}"

# Trajectory control
ros2 action send_goal /joint_trajectory_controller/follow_joint_trajectory \
  control_msgs/action/FollowJointTrajectory <goal>
```

### Claude Instructions (Phase 5)
```
Finalize the ros2_control hardware interface integration.
1. Create denso_robot.ros2_control.xacro that configures:
   - EthercatDriver plugin with master_id and control_frequency
   - DensoRc8Slave plugin with position, alias, slave_config path
   - Joint definitions with command_interface (position) and state_interfaces
     (position, velocity, effort)
   - GPIO for I/O (mini_io, hand_io, status_io)
2. Create controllers.yaml with:
   - joint_state_broadcaster
   - forward_position_controller (for testing)
   - joint_trajectory_controller (for MoveIt2)
3. Update the launch file to load hardware interface, spawn controllers.
4. Implement proper on_activate: transition CiA402 to OP_ENABLED.
5. Implement proper on_deactivate: transition CiA402 to SWITCH_ON_DISABLED.
```

---

## Phase 5b: CSV (Cyclic Synchronous Velocity) Mode
**Goal:** Add velocity command interface alongside CSP. Verify CSV mode works on J6 first, then all axes. This requires the vendor-unlocked firmware.

### Prerequisites
- [ ] **CRITICAL:** Confirm with DENSO how CSV PDO mapping works (see CLAUDE.md "Open question" section)
- [ ] Obtain updated ESI XML if vendor provides one
- [ ] CSP mode (Phase 5) fully stable

### Tasks
- [ ] Write to Modes of Operation (0x6060 etc.) via SDO to set mode 0x09 (CSV)
- [ ] Read back Modes of Operation Display (0x6061 etc.) to confirm mode accepted
- [ ] Determine RxPDO behavior in CSV mode:
  - If new PDO mapping: update PDO registration in plugin
  - If Target Position field reinterpreted as Target Velocity: handle in processData()
  - If separate SDO-only: implement SDO write path for Target Velocity
- [ ] Add `command_interface: velocity` for each joint in ros2_control
- [ ] Implement mode selection parameter: `control_mode` (csp | csv)
- [ ] Mode switch requires: disable axis → write new mode via SDO → re-enable axis
- [ ] Test J6 in CSV mode: send small constant velocity, verify smooth motion
- [ ] Test all axes in CSV mode
- [ ] Add velocity safety limits: max velocity per joint per cycle
- [ ] Add `forward_velocity_controller` to controllers.yaml
- [ ] Stress test CSV mode for 30+ minutes

### Safety Measures for CSV
- **CRITICAL:** In CSV mode, sending velocity=0 stops the motor. But unlike CSP, the robot drifts if the command is lost.
- Implement watchdog: if no velocity command for 2 cycles → ramp velocity to 0
- Maximum velocity clamp per joint (configurable parameter)
- On fault or communication loss → immediate velocity=0 for all axes

### Verification
```bash
ros2 launch denso_robot_bringup denso_robot_bringup.launch.py control_mode:=csv

# Verify mode is CSV
ros2 topic echo /denso_debug/modes_of_operation
# Expected: all enabled axes show 0x09

# Send small velocity command to J6
ros2 topic pub --once /forward_velocity_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0,0,0,0,0,0.1]}"
# J6 should rotate slowly

# Stop
ros2 topic pub --once /forward_velocity_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0,0,0,0,0,0]}"
```

### Claude Instructions (Phase 5b)
```
Read CLAUDE.md and ROADMAP.md Phase 5b.
Add CSV (Cyclic Synchronous Velocity, mode 0x09) support to DensoRc8Slave.
Requirements:
1. Add parameter: control_mode (string, default "csp", options: "csp" | "csv")
2. During configure(), if mode is "csv":
   - After reaching SWITCHED_ON state, write Modes of Operation = 0x09 via SDO
   - Read back Modes of Operation Display to confirm
   - Handle PDO differences based on vendor's answer (see CLAUDE.md open question)
3. Add command_interface: velocity for each joint (in addition to position)
4. In CSV processData():
   - Read velocity command from command_interface
   - Apply velocity safety clamp (max_velocity_per_cycle parameter)
   - Write to appropriate PDO field or SDO
5. Implement velocity watchdog: if no new command for N cycles, ramp to 0.
6. Add forward_velocity_controller config.
7. Keep CSP mode fully functional — mode is selected at startup, not runtime.
IMPORTANT: velocity=0 must always be the safe default. Never leave a non-zero
velocity command if communication is interrupted.
```

---

## Phase 6: MoveIt2 / Cartesian Control
**Goal:** Cartesian-space motion planning and execution using MoveIt2.

### Tasks
- [ ] Verify denso_robot_moveit_config works with new hardware interface
- [ ] Test planning and execution in RViz
- [ ] Verify trajectory scaling and velocity limits
- [ ] Test collision avoidance
- [ ] Document maximum joint velocities and accelerations (from ESI `Max Motor Speed` / `Max Acceleration`)

### Verification
```bash
ros2 launch denso_robot_bringup denso_robot_bringup.launch.py
ros2 launch denso_robot_moveit_config denso_robot_moveit.launch.py

# Plan and execute in RViz
```

### Claude Instructions (Phase 6)
```
Integrate with the existing denso_robot_moveit_config.
1. Ensure joint names in MoveIt config match ros2_control joint names.
2. Set velocity/acceleration limits from the ESI Max Motor Speed / Max Acceleration
   objects (0x6080, 0x60C5 etc.), readable via SDO.
3. Verify joint_trajectory_controller is compatible with MoveIt2 trajectory output.
4. Update launch file to optionally start MoveIt2.
```

---

## Phase 7: ATI F/T Sensor EtherCAT Integration
**Goal:** Add ATI Force/Torque sensor as a second EtherCAT slave on the same bus, using IgH master.

### Tasks
- [ ] Obtain/create ATI F/T sensor ESI file
- [ ] Analyze ATI EtherCAT PDO mapping (Fx, Fy, Fz, Tx, Ty, Tz)
- [ ] Create EcSlave plugin for ATI sensor (or use GenericEcSlave with YAML config)
- [ ] Add sensor to ros2_control URDF as a `<sensor>` element
- [ ] Publish `geometry_msgs/WrenchStamped` on a ROS2 topic
- [ ] Verify force/torque readings with known loads
- [ ] Apply calibration matrix if needed
- [ ] Test at full cycle rate

### Claude Instructions (Phase 7)
```
Create an ATI F/T sensor EtherCAT plugin for ethercat_driver_ros2.
1. If ATI uses standard CoE objects, try GenericEcSlave with YAML config first.
2. If custom logic needed (calibration matrix, bias removal), create AtiFtSlave plugin.
3. Expose 6 state_interfaces: force.x, force.y, force.z, torque.x, torque.y, torque.z.
4. Create a force_torque_sensor_broadcaster config to publish WrenchStamped.
5. The ATI sensor is a SEPARATE slave on the same EtherCAT bus (different position).
Reference: existing ATI_ROS2 repo for PDO mapping and calibration logic.
```

---

## Phase 8: Combined Multi-Device System
**Goal:** Robot + F/T sensor on same EtherCAT bus, synchronized operation.

### Tasks
- [ ] Configure EtherCAT bus with RC8 at position 0, ATI at position 1
- [ ] Verify both slaves reach OP state
- [ ] Test concurrent robot motion + force reading
- [ ] Implement force-reactive behaviors (optional: impedance control)
- [ ] Stress test combined system

### Verification
```bash
ros2 launch denso_robot_bringup denso_robot_full.launch.py

# Robot moving + force data streaming
ros2 topic echo /force_torque_sensor/wrench
ros2 topic echo /joint_states
```

---

## Claude Code Workflow

### Per-Phase Development Cycle
```
1. Read CLAUDE.md and ROADMAP.md for current phase context
2. Implement the phase deliverables
3. Build and fix compilation errors
4. Human verifies on real hardware
5. Human reports results back
6. Fix issues / iterate
7. Human confirms phase complete → tag commit → advance to next phase
```

### Claude Code Session Start Prompt Template
```
I am working on Phase N of the Denso RC8 EtherCAT ROS2 driver project.
Read CLAUDE.md and ROADMAP.md for full context.
Current status: [describe what's done and what needs to be done next]
The last hardware test showed: [results]
Please implement: [specific task from the phase]
```

### Important Constraints for Claude
- This is REAL HARDWARE. Wrong controlword values can cause uncontrolled motor movement.
- Always echo actual position to target position for disabled axes.
- Never enable a motor without explicit human confirmation.
- When in doubt, keep motors disabled and ask for guidance.
- All position values are encoder counts (DINT32). Conversion to radians needs a per-joint factor that will be determined experimentally in Phase 2.
