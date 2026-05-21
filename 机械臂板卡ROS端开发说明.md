# 机械臂板卡 ROS 端开发说明

> **文档定位**：上位机（Web UI / REST `/api/manipulator/*`）与跨板卡话题协议已在独立工程中实现。  
> **本文档范围**：本仓库（openArm ROS 工作空间）在**机械臂板卡**上仍需开发的节点、接口与联调项。  
> **协议依据**：[`MANIPULATOR_PROTOCOL.md`](./MANIPULATOR_PROTOCOL.md)

---

## 1. 背景与分工边界

### 1.1 系统角色

| 角色 | 职责 | 本仓库是否包含 |
|------|------|----------------|
| 上位机 | REST 接口、话题发布 `start`/`command`、订阅 `status`、链路健康检查 | 否（已实现） |
| 机械臂板卡（ROS） | 订阅 `start`/`command`、执行动作、发布 `status` | **待开发** |
| 底盘板卡 | 导航、`/cmd_vel` 等 | 否 |

### 1.2 通信链路（板卡视角）

```
上位机  --publish-->  /robot_arm/start     --subscribe-->  [manipulator_board_node]
上位机  --publish-->  /robot_arm/command   --subscribe-->  [manipulator_board_node]
上位机  --subscribe--> /robot_arm/status    --publish-->   [manipulator_board_node]
                              |
                              v
                    /openarm/pick_place (action)
                    /openarm/stop | goto_home | gripper (service)
                              |
                              v
                    MoveIt + ros2_control + 硬件
```

话题命名空间由环境变量 `MANIPULATOR_TOPIC_NS` 决定，默认 `/robot_arm`。  
板卡 launch 必须与上位机使用**相同**命名空间与 **ROS 域**（`ROS_DOMAIN_ID` 等）。

### 1.3 本仓库已有能力（复用，不重复开发）

以下包已在板卡本地运行，新节点只做**协议适配**，不替代技能层：

| 包 | 作用 |
|----|------|
| `openarm_ros2` / `openarm_bringup` | 控制器、MoveIt 启动 |
| `openarm_skills` | `/openarm/pick_place`、`/openarm/stop`、`/openarm/goto_home`、`/openarm/gripper` |
| `openarm_perception` | `pose_source=camera` 时可选 |
| `openarm_api` | 同机 JSON 网关（`/openarm/command`），**跨板卡场景非必须**，可保留作本地联调 |

### 1.4 固定位置抓取与放置：实现状态

> **需求定义**：在 `base_link`（或约定参考系）下，从**固定位姿 A 抓取**物体，再**放到固定位姿 B**（不依赖相机感知）。  
> **结论摘要**：运动与技能逻辑在 `openarm_skills` 中**已实现**；跨板卡协议入口 `manipulator_board_node` **未实现**；单独「只抓 / 只放」与「空 params 默认点位」**未实现**。

#### 1.4.1 已实现（技能层，可直接复用）

`skill_server_node` 提供 Action `/openarm/pick_place`。当 Goal 中 `pose_source == "upper_computer"` 时，**直接使用调用方传入的 `grasp_pose` / `place_pose`**，不调用感知服务。

实现位置：[`src/openarm_skills/src/skill_server_node.cpp`](./src/openarm_skills/src/skill_server_node.cpp) 中 `execute()` → `doPick()` → `doPlace()`。

| 阶段 | 行为 |
|------|------|
| 抓取 | 移到抓取点上方（关节规划）→ 直线下降到抓取位 → 夹爪闭合 → 抬起 |
| 搬运 | 通过 `place.approach` 关节规划移到放置点上方 |
| 放置 | 直线下降到放置位 → 夹爪打开 → 抬起 |

固定位姿相关 Goal / `params` 字段：

| 字段 | 说明 |
|------|------|
| `arm` | `left` \| `right`（Action 仅支持单臂，非 `both`） |
| `pose_source` | 固定点模式填 `upper_computer` |
| `grasp_pose` / `place_pose` | `xyz`（米）+ `rpy`（弧度），参考系默认 `base_link` |
| `approach_offset_m` / `retreat_offset_m` | 接近/撤离高度，默认 0.05 m |
| `speed_scale` | 慢速段缩放，默认 0.10 |

同机 JSON 网关 [`json_bridge_node.py`](./src/openarm_api/openarm_api/json_bridge_node.py) 已将 `params.grasp_pose` / `params.place_pose` 写入 Action Goal（`pose_source` 默认 `upper_computer`）。  
Schema 要求固定位姿模式下必须提供两位姿：[`pick_place.schema.json`](./src/openarm_api/openarm_api/schemas/pick_place.schema.json)。

#### 1.4.2 未实现或不完整

| 能力 | 状态 | 说明 |
|------|------|------|
| 完整 `pick_place`（A 点抓 → B 点放） | **已实现** | 需板卡已启动 MoveIt + `openarm_skills` |
| 协议话题 `command=pick_place` | **未实现** | 缺 `manipulator_board_node`，上位机命令尚不能自动转到 Action |
| `params: {}` 即用板卡默认点位 | **未实现** | 位姿须由调用方传入，或待在 `manipulator_board.yaml` 中配置（见 §6.2） |
| 单独 `pick`（只抓不放） | **未实现** | `openarm_skills` 的 `execute()` 始终执行完整抓+放；`json_bridge` 对 `pick`/`place`/`pick_place` 均调同一 Action |
| 单独 `place`（只放不抓） | **未实现** | 同上，无「已持物」状态机 |
| 板卡内置 YAML 默认抓取/放置点 | **未实现** | 需在适配层或上位机 `params` 中提供 |

#### 1.4.3 运行前提

固定位姿抓取放置要能执行，板卡上须先启动：

1. MoveIt + 控制器（如 `openarm_bimanual_moveit_config` 对应 launch）
2. `ros2 launch openarm_skills skills.launch.py`

缺任一项时 `/openarm/pick_place` 不可用，表现为「功能未生效」。

#### 1.4.4 验证方式（ROS 直连，不经过上位机协议）

**方式一：CLI 发送 Action**

```bash
ros2 action send_goal /openarm/pick_place openarm_skills/action/PickPlace "{
  cmd_id: 'test-fixed',
  arm: 'right',
  pose_source: 'upper_computer',
  grasp_pose: {position: {x: 0.42, y: 0.10, z: 0.20}, orientation: {x: 0, y: 0.7071, z: 0, w: 0.7071}},
  place_pose: {position: {x: 0.30, y: -0.20, z: 0.20}, orientation: {x: 0, y: 0.7071, z: 0, w: 0.7071}},
  approach_offset_m: 0.05,
  retreat_offset_m: 0.05,
  speed_scale: 0.10,
  timeout_s: 60.0
}" --feedback
```

**方式二：同机 `openarm_api` JSON**

`cmd_type: pick_place`，`pose_source: upper_computer`，`params` 含 `grasp_pose`、`place_pose`（`xyz` + `rpy`）。经 `/openarm/command` 或 `enable_http` 的 `POST /command`。

**方式三：上位机协议（跨板卡，待适配层）**

上位机 `POST /api/manipulator/command` 中 `command: pick_place` 的 `params` 须携带等价位姿；`manipulator_board_node` 需映射为：

| 协议 `params` | Action / 网关字段 |
|---------------|-------------------|
| `grasp_pose` | `grasp_pose`（`xyz`/`rpy` → `geometry_msgs/Pose`） |
| `place_pose` | `place_pose` |
| （可选）`approach_offset_m` 等 | 同名 Goal 字段 |
| — | `pose_source` 固定为 `upper_computer` |
| — | `arm` 取自节点参数 `default_arm` 或协议扩展字段 |

#### 1.4.5 对板卡适配开发的含义

| 你的目标 | 板卡还需开发的内容 |
|----------|-------------------|
| 上位机下发固定位姿，完成抓+放 | **仅需** `manipulator_board_node` 将 `pick_place` + `params` 转成 `/openarm/pick_place`（可复用 `json_bridge` 的 `_do_pick_place` 逻辑） |
| 上位机只发 `pick_place` 且无 params | 须在 `manipulator_board.yaml` 配置默认 `grasp_pose`/`place_pose`（§6.2 策略） |
| 只抓或只放一步 | 须扩展 `openarm_skills` 或适配层明确拒绝/拆分（§6.1） |

阶段二任务（§10）中的 `pick_place` 转发，**不包含**重新实现运动规划，仅接好协议与现有 Action。

---

## 2. 待开发总览

| 序号 | 交付物 | 优先级 | 状态 |
|------|--------|--------|------|
| 1 | ROS 包 `openarm_manipulator_board` | P0 | ✅ 已完成 |
| 2 | 节点 `manipulator_board_node` | P0 | ✅ 已完成 |
| 3 | Launch + 参数 YAML | P0 | ✅ 已完成 |
| 4 | 协议 JSON 校验 / 错误码映射 | P0 | ✅ 已完成 |
| 5 | `pick`/`place` 与空 `params` 产品策略（见 §1.4） | P1 | 🔶 策略已实现（pick/place 返回 1002；空 params 可配置默认点）|
| 6 | 联调检查清单与日志规范 | P1 | 见 §9 |

---

## 3. 新包设计建议

### 3.1 包名与目录结构

建议新建独立包，与 `openarm_api`（上位机/同机网关）解耦：

```text
src/openarm_manipulator_board/
├── package.xml
├── setup.py
├── config/
│   └── manipulator_board.yaml      # board_id、默认 arm、周期、超时、默认位姿等
├── launch/
│   └── manipulator_board.launch.py
└── openarm_manipulator_board/
    ├── __init__.py
    ├── manipulator_board_node.py   # 主节点
    ├── protocol.py                 # 报文解析、字段常量、校验
    ├── protocol_errors.py        # MANIPULATOR_PROTOCOL §9 错误码
    ├── skill_adapter.py            # 协议命令 -> /openarm/* 调用
    └── idempotency_cache.py        # request_id 去重缓存
```

### 3.2 依赖

- `rclpy`
- `std_msgs`（`String`）
- `openarm_skills`（action / srv 类型）
- 可选：`jsonschema`（与 `openarm_api` 一致）

### 3.3 节点参数（建议）

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `topic_ns` | string | `/robot_arm` | 与 `MANIPULATOR_TOPIC_NS` 对齐 |
| `board_id` | string | `arm-controller-01` | 写入 status |
| `status_publish_hz` | double | 5.0 | 周期 status |
| `default_arm` | string | `right` | 协议未带 `arm` 时使用 |
| `default_pose_source` | string | `upper_computer` | pick_place 默认 |
| `command_timeout_s` | double | 60.0 | 长动作总超时 |
| `idempotency_cache_size` | int | 256 | request_id 缓存条数 |
| `allow_motion_when_disabled` | bool | `false` | `enable=false` 时是否拒绝 command |

---

## 4. 话题接口实现要求

### 4.1 订阅 `${topic_ns}/start`

- 消息类型：`std_msgs/String`，`data` 为 JSON。
- 必填字段：`type=manipulator_start`、`request_id`、`timestamp`（建议校验，缺失可 `rejected`）。
- 建议忽略未知字段（兼容 1.0/1.1）。

**语义：**

| 字段 | 行为 |
|------|------|
| `enable: true` | 置内部 `control_enabled=true`，允许执行 command |
| `enable: false` | 调用 `/openarm/stop`；`control_enabled=false`；`state=stopped` |
| `mode` | 当前仅记录（如 `manual`），可扩展 |

处理完成后更新 status（见 §5），`last_request_id` = 该 `request_id`。

### 4.2 订阅 `${topic_ns}/command`

- 消息类型：`std_msgs/String`，JSON。
- 必填：`type=manipulator_command`、`command`、`request_id`。

**命令字与内部映射：**

| `command` | 内部调用 | 备注 |
|-----------|----------|------|
| `pick_place` | Action `/openarm/pick_place` | 需 `params` 或默认策略（§6.2） |
| `pick` | 同上或拆分子流程 | **待产品/实现定稿**（§6.1） |
| `place` | 同上或拆分子流程 | **待产品/实现定稿**（§6.1） |
| `home` | Service `/openarm/goto_home` | 默认 `arm=both` 或可配置 |
| `stop` | Service `/openarm/stop` | 最高优先级；幂等 |
| `gripper` | Service `/openarm/gripper` | `params.action`: `open` \| `close` |
| `get_status` | 无运动 | 立即发布一帧完整 status |

**前置条件：**

- `control_enabled == false` 且非 `stop`/`get_status` → `ack=rejected`，`error_code=1004`（安全互锁）或项目约定码。
- 正在执行长动作且非 `stop` → `ack=rejected`，`error_code=1003`（机械臂忙）。

### 4.3 发布 `${topic_ns}/status`

- 消息类型：`std_msgs/String`，JSON。
- `type` 固定：`manipulator_status`。
- **周期发布** + **命令受理/结束瞬间各发一帧**。

**建议字段（与协议 §8 对齐）：**

```json
{
  "type": "manipulator_status",
  "state": "idle",
  "running": false,
  "ack": "done",
  "error_code": 0,
  "error_message": "",
  "last_request_id": "cmd-xxx",
  "request_id": "cmd-xxx",
  "board_id": "arm-controller-01",
  "timestamp": 1714471240000
}
```

| 字段 | 取值说明 |
|------|----------|
| `state` | `idle` \| `running` \| `paused` \| `stopped` \| `error` |
| `running` | 是否与 action 执行中一致 |
| `ack` | `accepted` \| `running` \| `done` \| `rejected` \| `timeout` |
| `timestamp` | **毫秒** Unix 时间戳（与上位机一致） |

---

## 5. 状态机与幂等

### 5.1 `request_id` 幂等

- 以 `request_id` 为键维护 LRU 缓存（建议 256 条）。
- 重复 `start`/`command`：不重复执行，直接发布缓存结果对应的 status（`ack=done`，相同 `last_request_id`）。
- `stop`：始终执行停止逻辑，但对外仍可幂等返回 success。

### 5.2 `ack` 流转（命令）

```
收到 command
    -> ack=accepted, state=idle|running, 发布 status
    -> 调用 skill / action
    -> ack=running, running=true, 发布 status
    -> [action feedback] 可选更新 state/running
    -> 成功: ack=done, running=false, state=idle
    -> 失败: ack=rejected 或 done+error, state=error
    -> 超时: ack=timeout, error_code=2001
```

长动作（`pick_place`）应订阅 Action **feedback**，将 `feedback.status` / `phase` 映射到 `running=true`（不必在 status 里暴露 phase，除非上位机扩展）。

### 5.3 `stop` 优先级

- 任意时刻可处理 `stop`。
- 取消当前 Action goal（若 rclpy 支持 cancel）。
- 调用 `/openarm/stop`。
- 尽快 `ack=done`，`state=stopped`，`running=false`。

---

## 6. 待产品确认项（实现前定稿）

### 6.1 `pick` 与 `place` 独立命令

当前 `openarm_skills` 仅提供完整 `PickPlace` action，不区分只抓/只放。

可选方案（择一写入配置或代码注释）：

| 方案 | 说明 |
|------|------|
| A | 暂不支持，`rejected` + `error_code=1002` |
| B | 与 `pick_place` 相同，忽略差异（不推荐，语义误导） |
| C | 扩展 `PickPlace` action 或 skill_server 支持 `mode=pick\|place` |

### 6.2 `pick_place` 且 `params: {}`

协议示例允许空对象。板卡需约定：

| 策略 | 行为 |
|------|------|
| 拒绝 | `error_code=1001`，提示缺少位姿 |
| 默认位姿 | 从 `manipulator_board.yaml` 读取 `grasp_pose` / `place_pose` |
| 相机模式 | 强制 `pose_source=camera`，需 `target_name` 等 |

### 6.3 错误码映射

协议公共码（§9）与 `openarm_skills` 内部码**数值不同**（例如双方均有 `1001` 但含义不同）。

**要求：** 在 `protocol_errors.py` 中实现映射表，status 只输出协议码；日志可同时打印 `skill_code`。

| 协议 code | 含义 | 映射来源示例 |
|-----------|------|----------------|
| 0 | 成功 | skill OK |
| 1001 | 参数错误 | JSON 校验失败 |
| 1002 | 命令不支持 | 未知 command |
| 1003 | 机械臂忙 | 正在执行且非 stop |
| 1004 | 安全互锁 | `enable=false` |
| 2001 | 执行超时 | action 超时 |
| 2002 | 硬件故障 | `EXECUTE_FAILED` 等 |
| 3001+ | 保留给上位机 | 板卡不使用 9001 通信异常 |

---

## 7. 与 `openarm_api` json_bridge 的关系

| 对比项 | `json_bridge_node`（已有） | `manipulator_board_node`（待开发） |
|--------|---------------------------|-----------------------------------|
| 入口 | `/openarm/command` 服务、可选 HTTP | `/robot_arm/start`、`/command` 话题 |
| 部署位置 | 上位机或同机 | **机械臂板卡** |
| 报文格式 | `cmd_id` / `cmd_type` | `request_id` / `type` / `command` |
| 出口 | 直连 `/openarm/*` | 同上 |
| 状态输出 | `/openarm/skill/event`（事件流） | `/robot_arm/status`（协议快照） |

**建议：** 将 `json_bridge_node` 中 `_do_pick_place`、`_do_stop` 等调用逻辑抽到 `skill_adapter` 共用模块，或板卡节点初期复制一版后重构，避免双份漂移。

---

## 8. Launch 与部署

### 8.1 板卡推荐启动顺序

```bash
# 1. 硬件 + MoveIt（按现场 launch 为准）
ros2 launch openarm_bimanual_moveit_config demo.launch.py

# 2. 技能层
ros2 launch openarm_skills skills.launch.py

# 3. （可选）感知
ros2 launch openarm_perception perception.launch.py

# 4. 协议适配层（待开发）
export MANIPULATOR_TOPIC_NS=/robot_arm
ros2 launch openarm_manipulator_board manipulator_board.launch.py
```

### 8.2 多机 ROS 环境

与上位机对齐：

- `ROS_DOMAIN_ID`
- DDS / 防火墙放行
- 主机名解析或 `ROS_LOCALHOST_ONLY=0`
- 时间同步（NTP），便于 `timestamp` 排障

### 8.3 不建议在板卡默认启动

- `openarm_api` 的 `enable_http:=true`（REST 在上位机）
- 与底盘相关的 launch

---

## 9. 联调验收标准

上位机已完成的前提下，板卡 ROS 侧通过以下项即视为交付：

### 9.1 话题可见性

```bash
export MANIPULATOR_TOPIC_NS=/robot_arm
ros2 topic list | grep robot_arm
# 应能看到 /robot_arm/status（板卡发布）
# 上位机发布 start/command 后，板卡节点日志有收到记录
```

### 9.2 启动闭环

1. 上位机或 `ros2 topic pub` 发送 `manipulator_start`（`enable: true`）。
2. `status` 中 `last_request_id` 与请求一致，`ack=done`。

### 9.3 命令闭环

1. 发送 `get_status`，收到完整 JSON。
2. 发送 `gripper`（`open`/`close`），`ack` 经历 `accepted` → `running` → `done`。
3. 发送 `stop`，机械臂停止，`state=stopped`。
4. 重复相同 `request_id`，不二次执行（日志可见 dedup）。

### 9.4 异常场景

| 场景 | 期望 |
|------|------|
| `enable=false` 后发 `home` | `rejected`，非 0 `error_code` |
| 执行中发第二条 `pick_place` | `1003` 或项目约定忙错误 |
| 非法 JSON | `rejected`，`1001` |
| 上位机停发 status 订阅方仅验证板卡仍周期发布 | 板卡持续发布 status |

### 9.5 日志规范

每条命令日志至少包含：`request_id`、`command`、`ack` 终态、`skill_code`（若有）、耗时 ms。

---

## 10. 开发任务拆分（建议排期）

### 阶段一（P0，可联调）✅ 已完成

- [x] 创建包 `openarm_manipulator_board`
- [x] 实现话题订阅/发布与 JSON 解析（`protocol.py`）
- [x] 实现 `manipulator_start`（enable 标志 + stop）
- [x] 实现 `get_status`、`stop`、`gripper`、`home`
- [x] 周期 status + `board_id`、毫秒 `timestamp`
- [x] `request_id` 幂等缓存（`idempotency_cache.py`）
- [x] Launch + YAML

### 阶段二（P0，完整动作）✅ 已完成

- [x] 实现 `pick_place` → `/openarm/pick_place`（含 feedback 更新 running/state）
- [x] 协议错误码映射（`protocol_errors.py` skill_to_protocol）
- [x] 执行超时 → `ack=timeout`（done_event.wait + EXEC_TIMEOUT）

### 阶段三（P1，产品依赖）🔶 部分完成

- [x] `pick`/`place` 策略：返回 `1002 CMD_NOT_SUPPORTED`（可按需改为方案 C）
- [x] 空 `params` 策略：`use_default_poses=false` 时返回 `1001`；设为 `true` 时从 YAML 读取默认点位
- [ ] 与 `openarm_api` 抽取共用 `skill_adapter`（当前两套独立，优先级 P2）
- [ ] 更新 `MANIPULATOR_PROTOCOL.md` 附录：板卡 launch 与错误码映射表

---

## 11. 参考文件

| 文件 | 说明 |
|------|------|
| [`MANIPULATOR_PROTOCOL.md`](./MANIPULATOR_PROTOCOL.md) | 跨板卡协议全文 |
| [`src/openarm_api/openarm_api/json_bridge_node.py`](./src/openarm_api/openarm_api/json_bridge_node.py) | 同机命令转发参考实现 |
| [`src/openarm_skills/README.md`](./src/openarm_skills/README.md) | 技能层 ROS 接口 |
| [`src/openarm_skills/include/openarm_skills/error_codes.hpp`](./src/openarm_skills/include/openarm_skills/error_codes.hpp) | 技能内部错误码（映射源） |
| [`src/openarm_skills/src/skill_server_node.cpp`](./src/openarm_skills/src/skill_server_node.cpp) | 固定位置 pick_place 执行逻辑 |
| [`src/openarm_api/openarm_api/schemas/pick_place.schema.json`](./src/openarm_api/openarm_api/schemas/pick_place.schema.json) | 固定位姿 params 校验 |

---

## 12. 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2026-05-20 | 初版：明确上位机已实现前提下 ROS 板卡待开发范围 |
| 1.1 | 2026-05-20 | 新增 §1.4：固定位置抓取与放置实现状态核对 |
| 1.2 | 2026-05-20 | 完成全部 P0 开发：`openarm_manipulator_board` 包已实现 |
