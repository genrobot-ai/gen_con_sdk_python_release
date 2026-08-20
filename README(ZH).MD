# Gen Gripper Controller Python SDK

> 用于 Gen Controller 夹爪的纯 Python SDK，支持相机图像、触觉传感、编码器反馈及开合距离控制，无需 ROS。

[English](README.MD)

[GitHub代码](https://github.com/genrobot-ai/gen_con_sdk_python_release)

License: [MIT License](LICENSE)

## 1 功能特性

- 纯 Python SDK，无 ROS 依赖
- 三相机图像流，支持实时预览
- 通过回调获取触觉数据（左 / 右）
- 夹爪开合距离编码器反馈
- 通过 `DataBus.set_target_distance()` 控制夹爪开合
- 通过 `left` / `right` 参数支持单爪 / 双爪
- 标定、设备 ID、触觉调试等工具脚本

## 2 环境要求


| 项目     | 要求                  |
| ------ | ------------------- |
| 系统     | Ubuntu 20.04（推荐）    |
| Python | 3.8+                |
| USB    | USB 3.0 接口          |
| 硬件     | Gen Controller 夹爪设备 |

## 注意事项

使用夹爪前，请注意以下事项：

1. **初始校准**：首次发送指令前，确保夹爪中间无物块（校准码：-66.66）。
2. **电源供应**：若错误码显示 -6.6，表示电机未上电。请接入 24V 电源到 XT30 接口。
3. **断电重启校准**：电机断电重启后需要重新校准。重新上电前请移除夹爪中间的物块。
4. **断电延迟**：电机断电后，至少等待 3 秒再次上电。避免在短时间内频繁切断/接通电源，以防电机损伤。
5. **更换夹爪后重新初始化**：更换不同型号夹爪后，请重新对 USB 串口或电机供电接口执行断电再上电，这样可确保夹爪重新校准正确的零点。

## 3 快速开始


> 首次使用请先完成 [USB 配置](docs/usb-setup_CN.md)。

```shell
git clone https://github.com/genrobot-ai/gen_con_sdk_python_release.git
cd gen_con_sdk_python_release
pip3 install -r requirements.txt
python3 start_gripper.py left
```

默认弹出三个预览窗口，编码器距离与触觉数据会打印到终端。设置固定开合距离：

```shell
python3 start_gripper.py left --distance 0.05
```

通过 `--gripper-type` 切换夹爪类型。不同夹爪的最大打开后距离如下：

| 夹爪类型 | 最大打开后距离 |
| --- | ---: |
| `default_gripper` | `0.103` m |
| `tactile_gripper` | `0.0998` m |
| `soft_gripper` | `0.103` m |



## 4 程序接口



### 4.1 传感器回调

将以下回调传入 `GripperSystem`（示例见 [start_gripper.py](start_gripper.py)）：


| 回调                        | 说明              | 数据格式                                      |
| ------------------------- | --------------- | ----------------------------------------- |
| `capture_frames_callback` | 相机帧采集循环         | 接收 `CameraCapture` 实例                     |
| `tactile_callback`        | 触觉数据（左 / 右）      | `bytes` — 448 字节原始触觉记录                    |
| `encoder_callback`        | 夹爪开合距离反馈        | `bytes` — 大端 float（`struct.unpack(">f")`） |


预览窗口对应三台相机：

| 窗口         | 相机   |
| ---------- | ---- |
| `camera_0` | 中间相机 |
| `camera_1` | 左侧相机 |
| `camera_2` | 右侧相机 |




### 4.2 夹爪控制

通过 `DataBus` 发送目标开合距离：

```python
if self.system.databus:
    self.system.databus.set_target_distance(value)
```

目标距离不要超过所选夹爪类型的最大打开后距离。

双爪场景下，分别以 `left` 或 `right` 启动独立进程 — 各自连接对应串口（`/dev/ttyDeviceLeft` 或 `/dev/ttyDeviceRight`）。

## 5 安装



### 5.1 安装系统与 Python 依赖

```shell
sudo apt update
sudo apt install python3-pip v4l-utils
pip3 install -r requirements.txt
```

`v4l-utils` 提供 USB 配置所需的 `v4l2-ctl` 命令。

### 5.2 拉取仓库

```shell
git clone https://github.com/genrobot-ai/gen_con_sdk_python_release.git
cd gen_con_sdk_python_release
pip3 install -r requirements.txt
```



## 6 USB 配置

首次使用前需为每个 USB 口配置 udev 规则，模板见 [config/99-usb-serial.rules](./config/99-usb-serial.rules)。

简要步骤：

1. 用 `udevadm` 和 `v4l2-ctl` 查询串口与相机的 `KERNELS` 值
2. 编辑 `config/99-usb-serial.rules`
3. 复制到 `/etc/udev/rules.d/` 并 reload

详细图文步骤见：

- [USB 配置指南 (ZH)](docs/usb-setup_CN.md)
- [USB Configuration Guide (EN)](docs/usb-setup.md)

双爪配置后的默认串口软链接：`/dev/ttyDeviceLeft`、`/dev/ttyDeviceRight`。

## 7 使用方法



### 7.1 单爪启动

```shell
cd gen_con_sdk_python_release

python3 start_gripper.py left                              # 固定开合 5 cm（默认）
python3 start_gripper.py left --distance 0.08              # 固定开合 8 cm
python3 start_gripper.py left --gripper-type tactile_gripper
python3 start_gripper.py left --gripper-type soft_gripper
python3 start_gripper.py left --sine-wave                    # 连续开合 10 秒
python3 start_gripper.py left --camera-fps 60                # V4 Controller：设为 60 fps 以获得 30 fps 输出
python3 start_gripper.py left --no-preview                   # 关闭预览窗口
python3 start_gripper.py left --print-tactile-info           # 在终端打印触觉网格
```



### 7.2 双爪启动

```shell
cd gen_con_sdk_python_release

# 终端 1 — 左夹爪
python3 start_gripper.py left

# 终端 2 — 右夹爪
python3 start_gripper.py right

# 可选：打印右夹爪触觉网格
python3 start_gripper.py right --print-tactile-info

# V4 Controller 帧率
python3 start_gripper.py left --camera-fps 60
```

共弹出六个预览窗口（每爪三个）。**不要**同时运行多个设备工具命令。

### 7.3 设备工具命令

使用 [scripts/camera_cmd.py](scripts/camera_cmd.py) 进行标定与设备 ID 查询，无需 ROS 或额外守护进程。

**单设备：**

```shell
python3 scripts/camera_cmd.py camerarc   # 640x480 中间相机标定
python3 scripts/camera_cmd.py camerarl   # 640x480 左侧相机标定
python3 scripts/camera_cmd.py camerarr   # 640x480 右侧相机标定
python3 scripts/camera_cmd.py camerar2c  # 1600x1296 中间相机标定
python3 scripts/camera_cmd.py camerar2l  # 1600x1296 左侧相机标定
python3 scripts/camera_cmd.py camerar2r  # 1600x1296 右侧相机标定
python3 scripts/camera_cmd.py MCUID      # 设备 ID
python3 scripts/camera_cmd.py VERSION    # 查询当前软件/固件版本
```

**双设备（左 / 右）：**

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

标定生成的 YAML 文件保存在 `scripts/calib_result/` 目录下。

说明：
- `camerarl`、`camerarc`、`camerarr` 是 `640x480` 分辨率下的标定文件。
- `camerar2l`、`camerar2c`、`camerar2r` 是 `1600x1296` 分辨率下的标定文件。
- `VERSION` 可查询当前软件版本；如果查询不到，说明版本过旧。



## 8 常见问题


| 问题                | 解决方法                                                      |
| ----------------- | --------------------------------------------------------- |
| 找不到串口             | 执行 `sudo apt remove brltty`，重新插拔设备                        |
| 相机或串口路径不对         | 检查 udev 规则，见 [docs/usb-setup_CN.md](docs/usb-setup_CN.md) |
| V4 Controller 帧率减半 | 向 `start_gripper.py` 传入 `--camera-fps 60`                    |
| 导入或模块错误           | 在仓库根目录执行 `pip3 install -r requirements.txt`               |
| 无传感器数据            | 确认 udev 规则已加载且存在 `/dev/ttyDevice*` 软链接                   |
| 串口权限不足            | 重新加载 udev 规则，或执行 `sudo chmod 666 /dev/ttyDeviceLeft`（或 Right） |




## 9 文档索引


| 说明 | 链接 |
| -------------- | ---------------------------------------------------------------------------------- |
| 用户手册 | [DAS Controller 用户手册](https://docs.genrobot.ai/zh/products/das-controller) |
| URDF | [URDF 模型](https://zcnma1sv5kma.feishu.cn/wiki/CKpbwye45iOlrckukIPc5hKCndh) |
| 问题解决指南 | [Gripper Controller 问题指南](https://zcnma1sv5kma.feishu.cn/docx/E6VJdNqIfoGrSDxKIHKck7tLn3b) |
| USB 配置 (ZH) | [docs/usb-setup_CN.md](docs/usb-setup_CN.md) |
| USB setup (EN) | [docs/usb-setup.md](docs/usb-setup.md)                                             |
| udev 规则模板      | [config/99-usb-serial.rules](config/99-usb-serial.rules)                           |
| 主入口脚本        | [start_gripper.py](start_gripper.py)                                               |
| 设备工具脚本       | [scripts/camera_cmd.py](scripts/camera_cmd.py)                                     |
| SDK 模块         | [scripts/](scripts/)                                                               |
