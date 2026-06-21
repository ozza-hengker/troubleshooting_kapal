# Amarine Command GUI v2.1

GUI to easily run frequently used ROS2, Gazebo, Vision, and ArduPilot commands with full SAUVC 2026 mission support.

## Features

- **Integrated Gazebo Simulation**: Headless and GUI modes for pool simulation
- **ArduSub Flight Control**: Software-in-the-loop (SITL) connected to Gazebo
- **MAVROS Bridge**: ROS2 interface to ArduSub
- **Mission Execution**: Run qualification and final mission profiles
- **Vision Integration**: Docker-based YOLO object detection
- **System Monitoring**: Real-time CPU, GPU, RAM, power, and temperature tracking
- **Multi-Console Interface**: Parallel execution and monitoring of multiple systems

## Architecture

```
Gazebo (Simulation) ← Plugin ← ArduSub (SITL)
                                    ↓ (MAVLink UDP)
                              MAVROS (Bridge)
                                    ↓ (ROS2)
                    sauvc26-code (Mission) ← YOLO (Detection)
```

## Quick Start

### 🎯 New Users: Fresh Git Clone?

**Start here**: [SETUP_FROM_GIT.md](./SETUP_FROM_GIT.md)

Complete setup guide for fresh git clones:
```bash
git clone https://github.com/FechL/sauvc26-world.git
git clone https://github.com/FechL/sauvc26-code.git
git clone https://github.com/FechL/amarine-gui.git
```

Then follow **[SETUP_FROM_GIT.md](./SETUP_FROM_GIT.md)** step-by-step.

### ⚡ Quick Launch (After Setup)

The GUI uses dynamic paths, so it works from any location:

```bash
cd ~/amarine-gui
python3 command_gui.py
```

Or create an alias in your `~/.bashrc`:
```bash
alias amarine='cd ~/amarine-gui && python3 command_gui.py'
```

### 2. Running a Mission

**Terminal 1: Start GUI**
```bash
amarine
```

**In GUI:**
1. Select **Gazebo World** → `Headless (Final)` → Click **Start**
2. In **ArduSub + Gazebo** console → Click **Start**
3. In **MAVROS** console → Click **Start**
4. Wait for all systems to initialize
5. In **ROS2** tab → Select **Arm Vehicle** → Click **Start**
6. In **ROS2** tab → Select **Final Mission** → Click **Start**
7. Monitor progress in the consoles

### 3. Full Documentation

| Document | Purpose |
|----------|---------|
| **[SETUP_FROM_GIT.md](./SETUP_FROM_GIT.md)** | 👈 **START HERE** - Fresh git clone setup |
| [QUICKSTART_SAUVC.md](./QUICKSTART_SAUVC.md) | Mission workflow & examples |
| [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) | Common errors & fixes |
| [README.md](./README.md) | This file - Features & architecture |

## System Requirements

- **ROS 2 Humble** with Gazebo Sim 8 (Harmonic)
- **ArduPilot/ArduSub** built for SITL simulation
- **MAVROS** installed: `sudo apt install ros-humble-mavros*`
- **sauvc26-code** built in ROS2 workspace
- **sauvc26-world** available (physics simulation models)
- **ardupilot_gazebo** plugin compiled

### Verify Installation

```bash
# Check all dependencies
gazebo --version      # Should be >= 8.0
ros2 --version        # Should be Humble
ros2 pkg list | grep sauvc26    # Should list sauvc26_code
ls ~/ardupilot_gazebo/build/libArduPilotPlugin.so  # Should exist
```

## GUI Layout

### Top Section (Control Panel)

| Component | Purpose |
|-----------|---------|
| **Gazebo World** | Select simulation environment (Headless or GUI) |
| **Camera Bridge** | Connect Gazebo camera to ROS2 `/front_camera` topic |
| **Open RQT** | Launch RQT for visualization and debugging |
| **Kill All** | Emergency stop all running processes |
| **Monitoring Panel** | Real-time system statistics (CPU, GPU, RAM, Power, Temp) |

### Bottom Section (Consoles)

1. **ArduSub + Gazebo** (Left)
   - Flight control connected to physics simulation
   - SITL output and initialization messages

2. **MAVROS** (Middle)
   - ROS2 bridge to flight control
   - Publishes position, publishes velocity commands
   - Connection status and diagnostics

3. **Vision** (Right)
   - Docker-based YOLO detection
   - Two detection models available
   - Publishes to `/yolo_target_coord`

4. **ROS2 Packages** (Bottom, expandable)
   - Build sauvc26_code
   - Arm vehicle
   - Run missions (Qualification, Final)
   - Echo sensor data

## Commands and Integration

### Gazebo Options
- `Headless (Qualification)`: Run qualification pool without GUI
- `Headless (Final)`: Run final pool without GUI
- `GUI (Qualification)`: Render qualification pool with graphics (requires GPU)
- `GUI (Final)`: Render final pool with graphics (requires GPU)

### ArduSub Options
- `ArduSub + Gazebo`: Direct SITL connected to Gazebo (recommended)
- `SITL`: Standalone SITL without Gazebo connection

### ROS2 Mission Commands
- `Build sauvc26_code`: Compile mission planning code
- `Arm Vehicle`: Arm the AUV before mission
- `Qualification Mission`: Run qualification competition
- `Final Mission`: Run final competition
- `Echo YOLO Target`: Subscribe to detection topic for debugging

## Customization

### Adding New Commands

Edit `command_gui.py` and update the `COMMANDS` dictionary:

```python
COMMANDS = {
    "YourCategory": {
        "Your Command Name": f"source {WORKSPACE_ROOT}/ros2_ws/install/setup.bash && your-command-here",
        # Add more commands...
    },
    # ...
}
```

### Modifying System Commands

The following scripts in `~/sauvc26-world/` are used and can be customized:
- `run_gazebo_headless.sh`: Headless Gazebo with environment setup
- `run_gazebo.sh`: GUI Gazebo with environment setup
- `run_ardupilot.sh`: ArduSub SITL with Gazebo integration

All scripts use dynamic path resolution, so they work from any location.

## Troubleshooting

### GUI Won't Start
```bash
# Check dependencies
pip3 list | grep PyQt
# If missing, install:
pip3 install -r requirements.txt
```

### Gazebo Won't Connect
```bash
# Check plugin path
echo $GZ_SIM_SYSTEM_PLUGIN_PATH
# Should show: /path/to/ardupilot_gazebo/build

# Clean Gazebo cache
rm -rf ~/.gazebo ~/.cache/gazebo*
```

### MAVROS Connection Failed
```bash
# Check UDP port
netstat -tun | grep 14550
# Should show listening port

# Check ROS2 domain
echo $ROS_DOMAIN_ID  # Should be empty or 0

# Restart MAVROS console in GUI
```

### Mission Won't Run
```bash
# Verify vehicle is armed
ros2 run sauvc26_code test

# Check pose topic is publishing
ros2 topic echo /mavros/local_position/pose

# Check sauvc26_code is built
ros2 pkg list | grep sauvc26
```

## Performance Notes

- **VM Environments**: Use Headless Gazebo for better performance
- **Low RAM**: Use Final mission (lighter than qualification)
- **CPU Intensive**: Monitor tab shows real-time resource usage
- **Network**: Ensure UDP port 14550 is not blocked (MAVLink communication)

## File Structure

```
amarine-gui/
├── command_gui.py          # Main GUI application
├── requirements.txt        # Python dependencies
├── launch_gui.sh          # Shell launcher script
├── setup_gui.sh           # Setup script
├── README.md              # This file
└── QUICKSTART_SAUVC.md    # Detailed SAUVC mission guide
```

## Version History

- **v2.1** (Current): Full sauvc26-world and sauvc26-code integration
- **v2.0**: Multi-console interface with monitoring
- **v1.0**: Basic command launcher

## Dependencies

```
PyQt5==5.15.7
PyQtWebEngine==5.15.6
psutil>=5.8.0
```

Install all dependencies:
```bash
pip3 install -r requirements.txt
```

## License & Attribution

Part of the AMarineUV team for Singapore AUV Challenge 2026.

## Support & Documentation

- **Setup**: [SETUP_FROM_GIT.md](./SETUP_FROM_GIT.md) - Complete setup from git clone
- **Quick Start**: [QUICKSTART_SAUVC.md](./QUICKSTART_SAUVC.md) - Mission workflow guide
- **Troubleshooting**: [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Error fixes and diagnostics
- sauvc26-world: [README.md](../../sauvc26-world/README.md)
- sauvc26-code: [Repository](../../sauvc26-code/)
- ArduPilot: https://ardupilot.org/
- MAVROS: http://wiki.ros.org/mavros
- ROS 2: https://docs.ros.org/

---

**Last Updated**: 2026-06-22  
**Author**: AMarineUV Team  
**Status**: Ready for SAUVC 2026
