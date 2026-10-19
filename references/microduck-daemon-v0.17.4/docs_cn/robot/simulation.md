# 模拟鸭子

MuJoCo 里的一只鸭子，由真正的守护进程驱动。你对它开发，和对桌上机器人开发完全一样：同一个 `robotd`、同样的策略、同样的 50 Hz 循环、同一个 `robotctl`、同一个控制台。只有身体不同。

这一页讲怎么用它。[`design/simulation.md`](../design/simulation.md) 讲它是/不是什么的孪生体，以及为什么建成这样。

## 它是什么

`robotd --sim host:port` 用 `duck_control::sim::RemoteIo` 替换舵机总线来运行守护进程：每个节拍，关节位置、速度和 IMU 经 TCP 套接字从 MuJoCo 进程进来，策略的目标出去。那条接缝之上的一切——控制循环、策略、安全、跌倒检测、运动学、里程计、整个 IPC 面——都是机器人跑的代码，未改动、也无法分辨。`tofd --sim` 从同一个模拟器得到它的 8×8 深度帧；`mediad --sim-camera` 得到一张渲染的头部摄像头画面，像真的那样偏转四分之一圈安装。`configd` 和 `updaterd` 也原样运行，这正是让一只模拟鸭子拥有序列号、名字和一个 Hugging Face 账户的原因——你可以从这个网络之外通过它访问。

MuJoCo 那一半住在 [`microduck_rl`](https://github.com/pollen-robotics/microduck_rl) 里，作为 `duck-body`：一个进程、一个窗口、一个场景里的 N 个鸭子身体，带着策略训练时用的 BAM 执行器模型。

**它擅长什么：** 守护进程和它们客户端里的一切——IPC、`robotctl`、控制台、更新器、合唱、站立和行走的策略、映射。**它告诉不了你什么：** 驱动里的任何东西。Dynamixel 总线、BLE 无线电、摄像头 ISP、NPU 和硬件编码器都不在、没建模；那些里的一个 bug 只在机器人上可见。

## 你需要什么

- 本仓库，为你的机器构建好（`cargo build` 自己发生）。
- 一个带 venv 的 `microduck_rl` checkout——在本仓库旁边，或 `DUCK_SIM_RL` 指向的地方。它提供 `duck-body`、场景，以及笔记本电脑本来缺的 `libonnxruntime`。摄像头在它的 `develop` 分支上。
- 给一只有摄像头的鸭子（非 Linux）：`brew install gstreamer libnice-gstreamer`。第一个是带 `webrtcsink`、srtp 和 x264 的合并 formula。第二个是 `webrtcbin` 需要的 ICE 代理，而第一个不依赖它——没有它鸭子起来、却推到空处，只在浏览器要视频时才失败。`scripts/duck-sim` 会检查两者。
- 仅容器形式：`sudo`、`systemd-nspawn`（包 `systemd-container`）和 `mmdebstrap`。脚本说缺哪个并打印安装它的那行。

## `up` 还是 `boot`？

两者用同一个 `robotd` 驱动同一个 MuJoCo 身体。区别在守护进程运行在什么*之下*。

| | `scripts/duck-sim`（`up`） | `scripts/duck-sim boot N` |
|---|---|---|
| 守护进程作为 | 普通进程、你的用户、每个一个 pidfile | 每只鸭子一个 `systemd-nspawn` 容器里的 systemd 服务，来自真实 unit 文件 |
| 需要 | 除仓库和 `microduck_rl` 外没有别的 | `sudo`、一次性的 Debian rootfs 构建 |
| 启动 | 几秒 | 第一次一分钟，之后几秒 |
| 身份 | 你笔记本的；多只鸭子是一个进程树 | 每只鸭子一个 machine-id、一个声音、一个套接字，以及它们之间的 `duck-ether` |
| 锻炼 | 控制循环、策略、IPC、`robotctl`、控制台 | 那些全部，外加 `User=`/组/`RuntimeDirectory=`/加固、更新器的 apply、健康门控、回滚和重启顺序、`journalctl` |

经验法则：**`up`** 当你处理控制循环、策略、IPC 或客户端时。**`boot`** 当你处理任何触及 systemd、更新器或 provisioning 的东西，或不止一只鸭子互相说话时。这个模拟器抓到的 bug 里有两次是一个守护进程的用户不存在、所以它的 unit 从没启动——那是 `up` 完全看不见的一类。

## 一只鸭子，无容器

```sh
scripts/duck-sim                # 一个 MuJoCo 窗口打开，鸭子站起来，它是你的了
scripts/duck-sim status         # 健康，以及它是否站着
scripts/duck-sim drive          # 向前走 8 秒（参数：vx vyaw，默认 0.15 0）
scripts/duck-sim ctl health     # robotctl 做的任何事，指向这只鸭子
scripts/duck-sim monitor        # robotctl monitor：关节、IMU、ToF、摇杆
scripts/duck-sim log            # robotd 的日志
scripts/duck-sim simlog         # MuJoCo 那一边
scripts/duck-sim realtime       # 世界跑得多快（见下）
scripts/duck-sim down
```

不带参数时脚本从你所在的分支构建守护进程、写一个点名本仓库策略的参数文件、在 `~/.cache/duck-sim` 下启动 `duck-body`、`tofd --sim` 和 `robotd --sim`，并启用站立策略。`ctl` 是指向那只鸭子套接字的 `robotctl`，那是与机器人唯一不同之处：在板上套接字在 `/run` 下，这里在状态目录下。

要用你自己的工具跟它说话，套接字是 `~/.cache/duck-sim/duck-a.sock`（robotd；`duck.sock` 是指向 `ctl` 说话的那只鸭子的链接）和 `~/.cache/duck-sim/duck-a-tof.sock`，身体在 TCP 端口 7801。

## 多只鸭子，每只是一台你登录的机器

```sh
scripts/duck-sim boot 4         # 一个世界里四只鸭子，每只在自己的容器里（sudo）
scripts/duck-sim shell          # 你在 duck-a 上
scripts/duck-sim shell duck-c   # 或任何其他
duck-a # robotctl health
duck-a # journalctl -u robotd -f
scripts/duck-sim down
```

`boot` 在一个 `systemd-nspawn` 容器里、在真实 systemd 下运行每只鸭子的守护进程，从一个 Debian 13 rootfs（构建一次、约三分钟、保存在状态目录下）启动，每只鸭子一个 overlay。那买到普通形式伪造不了的部分：带 `User=`、组、`RuntimeDirectory=` 和加固的真实 unit 文件，在一个真实 init 之下——所以 `robotctl update apply`、健康门控和重启顺序像在机器人上那样表现。鸭子是 `duck-a`、`duck-b`、……，每只有自己的 machine-id 和自己的声音。

不止一只鸭子时脚本还启动 `duck-ether`，一个在容器之间携带合唱 BLE 信标的假无线电。它刻意是一台*坏*无线电——丢包和延迟的信标——因为一台完美的会藏起真无线电暴露的 bug。

第一次，`boot` 构建 rootfs，可能请你安装 `mmdebstrap`。每只鸭子都是一个 systemd unit，所以 `down` 是一个 `systemctl stop`，从来不是组合键。

## 一个可看的房间，和看的眼睛

```sh
DUCK_SIM_SCENE=apartment DUCK_SIM_CAMERAS=a scripts/duck-sim boot 2
```

默认世界是一块光地板。`apartment` 是 7×6 米里的六个房间，门口刻意偏置，好让一个姿态能从鸭子 45° 的前向视野里认出来；任何带斜杠的东西是你自己场景的路径，而 `microduck_rl` 的 `scene_*.xml` 文件是内置的那些。

摄像头按鸭子选择加入（`a`、`a,c` 或 `all`），因为一帧渲染花 12 ms，而步进四只鸭子的物理只要 0.3 ms：一个摄像头是核心的三分之一，四个是差不多一整个。每只有摄像头的鸭子得到自己的 `mediad`，它的控制台按索引服务在 `http://127.0.0.1:8080`、`8081`、……，正是机器人服务的那个页面。

## 从任何地方访问它

一只模拟鸭子登录一个 Hugging Face 账户并出现在该账户的机器人列表里，和桌上机器人一样。控制台、一个 App 或一个 Space 然后通过 WebRTC 到达它，无需在这个网络上。

```bash
DUCK_SIM_CAMERAS=a scripts/duck-sim
```

```bash
scripts/duck-sim ctl account login
```

打开 `hf.co/oauth/device`，输入它打印的码，鸭子几秒内就被列出。`scripts/duck-sim ctl account status` 说它属于哪个账户，`scripts/duck-sim ctl system info` 打印序列号和 `body MuJoCo` 那行。

在列表里它带一个 `simulated` 标志和一个由序列号派生的名字（`duck-eb55`），所以它不会读作架子上的机器人。除此之外它和任何鸭子一样：同样的控制通道、同样的 H.264、同样的策略。

从 rendezvous 按 token 键定对端，得到两条规则：

- **一个账户一次一只鸭子。** 同一账户上的第二次 `login` 取代第一次，而两者看起来都没坏——它们轮流被列出。第二只鸭子要第二个账户。
- **消费这只鸭子的 Space 需要它自己的 token**，不是鸭子的；一个已发布 Space 用访客自己的登录。

`scripts/duck-sim ctl account logout` 把它从列表上取下。

## 旋钮

环境变量，全部可选：

| 变量 | 默认 | 作用 |
|---|---|---|
| `DUCK_SIM_RL` | `~/Pollen/microduck_rl` | `duck-body`、场景和 ONNX runtime 在哪。 |
| `DUCK_SIM_STATE` | `~/.cache/duck-sim` | 套接字、日志、参数、rootfs 和鸭子的 overlay。刻意短：unix 套接字路径上限约 108 字节。 |
| `DUCK_SIM_DUCKS` | `1` | 多少只鸭子；`boot N` 也设它。 |
| `DUCK_SIM_SCENE` | 光地板 | 一个场景名（`apartment`）或一个路径。 |
| `DUCK_SIM_CAMERAS` | 无 | 哪些鸭子渲染摄像头：`a`、`a,c`、`all`。 |
| `DUCK_SIM_DUCK` | `duck-a` | `ctl` 和 `monitor` 跟哪只鸭子说话。 |
| `DUCK_SIM_KEYFRAME` | `SIT` | 一只鸭子从哪开始：`SIT` 折在地板上（站立策略从中起身）、`HOME`、`STAND`、`FOLD`。 |
| `DUCK_SIM_VIEWER` | `1` | `0` 让 MuJoCo 无头运行。 |
| `DUCK_SIM_PORT` | `7801` | 第一只鸭子的身体端口；每只 +1。 |
| `DUCK_SIM_FRAME_PORT` | `7901` | 第一个摄像头的帧端口；每个 +1。 |

## 值得知道的事

**实时性要紧。** 守护进程的循环是墙上时钟。一个跑在 1.0× 实时之下的模拟器不只是看起来慢：策略在驱动一台动得比它们预期少的机器人，它们平衡不了它。`scripts/duck-sim realtime` 报告倍数，`boot` 打印它。鸭子太多、或摄像头太多，鸭子不会变慢——它们在 45 Hz 门控处变*不健康*，而在容器形式里更新器开始把发布滚回去。更少的鸭子、更少的摄像头、或一个无头 viewer 是修法，按那个顺序。

**鸭子不热加入。** MuJoCo 编译它的模型，所以改鸭子数量会重启模拟器。守护进程挺得过：`RemoteIo` 在下一个节拍重连，而一只身体短暂消失的鸭子报告不健康而不是死掉，和总线上没电的机器人一样。

**`--sim` 不是 `--fake`。** `robotd --fake` 是一只空无一物的机器人：没有物理，位置完美回显，什么都倒不了。它给单元测试和完全不需要身体的笔记本工作。`--sim` 是孪生体。两个标志互斥。

**一只鸭子不该以为自己是台笔记本。** 鸭子的声音和它的合唱身份由硬件序列号派生，而一台机器上的多只鸭子会共享它。脚本给每只鸭子设 `DUCK_IDENTITY` 让它们不同；机器人上没有任何东西设它。

## 手工，不用脚本

两半各取一个 `host:port` 并可单独运行，出问题时很有用：

```sh
# 身体，来自 RL 仓库的 venv
duck-body --ducks 1 --port 7801 --keyframe SIT
# 守护进程，来自本仓库
target/debug/tofd --sim 127.0.0.1:7801 --socket /tmp/d/duck-a-tof.sock
DUCK_RUNTIME_DIR=/tmp/d ORT_DYLIB_PATH=<libonnxruntime.so> \
    target/debug/robotd --sim 127.0.0.1:7801 --params <params.toml> --socket /tmp/d/duck-a.sock
# 它是谁，属于哪个账户
target/debug/configd --socket /tmp/d/duck-a-config.sock --state-dir /tmp/d/duck-a \
    --simulated sim-duck-a --fake-net --fake-pads
target/debug/updaterd --config /tmp/d/updater.toml --socket /tmp/d/duck-a-updater.sock \
    --token /tmp/d/duck-a/hf-token
# 一个摄像头，如果 duck-body 以 --cameras a 启动
printf '[media]\nquality = "360p30"\n' > /tmp/d/mediad.toml
target/debug/mediad --sim-camera 127.0.0.1:7901 --config /tmp/d/mediad.toml \
    --robot-socket /tmp/d/duck-a.sock --tof-socket /tmp/d/duck-a-tof.sock \
    --config-socket /tmp/d/duck-a-config.sock --updater-socket /tmp/d/duck-a-updater.sock \
    --token /tmp/d/duck-a/hf-token
```

非 Linux 上，`mediad` 要带它的管线构建：`cargo build -p mediad --features gstreamer`。没有那个 feature 它启动、说明、然后退出。

`--token` 在两个守护进程上点同一个文件——`updaterd` 把凭据写到那里、`mediad` 读它。把它们指向不同文件，鸭子登录了却从不注册。

摄像头的几何在两边必须匹配——`mediad` 推 `[media] quality` 档（`360p30` 是 640×360），身体必须以同样尺寸渲染，因为帧是原始到达、没有握手，而 `mediad` 会拒绝一个尺寸错误的帧，而不是显示一张没人能读的图。`scripts/duck-sim` 是其余参数的记录；临场发挥前先读它。
