# 仿真：同样的守护进程，MuJoCo 中的机身

**状态：** 已构建并在使用中。`robotd --sim`、`tofd --sim` 和 `mediad --sim-camera` 已就绪，
`microduck_rl` 的 `duck-body` 提供另一半；容器（§8）、以太（§5）和每只鸭子的摄像头
都通过 `scripts/duck-sim` 运行。实测：守护进程以 `50.0 of 50.0 Hz · 0 missed` 的频率
对应 MuJoCo 机身运行，容器中的四只鸭子通过以太唱出一整段合唱。如何使用见
[`docs/robot/simulation.md`](../robot/simulation.md)；本页讲的是设计。

目标是让你开发一只鸭子时，就像开发真实机器人一样：相同的二进制、相同的 unit、
相同的 `robotctl`、相同的 `duckctl open`——只是机身位于 MuJoCo 中，而不是在桌面上。
不是 mock，也不是测试框架。而是一个孪生体，有着明确的边界。

## 1. 使用它的样子

一条命令就能得到一只鸭子。之后它就是一个机器人，你已经熟悉那些命令了。

**当前可用**（`scripts/duck-sim`，一只鸭子，无容器）：

```
scripts/duck-sim               # 打开一个窗口，鸭子站起来，它就是你的了
scripts/duck-sim drive         # 让它前进，然后停下
scripts/duck-sim ctl health    # 或 robotctl 能做的任何事
scripts/duck-sim log
scripts/duck-sim down
```

它会构建缺失的部分、写入参数、在 RL 仓库的 venv 中找到 ONNX Runtime、启动两半并让鸭子
站起来。它存在的意义就是阻止任何人输入三样无法猜测的东西：策略在机器人上位于
`/opt/robot/daemon/current`，在笔记本上则位于本仓库；`ort` 会 dlopen 一个笔记本上没有的
libonnxruntime；而 unix socket 路径上限约为 108 字节，因此状态目录必须短。

**它的去向**，一旦容器就绪：

```
duck-sim up 4                  # 四只鸭子，共享一个 MuJoCo 窗口
duck-sim shell duck-a          # 你进入 duck-a
```

进去之后，没什么特别之处：

```
duck-a # robotctl health
duck-a # robotctl configure
duck-a # robotctl chorale
duck-a # journalctl -u robotd -f
```

而在你自己的 shell 里，和桌上放着的机器人完全一样：

```
duckctl open duck-a
scripts/dev-push.sh microduck@duck-a
```

这个 harness 给自己定下的四条规则，因为一个需要 runbook 的开发工具没人会用：

**没有需要任何人记住的初始化步骤。** 在一台从未运行过的机器上执行 `duck-sim up`，
它会构建 rootfs、获取缺失的部分并说明——只做一次。凡是它确实无法替你完成的，
它会打印成可直接粘贴的确切命令行，风格与 `scripts/` 其余部分保持一致。

**默认值即常见情形。** 不指定数量就是一只鸭子。不指定场景就是公寓。不指定
`--cameras` 就是没有摄像头，因为大多数会话并不需要它们，而它们才是成本所在。

**没有什么是停不下来的。** 每只鸭子都作为 systemd unit 运行，所以 `duck-sim down`
就是一条 `systemctl stop`——而不是某个按键组合。这条规则是用一个必须在第二个终端
kill 掉的容器换来的，因为 `Ctrl-]` 在法语键盘上是 `AltGr + )`。

**鸭子可以按名字寻址。** `duck-a`、`duck-b`，每只都有自己的 machine-id——所以
`robotctl quack` 在每只上声音都不一样，因为鸭子的声音由其序列号生成。
合唱里的四只鸭子不应该是同一副嗓子的四个副本。

## 2. 接缝在哪里

`duck_control::io::RobotIo`——六个方法，也是唯一允许模拟器存在的地方：

```rust
fn read(&mut self) -> Result<Sensors>;          // 关节和 IMU，一次事务
fn write(&mut self, targets: &JointTargets) -> Result<()>;
fn set_gain(&mut self, kp: u16) -> Result<()>;
fn set_torque(&mut self, on: bool) -> Result<()>;
fn slow_sensors(&mut self) -> Result<SlowSensors>;   // 电压、逐关节温度
```

在其之上，一切不变：50 Hz 循环、ONNX 策略、`Safety`、跌倒检测、里程计、运动学、maploc、
每一次 IPC 调用、整个 `robotctl` 和 `duckctl`。在其之下只有一个东西——`DynamixelIo`——
IMU 并不与它分离，因为在这台机器人上 IMU 是一个 Dynamixel 节点，与十五个舵机在同一次
`sync_read` 中读取。

`FakeIo` 已经是这个 trait 的完整实现，这就是 `cargo test` 不需要硬件的原因。
`RemoteIo` 是第三个。

**当一个传感器的守护进程本身就是它的驱动时，两者都不替换。** `tofd --fake` 已经在
循环层合成了帧，而 `tof/src/sensor.rs` 说得明明白白：板外的 `Sensor`“不是假传感器，
也绝不能变成假传感器”。仿真深度就喂给那个已有的循环。同样的推理适用于任何驱动
无法与其硬件分离的东西。

## 3. 机身协议

TCP，换行分隔的 JSON，每次调用一个请求、一个应答。`duck_control::sim` 是它的实现，
也承载着推理；简而言之：

* **TCP**，因为 unix 路径受限于 `SUN_LEN`（约 108 字节），也因为模拟器必须能从守护进程
  所在的任何环境之外被访问——Linux 上是容器，macOS 上则是主机上运行 MuJoCo 的 Linux VM。
* **JSON**，因为一个 tick 约 1 KB，即 50 KB/s，代价换来了能用 `nc` 读取一帧、能用二十行
  Python 写另一端的能力。在两个仓库、两种语言之间共享一个打包 struct，正是这个项目
  以前浪费过好几天的地方。
* **`TCP_NODELAY`**，不是微优化：Nagle 会把一次小写入延迟最多约 40 ms，是 tick 的两倍，
  而它会表现为一个慢速模拟器。

请求形如 `{"op":"hello"|"read"|"write"|"gain"|"torque"|"slow", …}`。握手携带 `protocol`
（当前为 1）并在两个方向都做校验，因为两半分属两个仓库，“你的模拟器太旧”和“你的
守护进程太旧”否则会是同一个症状。

**模拟器以机器人自己的单位报告**——弧度、rad/s、mA，IMU 已经解析到躯干坐标系。MuJoCo
知道它自己模型的顺序、缩放和坐标系；在这一侧做转换层会是这些信息的第二个驻留地，
并随之漂移。

**一个死掉的模拟器就是一个坏 tick。** MuJoCo 会编译它的模型，所以改变鸭子数量会重启它，
鸭子必须熬过这个过程。断开的连接会作为错误返回给调用者，并在下一次调用时重连——没有
退避线程，因为控制循环本身就是重试定时器。

## 4. 伪造无线电零成本

在场本身就是 `robotd` 自己 socket 上的一个 IPC 契约：`chorale.subscribe`、
`chorale.beacon`（要广播什么）以及 `chorale.heard`（听到了什么，带一个 *age* 而非时间戳）。
而 `btd` 是 robotd 的**客户端**，不是服务器——所以仿真以太不需要冒充任何东西，也不
抢占任何 socket 路径。一个进程为每只鸭子各持有一条连接，收集每只鸭子想广播的内容，
并依据它们机身之间的距离推导出 RSSI 递送给其它鸭子。

无需改动 `robotd`，无需协议工作，而且有三点比真实无线电更适合开发：基于 age 的同步
路径得到真实演练；来自真值（ground truth）的 RSSI 让距离截断和不对称链路变成一个
旋钮而非一个环境搭建难题；`from` 被记录为仅用于去重的身份标识，所以定时轮换它就能
把那个耗费了一天的地址轮换 bug 变成回归测试。

## 5. 无线电必须够烂，否则 bug 会藏起来

完美的以太会掩盖真实无线电造成的 bug，这是实测而非空谈。孪生体里的四只鸭子每次都
收敛到同一段——同时启动、错峰启动，没有区别——因为每只鸭子都能即时、无损地看到其它
所有鸭子。导致现场 bug 的那一个特性，正是模拟器没有建模的那一个。

`duck-ether --discovery <s> --loss <f> --seed <n>` 把它变成一个烂无线电：一只鸭子需要
一段时间才会被 *注意到*，按对计算、从它上线时起算，并且一定比例的投递会被丢弃。
按对，因为正是这种不对称会分裂一个群体——所有人共享同一个延迟无法产生这种效果。
带种子，因为一个时好时坏的无线电只有在它的不可靠性可复现时才对调试有用。

**复现**，四只鸭子，`a` 和 `b` 唱了十二秒之后 `c` 和 `d` 加入：

```
duck-ether --discovery 90 --loss 0.3 --seed 3
```

在 main 的 chorale 上：

| 鸭子 | 声部 | 小节 | roster |
|---|---|---|---|
| a | *没有在唱* | — | 1 在范围内 |
| b | 低音 | 4 | 3 个声音 |
| c | 中音 | 4 | 2 个声音 |
| d | 低音 | 2 | 2 个声音 |

这正是那份现场报告——“有时什么都没发生，有时是两首不同的歌”——外加互不一致的 roster
和一个重复的声部。

**它尚未证明的东西。** 合并 `chorale-election` 后，同样的场景依然分裂（第 5、12 和 8 小节：
三条时间线）。但在 `--discovery 20 --loss 0.4` 下 *两者*都收敛，所以九十秒的发现时间比
那个分支写定时所设想的更苛刻，这并不能证明该修复在真实机器人上失效。值得做的实验是
一次扫描——找出每个版本在哪个发现值下停止收敛——现在这只是一个数字上的循环，而不再
需要四台机器人和一个房间。

## 6. 架构跟随宿主

这个项目构建的每个 artifact 都是 aarch64。诱人的想法是在 `qemu-user` 下于 x86 笔记本上
运行板子自己的二进制——相同的字节、完美的溯源。实测了两次，两次结果指向相反方向。

**单独跑守护进程没问题。** CI 的真实 aarch64 artifact，在 x86 笔记本上模拟：

| | |
|---|---|
| `robotctl health` | `50.0 of 50.0 Hz · 3804 ticks · 0 missed` |
| 主机 CPU | 单核的 4.7%，加载了九个 ONNX 会话，策略在驱动 |
| 策略推理 | 相对 20 ms tick 为 0.029 ms（原生实测） |

吞吐量从来不是风险所在——而真实 tick 中慢的那部分，Dynamixel 的 sync-read，在这里只是一条
本地 socket。

**在 systemd 下则不行。** 在 `systemd-nspawn` 中启动，aarch64 systemd 257 在 8.7 秒内起来，
随后什么都启动不了：

```
robotd.service:            (code=exited, status=226/NAMESPACE)
systemd-journald.service:  (code=exited, status=243/CREDENTIALS)
systemd-logind, systemd-tmpfiles, console-getty: 同样如此
```

qemu-user 8.2 无法翻译 systemd 257 用于 per-unit 命名空间和凭据的新挂载 API（`fsopen`、
`move_mount`、`open_tree`）。per-unit 加固正是用容器启动而不是在终端里跑七个进程的全部
理由，失去它就失去了意义。更新的 qemu 或许能修复；在 Debian 13 和 Ubuntu 25.04 中静态包
都是过渡性的、真实二进制是动态链接的，所以这是个从源码构建的问题，而不是一条 apt 命令。

所以孪生体运行**宿主的架构**：

* **x86 宿主**——一个 amd64 容器和一次原生构建。真实的 unit、真实的加固、真实的 journal；
  除了 CI 的确切字节之外的一切。
* **arm64 宿主（Apple Silicon）**——一个 arm64 容器，原生运行机器人自己签名过的 artifact。
  完整溯源、无模拟、命令完全一致。

对更新路径的影响很小，值得说清楚：`robotctl update apply` 端到端运行——预检、签名、
artifact 哈希、兼容性、健康门、自动回滚——因为这就是 `dev-push.sh` 的安装方式。x86 孪生体
做不到的是安装一个 *已发布*的 release。`board-test.sh` 已经在 CI 中以真实架构覆盖了真实
artifact。

## 7. 它是什么的孪生体，又不是什么的孪生体

完全相同，因为它是同一路径上的同一份代码：控制循环、策略、安全、跌倒检测、运动学、
里程计、maploc、整个 IPC 表面及其客户端、chorale 的选举和节拍、带真实 `User=`、组、
`RuntimeDirectory=` 和加固的 systemd unit，以及更新器。

被建模——真实代码路径、合成输入：执行器响应（拟合到真实 XL330 的 BAM 模型）、IMU、
ToF 深度、RSSI、摄像头图像，以及 x86 宿主上的 release 溯源。

缺失——完全没有被演练：Dynamixel 总线驱动、BLE 无线电、摄像头 ISP 和 rkaiq 的 3A、
NPU、硬件编码器及其 RGA 路径、热管理和电池。

**对这份清单的一个有用检验：** 把一周的真实 bug 过一遍。一个因为破坏了编码器到 RGA 的
零拷贝路径而损失 22 fps 的 `videoflip`；一个漏掉 stream-start 事件的 3A 引擎；一个收敛一次
就停下的自动曝光循环；一个分数通道坍缩成两个值的 INT8 head；一个丢读数的舵机总线。孪生体
一个都抓不到。这不是设计的缺陷，而是它的边界——硬件仍然是驱动真实存在的唯一地方。

## 8. 一只仿真鸭子就是一只鸭子，并且只声明一次

MuJoCo 中的鸭子会向 rendezvous 注册，并出现在其主人的 Hugging Face 机器人列表中，紧挨着
真实的那几只。这很值得拥有——整条远程访问路径都能在没有硬件的情况下演练——同时也很值得
做到毫不含糊，因为一个分不清两者的客户端最终会让某人驱动着一个仿真、却纳闷架子上的
机器人为何一动不动。

**一个事实，只声明一次。** `configd --simulated <serial>` 是它被声明的唯一地方。
`system.info` 携带它，`mediad` 在那里获知它——它启动时本来就会为名字问这个问题——并把
`simulated: true` 放进它注册的 `meta` 中。`robotctl system info` 会打印它。其它任何地方
都不推断它，尤其没有客户端会以 `sim-` 开头的序列号作为判断依据：那是一个需要三处达成
一致的约定，而其中一处迟早会搞错。

`mediad --sim-camera` 也会设置它。不是第二个真相来源——而是针对唯一可能出错的那个情形的
兜底：`configd` 只被问一次、还带超时，而那台机器上所有守护进程都在同一瞬间启动，一次
迟到的应答会把一只仿真鸭子注册成硬件。两者不可能产生分歧，因为不存在这样的组合：帧来自
模拟器、而它们所属的机器人是真实的。

**身份正是这个标志需要带值的原因。** 仿真鸭子没有 SoC 序列号，也没有逐鸭子的
`/etc/machine-id`——同一场景里的四只鸭子共享它们笔记本的 machine-id，于是会一个接一个地
把彼此挤出列表；而 macOS 根本没有这个文件，于是谁都注册不上。`sim-duck-a` 对每只鸭子
稳定，且跨重启稳定，这正是 `hardware_id` 必须满足的性质（`remote-access-design.md` §3.7）。
机器人的名字由它派生，正如真实机器人的名字派生方式一样，所以孪生体里的鸭子是
`duck-eb55`，而非运行它的那台机器的主机名。

**账号是机器人自己的流程，而不是交给它的一个凭据。** `updaterd` 在模拟器里以 `--token`
指向状态目录运行，而 `robotctl account login` 是对 Hugging Face 的 RFC 8628：一个代码，
在手机或浏览器上输入，和真实机器人上一模一样。`mediad` 轮询那个文件，一旦出现就注册，
所以会话中途登录的鸭子无需重启。凭据路径没有任何部分是模拟的，这正是关键——它是远程
访问里最容易在细微处出错、又最难在板子上测试的那一半。

需要知道的后果是 rendezvous 的，而不是我们的：peer 以 token 为键，所以同一账号同一时间
只列出一只鸭子。每只鸭子跑它自己的 device flow，所以这是一条关于账号的规则，而不是
模拟器的限制。

## 9. GStreamer 不是 Linux 专属的部分

`mediad` 的 pipeline 曾被整体用 `#[cfg(target_os = "linux")]` 门控，这在唯一来源是板载摄像头
时对守护进程成立，但在 `Source::Sim` 到来后就不再成立。`pipeline.rs` 里约 230 行是真正的
硬件代码：`v4l2src`、驱动的缓冲区池和 `GstVideoMeta` 分配查询，以及从媒体拓扑读出的传感器
模式。其余部分——appsrc、tee、appsink、`webrtcsink`、encoder-setup 信号——是 GStreamer，
在 GStreamer 运行的任何地方都能跑。

所以门控改为 `any(target_os = "linux", feature = "gstreamer")`：在机器人的 OS 上默认假定，
在别处按需启用。开发者的机器用两个 Homebrew formula 就能得到真实的守护进程、真实的控制台
和针对仿真摄像头的真实 WebRTC，而在 Linux 之外的 `Source::Camera` 会是一个明确的错误，
而非一个缺失的分支。

两个，而不是一个，而第二个是个值得记下来的陷阱：Homebrew 的 `gstreamer` 把 `libgstnice.dylib`
链接到一个它并不依赖的独立 `libnice-gstreamer` formula 里。没有那个 formula，链接悬空，
`webrtcbin` 没有 ICE agent，而一切看似正常——pipeline、摄像头、rendezvous 注册——直到第一个
消费者要一个 pad，会话就在某个 GStreamer 线程里以“libnice elements are not available”死掉。
构建检查发现不了它，所以 `scripts/duck-sim` 检查的是元素而非 pkg-config 文件。`exposure.rs`
保持仅 Linux，这不是打包问题——它通过 `ioctl` 写 V4L2 控制项，而仿真源背后没有传感器可测。

在那里默认关闭，是有意为之：Homebrew 的 `gstreamer` 是一个合并 formula，会把 gtk4 和 ffmpeg
一并拉进来，而在 `robotd` 上工作的人不该为了跑 `cargo check` 就需要它。代价是没有 CI 任务
编译这一组合——CI 全程是 `ubuntu-*`——所以靠的是开发者自己跑它来保持可用。一个执行
`cargo check -p mediad --features gstreamer` 的 `macos-latest` 任务能改变这一点，但它被有意
省略了：CI 已经够慢，而 macOS runner 更慢。

## 10. harness

一个 MuJoCo 进程、一个窗口、一个场景里的 N 个鸭子机身，所以鸭子共享物理并会相互碰撞。
`microduck_rl` 拥有那一半：它已经有场景、BAM 执行器模型和 mjlab，而向守护进程提供一个机身
正是它今天所做的 sim2real 的镜像。

每只鸭子是一个 `systemd-nspawn` 容器——没有守护进程、没有镜像格式，容器就是一个目录——
从一个 Debian 13 Trixie rootfs（`mmdebstrap`，非特权，238 MB，三分钟内）启动，每只鸭子一个
overlay。每只作为一个服务运行（`systemd-run --unit=duck-a …`），所以 `duck-sim down` 就是一条
`systemctl stop`：一只退不出去的鸭子算不上一只好鸭子。

`machinectl shell duck-a` 是进入的方式，`duckctl open duck-a` 是观看的方式，而
`scripts/dev-push.sh microduck@duck-a` 是安装一次构建的方式。

两个限制源自物理而非管道。**鸭子不会热加入**——MuJoCo 会编译它的模型，所以改变数量会重启
模拟器，这正是 `RemoteIo` 要重连的原因。而**摄像头渲染才是扩展的墙**，不是双足：N 个 30 fps
的离屏渲染远比 N 个十五自由度机身更贵，所以摄像头应该按鸭子选择启用。45 Hz 的健康门让两者
都成为硬边界而非软边界——鸭子太多时它们不会变慢，而是直接 *unhealthy*，更新器就开始回滚
release。

## 11. 选错模型的三种方式

每一种都表现为“鸭子仰面朝天”，而每一种都耗费了一个小时。

**`scene_walk.xml` 没有碰撞。** 它包含 RL 工作训练所用的模型，其执行器默认类带有
`contype="0" conaffinity="0"`。鸭子穿过场景里确实存在的地板下沉——躯干 z 在一秒内从 0.120
到 -0.105——而守护进程正确地报告一个躺着的机器人。`scene.xml` 包含
`robot_allcollisions.xml`，这才是孪生体想要的。

**`qpos0` 不是一个姿态。** 每个关节都为零是这个机器人从来不会处于的形状；守护进程从它的
home 坐标系测出 0.41 rad，于是相当合理地试图把一个已经折叠的机器人站起来。场景带有
`INIT`、`STAND`、`SIT` 和 `FOLD`，而 `STAND` 匹配 `duck_control::DEFAULT_POSITION`——其右腿是
*镜像*的，不是对称的，这值得读一读而不是想当然。

**扭矩应当在启动时就打开。** `robotd` 启动时从不启用扭矩，因为被更新重启的守护进程必须让
站着的机器人继续站着——舵机已经在维持姿态。一个软趴趴启动的模拟器，它的鸭子在第一次读取
之前就已经倒在地上了。

守护进程真正为之编写的那次启动是 `--keyframe SIT`：一只被发现折叠着的鸭子，它识别出来，
并用 sitstand 策略站起来。

## 12. 可复用的已知材料

`~/MISC/microduck_maploc`（在其它所有方面都已过时）有两样值得拿：`sim/tof_sensor.py` 里一个
仿真的 VL53L5CX——8×8 区域、45° 方形视场角、4 m 量程、随距离增长的噪声、15 Hz——以及
`sim/assets/apartment.xml`，一个有 83 个 geom 的室内场景。对 maploc、对游荡、对任何会躲藏的
东西而言，一个房间远比一块地平面有价值。
