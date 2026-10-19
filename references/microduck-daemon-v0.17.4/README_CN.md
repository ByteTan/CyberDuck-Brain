<p align="center">
  <img src="https://github.com/user-attachments/assets/c2f7c245-8217-46a1-8d1e-e0ba967cd969" alt="microduck" width="820">
</p>

<h1 align="center">Microduck</h1>

<p align="center">
  <em>一只通过强化学习策略移动的小型双足机器人。</em>
</p>

<p align="center">
  <a href="https://pollen-robotics.com/microduck"><b>在这里购买</b></a> ·
  <a href="docs/robot/cheatsheet.md">速查表</a> ·
  <a href="https://github.com/pollen-robotics/microduck_rl">策略训练</a> ·
  <a href="docs/design/architecture.md">工作原理</a> ·
  <a href="CONTRIBUTING.md">贡献指南</a>
</p>

<p align="center">
  <a href="https://github.com/pollen-robotics/microduck/actions/workflows/ci.yml"><img src="https://github.com/pollen-robotics/microduck/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
</p>

---

**这个仓库是鸭子的「大脑」。** 一只约 25 厘米、800 克重的机器人，由运行在 Rockchip RK3566 上的一组守护进程驱动：一个 50 Hz 的控制循环通过神经网络策略驱动十五个舵机，此外还有无线通信、摄像头，以及一套把新软件安全送到机器人上、避免变砖的更新机制。

运行一只 Microduck 所需的一切都在这里。**如果你想要一只，
[在这里购买](https://pollen-robotics.com/microduck)。**

它运行的策略是在隔壁仓库训练的：
**[microduck_rl](https://github.com/pollen-robotics/microduck_rl)**——包含 MuJoCo 与 PPO、sim2real 配方，以及本仓库所加载的 ONNX 导出。

## 它能做什么

<table>
<tr>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/356a6011-8e0d-4b28-bda9-da78646583a3" controls width="100%"></video>
</td>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/abfbf250-1b1c-42cb-8430-00267e2b148a" controls width="100%"></video>

</td>
</tr>
<tr>
<td><b>它会走路。</b> 拿起游戏手柄就能驾驶。</td>
<td><b>它会滚动。</b> 装上轮子、请求滚轮模式，它就会加载另一个「大脑」。</td>
</tr>
<tr>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/7e70c1da-e120-428f-ae0b-f4de62f25984" controls width="100%"></video>
</td>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/3eef63a5-6f84-47cf-90de-e717e6d7f8f0" controls width="100%"></video>
</td>
</tr>
<tr>
<td><b>它会捡东西。</b> 喙贴地面，一键完成。</td>
<td><b>它会自己站起来。</b> 把它推倒，它会自己重新站立。</td>
</tr>
</table>

它还会坐下、踢球、听指令向前翻滚，并用它独有的声音嘎嘎叫。

## 到哪里找东西

### 你有一只鸭子

| | |
|---|---|
| [速查表](docs/robot/cheatsheet.md) | 每一条 `robotctl` 命令：驾驶、配置、语音、合唱、特雷门琴、WiFi、更新、日志。从这里开始。 |
| [游戏手柄](docs/robot/cheatsheet.md#gamepad-configd) | 完整的按键映射，以及配对手柄——[每只手柄一次](docs/robot/pair-a-gamepad.md)，还有它无法配对时该怎么做。 |
| [`duckctl`](docs/robot/duckctl.md) | 从笔记本电脑通过蓝牙控制机器人，无需网络、无需 ssh。 |
| [更新](docs/robot/cheatsheet.md#updates-updaterd) | 安装、回滚、固定版本。每一次更新都经过校验、健康门控，且可逆。 |

### 你在它之上开发

| | |
|---|---|
| [microduck_rl](https://github.com/pollen-robotics/microduck_rl) | 策略的来源：MuJoCo、PPO、域随机化，以及本仓库所加载的 ONNX 导出。 |
| [工作原理](docs/design/architecture.md) | 一页看懂整个系统——各守护进程、总线、一次更新如何到达机器人——之后每个部分单独一页。 |
| [搭建开发板](docs/robot/install-dev.md) | 从一块空白开发板，到一只能接受分支构建的机器人。 |
| [开发者速查表](docs/robot/cheatsheet-dev.md) | 分支构建、候选发布、从笔记本电脑驾驶，以及更新后的重启陷阱。 |
| [推送你的分支](docs/robot/dev-push.md) | 在你的机器上构建，通过 ssh 安装，大约一分钟。 |
| [模拟鸭子](docs/robot/simulation.md) | 桌上没有机器人？`scripts/duck-sim` 会在 MuJoCo 里让真正的守护进程驱动一个身体——窗口里一只鸭子，或者四只，当作你可以登录的机器。 |
| [CONTRIBUTING.md](CONTRIBUTING.md) | 构建、测试、目录布局、约定、发布。 |
| [文档索引](docs/README.md) | 全部内容，包括设计文档和未决问题。 |

## 底层原理

Rust、无框架、单一 workspace。`robotd` 负责控制循环和电机总线；`updaterd` 安装经过签名的发布，并在机器人启动后不健康时回滚；`configd` 负责 WiFi 和身份；`btd` 是手机所使用的蓝牙通道；`padd` 读取游戏手柄；`mediad` 通过 WebRTC 推送摄像头画面；`tofd` 提供深度传感器服务。它们通过 Unix 套接字上的同一套 JSON-RPC 契约通信，而每个客户端——App、控制台、游戏手柄、你的脚本——发出的都是完全相同的调用。

那些有意思的设计决策都记录在案：[`docs/design/`](docs/design/) 解释事情为什么是这样，而 [`docs/project/`](docs/project/) 记录了曾经出过什么问题、以及什么方案能解决它。

## 关于鸭子的一句话

制造这台机器人的过程中，没有鸭子受到伤害。有几位鸭子接受了咨询。
