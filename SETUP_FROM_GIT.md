# AMarineGUI - Complete Setup Guide from Git Clone

Full setup guide untuk SAUVC 2026 AMarineGUI setelah fresh git clone dari GitHub.

**Status**: ✅ Ready for SAUVC 2026 Competition  
**Last Updated**: 2026-06-22

---

## 📋 Quick Overview

Ini adalah integrated system untuk autonomous underwater vehicle (AUV) yang menggabungkan:
- **Gazebo Sim** (pool physics simulation)
- **ArduSub** (flight control system)
- **MAVROS** (ROS2 bridge)
- **sauvc26-code** (mission planning AI)
- **AMarineGUI** (unified control interface)

---

## 🚀 Quick Start (5 Minutes)

```bash
# 1. Clone repositories
cd ~
git clone https://github.com/FechL/sauvc26-world.git
git clone https://github.com/FechL/sauvc26-code.git
git clone https://github.com/FechL/amarine-gui.git

# 2. Run setup script
cd ~/amarine-gui
bash setup.sh

# 3. Launch GUI
python3 command_gui.py
```

**Then follow the in-GUI workflow** (see [GUI Workflow](#gui-workflow) section)

---

## 📦 Prerequisites

### System Requirements

- **OS**: Ubuntu 22.04 LTS or later
- **RAM**: 8GB minimum (16GB recommended)
- **Disk Space**: 10GB free
- **GPU**: Optional (recommended for GUI mode)

### Required Software (Pre-Installation)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install core dependencies
sudo apt install -y \
  build-essential \
  cmake \
  git \
  python3-pip \
  python3-dev \
  ros-humble-desktop \
  gazebo-all

# Verify versions
gazebo --version        # >= 8.0
ros2 --version         # Humble
python3 --version      # >= 3.8
```

### Optional but Recommended

```bash
# For better build performance
sudo apt install -y ccache

# For visualization
sudo apt install -y \
  rviz2 \
  rqt

# For debugging
sudo apt install -y \
  net-tools \
  htop \
  tmux
```

---

## 📥 Step 1: Clone Repositories

```bash
# Create workspace
mkdir -p ~/sauvc_workspace
cd ~

# Clone all three repositories
echo "Cloning sauvc26-world..."
git clone https://github.com/FechL/sauvc26-world.git

echo "Cloning sauvc26-code..."
git clone https://github.com/FechL/sauvc26-code.git

echo "Cloning amarine-gui..."
git clone https://github.com/FechL/amarine-gui.git

# Verify all cloned successfully
ls -la ~/ | grep sauvc
ls -la ~/amarine-gui
```

**Expected Output:**
```
drwxr-xr-x sauvc26-code
drwxr-xr-x sauvc26-world
drwxr-xr-x amarine-gui
```

---

## 🛠️ Step 2: Setup ROS2 Workspace

### Create Workspace Structure

```bash
# Create ROS2 workspace
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# Create symlinks to git repos (CRITICAL!)
ln -sf ~/sauvc26-code .
ln -sf ~/sauvc26-world .

# Verify symlinks
ls -la ~/ros2_ws/src | grep sauvc
```

**Expected Output:**
```
sauvc26-code -> /home/ozza/sauvc26-code
sauvc26-world -> /home/ozza/sauvc26-world
```

### Setup Shell Environment

```bash
# Add to ~/.bashrc
cat >> ~/.bashrc << 'EOF'

# ===== SAUVC 2026 Setup =====
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

# Gazebo environment
export GZ_SIM_RESOURCE_PATH=~/sauvc26-world/models:~/sauvc26-world/worlds
export GZ_SIM_SYSTEM_PLUGIN_PATH=~/ardupilot_gazebo/build

# ArduPilot PATH
export PATH=$PATH:~/ardupilot/Tools/autotest

# ROS2 Domain ID (for multi-robot testing)
# export ROS_DOMAIN_ID=0

# End SAUVC 2026 Setup
EOF

# Apply changes
source ~/.bashrc
```

### Verify Environment

```bash
# Test ROS2 is available
ros2 --version
# Should output: ROS 2 humble

# Test environment variables
echo "GZ_SIM_RESOURCE_PATH: $GZ_SIM_RESOURCE_PATH"
echo "GZ_SIM_SYSTEM_PLUGIN_PATH: $GZ_SIM_SYSTEM_PLUGIN_PATH"
```

---

## 🏗️ Step 3: Install Dependencies

### Install Python Dependencies

```bash
# Install GUI requirements
pip3 install -r ~/amarine-gui/requirements.txt

# Verify PyQt5
python3 -c "from PyQt5.QtWidgets import QApplication; print('✓ PyQt5 installed')"

# Verify psutil
python3 -c "import psutil; print(f'✓ psutil {psutil.__version__} installed')"
```

### Install ROS2 Packages

```bash
# Install MAVROS and dependencies
sudo apt install -y ros-humble-mavros*

# Install colcon build tool (should already be installed)
sudo apt install -y python3-colcon-common-extensions

# Verify colcon
colcon --version
# Should output: colcon X.X.X
```

---

## 🔨 Step 4: Build ROS2 Packages

### Build sauvc26-code

```bash
cd ~/ros2_ws

# Build with verbose output
colcon build --packages-select sauvc26_code --event-handlers console_direct+

# Wait for: "Finished <<< sauvc26_code"
# Typical time: 1-2 minutes
```

### Build sauvc26-world

```bash
# Build world package
colcon build --packages-select sauvc26_world --event-handlers console_direct+

# Wait for: "Finished <<< sauvc26_world"
# Typical time: 30 seconds
```

### Verify Builds

```bash
# Check packages are available
source ~/ros2_ws/install/setup.bash
ros2 pkg list | grep sauvc

# Expected output:
# sauvc26_code
# sauvc26_world
```

---

## ⚙️ Step 5: Verify Helper Scripts

### Check Script Permissions

```bash
# Make helper scripts executable
chmod +x ~/sauvc26-world/run_gazebo_headless.sh
chmod +x ~/sauvc26-world/run_gazebo.sh
chmod +x ~/sauvc26-world/run_ardupilot.sh
chmod +x ~/amarine-gui/command_gui.py

# Verify permissions
ls -la ~/sauvc26-world/*.sh
ls -la ~/amarine-gui/command_gui.py
# All should have 'x' permission
```

### Test Path Resolution

```bash
# Test that scripts correctly find workspace paths
cd ~/sauvc26-world
bash -c 'SCRIPT_DIR="$(cd "$(dirname "./run_ardupilot.sh")" && pwd)"; WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"; echo "Workspace root: $WORKSPACE_ROOT"; test -f "$WORKSPACE_ROOT/ardupilot/Tools/autotest/sim_vehicle.py" && echo "✓ sim_vehicle.py found"'

# Expected output:
# Workspace root: /home/ozza
# ✓ sim_vehicle.py found
```

---

## ✅ Step 6: System Verification Checklist

Run these checks to ensure everything is installed correctly:

```bash
#!/bin/bash
echo "========================================="
echo "SAUVC 2026 System Verification Checklist"
echo "========================================="

# 1. Gazebo
echo ""
echo "1. Checking Gazebo..."
gazebo --version && echo "✓ Gazebo installed" || echo "✗ Gazebo NOT found"

# 2. ROS2
echo ""
echo "2. Checking ROS2..."
ros2 --version && echo "✓ ROS2 Humble installed" || echo "✗ ROS2 NOT found"

# 3. MAVROS
echo ""
echo "3. Checking MAVROS..."
ros2 pkg list | grep mavros > /dev/null && echo "✓ MAVROS installed" || echo "✗ MAVROS NOT found"

# 4. Repositories
echo ""
echo "4. Checking repositories..."
test -d ~/sauvc26-world && echo "✓ sauvc26-world cloned" || echo "✗ sauvc26-world NOT found"
test -d ~/sauvc26-code && echo "✓ sauvc26-code cloned" || echo "✗ sauvc26-code NOT found"
test -d ~/amarine-gui && echo "✓ amarine-gui cloned" || echo "✗ amarine-gui NOT found"

# 5. ROS2 packages
echo ""
echo "5. Checking ROS2 packages..."
source ~/ros2_ws/install/setup.bash
ros2 pkg list | grep sauvc26_code > /dev/null && echo "✓ sauvc26_code package" || echo "✗ sauvc26_code package NOT found"
ros2 pkg list | grep sauvc26_world > /dev/null && echo "✓ sauvc26_world package" || echo "✗ sauvc26_world package NOT found"

# 6. Python packages
echo ""
echo "6. Checking Python packages..."
python3 -c "from PyQt5.QtWidgets import QApplication" 2>/dev/null && echo "✓ PyQt5" || echo "✗ PyQt5 NOT found"
python3 -c "import psutil" 2>/dev/null && echo "✓ psutil" || echo "✗ psutil NOT found"

# 7. World files
echo ""
echo "7. Checking world files..."
test -f ~/sauvc26-world/worlds/sauvc_final.world && echo "✓ sauvc_final.world" || echo "✗ sauvc_final.world NOT found"
test -f ~/sauvc26-world/worlds/sauvc_qualification.world && echo "✓ sauvc_qualification.world" || echo "✗ sauvc_qualification.world NOT found"

# 8. ArduPilot setup
echo ""
echo "8. Checking ArduPilot..."
test -f ~/ardupilot/Tools/autotest/sim_vehicle.py && echo "✓ ArduSub simulator found" || echo "✗ ArduSub simulator NOT found"
test -f ~/ardupilot_gazebo/build/libArduPilotPlugin.so && echo "✓ ArduPilot Gazebo plugin" || echo "✗ ArduPilot Gazebo plugin NOT found"

echo ""
echo "========================================="
echo "Verification complete!"
echo "========================================="
```

Save as `~/verify_sauvc.sh` and run:
```bash
chmod +x ~/verify_sauvc.sh
~/verify_sauvc.sh
```

---

## 🎮 Step 7: Launch AMarineGUI

### First Launch

```bash
cd ~/amarine-gui
python3 command_gui.py
```

**Expected Output:**
```
Warning: Ignoring XDG_SESSION_TYPE=wayland on Gnome. Use QT_QPA_PLATFORM=wayland to run on Wayland anyway.
# (This warning is normal and can be ignored)

# GUI window should open after 5-10 seconds
```

### GUI Workflow

Once GUI is open, follow this exact sequence:

#### 1️⃣ Start Gazebo World (Top-Left)

```
Dropdown: Select "Headless (Final)" or "GUI (Final)"
Button: Click "Start"
Wait: 10-15 seconds
Check: Console shows "Starting Gazebo HEADLESS"
```

**What's happening:**
- Gazebo server initializes water physics
- ArduPilot plugin loads
- Virtual pool is ready

#### 2️⃣ Start ArduSub + Gazebo (Left Console)

```
Button: Click "Start"
Wait: 20-30 seconds
Check: Console shows "Ready to fly"
```

**What's happening:**
- ArduSub SITL (Software In The Loop) starts
- Connects to Gazebo for physics simulation
- Flight control system initializes

#### 3️⃣ Start MAVROS (Middle Console)

```
Button: Click "Start"
Wait: 10-20 seconds
Check: Console shows "Got HEARTBEAT, connected. FCU: ArduPilot"
Note: May crash after connection (known issue) - restart is OK
```

**What's happening:**
- MAVROS bridge starts
- Connects to ArduSub via MAVLink
- ROS2 topics become available

**If MAVROS crashes:**
- Click "Kill" on MAVROS console
- Wait 2 seconds
- Click "Start" again
- Usually works on second try

#### 4️⃣ Arm Vehicle (Bottom-Left ROS2)

```
Dropdown: Select "Arm Vehicle"
Button: Click "Start"
Wait: 5 seconds
Check: Console shows "[info] ARMED"
```

**What's happening:**
- Vehicle receives arm command
- Safety checks pass
- Motors are ready to operate

#### 5️⃣ Run Final Mission (Bottom-Middle ROS2)

```
Dropdown: Select "Final Mission"
Button: Click "Start"
Watch: Console shows mission progress
Check: Vehicle moves in Gazebo
```

**What's happening:**
- Mission planning node starts
- Vehicle navigates autonomously
- Completes competition tasks

---

## 🐛 Common Issues & Fixes

### Issue 1: "Package 'sauvc26_code' not found"

**Cause**: Symlink not created or build failed

**Fix**:
```bash
# Recreate symlink
cd ~/ros2_ws/src
rm -f sauvc26-code
ln -sf ~/sauvc26-code .

# Rebuild
cd ~/ros2_ws
colcon build --packages-select sauvc26_code
```

### Issue 2: "/bin/sh: i: source: not found"

**Cause**: Already fixed in this version (v2.1+)

**Check**:
```bash
grep "executable='/bin/bash'" ~/amarine-gui/command_gui.py
```

Should show the line. If not, update command_gui.py.

### Issue 3: "Error starting tegrastats"

**Cause**: Normal on non-Jetson systems

**Fix**: Already handled - GUI falls back to psutil

### Issue 4: "./run_ardupilot.sh: line 15: cd: /home/ardupilot: No such file"

**Cause**: Incorrect path in script

**Fix**:
```bash
# Check script
head -3 ~/sauvc26-world/run_ardupilot.sh

# Should show path calculation as: WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
# NOT: WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
```

### Issue 5: "MAVROS crash - std::future_error"

**Cause**: Known MAVROS 2.x issue with ArduPilot SITL

**Fix**:
- Let it crash (no harm done)
- Click "Kill" then "Start" again
- Usually works on second attempt
- Or run mission without MAVROS (direct ArduSub connection)

### For More Issues

See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for comprehensive troubleshooting guide.

---

## 📚 Complete Documentation

After setup, read these guides in order:

1. **[README.md](./README.md)** - Overview and features
2. **[QUICKSTART_SAUVC.md](./QUICKSTART_SAUVC.md)** - Mission workflow details
3. **[TROUBLESHOOTING.md](./TROUBLESHOOTING.md)** - Detailed troubleshooting

---

## 🧪 Manual Testing (Optional)

If you want to test components individually before using GUI:

### Test 1: Gazebo Only

```bash
export GZ_SIM_RESOURCE_PATH=~/sauvc26-world/models:~/sauvc26-world/worlds
export GZ_SIM_SYSTEM_PLUGIN_PATH=~/ardupilot_gazebo/build
cd ~/sauvc26-world
./run_gazebo_headless.sh ./worlds/sauvc_final.world
# Should start without errors
```

### Test 2: ArduSub Only

```bash
source ~/ros2_ws/install/setup.bash
sleep 10  # Wait for Gazebo to start in another terminal
cd ~/sauvc26-world
./run_ardupilot.sh
# Should show "Ready to fly"
```

### Test 3: MAVROS Only

```bash
source ~/ros2_ws/install/setup.bash
ros2 launch mavros apm.launch fcu_url:=udp://:14550@localhost:14555
# Should show successful connection
```

### Test 4: ROS2 Topics

```bash
source ~/ros2_ws/install/setup.bash
ros2 topic list | grep mavros
# Should list many topics like /mavros/local_position/pose
```

---

## 🚀 Performance Tuning

### For Faster Builds

```bash
# Use multiple cores
colcon build --packages-select sauvc26_code -j 8

# Enable ccache
sudo apt install ccache
export CC=/usr/bin/ccache/gcc
export CXX=/usr/bin/ccache/g++
```

### For Faster Simulation

```bash
# Run headless instead of GUI
# Use "Headless (Final)" in GUI dropdown

# Reduce Gazebo plugin count
# Edit ~/sauvc26-world/worlds/sauvc_final.world
# Comment out unnecessary plugins
```

### Monitor Performance

```bash
# In separate terminal, watch CPU/memory
watch -n 1 'ps aux | grep -E "gazebo|ardupilot|mavros" | grep -v grep'
```

---

## 📞 Support & Documentation

### Internal Documentation

- [AMarineGUI Architecture](./README.md#architecture)
- [Mission Code Structure](../../sauvc26-code/)
- [Gazebo World Models](../../sauvc26-world/README.md)

### External Resources

- [ROS 2 Documentation](https://docs.ros.org/)
- [Gazebo Documentation](https://gazebosim.org/docs/)
- [MAVROS Documentation](http://wiki.ros.org/mavros)
- [ArduPilot Documentation](https://ardupilot.org/)

### Getting Help

1. Check [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
2. Run verification script: `~/verify_sauvc.sh`
3. Check logs: `tail -50 ~/.ros/log/*/rosout.log`
4. Run diagnostic commands from troubleshooting guide

---

## 📋 Checklist: Before Competition

- [ ] All repositories cloned from GitHub
- [ ] ROS2 workspace setup complete
- [ ] All packages built successfully
- [ ] Verification script passes all checks
- [ ] GUI launches without errors
- [ ] Can complete full workflow (Gazebo → ArduSub → MAVROS → Arm → Mission)
- [ ] Vehicle successfully completes qualification mission
- [ ] Vehicle successfully completes final mission
- [ ] Can arm/disarm reliably
- [ ] Performance is smooth (monitoring shows healthy CPU/RAM)

---

## 🎯 Next Steps

1. **Run verification**: `~/verify_sauvc.sh`
2. **Launch GUI**: `python3 ~/amarine-gui/command_gui.py`
3. **Follow workflow**: See Step 7 above
4. **Read documentation**: README.md → QUICKSTART_SAUVC.md → TROUBLESHOOTING.md
5. **Test missions**: Qualification first, then Final
6. **Optimize**: Use performance tips if needed

---

## 🔄 Updates & Branches

### Check for Updates

```bash
# Update all repositories
cd ~/sauvc26-world && git pull origin main
cd ~/sauvc26-code && git pull origin main
cd ~/amarine-gui && git pull origin main

# Rebuild after pulling
cd ~/ros2_ws
colcon build --packages-select sauvc26_code sauvc26_world
```

### Switch to Competition Branch (if available)

```bash
cd ~/sauvc26-code
git checkout competition-branch
cd ~/ros2_ws && colcon build --packages-select sauvc26_code
```

---

## 📊 System Architecture

```
┌─────────────────────────────────┐
│     AMarineGUI (PyQt5)          │  ← Control Interface
└──────────────┬──────────────────┘
               │
        ┌──────┴─────────┐
        │                │
   ┌────▼────┐    ┌─────▼──────┐
   │ Gazebo  │    │ ArduSub    │   ← Simulation & Flight Control
   │ Harmonic│    │ SITL       │
   └────┬────┘    └────┬───────┘
        │              │ (MAVLink UDP)
        │         ┌────▼───────┐
        │         │  MAVROS    │   ← ROS2 Bridge
        │         └────┬───────┘
        │              │ (ROS2 Topics)
        │         ┌────▼──────────────┐
        │         │ sauvc26-code      │  ← Mission Planning
        │         │ (Arm, Qualification,
        │         │  Final)           │
        │         └───────────────────┘
        │
   ┌────▼──────────────┐
   │  Water Physics    │
   │  (Buoyancy,       │
   │   Hydrodynamics)  │
   └───────────────────┘
```

---

## 📝 License & Attribution

Part of **AMarineUV Team** for **Singapore AUV Challenge 2026**

---

**Version**: 2.1  
**Last Updated**: 2026-06-22  
**Status**: ✅ Production Ready  
**Next Competition**: Singapore AUV Challenge 2026
