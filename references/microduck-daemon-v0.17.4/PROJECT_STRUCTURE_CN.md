# 项目结构速览：每个文件夹对应机器人的哪个功能模块

> 本文档整理自 `Cargo.toml`、各 crate 的 `lib.rs` 头部注释与目录结构，用于快速查询「某个文件夹是干什么的、对应机器人的哪部分」。
>
> 这是一个 Rust workspace（`Cargo.toml` 明确写着「一个 crate = 一个服务/工具」）。整只鸭子机器人由 8 个守护进程组成，每个守护进程基本对应一个 crate。

## 一、核心控制（大脑 + 身体）

| 目录 | 作用 | 对应机器人模块 |
|---|---|---|
| `robotd/` | 机器人控制守护进程。50 Hz 主循环，运行 ONNX 策略（走路/站立/坐下等步态）、安全钳制、跌倒反射、intent deadman | 主脑 |
| `robotd-params/` | `robotd` 的启动参数 schema、默认值和校验 | 主脑的配置层 |
| `duck-control/` | 控制核心库：模型、舵机总线（Dynamixel）、传感器读取，`RobotIo` 抽象层 | 身体模型 + 驱动 |
| `kinematics/` | 正向运动学（由 MJCF 编译），把关节角算成足部/躯干位置 | 身体姿态计算 |
| `odometry/` | 里程计，用腿的接触点 + IMU 估算机器人在世界中的位置 | 定位 |
| `pad-imu/` | 手柄内置 IMU → 姿态四元数，用于「晃手柄控制头部」 | 手柄姿态输入 |

## 二、感知（眼睛 + 耳朵 + 触觉）

| 目录 | 作用 | 对应机器人模块 |
|---|---|---|
| `mediad/` | 摄像头、麦克风、WebRTC 流媒体 + 远程网关（`webrtcsink`） | 视觉/听觉 + 远程访问 |
| `uyvy/` | 摄像头像素处理：UYVY → RGB 采样、画面旋转 | 视觉底层 |
| `tof/` | 头部 ToF 深度传感器（VL53L5CX/L8CX，8×8 距离矩阵）驱动 | 深度感知 |
| `duck-detect/` | 在 NPU 上运行鸭子检测模型（`.rknn`，INT8），在画面里找别的鸭子 | 视觉识别 |
| `pet-detect/` | 抚摸检测：板载麦克风的微型音频分类器，识别「摸头」 | 触觉/听觉识别 |
| `sounds/` | 机器人发声合成器，由序列号种子生成每只鸭子独特嗓音 | 嗓音 |

## 三、手柄与输入

| 目录 | 作用 | 对应机器人模块 |
|---|---|---|
| `padd/` | 手柄守护进程：物理手柄 → 机器人 intent | 手柄输入 |
| `netpadd/` | 通过 UDP 从局域网程序接收手柄输入（虚拟手柄） | 网络手柄输入 |
| `pad-map/` | 手柄按键/摇杆到 intent 的映射，`padd` 和 `netpadd` 共享 | 手柄映射 |
| `nfc/` | NFC 读卡器（CLRC663），读 NTAG215 标签 | NFC 输入 |

## 四、通信与身份

| 目录 | 作用 | 对应机器人模块 |
|---|---|---|
| `btd/` | BLE 传输适配器：手机通过 GATT 访问机器人 JSON-RPC API 的「前门」 | 蓝牙通信 |
| `duck-ble/` | BLE 线协议契约：GATT UUID、NDJSON 分块、广播 payload | 蓝牙协议层 |
| `duck-ipc-proto/` | 各服务与客户端之间的 IPC 契约（JSON-RPC 方法定义） | 内部通信协议 |
| `configd/` | WiFi 配置 + 机器人身份（名字、序列号、配对 PIN） | 网络 + 身份 |
| `duck-ether/` | 仿真鸭子用的「无线电」，模拟广播/邻近感知 | 仿真通信 |

## 五、运维与工具

| 目录 | 作用 | 对应机器人模块 |
|---|---|---|
| `updater/` | 更新引擎：签名校验、健康门控、安全回滚 | 系统更新 |
| `robotctl/` | 机器上的本地 CLI（`update`、`policy`、`configure` 等命令） | 本地管理工具 |
| `duckctl/` | 从笔记本操控机器人的 CLI（`scan`/`ip`/`open`/`wifi` 等） | 远程客户端 |
| `xtask/` | 构建工具：打包、签名、发布 release | 构建/发布 |
| `test-support/` | 测试用的签名 release 夹具，不发布 | 测试支撑 |

## 六、支持目录（非 crate）

| 目录 | 作用 |
|---|---|
| `scripts/` | 板子配置与运维脚本：`install.sh`（裸板安装）、`setup-board.sh`、`seed-policies.sh`、`migrate-network.sh`、`dev-push.sh`（开发推送）、`duck-sim`（仿真启动）等 |
| `deploy/` | 部署配置：systemd unit、`robotd.toml`、`updater.toml`、可信签名密钥、音频/overlay |
| `hooks/` | 更新钩子 `postinstall`、`preinstall`（在更新中安装 unit、ONNX Runtime 等） |
| `spaces/` | Hugging Face Space 相关内容（远程访问的 rendezvous/演示） |
| `docs/` | 设计文档：`design`（设计）、`project`（项目记录）、`robot`（使用手册）、`ideas`、`superpowers` |
| `docs_cn/` | 设计文档中文翻译（进行中） |

## 七、快速对照（一句话版）

- `robotd` 是主脑，`duck-control` / `kinematics` / `odometry` 是它的身体模型。
- `mediad` + `uyvy` / `tof` / `duck-detect` / `pet-detect` / `sounds` 是眼睛、耳朵、触觉和嗓音。
- `padd` / `netpadd` / `pad-map` / `nfc` / `pad-imu` 是输入设备。
- `btd` / `duck-ble` / `duck-ipc-proto` / `configd` 是通信与身份。
- `updater` / `robotctl` / `duckctl` / `xtask` 是运维与工具。

## 八、8 个守护进程总览

| 守护进程 | 对应 crate | 职责 |
|---|---|---|
| `robotd` | `robotd/` | 机器人控制主脑 |
| `configd` | `configd/` | WiFi 与机器人身份 |
| `btd` | `btd/` | BLE 传输前门 |
| `padd` | `padd/` | 手柄输入 |
| `netpadd` | `netpadd/` | 网络手柄输入 |
| `mediad` | `mediad/` | 摄像头/麦克风/WebRTC 远程网关 |
| `tofd` | `tof/` | ToF 深度传感器 |
| `updaterd` | `updater/` | 系统更新引擎 |
