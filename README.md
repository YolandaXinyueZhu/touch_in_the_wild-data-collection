# Tactile Hardware Code and Data Collection

[Project page](https://binghao-huang.github.io/touch_in_the_wild/) | [Paper](https://binghao-huang.github.io/touch_in_the_wild/) 

<img width="90%" src="assets/system.png" alt="System image"/>

[Xinyue Zhu](https://binghao-huang.github.io/touch_in_the_wild/)<sup>\* 1</sup>,
[Binghao Huang](https://binghao-huang.github.io/)<sup>\* 1</sup>,
[Yunzhu Li](https://yunzhuli.github.io/)<sup>1</sup>

<sup>\*</sup>Equal contribution <sup>1</sup>Columbia University

*This README serves as **both** the ROS 2 tutorial **and** the data collection guide for our visuo‑tactile gripper.*

## 📑 Quick Outline

1. [Gripper & Tactile Hardware](#hardware)  
2. [Persistent Port Naming](#udev)
3. [ROS 2 Setup](#ros)
4. [Data Collection Protocol](#protocol)
5. [Troubleshooting](#trouble)
6. [License](#license)

## <a id="hardware"></a>🔧 Gripper & Tactile Hardware

To build the visuo-tactile gripper, follow this guide: [Visuo-Tactile Gripper Guide](https://docs.google.com/document/d/1Gxd2PzVo-sH_XbvnVw_r3eI5i3I3t0rr7JdhpN1PJMA/edit?tab=t.0#heading=h.w30qb8tzs5g5)  

You’re welcome to request a sample of our tactile hardware, [FlexiTac](https://flexitac.github.io/).

Alternatively, you can use the previous version of the hardware to build two tactile sensors (“right_finger” and “left_finger”) by following the [Hardware Assembly Tutorial](https://docs.google.com/document/d/1XGyn-iV_wzRmcMIsyS3kwcrjxbnvblZAyigwbzDsX-E/edit?tab=t.0#heading=h.ny8zu0pq9mxy).

For Python & ROS 1 code, see [3D‑ViTac\_Tactile\_Hardware](https://github.com/binghao-huang/3D-ViTac_Tactile_Hardware).

## <a id="udev"></a>🔌 Persistent Port Naming

### Why persistent names?

Linux assigns FTDI adapters to `/dev/ttyUSB*` in detection order. After a reboot (or if you swap cables) the numbers shift — a sensor that was `/dev/ttyUSB0` might come back as `/dev/ttyUSB5`. By binding each adapter’s unique **FTDI serial number** to a udev rule we get stable aliases like `/dev/right_finger` and `/dev/left_finger`.

### One‑time setup for two sensors

1. **Plug in only one sensor (right finger).**
2. **Find its current device node**

   ```bash
   ls -l /dev/ttyUSB*
   ```
3. **Read the adapter’s serial number**

   ```bash
   udevadm info -a -n /dev/ttyUSB0 | grep -E "ATTRS?\{serial\}|ID_SERIAL"
   # → AQ01L3KE  (example)
   ```
4. **Create or edit** `/etc/udev/rules.d/99-tactile.rules` and add:

   ```bash
   SUBSYSTEM=="tty", ATTRS{serial}=="AQ01L3KE", ENV{ID_MM_DEVICE_IGNORE}="1", ATTR{device/latency_timer}="1", SYMLINK+="right_finger"
   ```
5. **Unplug the first sensor and plug in the second (left finger)**, then repeat steps 2‑4 with its serial number:

   ```bash
   SUBSYSTEM=="tty", ATTRS{serial}=="B003NWB4", ENV{ID_MM_DEVICE_IGNORE}="1", ATTR{device/latency_timer}="1", SYMLINK+="left_finger"
   ```
6. **Reload the rules and apply them:**

   ```bash
   sudo udevadm control --reload
   sudo udevadm trigger
   ```
7. **Verify**

   ```bash
   ls -l /dev/right_finger /dev/left_finger
   ```

You should now see two stable links:

```
/dev/right_finger -> ttyUSB?
/dev/left_finger  -> ttyUSB?
```

Use these in your launch files and scripts; they remain constant regardless of USB port or reboot.


## <a id="ros"></a>🛠️ ROS 2 Setup

Tested on **Ubuntu 22.04**.

1. **Install ROS 2 Humble**
   Follow the canonical ROS tutorial for Ubuntu 22.04:
   [https://docs.ros.org/en/humble/Installation.html](https://docs.ros.org/en/humble/Installation.html)

2. **Create or choose a ROS 2 workspace**

   ```bash
   # Example: make a workspace in your home directory
   git clone git@github.com:YolandaXinyueZhu/touch_in_the_wild_ros2.git
   cd ~/touch_in_the_wild_ros2
   ```

3. **Build the packages once (any one terminal)**

   ```bash
   colcon build
   ```

4. **Open **four** new terminals** (call them *T1 … T4*) and in **each** run:

   ```bash
   # Terminal-specific session setup
   colcon build 
   source install/setup.bash  
   ```

   > *Why four?* — Each terminal will host one ROS node or script (sensor, viewer, QR display, recorder). Running them in separate shells keeps logs readable and lets you stop only the recorder when a session ends.


### Terminal Commands

| Terminal | Node / Script      | Command                           | What it does                                                                                                         |
| -------- | ------------------ | --------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **1**    | tactile\_sensor    | `ros2 run tactile tactile_sensor` | Reads data from `/dev/right_finger` & `/dev/left_finger`, publishes `/tactile_input_left` & `/tactile_input_right`.  |
| **2**    | tactile\_viewer    | `ros2 run tactile tactile_viewer` | Opens an OpenCV window showing left and right tactile frames in real time.                                           |
| **3**    | qr                 | `ros2 run tactile qr`             | Displays a full‑screen QR code with the current Unix timestamp. **Scan before each demo.**                           |
| **4**    | record\_tactile.sh | `./record_tactile.sh`             | Starts `ros2 bag record` on both tactile topics, prints a running timer, and after Ctrl +C converts the bag to JSON. |

> **Start order:** 1 → 2 → 3 → 4. Launch Terminal 4 only after the first three report “ready”. Do **not** stop Terminal 4 until the *entire* data‑collection session finishes.


## <a id="protocol"></a>📷 Data Collection Protocol

> Prerequisite: Prepare an Ubuntu machine with the ROS 2 Humble installation above, then connect the Arduino boards of both tactile sensors to that machine.  

1. **Start all four ROS terminals** (see §ROS 2 Setup). Keep the recorder running **throughout the session**.
2. **For each demonstration:**

   1. Hold the gripper roughly 1 m from the screen, directly in front of the QR-code board shown on **Terminal 3**, and keep it there for about 2 seconds so the GoPro 9 can record the scan.
      *QR tags embed the current Unix time so we can align video frames later.*
   2. When ready, cover the GoPro lens with your palm briefly; the resulting black frame marks the start of the episode
   3. Perform the manipulation demonstration. (For details about collection demonstrations with UMI gripper, reference: [UMI data collection guide](https://swanky-sphere-ad1.notion.site/UMI-Data-Collection-Instruction-4db1a1f0f2aa4a2e84d9742720428b4c))
3. After the last demo, press Ctrl +C only in **Terminal 4** to stop recording and convert the rosbag to JSON.
4. **Gather your dataset**: move all GoPro .mp4 files plus the generated tactile JSON file into a single directory — this is the input to the SLAM pipeline.

> **Why the QR node is mandatory:** GoPro 9 does not embed reliable timestamps. The QR timestamp + black‑frame marker is a hardware‑free method that allows sub‑frame alignment between tactile and video streams.

Refer to [Touch in the Wild](https://github.com/YolandaXinyueZhu/touch_in_the_wild) for SLAM pipeline setup.


## <a id="trouble"></a>❓ Troubleshooting

| Symptom                             | Likely Cause              | Quick Fix                                     |
| ----------------------------------- | ------------------------- | --------------------------------------------- |
| `serial.SerialException` on startup | Wrong `/dev/<name>`       | Check udev rules; `ls -l /dev/right_finger`   |
| Viewer is black                     | Calibration still running | Wait ≈ 10 s after launching `tactile_sensor`  |
| No `/tactile_input_*` topics        | Workspace not sourced     | `source install/setup.bash` in every terminal |
| Bag size 0 B                        | Stopped wrong terminal    | Press Ctrl +C **only** in Terminal 4          |
| SLAM fails to find the QR code      | Glare / out of focus      | Re‑scan QR closer; avoid reflections          |

Open an issue if you encounter anything not covered here—screenshots & logs help us help you 🙂

## <a id="ackowledgement"></a>🏷️ Acknowledgement

This hardware codebase build upon [3D‑ViTac\_Tactile\_Hardware](https://github.com/binghao-huang/3D-ViTac_Tactile_Hardware).

## <a id="license"></a>🏷️ License

FlexiTac © 2025 by Columbia University is licensed under CC BY-NC 4.0. To view a copy of this license, visit https://creativecommons.org/licenses/by-nc/4.0/


