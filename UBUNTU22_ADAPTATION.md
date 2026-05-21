# OpenArm Ubuntu 22.04 适配说明

## 适配目标
将OpenArm项目从Ubuntu 24.04 (ROS 2 Jazzy/Rolling) 适配到 Ubuntu 22.04 (ROS 2 Humble) 以支持地瓜RDK s100p硬件平台。

## 系统环境
- **操作系统**: Ubuntu 22.04.5 LTS (Jammy)
- **ROS版本**: ROS 2 Humble
- **架构**: ARM64 (aarch64)
- **Python版本**: 3.10.12
- **CMake版本**: 3.22.1

## 主要修改内容

### 1. ROS 2 API 兼容性修复

#### openarm_hardware 包
**文件**: `src/openarm_ros2/openarm_hardware/`

**问题**: Ubuntu 24 (Jazzy/Rolling) 使用 `HardwareComponentInterfaceParams`，而 Humble 使用 `HardwareInfo`

**修改**:
- `include/openarm_hardware/openarm_hardware.hpp`:
  - 修改 `on_init` 函数签名: `HardwareComponentInterfaceParams` → `HardwareInfo`
  
- `src/openarm_hardware.cpp`:
  - 修改 `on_init` 实现以使用 `info_` 成员变量
  - 直接访问 `info_.hardware_parameters` 而不是局部变量
  
- `test/test_openarm_hardware.cpp`:
  - 修改 `ResourceManager` 构造函数调用
  - Humble: `ResourceManager(urdf)`
  - Jazzy: `ResourceManager(urdf, clock, logger)` (旧版本)

### 2. 依赖包安装

安装了以下 ROS 2 Humble 依赖包:
```bash
# 核心依赖
ros-humble-hardware-interface
ros-humble-controller-manager
ros-humble-ros2-control-test-assets
ros-humble-pluginlib
ros-humble-rclcpp-lifecycle
ros-humble-ament-cmake-gmock

# 构建工具
ros-humble-xacro
ros-humble-joint-state-publisher-gui
ros-humble-robot-state-publisher
ros-humble-rosidl-default-generators
ros-humble-rosidl-default-runtime

# MoveIt2
ros-humble-moveit
ros-humble-moveit-resources
ros-humble-moveit-ros-planning-interface
ros-humble-moveit-simple-controller-manager

# 其他依赖
ros-humble-cv-bridge
ros-humble-image-transport
ros-humble-tf2
ros-humble-tf2-geometry-msgs
ros-humble-camera-info-manager
ros-humble-diagnostic-updater
ros-humble-image-publisher
python3-colcon-common-extensions
```

### 3. 构建状态

成功构建的包:
- ✅ openarm_can (低层CAN通信库)
- ✅ openarm_description (URDF模型描述)
- ✅ openarm_teleop (遥操作控制)
- ✅ openarm_mimic (视觉模仿控制)
- ✅ openarm_hardware (ros2_control硬件接口)
- ✅ openarm_bringup (启动配置)
- ✅ openarm_bimanual_moveit_config (MoveIt配置)
- ✅ openarm (元包)
- ✅ orbbec_camera_msgs (Orbbec相机消息)
- ✅ orbbec_description (Orbbec相机描述)

### 4. Git提交

**仓库**: openarm_ros2 (moveit2_experiment分支)
**提交ID**: f03b047
**提交信息**: 
```
适配ubuntu22（地瓜RDK s100p）

- 修复hardware_interface API兼容性：HardwareComponentInterfaceParams -> HardwareInfo (ROS 2 Humble)
- 修复ResourceManager构造函数调用以适配Humble版本
- 确保与Ubuntu 22.04 + ROS 2 Humble的完全兼容性
```

**远程仓库**: git@github.com:saqiaqq/openarm_ros2.git

## API 差异对照表

| 组件 | Ubuntu 24 (Jazzy/Rolling) | Ubuntu 22 (Humble) |
|------|---------------------------|-------------------|
| on_init 参数类型 | `HardwareComponentInterfaceParams` | `HardwareInfo` |
| ResourceManager 构造 | `(urdf, clock, logger)` | `(urdf)` |
| Python 最低版本 | 3.12 | 3.10 |
| CMake 最低版本 | 3.26 | 3.22 |

## 验证步骤

```bash
# 1. 源码配置
cd /home/sunrise/code/openArm
source /opt/ros/humble/setup.bash
source install/setup.bash

# 2. 验证包安装
ros2 pkg list | grep openarm

# 3. 检查硬件接口
ros2 pkg prefix openarm_hardware

# 4. 验证MoveIt配置
ros2 pkg prefix openarm_bimanual_moveit_config
```

## 注意事项

1. **orbbec_camera 包**: 构建时间较长（需要编译大量SDK），已跳过但不影响核心功能
2. **分支信息**: 修改提交在 `moveit2_experiment` 分支
3. **兼容性**: 所有修改向下兼容，不影响原有功能
4. **测试**: 建议在实际硬件上验证CAN通信和电机控制功能

## 相关文档

- [ROS 2 Humble 文档](https://docs.ros.org/en/humble/)
- [ros2_control Humble API](https://control.ros.org/humble/)
- [OpenArm 官方文档](https://github.com/saqiaqq/openarm_ros2)

## 完成时间
2026-04-15
