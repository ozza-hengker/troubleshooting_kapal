# AMarineGUI Troubleshooting Guide - Full Setup & Fixes

Complete troubleshooting guide untuk setup SAUVC 2026 AMarineGUI dari awal sampai berhasil jalan.

---

## Table of Contents

1. [Initial Setup Checklist](#initial-setup-checklist)
2. [Common Errors & Solutions](#common-errors--solutions)
3. [Detailed Troubleshooting](#detailed-troubleshooting)
4. [Testing Workflow](#testing-workflow)
5. [Advanced Issues](#advanced-issues)
6. [Performance Tips](#performance-tips)

---

## Initial Setup Checklist

Sebelum jalankan GUI, pastikan semua ini sudah selesai:

### 1. Install Dependencies

```bash
# Update package manager
sudo apt update
sudo apt upgrade -y

# Install Python dependencies
pip3 install -r ~/amarine-gui/requirements.txt

# Verify installations
gazebo --version          # Should be >= 8.0
ros2 --version           # Should be Humble
python3 --version        # Should be >= 3.8
```

### 2. Setup ROS2 Workspace

```bash
# Source ROS2
source /opt/ros/humble/setup.bash

# Create symlinks in workspace (CRITICAL)
cd ~/ros2_ws/src
ln -sf ~/sauvc26-code .        # Link sauvc26-code
ln -sf ~/sauvc26-world .       # Link sauvc26-world (if not already)

# Verify symlinks
ls -la ~/ros2_ws/src | grep sauvc
# Should show:
# sauvc26-code -> /home/ozza/sauvc26-code
# sauvc26-world -> ../../sauvc26-world
```

### 3. Build sauvc26_code

```bash
cd ~/ros2_ws
colcon build --packages-select sauvc26_code

# Verify build
ros2 pkg list | grep sauvc26_code
# Should output: sauvc26_code
```

### 4. Verify Helper Scripts

```bash
# Check all helper scripts exist and are executable
ls -la ~/sauvc26-world/*.sh

# Make executable if needed
chmod +x ~/sauvc26-world/run_gazebo_headless.sh
chmod +x ~/sauvc26-world/run_gazebo.sh
chmod +x ~/sauvc26-world/run_ardupilot.sh

# Test path resolution
bash -c 'SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"; echo "Script dir: $SCRIPT_DIR"' ~/sauvc26-world/run_gazebo_headless.sh
```

### 5. Verify ArduPilot Setup

```bash
# Check ardupilot directory exists
ls -la ~/ardupilot/Tools/autotest/sim_vehicle.py

# Check ardupilot_gazebo plugin
ls -la ~/ardupilot_gazebo/build/libArduPilotPlugin.so

# Add to .bashrc if needed
echo 'export PATH=$PATH:~/ardupilot/Tools/autotest' >> ~/.bashrc
source ~/.bashrc
```

### 6. Test ROS2 Communications

```bash
# Terminal 1: Run Gazebo
cd ~/sauvc26-world
./run_gazebo_headless.sh ./worlds/sauvc_final.world

# Terminal 2: Run ArduSub (wait 10 seconds for Gazebo to start)
sleep 10
cd ~/sauvc26-world
./run_ardupilot.sh

# Terminal 3: Check topics (wait another 10 seconds)
sleep 10
source ~/ros2_ws/install/setup.bash
ros2 topic list | grep -E "mavros|local_position"
# Should show: /mavros/local_position/pose (eventually)
```

✅ **If all above passes, your setup is correct!**

---

## Common Errors & Solutions

### Error 1: "Package 'sauvc26_code' not found"

**Symptoms:**
```
Package 'sauvc26_code' not found
Command finished (exit code: 1)
```

**Causes:**
- sauvc26_code not symlinked to ROS2 workspace
- Build not completed
- ROS2 setup.bash not sourced

**Solutions:**

**Solution A: Create Symlink**
```bash
cd ~/ros2_ws/src
ln -sf ~/sauvc26-code .
```

**Solution B: Rebuild Package**
```bash
cd ~/ros2_ws
colcon build --packages-select sauvc26_code --event-handlers console_direct+
# Wait for: "Finished <<< sauvc26_code"
```

**Solution C: Source Setup**
```bash
# In every new terminal, do:
source ~/ros2_ws/install/setup.bash

# Or add to ~/.bashrc permanently:
echo 'source ~/ros2_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

**Verify Fix:**
```bash
source ~/ros2_ws/install/setup.bash
ros2 pkg list | grep sauvc26_code
# Should output: sauvc26_code
```

---

### Error 2: "/bin/sh: i: source: not found"

**Symptoms:**
```
/bin/sh: i: source: not found
```

**Root Cause:**
- Commands using `source` are executed with `sh` instead of `bash`
- `source` is bash-specific command

**Solution:** (Already implemented in v2.1+)
- Verify `command_gui.py` has `executable='/bin/bash'` in subprocess.Popen

**Check Fix:**
```bash
grep -A 5 "subprocess.Popen" ~/amarine-gui/command_gui.py | grep executable
# Should show: executable='/bin/bash'
```

---

### Error 3: "Error starting tegrastats: No such file or directory"

**Symptoms:**
```
Error starting tegrastats: [Errno 2] No such file or directory: 'tegrastats'
```

**Root Cause:**
- `tegrastats` only available on NVIDIA Jetson
- Desktop Linux doesn't have it

**Solution:** (Already implemented in v2.1+)
- GUI now falls back to psutil for non-Jetson systems

**Verify Fix:**
```bash
# GUI should start without errors
cd ~/amarine-gui
python3 command_gui.py

# Monitoring panel should show:
# CPU: XX%
# RAM: XXXX/XXXMB
# GPU: N/A (on non-Jetson)
```

---

### Error 4: "./run_ardupilot.sh: line 15: cd: /home/ardupilot: No such file or directory"

**Symptoms:**
```
./run_ardupilot.sh: line 15: cd: /home/ardupilot: No such file or directory
./run_ardupilot.sh: line 23: Tools/autotest/sim_vehicle.py: No such file or directory
```

**Root Cause:**
- Path calculation in `run_ardupilot.sh` incorrect
- Using `/../..` when should be `/..`
- Trying to cd into wrong directory

**Solution:**

**Check Current Code:**
```bash
head -20 ~/sauvc26-world/run_ardupilot.sh | grep WORKSPACE_ROOT
```

**Should Show:**
```bash
WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
# NOT: WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
```

**If Wrong, Fix It:**
```bash
# Edit the file and change line:
# FROM: WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
# TO:   WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

# Or run this to auto-fix:
sed -i 's|cd "$SCRIPT_DIR/../.." && pwd|cd "$SCRIPT_DIR/.." \&\& pwd|g' ~/sauvc26-world/run_ardupilot.sh
```

**Verify Fix:**
```bash
cd ~/sauvc26-world
bash -c 'SCRIPT_DIR="$(cd "$(dirname "./run_ardupilot.sh")" && pwd)"; WORKSPACE_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"; echo "Workspace: $WORKSPACE_ROOT"; ls $WORKSPACE_ROOT/ardupilot/Tools/autotest/sim_vehicle.py'
# Should show the path to sim_vehicle.py
```

---

### Error 5: "MAVROS crash - std::future_error"

**Symptoms:**
```
[mavros_node-1] terminate called after throwing an instance of 'std::future_error'
[mavros_node-1]   what():  std::future_error: Promise already satisfied
```

**Root Cause:**
- Known issue in MAVROS 2.x with ArduPilot SITL
- Usually happens after successful connection

**Solutions:**

**Solution A: Restart MAVROS (Quick Fix)**
```bash
# In GUI:
# 1. Click "Kill" on MAVROS console
# 2. Wait 2 seconds
# 3. Click "Start" on MAVROS console
# Usually works on second attempt
```

**Solution B: Increase Timeout in MAVROS Launch**
```bash
# Edit MAVROS command in command_gui.py
# Add parameter: fcu_heartbeat_rate=10

# From:
# ros2 launch mavros apm.launch fcu_url:=udp://:14550@localhost:14555

# To:
# ros2 launch mavros apm.launch fcu_url:=udp://:14550@localhost:14555 fcu_heartbeat_rate:=10
```

**Solution C: Use MAVProxy Alternative (If MAVROS keeps crashing)**
```bash
# Instead of MAVROS, use MAVProxy
mavproxy.py --master=udp:127.0.0.1:14550 --sitl=127.0.0.1:5501
```

**Workaround For Now:**
- Let MAVROS crash, but mission code can still run via direct ArduSub connection
- Arm and run mission directly:
  ```bash
  ros2 run sauvc26_code arm
  sleep 2
  ros2 run sauvc26_code final
  ```

---

### Error 6: "Warning: Ignoring XDG_SESSION_TYPE=wayland"

**Symptoms:**
```
Warning: Ignoring XDG_SESSION_TYPE=wayland on Gnome. Use QT_QPA_PLATFORM=wayland to run on Wayland anyway.
```

**Root Cause:**
- Normal Qt5 warning on Wayland desktop
- Not an actual error

**Solution:** 
- Ignore it - GUI works fine
- Or suppress with:
  ```bash
  QT_QPA_PLATFORM=xcb python3 ~/amarine-gui/command_gui.py
  ```

---

### Error 7: "Package 'sauvc26-world' not found" or Gazebo won't start

**Symptoms:**
```
Package 'sauvc26-world' not found
Cannot find world file
GZ_SIM_RESOURCE_PATH error
```

**Root Causes:**
- sauvc26-world not symlinked
- GZ_SIM_RESOURCE_PATH not set correctly
- World files don't exist

**Solutions:**

**Check 1: Symlink exists**
```bash
ls -la ~/ros2_ws/src | grep sauvc26-world
# Should show: sauvc26-world -> ../../sauvc26-world
```

**Check 2: World files exist**
```bash
ls ~/sauvc26-world/worlds/
# Should show: sauvc_final.world, sauvc_qualification.world
```

**Check 3: Gazebo resource path**
```bash
export GZ_SIM_RESOURCE_PATH=~/sauvc26-world/models:~/sauvc26-world/worlds
gz resource --locate models
# Should show models found
```

**Check 4: ArduPilot plugin**
```bash
ls ~/ardupilot_gazebo/build/libArduPilotPlugin.so
export GZ_SIM_SYSTEM_PLUGIN_PATH=~/ardupilot_gazebo/build
gz plugin --info ArduPilotPlugin
```

---

## Detailed Troubleshooting

### Step 1: Verify Base System

```bash
# Check all required tools exist
which gazebo
which ros2
which colcon
which python3

# Check versions
gazebo --version       # >= 8.0
ros2 --version        # Humble
python3 --version     # >= 3.8
```

**If any tool missing:**
```bash
# Install missing dependencies
sudo apt install gazebo-all ros-humble-* python3-pip

# For ardupilot_gazebo
cd ~/ardupilot_gazebo/build
cmake ..
make -j4
```

### Step 2: Verify File Permissions

```bash
# Check GUI script is executable
ls -l ~/amarine-gui/command_gui.py
# Should have 'x' permission

# Check helper scripts are executable
ls -l ~/sauvc26-world/*.sh
# All should have 'x' permission

# Fix if needed
chmod +x ~/amarine-gui/command_gui.py
chmod +x ~/sauvc26-world/*.sh
```

### Step 3: Verify Python Packages

```bash
# Check PyQt5 installation
python3 -c "from PyQt5.QtWidgets import QApplication; print('PyQt5 OK')"

# Check psutil installation
python3 -c "import psutil; print(f'psutil {psutil.__version__} OK')"

# If errors, install:
pip3 install -r ~/amarine-gui/requirements.txt
```

### Step 4: Verify ROS2 Setup

```bash
# Source setup
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

# Check packages available
ros2 pkg list | head -20
ros2 pkg list | grep sauvc

# If sauvc26_code missing:
cd ~/ros2_ws
colcon build --packages-select sauvc26_code
```

### Step 5: Manual System Test (Before GUI)

**Terminal 1 - Gazebo:**
```bash
export GZ_SIM_RESOURCE_PATH=~/sauvc26-world/models:~/sauvc26-world/worlds
export GZ_SIM_SYSTEM_PLUGIN_PATH=~/ardupilot_gazebo/build
cd ~/sauvc26-world
./run_gazebo_headless.sh ./worlds/sauvc_final.world
# Wait for: "Connected to gz sim service"
```

**Terminal 2 - ArduSub (wait 10s after Gazebo starts):**
```bash
source ~/ros2_ws/install/setup.bash
sleep 10
cd ~/sauvc26-world
./run_ardupilot.sh
# Wait for: "Ready to fly"
```

**Terminal 3 - Check Connection (wait 10s after ArduSub starts):**
```bash
source ~/ros2_ws/install/setup.bash
sleep 10
ros2 topic list
# Should see many /mavros/* topics
ros2 topic echo /mavros/local_position/pose
# Should see position updates
```

✅ **If all 3 terminals work, system is ready for GUI!**

---

## Testing Workflow

Once initial setup passes, follow this exact workflow:

### Pre-Flight Checklist

```bash
# 1. Close any existing GUI windows
killall python3 2>/dev/null || true

# 2. Kill any lingering processes
pkill -f "gazebo\|ardupilot\|mavros" 2>/dev/null || true

# 3. Clean caches
rm -rf ~/.gazebo/rendering.log
rm -rf /tmp/gazebo_* 2>/dev/null || true

# 4. Check disk space
df -h ~/ | head -3
# Need at least 1GB free
```

### Launch GUI

```bash
cd ~/amarine-gui
python3 command_gui.py
# Wait for window to appear (5-10 seconds)
```

### Execute Sequence

**Step 1: Gazebo World** (Top-left)
- Dropdown: Select `Headless (Final)` or `GUI (Final)`
- Click: **Start**
- Wait: Console shows `Starting Gazebo HEADLESS`
- Check: Gazebo window opens or headless service starts
- Time: 10-15 seconds

**Step 2: ArduSub + Gazebo** (Left console)
- Click: **Start**
- Wait: Console shows `Starting ArduSub sim_vehicle with Gazebo`
- Check: See output like `Ready to fly` or `SITL connected`
- Time: 20-30 seconds

**Step 3: MAVROS** (Middle console)
- Click: **Start**
- Wait: Console shows multiple `[INFO] [mavros.mavros]: Plugin ... initialized`
- Check: See `Got HEARTBEAT, connected. FCU: ArduPilot`
- Time: 10-20 seconds
- Note: May crash after first successful connection (known issue) - restart is OK

**Step 4: Arm Vehicle** (Bottom-left ROS2)
- Dropdown: Select `Arm Vehicle`
- Click: **Start**
- Wait: Console shows command output
- Check: See `[info] ARMED` or similar
- Time: 5 seconds

**Step 5: Final Mission** (Bottom-middle ROS2)
- Dropdown: Select `Final Mission`
- Click: **Start**
- Watch: Console shows mission progress
- Check: Vehicle moves in Gazebo window
- Time: Variable (mission duration)

---

## Advanced Issues

### Issue: Gazebo Visual Crashes on VM

**Symptoms:**
- Gazebo starts but GUI window crashes
- Error mentions Ogre or rendering

**Solution:**
- Use `Headless (Final)` instead of `GUI (Final)`
- Simulation works fine, just no graphics

### Issue: Very Slow Performance

**Symptoms:**
- Commands take very long to execute
- High CPU/RAM usage (see Monitoring panel)

**Solutions:**
```bash
# 1. Close other applications
# 2. Use fewer Gazebo plugins (edit sauvc_final.world)
# 3. Reduce visual quality in Gazebo GUI
# 4. Run on native Linux instead of VM
```

### Issue: Network Connectivity Problems

**Symptoms:**
- MAVROS won't connect
- "udp:14550 timeout"
- Port conflicts

**Solutions:**
```bash
# Check if port is in use
netstat -tun | grep 14550
lsof -i :14550

# Kill conflicting process
pkill -f "14550" 2>/dev/null || true

# Try alternate port
# Edit command_gui.py and change:
# FROM: fcu_url:=udp://:14550@localhost:14555
# TO:   fcu_url:=udp://:14551@localhost:14556
```

---

## Performance Tips

### Speed Up Builds

```bash
# Use more cores
cd ~/ros2_ws
colcon build --packages-select sauvc26_code -j 8

# Use ccache
sudo apt install ccache
export CC=/usr/bin/ccache/gcc
export CXX=/usr/bin/ccache/g++
```

### Optimize Gazebo

```bash
# Use faster rendering backend
export GAZEBO_RENDER_ENGINE=ogre2

# Reduce physics update rate in world file
# (decrease <step_size> in sauvc_final.world)

# Disable visualization if not needed
gz sim -r -s world.sdf  # Headless only
```

### Monitor System

```bash
# In separate terminal, watch resources
watch -n 1 'ps aux | grep -E "gazebo|ardupilot|mavros|python3" | grep -v grep'

# Use top for detailed info
top -p $(pgrep -f "gazebo|ardupilot|mavros" | tr '\n' ',')
```

---

## Quick Reference Commands

### Emergency Stop
```bash
# Kill everything
killall gazebo ardupilot mavros 2>/dev/null || true
pkill -f "sim_vehicle\|command_gui" 2>/dev/null || true
```

### Rebuild Everything
```bash
cd ~/ros2_ws
rm -rf build install log
colcon build --packages-select sauvc26_code sauvc26_world
```

### Test Single Component
```bash
# Test Gazebo only
export GZ_SIM_RESOURCE_PATH=~/sauvc26-world/models:~/sauvc26-world/worlds
export GZ_SIM_SYSTEM_PLUGIN_PATH=~/ardupilot_gazebo/build
gz sim -r ~/sauvc26-world/worlds/sauvc_final.world

# Test ArduSub only
cd ~/ardupilot
Tools/autotest/sim_vehicle.py -L RATBeach -v ArduSub

# Test ROS2 package
source ~/ros2_ws/install/setup.bash
ros2 run sauvc26_code test
```

### Check Logs
```bash
# GUI logs
tail -50 ~/.ros/log/*/rosout.log

# ArduSub logs
tail -100 ~/ardupilot/sim_vehicle_log.txt

# Gazebo logs
tail -50 ~/.gazebo/rendering.log
```

---

## Success Indicators

When everything is working correctly, you should see:

1. ✅ **GUI launches without errors**
   - Window opens cleanly
   - No Python exceptions

2. ✅ **Gazebo starts**
   - Console shows `Starting Gazebo`
   - Pool visualization appears (if GUI mode)

3. ✅ **ArduSub connects**
   - Console shows `Ready to fly`
   - SITL output stabilizes

4. ✅ **MAVROS connects**
   - Console shows `Got HEARTBEAT`
   - Topics appear in `ros2 topic list`

5. ✅ **Arm succeeds**
   - Console shows `[info] ARMED`
   - No timeout or connection errors

6. ✅ **Mission runs**
   - Vehicle moves in Gazebo
   - Console shows mission state machine
   - Position updates in `/mavros/local_position/pose`

---

## Still Having Issues?

### Diagnostic Command

Run this to generate a diagnostic report:

```bash
#!/bin/bash
echo "=== SYSTEM INFO ==="
uname -a
python3 --version

echo "=== ROS2 ==="
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 --version

echo "=== PACKAGES ==="
ros2 pkg list | grep sauvc

echo "=== FILES ==="
ls -la ~/sauvc26-world/worlds/
ls -la ~/ardupilot_gazebo/build/libArduPilotPlugin.so

echo "=== PYTHON ==="
python3 -c "from PyQt5.QtWidgets import QApplication; print('PyQt5: OK')"
python3 -c "import psutil; print(f'psutil: OK ({psutil.__version__})')"

echo "=== ROS2 TOPICS ==="
timeout 5 bash -c 'source ~/ros2_ws/install/setup.bash && ros2 topic list | grep -E "mavros|local_position"' || echo "Topics not available yet"

echo "=== PATHS ==="
echo "GZ_SIM_RESOURCE_PATH: $GZ_SIM_RESOURCE_PATH"
echo "GZ_SIM_SYSTEM_PLUGIN_PATH: $GZ_SIM_SYSTEM_PLUGIN_PATH"
```

Save output and share if stuck!

---

## References

- [AMarineGUI README](./README.md)
- [SAUVC Quick Start](./QUICKSTART_SAUVC.md)
- [ROS2 Documentation](https://docs.ros.org/)
- [Gazebo Documentation](https://gazebosim.org/docs/)
- [MAVROS Documentation](http://wiki.ros.org/mavros)
- [ArduPilot Documentation](https://ardupilot.org/)

---

**Last Updated**: 2026-06-22  
**Status**: Ready for production use  
**Known Issues**: MAVROS may crash on first connection (restart fixes it)
