# 机械臂独立板卡对接协议（上位机）

本文档用于对接 **机械臂独立控制板卡**。  
机械臂控制链路与底盘控制链路完全分离：机械臂命令不应下发到底盘板卡。

## 1. 适用范围与目标

- 适用于上位机与机械臂控制板卡位于同一局域网、不同板卡部署的场景。
- 协议层定义消息结构和语义，不绑定具体硬件型号。
- 传输层依赖 ROS 话题链路（JSON 负载封装在 `std_msgs/String`）。

目标：

- 保证机械臂与底盘控制解耦。
- 保证跨板卡通信可联通、可观测、可诊断。
- 在保持现有 API 的前提下，补充可靠性约束。

## 2. 架构与链路边界

- 底盘板卡：负责导航、速度控制、建图等底盘能力。
- 机械臂板卡：负责机械臂动作执行、夹爪控制、动作状态回传。
- 上位机（本项目）：统一 UI，分别向不同控制链路下发命令。

建议链路：

- 底盘链路：`/cmd_vel`、导航/建图相关接口（保持现有）。
- 机械臂链路：`/robot_arm/*`（本文档定义，支持环境变量改名）。

## 3. 跨板卡通信前提（同一局域网）

本协议支持跨不同板卡通信，但需要满足以下网络前提：

- 上位机与机械臂板卡网络互通（可达、无 ACL 阻断）。
- ROS 运行环境互通（同 ROS 网络域，名称解析和端口可达）。
- 双方对齐话题命名空间（建议统一 `MANIPULATOR_TOPIC_NS`）。
- 双方时间建议同步（NTP），便于按 `timestamp` 排障。

建议最小联通检查：

1. IP 层可达：上位机能访问机械臂板卡 IP。
2. ROS 话题可见：上位机可看到 `${MANIPULATOR_TOPIC_NS}/status`。
3. 控制闭环可达：发送 `get_status` 后状态有回包且 `request_id` 对应。

## 4. 协议总览

机械臂消息统一使用 `std_msgs/String`，内容为 JSON 字符串。

- 启动控制（上位机 -> 机械臂板卡）
  - 话题：`${MANIPULATOR_TOPIC_NS}/start`
- 动作控制（上位机 -> 机械臂板卡）
  - 话题：`${MANIPULATOR_TOPIC_NS}/command`
- 状态上报（机械臂板卡 -> 上位机）
  - 话题：`${MANIPULATOR_TOPIC_NS}/status`

默认 `MANIPULATOR_TOPIC_NS=/robot_arm`。

## 5. 通用消息规范（新增）

### 5.1 公共字段

所有上位机下发消息建议包含：

- `type`：消息类型。
- `request_id`：请求唯一 ID（必须全局唯一，建议前缀 + 时间戳）。
- `timestamp`：毫秒时间戳。
- `source`：请求来源，建议 `web_ui`、`api`、`task_engine`。
- `protocol_version`：协议版本，建议固定为 `1.1`（兼容 1.0）。

### 5.2 幂等与重试约束（新增）

- 机械臂板卡应以 `request_id` 作为幂等键，重复消息不重复执行。
- 上位机超时后允许重发同一个 `request_id`（建议最多 2 次）。
- `stop` 必须高优先级、幂等、安全可重复下发。

### 5.3 状态回执约束（新增）

机械臂板卡处理控制命令后，应在状态中回传：

- `last_request_id`：最近完成处理的请求 ID。
- `ack`：`accepted|running|done|rejected|timeout`。

> 说明：为兼容已有实现，`request_id` 字段保留；新实现建议优先使用 `last_request_id` 表达“最新处理结果关联”。

## 6. 启动控制协议

### 6.1 请求体（上位机下发）

```json
{
  "type": "manipulator_start",
  "enable": true,
  "mode": "manual",
  "source": "web_ui",
  "protocol_version": "1.1",
  "request_id": "start-1714471200123",
  "timestamp": 1714471200123
}
```

字段说明：

- `type`：固定 `manipulator_start`。
- `enable`：`true` 启动机械臂控制；`false` 停止机械臂控制。
- `mode`：控制模式，当前建议 `manual`。
- `source`：请求来源。
- `protocol_version`：协议版本。
- `request_id`：请求唯一 ID（用于上下游日志关联）。
- `timestamp`：毫秒时间戳。

### 6.2 语义约束

- 启动命令仅作用于机械臂板卡，不影响底盘运动状态。
- 停止命令应使机械臂进入安全可控状态（例如停止轨迹执行）。

## 7. 动作控制协议

### 7.1 通用请求体

```json
{
  "type": "manipulator_command",
  "command": "pick_place",
  "params": {},
  "source": "web_ui",
  "protocol_version": "1.1",
  "request_id": "cmd-1714471220999",
  "timestamp": 1714471220999
}
```

字段说明：

- `type`：固定 `manipulator_command`。
- `command`：动作命令字。
- `params`：命令参数对象。
- `source`：请求来源。
- `protocol_version`：协议版本。
- `request_id`：请求唯一 ID。
- `timestamp`：毫秒时间戳。

### 7.2 命令字定义

支持命令：

- `pick_place`
- `pick`
- `place`
- `home`
- `stop`
- `gripper`
- `get_status`

其中 `gripper` 示例：

```json
{
  "type": "manipulator_command",
  "command": "gripper",
  "params": {
    "action": "open"
  },
  "source": "web_ui",
  "protocol_version": "1.1",
  "request_id": "cmd-1714471230555",
  "timestamp": 1714471230555
}
```

`params.action` 建议取值：

- `open`
- `close`

## 8. 状态上报协议

机械臂板卡向 `${MANIPULATOR_TOPIC_NS}/status` 发布状态。

建议报文：

```json
{
  "type": "manipulator_status",
  "state": "idle",
  "running": false,
  "ack": "done",
  "error_code": 0,
  "error_message": "",
  "last_request_id": "cmd-1714471220999",
  "request_id": "cmd-1714471220999",
  "board_id": "arm-controller-01",
  "timestamp": 1714471240000
}
```

建议字段：

- `state`：`idle|running|paused|stopped|error`。
- `running`：是否处于动作执行态。
- `ack`：`accepted|running|done|rejected|timeout`。
- `error_code`：0 为正常，非 0 表示故障。
- `error_message`：错误描述。
- `last_request_id`：最近处理请求 ID（推荐）。
- `request_id`：兼容字段，可与 `last_request_id` 一致。
- `board_id`：机械臂板卡标识（跨板卡排障推荐）。
- `timestamp`：毫秒时间戳。

## 9. 错误码建议（新增）

建议保留设备自定义空间，但统一公共错误码区间：

- `0`：成功。
- `1001`：参数错误（缺字段/字段非法）。
- `1002`：命令不支持。
- `1003`：机械臂忙（不可受理新命令）。
- `1004`：安全互锁触发。
- `2001`：执行超时。
- `2002`：硬件故障。
- `9001`：通信链路异常（上位机侧判定）。

## 10. 上位机 REST 接口（已实现）

前端通过 REST 调用后端，后端再转发到机械臂话题：

- `POST /api/manipulator/start`
  - body：
  ```json
  { "enable": true, "mode": "manual", "source": "web_ui" }
  ```
- `POST /api/manipulator/command`
  - body：
  ```json
  { "command": "pick_place", "params": {} }
  ```
- `GET /api/manipulator/status`
  - 返回最近一次机械臂状态原始报文（用于调试）。

兼容建议：

- 后端可自动补齐 `protocol_version`、`request_id`、`timestamp`。
- 旧板卡若不识别新增字段，应按“忽略未知字段”处理。

## 11. 联调与上线建议（独立板卡场景）

- 机械臂板卡与底盘板卡日志分开记录，使用 `request_id` 串联链路。
- 对 `stop` 命令设置最高优先级，并保证幂等。
- `get_status` 返回完整状态快照，便于 UI 刷新。
- 若机械臂板卡离线，上位机显示“机械臂链路异常”，但不阻断底盘控制。
- 对跨板卡场景增加链路健康检查：
  - 上位机定时检查 `status` 新鲜度（例如 2 秒窗口）。
  - 超过阈值后标记离线，并提示网络/ROS 连接异常。

## 12. 工程配置项

后端支持通过环境变量设置机械臂话题命名空间：

- `MANIPULATOR_TOPIC_NS`，默认值：`/robot_arm`。

示例：

```bash
export MANIPULATOR_TOPIC_NS=/robot_arm
```

则机械臂三条话题为：

- `${MANIPULATOR_TOPIC_NS}/start`
- `${MANIPULATOR_TOPIC_NS}/command`
- `${MANIPULATOR_TOPIC_NS}/status`
