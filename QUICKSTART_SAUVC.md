# AMarineGUI Quick-Start Guide for SAUVC 2026 Missions

This guide explains how to use the AMarineGUI to run the SAUVC 2026 autonomous vehicle missions with full integration between Gazebo simulation, ArduSub flight control, MAVROS, and sauvc26-code mission planning.

## Prerequisites

Ensure you have all these components properly installed:

- **Gazebo Sim 8 (Harmonic)** with water physics plugins
- **ArduPilot/ArduSub** built and configured
- **MAVROS** installed and configured for ArduSub
- **ROS 2 Humble** with sauvc26-code built
- **sauvc26-world** symlinked to your ROS2 workspace

### Quick Setup Check

```bash
# Check Gazebo is installed
gazebo --version

# Check ROS2 is sourced
source ~/ros2_ws/install/setup.bash && ros2 --version

# Check sauvc26-code is built
ros2 pkg list | grep sauvc26

# Check ardupilot_gazebo plugin
ls ~/ardupilot_gazebo/build/libArduPilotPlugin.so
```

## Workflow: Running a SAUVC Mission

### Step 1: Launch the AMarineGUI

```bash
cd ~/amarine-gui
python3 command_gui.py
```

### Step 2: Start the Simulation Environment

In the GUI:

1. **Gazebo World Selection** (Top Left)
   - Select: `Headless (Final)` (for VMs) or `GUI (Final)` (for desktop with GPU)
   - Click **Start**
   - Wait for: `Starting Gazebo HEADLESS` message in console

2. **ArduSub + Gazebo** (Left Console)
   - Click **Start**
   - Wait for messages like:
     - `ArduSub starting...`
     - `Ready to fly`
     - Should connect to Gazebo automatically

3. **MAVROS** (Middle Console)
   - Click **Start**
   - Wait for:
     - `Connecting to FCU` message
     - Should show `Connected!`

### Step 3: Arm the Vehicle

In **ROS2** tab (Right side, bottom):

1. Select **Arm Vehicle** from dropdown
2. Click **Start**
3. Check output for: `[info] ARMED`

### Step 4: Run the Mission

Choose one of:

#### Option A: Qualification Mission
- From **ROS2** dropdown: Select **Qualification Mission**
- Click **Start**
- Vehicle will:
  1. Rotate to scan for gates
  2. Navigate through qualification gates
  3. Complete mission tasks

#### Option B: Final Mission
- From **ROS2** dropdown: Select **Final Mission**
- Click **Start**
- Vehicle will:
  1. Scan for gates
  2. Pass through gates
  3. Find and approach drums
  4. Navigate to flares in order
  5. Avoid obstacles
  6. Complete all tasks

### Step 5: Monitor the Mission

- **Pose Data**: Available via MAVROS topics (use RQT or Echo YOLO)
- **YOLO Detections**: Watch `/yolo_target_coord` topic
- **Vehicle Status**: Check console output in MAVROS tab

### Step 6: Stop the Mission

- Click **Kill** on any running console to stop
- Or click **Kill All** to stop everything

## Detailed Console Guide

### Top Section Controls

| Button | Purpose |
|--------|---------|
| **Gazebo World** Dropdown | Choose simulation environment |
| **Start/Kill** | Toggle Gazebo on/off |
| **Camera Bridge** | Bridge Gazebo camera to ROS2 topics |
| **Open RQT** | Launch RQT for visualization |
| **Kill All** | Emergency stop all processes |
| **Monitoring** | Shows CPU/GPU/RAM/Power/Temp usage |

### Bottom Consoles

1. **ArduSub + Gazebo** (Left)
   - Flight control system connected to Gazebo
   - Shows SITL (Software In The Loop) output
   - Command: `./run_ardupilot.sh`

2. **MAVROS** (Middle)
   - Bridge between ArduSub and ROS2
   - Publishes: `/mavros/local_position/pose`
   - Subscribes to: `/mavros/setpoint_raw/local`
   - Command: `ros2 launch mavros apm.launch fcu_url:=udp://:14550@localhost:14555`

3. **Vision** (Right)
   - Docker-based YOLO detection system
   - Publishes to: `/yolo_target_coord` (JSON format)
   - Files: `detect_ros.py`, `detect_ros_2.py`

4. **ROS2 Package Selection** (Bottom, left to right)
   - Build sauvc26_code
   - Run individual mission nodes (Arm, Qualification, Final)
   - Echo sensor data topics

## Command Reference

All commands automatically source the ROS2 environment. Here are the key ROS2 commands used:

### Build Commands
```bash
# Full build
cd ~/ros2_ws && colcon build --packages-select sauvc26_code

# Build with output
cd ~/ros2_ws && colcon build --packages-select sauvc26_code --event-handlers console_direct+
```

### Mission Commands
```bash
# Arm the vehicle
ros2 run sauvc26_code arm

# Qualification mission
ros2 run sauvc26_code qualification

# Final mission
ros2 run sauvc26_code final

# Test basic connectivity
ros2 run sauvc26_code test
```

### Sensor Commands
```bash
# Watch YOLO detections
ros2 topic echo /yolo_target_coord

# Watch vehicle position
ros2 topic echo /mavros/local_position/pose

# List all active topics
ros2 topic list
```

## Troubleshooting

### "Cannot connect to Gazebo"
- Check if `run_gazebo_headless.sh` executed successfully
- Verify `GZ_SIM_SYSTEM_PLUGIN_PATH` is set correctly
- Try cleaning Gazebo cache: `rm -rf ~/.gazebo ~/.cache/gazebo*`

### "MAVROS won't connect"
- Ensure ArduSub is fully initialized (wait for "Ready to fly")
- Check UDP port 14550 is listening: `netstat -tun | grep 14550`
- Restart MAVROS console

### "YOLO topics not publishing"
- Verify Vision Docker container is running
- Check if `docker ps` shows the vision container
- Verify `/yolo_target_coord` topic exists: `ros2 topic list | grep yolo`

### "Mission node won't run"
- Ensure vehicle is ARMED first
- Check if `/mavros/local_position/pose` topic is active
- Verify sauvc26_code is built: `ros2 pkg list | grep sauvc26`
- Build it if needed: **ROS2** tab → **Build sauvc26_code**

### "Everything crashes on VMs"
- Use **Headless** Gazebo mode instead of GUI
- Disable GPU monitoring in GUI (if causing issues)
- Check system resources: look at the Monitoring panel on the right

## Advanced: Manual Command Execution

You can run commands directly in terminal without the GUI:

```bash
# Terminal 1: Start Gazebo
cd ~/sauvc26-world
./run_gazebo_headless.sh ./worlds/sauvc_final.world

# Terminal 2: Start ArduSub
cd ~/sauvc26-world
./run_ardupilot.sh

# Terminal 3: Start MAVROS
source ~/ros2_ws/install/setup.bash
ros2 launch mavros apm.launch fcu_url:=udp://:14550@localhost:14555

# Terminal 4: Arm and run mission
source ~/ros2_ws/install/setup.bash
ros2 run sauvc26_code arm
sleep 2
ros2 run sauvc26_code final
```

## Mission Implementation Details

### Communication Flow

```
Gazebo (Physics Simulation)
  ↓ (Plugin) ↓
ArduSub (SITL Flight Control)
  ↓ (MAVLink over UDP) ↓
MAVROS (ROS2 Bridge)
  ↓ (ROS2 Topics) ↓
sauvc26-code (Mission Node)
  ↓ (Vision Input) ↓
YOLO (Docker - Object Detection)
```

### ROS2 Topics Used

| Topic | Type | Direction | Purpose |
|-------|------|-----------|---------|
| `/mavros/local_position/pose` | PoseStamped | Subscribe | Vehicle position |
| `/mavros/setpoint_raw/local` | PositionTarget | Publish | Velocity commands |
| `/yolo_target_coord` | String (JSON) | Subscribe | Object detections |

### State Machines

The mission nodes use state machines to progress through tasks:

**Qualification Mission States:**
1. Arm and descend to depth
2. Rotate to scan for gates
3. Navigate through gate 1
4. Navigate through gate 2
5. Return to surface

**Final Mission States:**
1. Arm and descend
2. Scan and find gate
3. Pass through gate
4. Find and approach first drum
5. Track and collect drums
6. Find and approach flares in order
7. Avoid obstacles
8. Surface and disarm

## Performance Tuning

### PID Controller Parameters (in mission code)

```python
# Depth control
KP_DEPTH = 0.5  # Proportional gain
KI_DEPTH = 0.1  # Integral gain
KD_DEPTH = 0.2  # Derivative gain

# Gate tracking
KP_GATE = 0.5
KI_GATE = 0.1
KD_GATE = 0.2
```

Modify these values in `sauvc26_code/qualification.py` or `final.py` to tune mission behavior.

## Next Steps

1. **Test connectivity**: Run `ros2 run sauvc26_code test`
2. **Practice armament**: Arm/disarm the vehicle a few times
3. **Test gate navigation**: Run qualification mission first
4. **Debug with RQT**: Use `rqt` to visualize TF tree, topics, and images
5. **Fine-tune PID values**: Adjust controller parameters based on mission performance

## Getting Help

If you encounter issues:

1. Check **Console Output**: Every command shows detailed logs
2. Use **RQT**: `Open RQT` button to visualize system state
3. Check **System Resources**: Monitor panel shows if system is overloaded
4. Verify **ROS2 Setup**: `echo $ROS_DOMAIN_ID` should be empty or 0
5. Check **Network**: `netstat -tun | grep 14550` for MAVLink connectivity

## Additional Resources

- [sauvc26-world README](../../sauvc26-world/README.md)
- [sauvc26-code repository](../../sauvc26-code/)
- [ArduPilot Documentation](https://ardupilot.org/)
- [MAVROS Documentation](http://wiki.ros.org/mavros)
- [ROS 2 Documentation](https://docs.ros.org/)

---

**Last Updated**: 2026-06-22  
**Version**: 2.1  
**Team**: AMarineUV - Singapore AUV Challenge 2026
