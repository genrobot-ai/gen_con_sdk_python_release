# Gen Gripper Controller Python SDK

> Pure Python SDK for Gen Controller gripper — camera streaming, tactile sensing, encoder feedback, and distance control. No ROS dependency.

[中文](README(ZH).MD)

[GitHub repository](https://github.com/genrobot-ai/gen_con_sdk_python_release)

License: [MIT License](LICENSE)

## 1 Features

- Pure Python SDK with no ROS dependency
- Three-camera image streaming with optional live preview
- Tactile sensor data via callbacks (left / right)
- Encoder feedback for gripper opening distance
- Gripper distance control via `DataBus.set_target_distance()`
- Single-gripper and dual-gripper support via `left` / `right` CLI argument
- Utility scripts for calibration, device ID, and tactile debugging

## 2 Requirements


| Item     | Requirement                   |
| -------- | ----------------------------- |
| OS       | Ubuntu 20.04 (recommended)    |
| Python   | 3.8+                          |
| USB      | USB 3.0 port                  |
| Hardware | Gen Controller gripper device |


## Important Notes

Before operating the gripper, observe the following:

1. **Initial Calibration**: On first command, ensure the gripper is unobstructed (calibration code: -66.66).
2. **Power Supply**: Error code -6.6 indicates unpowered motor. Connect 24V power to the XT30 connector.
3. **Power Cycle Recalibration**: After power loss and restart, recalibration is required. Remove any objects from the gripper before power restoration.
4. **Power Cycle Delay**: Wait at least 3 seconds after power-off before powering on the motor again. Avoid rapid power cycling to prevent motor damage.
5. **Gripper Change Reinitialization**: After replacing the gripper with another model, power-cycle the USB serial port or the motor power interface. This ensures the gripper recalibrates the correct zero point.


## 3 Quick Start

> First-time users must complete [USB configuration](docs/usb-setup.md) before launching the driver.

```shell
git clone https://github.com/genrobot-ai/gen_con_sdk_python_release.git
cd gen_con_sdk_python_release
pip3 install -r requirements.txt
python3 start_gripper.py left
```

Three preview windows appear by default. Encoder distance and tactile data are printed to the terminal. Set a fixed opening distance:

```shell
python3 start_gripper.py left --distance 0.05
```

Select the gripper type with `--gripper-type`. The maximum opening distance depends on the gripper:

| Gripper type | Maximum opening distance |
| --- | ---: |
| `default_gripper` | `0.103` m |
| `tactile_gripper` | `0.0998` m |
| `soft_gripper` | `0.103` m |



## 4 Program Interface



### 4.1 Sensor Callbacks

Pass these callbacks to `GripperSystem` (see [start_gripper.py](start_gripper.py) for examples):


| Callback                  | Description                          | Data format                                      |
| ------------------------- | ------------------------------------ | ------------------------------------------------ |
| `capture_frames_callback` | Camera frame capture loop            | Receives `CameraCapture` instance                |
| `tactile_callback`        | Tactile sensor data (left / right)   | `bytes` — 448-byte raw tactile record            |
| `encoder_callback`        | Gripper opening distance feedback    | `bytes` — big-endian float (`struct.unpack(">f")`) |


Preview windows map to three cameras:

| Window    | Camera        |
| --------- | ------------- |
| `camera_0` | Center camera |
| `camera_1` | Left camera   |
| `camera_2` | Right camera  |




### 4.2 Gripper Control

Send target opening distance through `DataBus`:

```python
if self.system.databus:
    self.system.databus.set_target_distance(value)
```

Use a target distance within the maximum opening distance for the selected gripper type.

For dual-gripper setups, run separate processes with `left` or `right` — each connects to its own serial port (`/dev/ttyDeviceLeft` or `/dev/ttyDeviceRight`).

## 5 Installation



### 5.1 Install system and Python dependencies

```shell
sudo apt update
sudo apt install python3-pip v4l-utils
pip3 install -r requirements.txt
```

`v4l-utils` provides `v4l2-ctl`, required for USB configuration.

### 5.2 Clone the repository

```shell
git clone https://github.com/genrobot-ai/gen_con_sdk_python_release.git
cd gen_con_sdk_python_release
pip3 install -r requirements.txt
```



## 6 USB Configuration

Configure udev rules once per USB port before first use. The template is at [config/99-usb-serial.rules](./config/99-usb-serial.rules).

Summary:

1. Query serial and camera `KERNELS` values with `udevadm` and `v4l2-ctl`
2. Edit `config/99-usb-serial.rules`
3. Copy to `/etc/udev/rules.d/` and reload rules

For step-by-step instructions with screenshots, see:

- [USB 配置指南 (ZH)](docs/usb-setup_CN.md)
- [USB Configuration Guide (EN)](docs/usb-setup.md)

Default serial symlinks after dual-gripper setup: `/dev/ttyDeviceLeft`, `/dev/ttyDeviceRight`.

## 7 Usage



### 7.1 Single Gripper

This is a runtime driver. It keeps the connection open, streams camera/tactile/encoder continuosly, and stays running until you stop it (Ctrl+C)

```shell
cd gen_con_sdk_python_release

python3 start_gripper.py left                              # Fixed open at 5 cm (default)
python3 start_gripper.py left --distance 0.08              # Fixed open at 8 cm
python3 start_gripper.py left --gripper-type tactile_gripper
python3 start_gripper.py left --gripper-type soft_gripper
python3 start_gripper.py left --sine-wave                    # Open/close continuously for 10 s
python3 start_gripper.py left --camera-fps 60                # V4 Controller: set 60 fps for 30 fps output
python3 start_gripper.py left --no-preview                   # Disable preview windows
python3 start_gripper.py left --print-tactile-info           # Print tactile grid to terminal
```



### 7.2 Dual Gripper

```shell
cd gen_con_sdk_python_release

# Terminal 1 — left gripper
python3 start_gripper.py left

# Terminal 2 — right gripper
python3 start_gripper.py right

# Optional: print tactile grid for right gripper
python3 start_gripper.py right --print-tactile-info

# V4 Controller frame rate
python3 start_gripper.py left --camera-fps 60
```

Six preview windows appear (three per gripper). Do **not** run multiple device utility commands simultaneously.

### 7.3 Device Utilities

Use [scripts/camera_cmd.py](scripts/camera_cmd.py) for calibration and device ID queries. This is a **one-shot CLI**, seperate fron the runtime driver. It opens the serial port, sends exactly one command, and exits.No ROS or separate daemon required.

**Single device:**

```shell
python3 scripts/camera_cmd.py camerarc   # 640x480 center camera calibration
python3 scripts/camera_cmd.py camerarl   # 640x480 left camera calibration
python3 scripts/camera_cmd.py camerarr   # 640x480 right camera calibration
python3 scripts/camera_cmd.py camerar2c  # 1600x1296 center camera calibration
python3 scripts/camera_cmd.py camerar2l  # 1600x1296 left camera calibration
python3 scripts/camera_cmd.py camerar2r  # 1600x1296 right camera calibration
python3 scripts/camera_cmd.py MCUID      # Device ID
python3 scripts/camera_cmd.py VERSION    # Query current software/firmware version
```

**Dual device (left / right):**

```shell
python3 scripts/camera_cmd.py left camerarc
python3 scripts/camera_cmd.py left camerarl
python3 scripts/camera_cmd.py left camerarr
python3 scripts/camera_cmd.py left camerar2c
python3 scripts/camera_cmd.py left camerar2l
python3 scripts/camera_cmd.py left camerar2r
python3 scripts/camera_cmd.py left MCUID
python3 scripts/camera_cmd.py left VERSION

python3 scripts/camera_cmd.py right camerarc
python3 scripts/camera_cmd.py right camerarl
python3 scripts/camera_cmd.py right camerarr
python3 scripts/camera_cmd.py right camerar2c
python3 scripts/camera_cmd.py right camerar2l
python3 scripts/camera_cmd.py right camerar2r
python3 scripts/camera_cmd.py right MCUID
python3 scripts/camera_cmd.py right VERSION
```

Calibration YAML files are saved under `scripts/calib_result/`.

Notes:
- `camerarl`, `camerarc`, `camerarr` are for `640x480` calibration files.
- `camerar2l`, `camerar2c`, `camerar2r` are for `1600x1296` calibration files.
- `VERSION` can query the current software version. If no response is returned, the device firmware/software is too old.



## 8 Troubleshooting


| Problem                         | Solution                                                                 |
| ------------------------------- | ------------------------------------------------------------------------ |
| Serial port not found           | Run `sudo apt remove brltty`, then replug the device                     |
| Camera or serial has wrong path | Re-check udev rules; see [docs/usb-setup.md](docs/usb-setup.md)          |
| V4 Controller frame rate halved | Pass `--camera-fps 60` to `start_gripper.py`                             |
| Import or module errors         | Run `pip3 install -r requirements.txt` in the repository root            |
| No sensor data                  | Confirm udev rules are loaded and `/dev/ttyDevice*` symlinks exist       |
| Permission denied on serial     | Reload udev rules or run `sudo chmod 666 /dev/ttyDeviceLeft` (or Right) |




## 9 Documentation


| Description           | Link                                                                                     |
| --------------------- | ---------------------------------------------------------------------------------------- |
| User Manual           | [DAS Controller User Manual](https://docs.genrobot.ai/en/products/das-controller)        |
| URDF                  | [URDF Model](https://zcnma1sv5kma.feishu.cn/wiki/CKpbwye45iOlrckukIPc5hKCndh)            |
| Troubleshooting Guide | [Problem Solving Guide](https://zcnma1sv5kma.feishu.cn/docx/E6VJdNqIfoGrSDxKIHKck7tLn3b) |
| USB 配置 (ZH)           | [docs/usb-setup_CN.md](docs/usb-setup_CN.md)                                             |
| USB setup (EN)        | [docs/usb-setup.md](docs/usb-setup.md)                                                   |
| udev rules template   | [config/99-usb-serial.rules](config/99-usb-serial.rules)                                 |
| Main entry script     | [start_gripper.py](start_gripper.py)                                                     |
| Device utility script | [scripts/camera_cmd.py](scripts/camera_cmd.py)                                           |
| SDK modules           | [scripts/](scripts/)                                                                     |
