# 文档

[README](../README.md) 是前门——microduck 是什么、该往哪去。如果你面前就有一只、想驾驶它，从[速查表](robot/cheatsheet.md)开始。

[`faq.md`](faq.md) 是另一扇前门：来自「对着」鸭子构建而非修改它的人提出的、以任务为形的问题——跑一个对开发板来说太重的模型、把摄像头接进自己的程序、为什么 Space 连不上。

它也是**发布者**的起点：[`policy-manifest.md`](policy-manifest.md) 是 microduck `.onnx` 旁边的 `manifest.json` 的契约，并且它拥有每一个字段。设计文档给出背后的推理并指向它。

## `robot/` —— 你有一只机器人

| | |
|---|---|
| [`cheatsheet.md`](robot/cheatsheet.md) | 每一条 `robotctl` 命令。 |
| [`pair-a-gamepad.md`](robot/pair-a-gamepad.md) | 每只手柄一次：配对模式、`pad pair`，以及它无法配对时该怎么做。 |
| [`udp-pad.md`](robot/udp-pad.md) | 通过 UDP 在局域网上的手柄：如何开启、`duckctl udp-pad`，以及编写你自己的发送端所用的线格式。 |
| [`cheatsheet-dev.md`](robot/cheatsheet-dev.md) | 需要开发板才能用的命令：分支构建、候选发布、开发推送。 |
| [`dev-push.md`](robot/dev-push.md) | 在你的机器上构建并通过 ssh 安装到开发板，无需 CI 运行。 |
| [`simulation.md`](robot/simulation.md) | 模拟鸭子：`scripts/duck-sim`、真正的守护进程驱动 MuJoCo 身体、容器里的一只或几只鸭子。 |
| [`duckctl.md`](robot/duckctl.md) | 每一条 `duckctl` 命令——从笔记本电脑通过蓝牙控制机器人。 |
| [`install-dev.md`](robot/install-dev.md) | 从零搭建一块开发板。 |
| [`install-by-hand.md`](robot/install-by-hand.md) | 同样的安装拆成单独的命令，供一步步测试。 |

## `design/` —— 你在修改守护进程

它如何运转、以及为什么。这些内容很少变化；当行为与设计文档不一致时，错的是文档。

**一个页面只拥有一个机制，其他页面链接到它。** 下表就是这份归属：如果某个事实属于这里列出的某一页，其他每一页就只说一句话并指向它，而不是再解释一遍。一个事实被写在六个地方，就会朝六个方向漂移，每一处都各自显得合理——这正是六份文档曾经信誓旦旦地说 `updaterd` 和 `btd` 会保留旧二进制直到下次重启的原因，而它们在两个发布版本之前就已经不再这么做了，其中就包括人们在诊断这个问题时读的那两页。所以当两份文档不一致时，没有拥有该机制的那一份就是 bug。

| | |
|---|---|
| [`architecture.md`](design/architecture.md) | 服务拆分、IPC 契约、状态归属、安全与权限。 |
| [`robotd-design.md`](design/robotd-design.md) | 控制循环：Dynamixel 总线与谁拥有串口、模型、感知、观测、策略、安全——以及还有什么挂在这个节拍上。 |
| [`updater-design.md`](design/updater-design.md) | 更新引擎：校验、原子切换、健康门控、回滚、发布格式。 |
| [`policy-channel-design.md`](design/policy-channel-design.md) | ONNX 策略从哪来：`policies` 组件、试用别人的策略、以及 `reset` 会放回什么。 |
| [`restart-order.md`](design/restart-order.md) | 在移动 `current` 的每一条路径上——以及启动时——哪个单元在哪一步重启。 |
| [`app-path-design.md`](design/app-path-design.md) | `btd` 与 `configd`——手机如何通过 BLE 配置机器人。 |
| [`mobile-app.md`](design/mobile-app.md) | 手机 App——它由什么构建、机器人还欠它什么。代码在 [`microduck-app`](https://github.com/pollen-robotics/microduck-app)。 |
| [`remote-webrtc.md`](design/remote-webrtc.md) | WebRTC 会话、信令与控制通道——对端如何驾驶和观察机器人。 |
| [`webrtc-console.md`](design/webrtc-console.md) | WebRTC 客户端：从机器人提供它、找到机器人、以及这个页面应当是什么。 |
| [`remote-access-design.md`](design/remote-access-design.md) | 从局域网之外访问鸭子：Hugging Face 账户、设备流，以及通往 rendezvous 服务的桥。 |
| [`boot-recovery-net.md`](design/boot-recovery-net.md) | 当启动的发布无法启动它的守护进程时，回退到 golden。 |
| [`simulation.md`](design/simulation.md) | 孪生体：守护进程与身体之间的接缝在哪、身体协议、假无线电、容器，以及它是/不是什么的孪生体。 |

## `project/` —— 你在运营这个项目

带日期的记录而非参考资料。它们描述某一个时刻，并刻意过时。

| | |
|---|---|
| [`roadmap.md`](project/roadmap.md) | 里程碑，以及今天什么能用、什么还只是设计。 |
| [`ci-setup.md`](project/ci-setup.md) | 发布流水线的一次性设置：密钥、机密、轮换。 |
| [`install-path-gap.md`](project/install-path-gap.md) | 为什么四个安装路径 bug 一路到达了开发板，以及是什么堵住了它。已关闭——它带来的规则在 [`updater-design.md`](design/updater-design.md) §9.1。 |
| [`slice-2-bringup.md`](project/slice-2-bringup.md) | 一块真实的 Radxa Zero 3W 对 slice 2 做了什么。 |
| [`update-over-ble.md`](project/update-over-ble.md) | 从手机驱动更新路径：它暴露了什么，以及最终决定无线回滚如何做。 |
| [`media-bringup.md`](project/media-bringup.md) | Radxa Zero 3W 对视频做了什么：VPU、MPP 需要什么、以及必须构建的两个插件。 |
| [`pad-minimal-pairing.md`](project/pad-minimal-pairing.md) | 游戏手柄能配对的最小开发板配置，通过一次去掉一项找到。 |
| [`idle-cpu.md`](project/idle-cpu.md) | 没有人使唤守护进程时它们在做什么：四件被停掉的事、两件被测量后原样保留的事，以及还有什么需要一块开发板。 |
| [`tof-on-demand.md`](project/tof-on-demand.md) | `tofd` 空闲的 5% 里有九成是头部 IMU、一成是深度，而 IMU 没有消费者。为什么激光和该单元被保留，而 IMU 加了个开关。 |

## `ideas/` —— 尚未设计

暂存处。某个将来需要设计文档的东西，在它还没有设计文档之前先写下来，好让思路不丢失，也不会被误当成已经做出的决定。

| | |
|---|---|
| [`autonomous_behavior.md`](ideas/autonomous_behavior.md) | 行为栈：运行时的「大脑」必须交出什么，以及合唱与特雷门琴工作留下的想法。 |

## 其他

| | |
|---|---|
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | 构建、测试、仓库布局、约定、发布。 |
| [`project/npu-bringup.md`](project/npu-bringup.md) | RK3566 NPU 上的鸭子检测器：运行什么、如何基准测试，以及仍缺失的帧路径。 |
| [`../deploy/README.md`](../deploy/README.md) | 机器人镜像被配置了什么，以及 provisioning 实际做了什么。 |
