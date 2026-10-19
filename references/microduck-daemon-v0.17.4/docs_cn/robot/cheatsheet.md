# 速查表

`robotctl`，它运行在机器人上。这里的每一条命令都取自发布它的分支上的 `--help`，而不是凭记忆。

只读命令无需权限。任何**改变**机器人的操作都需要 `sudo`（或者对于 `configd`，是一个在 `--allow-user`/`--allow-group` 中的用户；对于 `updaterd`，是 `updater.toml` 里的 `allow_uids`/`allow_gids`）。

分支构建、候选发布以及更新后的重启陷阱在 [`cheatsheet-dev.md`](cheatsheet-dev.md) 里——它们需要一块开发板。从笔记本电脑通过蓝牙、无需网络和 ssh 控制同一台机器人，见 [`duckctl.md`](duckctl.md)。

## 在机器人上 —— `robotctl`

### 首先要运行的东西

```
robotctl version
```

每个守护进程正在*运行*的版本对*已安装*的版本，以及两者不一致时的警告。在相信任何其他诊断之前先运行它——一次更新后一个守护进程仍在跑旧代码，看起来和你刚发布的修复里的 bug 一模一样。见下面的「更新之后」。

```
robotctl health
```

硬件和软件汇总成一份报告。当机器人不健康或不可达时以非零退出，因此它可以作为脚本的门控——一个发烫的电机或一个被固定的组件会被*报告*，而不是被*判定*，且不影响退出码。`--json` 用于生成支持包；`--check` 会先检查更新源，所以关于它的那一行会说明它此刻是否有响应。

### 观察循环

```
robotctl monitor
```

客户端请求了什么，与实际应用了什么并排显示，两者不一致时点明原因——安全机制一直在做钳制，没有这一信息，「摇杆往前推、机器人却不动」是无法读懂的。限制会被完整拼写出来而不是只给个名字：`deadman — no intent arrived recently, velocity zeroed`。

这一帧上还有：每个关节的实测值对下达值、IMU 的投影重力以及据此得出的跌倒判定，以及作为轨迹显示的实际循环速率，这样一次已经恢复的卡顿仍然可见。投影重力是这个流上唯一的 IMU 量——竖直大约是 `[0, 0, -1]`，`fallen` 就是据此判定的。过期读取计数器以及让它们有意义的比值在 `robotctl health` 里。

表头最后一行是机器人的*状况*而不是*行为*：电池组以伏特和分数表示的电荷、最热的舵机以及板子自身的温度。它来自 `robot.health`，每两秒轮询一次，因为这些东西都不在状态流上——而任何不对劲的东西都会在这一行被点名，无论是 `unhealthy: control loop at 43.9 Hz`、`degraded: no robot on the motor bus after 3 attempts`，还是 `orientation frozen — 25 stale reads`。最后这一条只出现在这一行、帧上没有别处：一块已经停止融合的板子仍然会回应总线，所以什么都不会报错，而上面的重力向量会无限期地保持一个貌似合理的姿态。

0% 就是 `BATTERY_EMPTY_V`，那是 `robotd` 让机器人坐下并断电的位置，所以这个数字是倒计时而不是量表——30% 变黄，15% 变红。一个尚未取得的读数会显示 `batt not read yet` 而不是 `0.00 V`，开机后的第一秒和一条无法应答的总线看起来都是这样。即使完全没有状态这一行也会绘制，而那正是它最要紧的情形：一块舵机断电的板子永远完不成一个控制节拍，所以流上什么都到不了，原因只出现在健康应答里。

底部边框会点名已加载的策略——各个 `.onnx` 文件，以及是否配置了站立网络——因为 `walk` 是一个两个发布版本都在报告、但步态不同的模式。没有策略的机器人会这么说，策略加载失败的机器人则会那么说，而流的 `held` 无法区分两者。

右侧，**机器人以它站立的姿态被画出来**——与策略训练时所用的同一套视觉模型，由实测关节角摆姿、由 IMU 重力向量倾斜。一条折反了的腿、一颗扎进地板里的头、一只侧躺的鸭子，在关节表里都只是一串数字；在这里每一个都一目了然。它默认开启，只要终端足够宽就出现（大约 110 列——表格优先，机器人拿剩余的宽度）。**`d`** 关掉它；**`[`** 和 **`]`**，或 `←`/`→`，环绕观察它。

只要 ToF 在输出帧，它看到的东西就会画进同一个场景——黄色代表命中、绿色代表地面——并针对机器人自身身体做深度测试，所以喙后面的点会被身体遮住。这正是「它在看我的手，还是在看它自己」得以回答的原因。它不需要按键：帧到达时点出现，帧停止时点消失。

在它下面，当这一列足够高时，**一张机器人到过哪里的地图**：由足部接触和 IMU 得出的里程计轨迹，用盲文点阵绘制。面板永远不会变大——世界会随着轨迹一起缩小，好让整条路径始终留在画面内。`+` 是它出发的位置，`●` 是机器人，带一条短射线表示朝向，屏幕向上就是它开机时的朝向。这里没有磁力计，所以这是相对运动、会漂移；它回答的是「它是不是走了一个圈」，而不是「它现在在哪」。

`q` 退出；窗口太短放不下全部时，`↑`/`↓` 滚动关节列表；`u` 在角度与弧度之间切换；`t` 打开 [ToF 矩阵](#tof-传感器-tofd)；`c` 打开[摄像头](#摄像头-mediad)；`d` 切换机器人视图，`[` / `]` 环绕观察；`p` 打开手柄的原始输入流——来自游戏手柄的每一条 evdev 报告，以及它们之间的间隔，这是唯一能看到无线电卡死的地方（[配对一个游戏手柄](pair-a-gamepad.md#驾驶时掉线)）。一个带惯性单元的手柄——Pro Controller 的仿品有，Xbox 没有——会把该块扩展成一个面板：一个线框手柄，随着你手里的那个一起倾斜和转动，显示它的俯仰、横滚和漂移的偏航、原始加速度和角速度，以及陀螺仪的静止偏置是否已学习完毕（拿稳半秒）。黄色条是手柄的前沿。角度在屏幕上以度显示——关节、头部以及偏航角速率。
被重定向或管道连接时，它改为每个节拍打印一行，于是 `> run.log` 和 `| grep FALLEN` 都能正常工作，而那些数字无论屏幕设成什么始终是弧度。关节向量在 `--json` 里，它携带整个状态，每行一个对象：

```
robotctl monitor --json --hz 50 > run.jsonl
```

### 配置机器人

```
sudo robotctl configure
```

这台机器人被改过什么，别的什么都没有：

```
robotctl configure --list
```

一台没人碰过的机器人什么都不打印——那就是答案。加 `--json` 可生成支持包。它不需要 root 也不需要终端，这正是重点：这是面对一台行为怪异的机器人时要问的第一个问题，而在此之前，回答它意味着通过 ssh 开一个全屏编辑器。

一个针对 `/etc/robot/robotd.toml` 的交互式编辑器：守护进程认识的每一个键，功能开关在前（策略开/关、行走/滚轮、limp-fall、音频、宠物检测、电池关机、摄像头和视频质量……），当前值对照默认值，还有一行文档。SPACE 切换，ENTER 输入值，`u` 把一个键恢复为默认值，`ctrl+f` 打开对屏幕上一切内容的模糊搜索（输入时选中项跟着走；ENTER 或 ESC 让它停留在那里）。黄色（标有 `•`）的值是这台机器人偏离默认值的键；其余都是内置默认值，而 `unset` 的可选项显示它们解析成的值 `(auto)`。

三个值得信赖的性质：

- **它不可能与守护进程不一致。** schema、默认值和校验来自 `robotd` 解析文件用的同一个 crate，而且键列表由测试钉死为完整——守护进程里新增一个 `[section]`，要么在这里出现，要么构建失败。
- **它不会吃掉你的文件。** 注释、顺序以及来自其他发布的键都原样保留；只有你改动的键才会被写回。恢复一个键会移除它（以及附在它上面的注释），而不是把默认值钉死，这样文件始终保持一份*决策*清单，而不是默认值的副本。
- **它写不出让 robotd 拒绝启动的文件。** 每次保存都会先经过守护进程自己的加载器校验，原子地进行（临时文件 + 重命名），并在拒绝时给出原因。

保存时会根据改动实际所需、由真正读取它的守护进程来提议后续动作：大多数键要重启（`[media]` 和 `[duck_detector]` 是 `mediad` 的，`[head_imu]` 在 zero3 上是 `tofd` 的、在 beta 上是 `robotd` 的），`[policy]` 要 `robotd` *重载*——电机保持供电——而 `[pad]`、`[pad_imu_head_control]` 以及其他 `pad_` 小节什么都不用做，手柄守护进程会在一秒内感知到。需要 `sudo`，因为文件归 root 所有——没有它编辑器会以只读方式打开，并在第一次写入时说明。`--file` 让它指向别处以供台架副本使用。随发布提供的 `deploy/robotd.toml` 仍然是每个旋钮*为什么存在*的参考；这里是用来拨动它们的。

#### 视频质量

```
sudo robotctl configure
```

设置 `media.quality` —— `1080p30`、`720p30`、`720p15` 或 `360p30` —— 并接受它提议的重启。把 `media.source` 设为 `test` 则改为推流测试图案，这正是一块没有摄像头的板子想要的：WebRTC 的*控制*通道搭载在视频轨上，所以一条无法启动的管线会同时损失两者。测试图案忽略 `media.quality`，以 256x144@5 运行——它只是为了把会话建立起来，画一个 720p 的会花掉真实摄像头五倍的 CPU。除非你设置，否则 `media.bitrate` 跟随质量；单位是比特每秒。

`media.congestion_control` 是该小节里的另一个旋钮，而它才是移动 CPU 的那个：`disabled` 会去掉带宽估计器，那是 `mediad` 里最大的单一消耗者（一个核心的 7.6%，而采集只有 0.3%），并把 `media.bitrate` 变成实际速率而不是起点。它的代价是自适应性——在一条劣化的链路上，画面会卡住而不是速率下降。

720p30 是这条管线被测量过的档位；一个撑不住的档位会跑得更慢而不是失败。`robotctl monitor` 在底部边框报告实际达到的速率，当它低于所请求的 90% 时，旁边会用黄色显示 `of <target>`。实际应用了什么：

```
journalctl -u mediad -b | grep streaming
```

### 策略与技能

一个**槽位（slot）**是机器人默认运行的东西——行走步态、站立网络。一个**技能（skill）**是它被要求时运行的东西：一次踢球、滚翻、鞠躬。两者都是 Hub 上的文件，两者都可以在不发布守护进程的情况下更换，下面的一切都不需要重启。

这台机器人此刻正在运行什么：

```
robotctl policy list
```

两张表：七个槽位，然后是技能——默认运行什么，被要求时运行什么。

```text
   SKILL        RUNS FOR  POLICY
   kick_left       0.5 s  ball_kick_left.onnx
   roulade           1 s  roulade.onnx
 * polite-bow        4 s  fffiloni/microduck-polite-bow-b1d864/main/policy.onnx
   ground_pick         —  由机器人自己驱动
   sit_toggle          —  由机器人自己驱动
```

`*` 标记配置有意见的每一行，底部有计数。一个你关掉的槽位会显示 `switched off`，而不是看起来像机器人从来没有过它。路径里的目录会被去掉，因为 `ORIGIN` 列已经说明了它是哪一个。

`ground_pick` 和 `sit_toggle` 没有时长，因为机器人自己驱动它们——它们像其他技能一样响应 `robot do`，但它们不是你能改的条目。

#### 更新的官方策略集

机器人走路用的策略集在 Hub 上，并各自独立地打版本：

```
robotctl policy check
```

```
sudo robotctl policy update
```

`check` 打印已安装什么、最新是什么，以及该仓库还提供什么；它什么都不改，在无法连上 Hub 时也会明白地说出来。`update` 取最新，除非你指定一个——`--version v1` 是回退的办法。机器人回到它的家姿势，重新读取每个槽位并再次驱动，而**你自己加载过的槽位会被原样保留**，因为它完全指向别处。

#### 更新的鸭子检测器

`mediad` 用来发现其他鸭子的模型同样在 Hub 上（`pollen-robotics/microduck-duck-detector`），并各自独立地打版本：

```
robotctl duck-detector check
```

```
sudo robotctl duck-detector update
```

与策略那一对形状相同——`--version <tag>` 指定一个，`check` 什么都不改。`update` 会重启 `mediad`，让控制台的视频中断片刻；之后检测器是否运行，取决于 `robotctl configure` 里的 `[duck_detector] enabled`。

#### 试你自己的文件

无需发布、无需编辑文件、无需重启：

```
sudo robotctl policy load walk /home/microduck/my_walking.onnx
```

如果机器人在走路，它会回到家姿势、加载它并再次驱动。如果它在做别的事——坐着、站着不动——什么都动：被替换的网络不是正在运行的那个，交换发生在底层。`sudo robotctl policy reset walk` 把那个槽位放回；不带槽位，则把全部七个放回。

#### 试别人的策略

别人为这台机器人发布策略：

```
robotctl policy search microduck
```

```
sudo robotctl policy load walk RemiFabre/microduck-flamingo-cycle
```

仓库名就够了——它一步完成抓取和加载。加 `@v2` 指定一个修订版，加 `:policy.onnx` 用于携带多个文件的仓库。`policy list` 会把它标为 `community`。

**陌生人发布的东西没有任何人验证过。** 让它试起来安全的是关节钳制、跌倒反射和形状门控——而不是描述。第一次把它放在支架上试。

一个 manifest 说它不能在这里运行的策略——观测宽度不对、守护进程更新、机器人不同——会在下载之前被拒绝。没有 manifest 的会被接受，并在加载时照常检查。

#### 添加一个技能

放进 walk 槽位的策略替换步态。作为*技能*添加的策略则与它并排存在，在被要求时运行，这正是大多数已发布一次性动作想要的：

```
sudo robotctl policy add polite-bow fffiloni/microduck-polite-bow-b1d864
```

```
robotctl robot do polite-bow
```

时长来自仓库的 manifest。一个一直保持到另有指令为止的策略——比如单脚站立——自己没有时长，所以给它一个，以及它读取的 twist：

```
sudo robotctl policy add flamingo RemiFabre/microduck-flamingo-cycle --hold 5 --command 1,1,0
```

`--command` 是它运行期间喂给网络的东西。大多数技能什么都不需要：它们在全零命令上训练，被选中本身就是触发。一个把它的 twist 读成别的东西的策略——flamingo 的是 `[flag, side, 0]`——需要把它拼出来，而它的 README 会说明这些槽位的意思。

`sudo robotctl policy remove <name>` 移除一个。本机器人这个发布自带的技能会在你移除后回来，因为移除条目只是移除了覆盖。

机器人必须处于驾驶状态技能才能运行——先在手柄上按 **Start**，否则请求会被拒绝并说明原因。

#### 把技能绑到按键上

```
robotctl pad bindings
```
```text
a           sit_toggle
b           ground_pick
x             (nothing)
y             (nothing)
lb            (nothing)
rb            (nothing)

`robotctl pad bind <button> <skill>` 改一个；padd 在一秒内感知到。
```

```
sudo robotctl pad bind x polite-bow
```
```text
a           sit_toggle
b           ground_pick
x           polite-bow
y             (nothing)
lb            (nothing)
rb            (nothing)
```

那会写一行，而且只写你指定的那个按键：

```toml
[pad]
x = "polite-bow"
```

放回去：

```
sudo robotctl pad reset
```

**什么都不重启**——`padd` 在一秒内注意到。

一个绑定指向本机器人没有的技能时，会在列表里被标出，而不是你按下时无声地什么都不做。

六个按键可绑定：`a`、`b`、`x`、`y`、`lb`、`rb`。**`lb`/`rb` 是肩键（bumper）**，不是模拟扳机，后者是嘴巴和嘎嘎声。空名字会关掉一个按键，`pad reset <button>` 放回一个。X 和 Y 出厂为空，所以它们是新技能的安放处；X 在按住时会重复发送，这正是滚翻这类链式技能想要的。

手柄的其余部分不可绑定：Start 让机器人站起来并切换策略，方向键（D-pad）选择摇杆的含义，长按 Select 切断扭矩并给机器人断电——那个能停下机器人的按键，恰恰是值得别让一次配置编辑就弄丢的按键。名字会对照机器人实际拥有的技能检查，所以打错字会被连同列表一起拒绝，而不是变成一个死按键。

#### 全部放回

```
sudo robotctl policy reset
```
```
sudo robotctl pad reset
```
```
robotctl configure --list
```

最后一个只打印这台机器人相对默认值改了什么、别的什么都没有——一台没人碰过的机器人打印一行说明这一点。当一台机器人行为怪异、而你不确定留下了什么设置时，它是首先要运行的东西。

#### 槽位，以及四件值得知道的事

槽位是 `walk`、`stand`、`sitstand`、`ground_pick`、`kick_left`、`kick_right` 和 `roulade`。`load` 把选择写进 `/etc/robot/robotd.toml`，所以它能挺过重启，也能挺过更新——一个发布替换二进制和它自带的策略，而不是指向别处的那一行。

- **重置一个已经重置过的东西什么都不做，并会说明。** 没有回位、没有重载，而且当确实没有东西要写时，也不需要 `sudo`。
- **一个不是 `obs[1,61] -> actions[1,14]` 的策略会在任何东西改变之前被拒绝。** 文件在机器人仍在跑旧策略时就被打开检查。
- **一个最终仍加载失败的策略会保留原来在跑的那个。** 试一个步态不可能让你丢掉已有的那个。
- **一个到下次启动时已经丢失的文件，代价是它的槽位，而不是机器人。** 槽位回退到本机器人自己的策略，`robotctl health` 报告*降级*并点名该文件，`policy reset <slot>` 清除它。一个加载不起来的官方策略仍然是**不健康**——那是一个坏掉的发布，更新器会回滚它。

`none` 关掉一个槽位——除了 `walk` 之外的每个槽位，`walk` 是其他槽位回退的去处、不能为空。有些策略需要这样：一个自己做站立的策略，希望站立网络别挡路，否则命令为零时机器人就会把自己交给它。

#### 为每一台机器人发布策略

官方策略集里的一个策略会到达每一台机器人，添加一个只需四步、无需发布守护进程：

1. 把 `.onnx` 上传到 `pollen-robotics/microduck-policies`。
2. 在它的 `manifest.json` 里加一个条目：
   ```json
   { "file": "polite-bow.onnx", "kind": "episodic", "duration_s": 4.0 }
   ```
   该策略集的 manifest 是 `schema_version: 2`；一个普通的一次性动作只需要这三个就够了。
3. 给它打标签——`hf repos tag create pollen-robotics/microduck-policies v4`。
4. 在机器人上：`sudo robotctl policy update`。

那个条目正是一次性动作所需的、且仅此而已：**一个带 `duration_s` 的 `episodic`，在它训练所用的全零命令上，变成一个机器人能按名字响应的技能**——准备好 `robot do` 和按键绑定，别的什么都不用改。一个 **`perpetual`** 的则是步态，需要一个槽位指向它。

一个守护进程必须去*驱动*的策略——随时间写一个相位，或翻转一个姿态标志——会在 `command.encoding` 下声明这一点，它的数字成为那只手臂的时序，而不是一个新技能。地面拾取和坐↔站就是这两者，把这些条目写错正是这里唯一值得小心对待的错误。

完整的 manifest、每个字段以及每一项在机器人上改变什么：见 [`../policy-manifest.md`](../policy-manifest.md)。

策略必须具有的形状以及加载时还会检查什么，见 [`../design/robotd-design.md`](../design/robotd-design.md) §2.3；策略从哪里来、`official` 是什么意思、技能如何声明自己，见 [`../design/policy-channel-design.md`](../design/policy-channel-design.md)。

### 给关节供电（`robotd`）

```
sudo robotctl robot init
```

```
sudo robotctl robot relax --yes
sudo robotctl robot reboot-motors           # 每个舵机；或 `reboot-motors 3 11` 只对那几个。先断扭矩，再 init / Start
```

`init` 给关节供电并在大约两秒内爬升到家姿势——**它会移动每一个关节**，所以让机器人待在支架上。它不需要策略，而这正是手柄 Start 在通往驾驶途中所做的事，所以手动做是台架上的活。

`relax` 切断电源，如果没东西扶着**机器人会瘫倒**，这正是它要求 `--yes` 的原因。它是除拔电源之外回到松软状态的唯一办法：再按一次 Start 会停止策略并让机器人保持站立，`robot.stop` 在保持站立的同时把速度清零。

两者都经由拥有电机总线的 `robotd`。`robotd init`——这个子命令——对守护进程没在运行的机器人仍然存在，而且它需要守护进程停着，因为一条 UART 上的两个写入者会互相破坏对方的应答：

```
sudo systemctl stop robotd && sudo /opt/robot/daemon/current/bin/robotd init && sudo systemctl start robotd
```

`reboot-motors` 也能通过蓝牙触达，即 `duckctl reboot-motors`（[`duckctl.md`](duckctl.md)）——`relax` 则不行。

**更换一个电机**不需要任何配置工具。把新舵机直接从盒子里装上（ID 1，57 600 波特率），给舵机供电，`robotd`——或 `robotd init`——就会找到那个不再应答的关节，把新舵机刷成该关节，设置它的寄存器并重启它。日志会说 `factory-fresh servo on the bus; flashing it as the missing joint`，然后是 `replacement servo adopted`。一次一个：缺两个关节时它无法分辨新舵机是给谁的，会等待并说明。

无论机器人是否跌倒过，`init` 都能工作——默认情况下跌倒是一份*报告*（在 `robotctl monitor` 里可见），而不是门控，与原型一致。在 `robotd.toml` 里设置了 `[safety] fall_limp` 或 `fall_recover` 的板子会武装门控：在那里，跌到的机器人会变松软，并拒绝 `init`/`enable`/技能，直到它被扶起来。

### 把机器人抱起来

一只走路中被抱起的机器人会察觉：被提起约 0.2 秒后，双腿落进站立站姿并保持——无论你怎么抱它，抱着身体或头、倒过来、转圈——而在它双脚着地约 0.2 秒后，策略把它接回去。`[pickup] enabled = false`（`sudo robotctl configure`，然后让它重启 `robotd`）关掉它，腿会在你手里继续走，直到你按 Start，就像过去那样。

```
journalctl -u robotd -f | grep -E 'picked up|put down|pickup'
```

`picked up — pausing the policy` 和 `put down — handing the robot back to the policy` 是两个边缘，各自带有越过的概率。暂停期间，`robotctl monitor` 在原本显示 `upright` 的位置显示 `PICKED UP`（策略读到的是 `held`），而 `robot.state` 携带 `safety.picked_up`。只有行走和站立被监视——一个技能、一次坐下或一次跌倒从不被暂停。

**默认开启。** 分类器只在仿真中训练过；能说明它出错的是：机器人在正常走路时来了一次暂停（行走冻结一瞬，然后继续），或机器人被放下后仍保持暂停。同一小节里的 `pause_threshold` 和 `resume_threshold` 移动这两个边缘；设计见 [`robotd-design.md` §2.4.2](../design/robotd-design.md)。

### 兽医手册（vet book）

鸭子的整个一生，用数字记录——由 `robotd` 从它第一次启动起记录，跨过每一次更新，直到板子被重新刷写：

```
robotctl vetbook           # 实时页面：生命、电机、日记——q 退出，←/→ 切换页签
robotctl vetbook | less    # 同样的内容作为纯文本
robotctl vetbook --json    # 手册以 robot.vetbook 应答的形式
```

**★ 纪录**保存来自每个传感器的个人最好成绩：挺过的最快旋转与最陡倾斜（身体 IMU）、无跌倒的最高速度和最长步行（里程计）、峰值电流和最快的关节（舵机）、声音、喙张合次数和手电筒时长、照片和推流视频（`mediad`）、激光看到的新表面以及最近和最远的东西（`tofd`），还有一次充电的最长续航。`mediad` 和 `tofd` 每分钟向 `robotd` 报告一次它们的数据（`robot.vetbookTally`）。

页面上的 `n` 在日记里写一行带日期的记录——「重粘了喙」、「新脚」——Enter 把它保存在机器人上。

**`s` 分享它。** 在一台已登录 Hugging Face 的机器人上（`robotctl account login`），`updaterd` 会把手册发布为该账户上的一个**公开**数据集，`<user>/<robot name>-vetbook`，页面会显示一个指向它在 `microduck-vetbook` Space 上卡片的链接，并带一个二维码以便在手机上打开。卡片是八种设计之一，每次分享随机绘制，配色由鸭子的名字决定，在万圣节、十二月和情人节还有季节皮肤；它承载鸭子的心情（从「Feeling great!」到「Vet time!」）、它最稀有的徽章和最了不起的趣事。旁边是一张 1080 × 1920 的 Instagram/TikTok 竖版故事图；Space 的页面上有发布它的按钮，粘贴的链接会展开成卡片。日记笔记留在机器人上。给鸭子改名会分享到一个新数据集；旧的保留。Space 的源码在 `spaces/vetbook/`。

**生命**：清醒的小时数，带扭矩、行走、站立、坐着、被抱在手里和在地上——行走指的是步态被要求移动，所以 velstand 站着不动仍算站立；走过的米数（接触里程计，仅在行走时）；跌倒、软着陆、被抱起、被抚摸、地面拾取、坐下、站起、歌曲和按名字计的每个技能；开机、更新和估算的充电次数；最热的舵机和板子以及电池的范围。**电机**：按关节计的扭矩小时数、轴行程、承载电流和此刻其中舵机的热度；它锁存的每次过载/过热/冲击/编码器跳闸（不包括输入电压——每个舵机在充满电的电池组上都会锁存它，因为它高于自身额定值）；以及它之前的舵机用了多久。**日记**：更新、舵机更换、充电和笔记，带日期。

**舵机更换会自己记下来。** 当 `robotd` 把一个出厂新舵机（id 1，57600 波特率）收编进一个缺失的关节时，它会用旧舵机做过的事给旧记录办理退休，并让新舵机从零开始。一个装进去时已经设成该关节 id 的舵机是隐形的——Dynamixel 没有序列号——它会沿用旧舵机的记录。

手册是 `/var/lib/robot/vetbook.json`，每分钟以及在 `robotd` 停止时保存一次，所以从电池端断电最多丢失一分钟。除非用 `robotd --vetbook <file>` 指定一个，否则 `--fake` 和 `--sim` 把它保存在内存里。

### 改摇杆映射

运行 `sudo robotctl configure`，然后搜索 `pad_axes` 来选择轴、方向和增益。头、身体和滚轮的限幅在 `pad_head`、`pad_body` 和 `pad_roller` 下；步行速度限幅仍在 `pad_drive` 下。这些与 `[pad]` 按键绑定共享 `/etc/robot/robotd.toml`，并在一秒内生效而无需重启——对蓝牙手柄和 UDP 手柄一视同仁。

例如，前进/后退在右摇杆、转向在左摇杆，且转向更柔和：

```toml
[pad_axes.drive.vx]
source = "right_y"

[pad_axes.drive.vyaw]
source = "left_x"
gain = 0.7

[pad_drive]
vx_min = -0.2
vx_max = 0.3
```

转向绑定保留默认的反转；设 `invert = false` 反转它。用 `source = "none"` 禁用一个命令。移除一个覆盖以恢复该模式的默认值。无效的编辑会被记日志，之前的映射保持有效。完整的求值顺序和 IMU 行为见[控制器配置](../design/robotd-design.md#42-params)。

### 游戏手柄（`configd`）

每个按键*做什么*——以及如何改它——在上面的**策略与技能**部分（`robotctl pad bindings`）。这一节讲的是如何让手柄连上。

```
robotctl pad status
```

```
sudo robotctl pad pair
```

```
sudo robotctl pad pair 78:86:2E:BB:13:28
```

```
sudo robotctl pad forget 78:86:2E:BB:13:28
```

配对是每只手柄一次，并有它自己的一页——[`pair-a-gamepad.md`](pair-a-gamepad.md)：哪个按键让手柄进入配对模式、不忘记第一只地添加第二只手柄，以及它无法配对时该怎么做（`/etc/bluetooth/main.conf` 里的 `Privacy` 设置比其他任何原因都更常是答案）。

`padd.service` 从启动时运行，驱动任何连接上的手柄，所以配对是唯一一步。手柄的每个部分各司其职——面键和肩键运行技能，方向键选择摇杆的含义，两个居中的小按键是长按——这样切断扭矩的操作不会在大拇指擦过时触发：

| 控制 | 作用 |
| --- | --- |
| **A** / 交叉 | 坐 ↔ 站 |
| **B** / 圆圈 | 地面拾取 |
| **X**、**Y**、**LB / RB** | 无——留给你自己的技能（`robotctl pad bind`，例如 `lb kick_left`） |
| **RT / LT** | 嘴巴（任一扳机）——RT 也嘎嘎叫；LT 按住时拖着「wheee」 |
| **方向键左** | 移动模式：左摇杆前进/后退和横移，右摇杆转向。手柄启动时所在模式 |
| **方向键上** | 头模式：左摇杆头的偏航和俯仰，右摇杆颈的俯仰和头的横滚。身体保持不动 |
| **方向键右** | 头 + 移动：左摇杆前进/后退和转向，右摇杆环顾（头偏航和俯仰）。在 `[pad_imu_head_control] enabled` 且手柄带 IMU 时，手柄的倾斜来摆头，摇杆保持移动映射——见下 |
| **方向键下** | 身体 + 头：左摇杆上/下起身和下蹲，左/右让身体侧倾；右摇杆环顾（头偏航和俯仰）。身体不走 |
| **Start**，按下 | 第一次按：扭矩开启并爬升 2 秒——机器人坐着（或几乎坐着）时到座位，否则到家姿势。第二次按：再次测量机器人——坐着的，sitstand 策略用约 2 秒把它安置进座位、起身、然后走；其他情况，策略立即开走。日志每次都会说它看到了什么。此后一按切换策略：停止它让机器人回到家姿势、僵硬（坐着的机器人保持坐着） |
| **Start**，按住 1.5 秒 | 家姿势、电机僵硬、策略关——「把一切放回」的按键，从任何地方。坐着时保持坐着、僵硬：下一次 Start 之后 A 让它站起来 |
| **Select**，在 2 到 4 秒之间松开 | 休息（`robot.rest`）：一个驾驶中的机器人坐下、缓入休息姿势，然后扭矩关闭、每个舵机重启——也是不拔电池就从一次跳闸的过载恢复的办法。不在驾驶中的机器人就地重启。Start 再让它站起来 |
| **Select**，按住 4 秒 | 断电（`robot.shutdown`）：同样的坐下和休息姿势，然后板子关闭。之后的发布不会再做什么 |

**一次一个动作。** 一个踢球、拾取或坐下在另一个运行期间会被拒绝，而坐着的机器人只接受 A（站起来）。在它坐着时停止策略让它保持坐着；切断扭矩或重启舵机则把座位忘得干干净净。

方向键是选择而不是切换：一次按下落到它箭头所指的模式，无论你之前在哪。离开一个模式会放回它移动过的东西——身体 + 头之后身体回到标称，而当你回到普通移动模式时头重新居中。Select 的一次短按对机器人什么都不做；日志会说要按住多久。

**用手柄本身来驱动头。** Pro Controller 带一个 IMU，配合

```bash
sudo robotctl configure      # Controller-IMU head control → enabled
```

头 + 移动（方向键右）把头交给手柄——倾斜它头就倾斜，转动它头就转动——同时摇杆继续像移动模式那样驱动身体。再按一次方向键右，手柄就**从手柄此刻的位置**驱动头：它的偏航只是陀螺仪的话、会漂移，而每次按下都重新居中，就是你在没有磁力计时战胜漂移的办法。同一小节里的 `gain` 是每手柄弧度对应的头弧度，默认 1。在 Xbox 手柄上，或开关关闭时，该模式下由右摇杆摆头。`padd` 在一秒内感知更改；无需重启。

没有停止按键：松开摇杆机器人就站住，如果 `padd` 死了，`robotd` 的 deadman 会停住它。**驾驶时手柄掉线**，一秒后机器人会问它去哪了（一声上扬的嘎嘎叫），如果当时站着就坐下——踢球和拾取会让它们先做完。更短的掉线是无声的。手柄回来时它用一声嘎嘎叫跟你打招呼；A 让机器人站起来。

**独自待着时，机器人会环顾四周。** 在策略运行、没要求动作、也没人摆头 2 秒后，它每过一两秒就会瞥一眼——大多是左右、一点点上下，脖子前倾、每次看时喙微微张开，偶尔带一个好奇的歪头——而在它保持一次注视时，视线还会在目标周围轻轻游移。它轻微漂移并呼吸（一毫米的高度起伏，仅站立时）。什么都没发生的时间越长，它越平静，过一会儿还会偶尔打哈欠——喙张开、头后仰、无声地。它淡入淡出，任何摇杆或头的命令都会把头要回来。在方向键的头模式里手柄保留头，所以这只在移动模式发生。`journalctl -fu robotd` 会记录每一段开始与停止。每个数字都在 `[idle]` 里——`sudo robotctl configure`，然后重启 `robotd`。

在滚轮机器人上（`robotd.toml` 里 `mode = "roller"`）摇杆自动采用滚轮塑形——非对称推/刹、无横移——而 B 触发下蹲。其他技能一并跟随：坐下和踢球在轮子上也能用，如原型所示。

**`robot.setMode` 在两者之间切换**，用于你刚给鸭子装上轮子或拆下轮子的时候：

```
duckctl call robot.setMode '{"mode":"roller"}'
```

机器人叫一声表示行走、两声表示滚轮，回到家姿势，在那里加载该模式的策略并再次驱动——几秒钟，全程扭矩开启、无需重启，手柄的摇杆塑形在一秒内跟上。`robotd.toml` 不会被触碰，所以重启后回到配置的模式；用 `robotctl configure`（或 `[policy] mode`）让它固定下来。

`pad status` 分别回答两个问题，因为一个已连接的手柄和一个死掉的驱动从外面看一模一样：

```
pad     Xbox Wireless Controller 78:86:2E:BB:13:28  connected
padd    active — driving whatever pad connects
```

要用非默认速度驾驶，在 `[pad_drive]` 里设置——`vx_max`/`vx_min` 前进后退、`vy_max`/`vy_min` 横移、`vyaw_max`/`vyaw_min` 转向，每个 `_min` 为负——用 `sudo robotctl configure`。`padd` 在一秒内感知更改，无需重启。

当怀疑的对象是链路本身时，实时观察它——`robotctl monitor`，然后 `p`。没有机器人时这也能用：在一块舵机未供电或 `robotd` 停着的板子上，monitor 会打开手柄块而不是拒绝。要得到一个跨时间窗的结论，从本仓库的一个克隆把测量脚本拷过去：

```
scp scripts/pad-link-test.sh microduck@<board>:/tmp/
```

掉线已经在 `padd` 的日志里——不需要手柄，而且立即回答：

```
sudo sh /tmp/pad-link-test.sh --history
```

或者现在就测量，全程两分钟保持摇杆在动：

```
sudo sh /tmp/pad-link-test.sh
```

它把掉线对内核为每次给出的原因计数，并计时手柄输入报告之间的间隔——那是 `padd` 看不见的失败，链路保持在线、机器人却走在一条过期的命令上。[`pair-a-gamepad.md`](pair-a-gamepad.md#驾驶时掉线) 会读这些数字。

当两块板子对同一只手柄表现不同时，差异在它下面的栈里：

```
scp scripts/pad-stack-report.sh microduck@<board>:/tmp/
```

```
sudo sh /tmp/pad-stack-report.sh
```

内核、BlueZ、控制器固件、LE 或 BR/EDR，以及手柄自身的固件版本——打印并保存到 `/tmp/pad-stack-<host>-<when>.log`。`--fingerprint` 只打印两块板子之间必须一致的值，供 `diff` 用。[`pair-a-gamepad.md`](pair-a-gamepad.md#这块板子和那块跑的是同一个栈吗) 有对比。

### 声音

```
robotctl quack
```

区分鸭子们最响亮的方式：每台机器人的声音库都从它的 SoC 序列号生成（`sounds ensure-bank`，每次发布安装都会运行），所以应答你的——用一种只属于它自己的声音——正是你 SSH 进去的那一台。一台没声音的机器人——音频关闭，或没有声音库——会这么说而不是打印 🦆，所以沉默总是意味着连错了鸭子。机器人在 `robotd` 起来时也会打招呼，断电前啄一下告别，而且——如果你让它这么做——当麦克风听到有人挠它的头时会咕咕叫。那一个在两种模式下都默认关闭（`audio.pet_detect = true` 打开它；分类器随发布提供）：常开的版本对每一次偶然的擦碰都咕咕叫，让人腻烦。开机问候有它自己的开关，给那些整天重启守护进程的人：

```
sudo robotctl configure
```

设 `audio.greet = false` 并接受保存时提议的重启。那会静音那一声嘎嘎叫，而留下触发器和麦克风不动，这是 `audio.enabled = false` 做不到的。音频硬件 bring-up——编解码驱动、设备树覆盖、混音器——是 `setup-board.sh` 的 audio 小节，每块板子一次。

要试听一个声音或手工重新生成声音库，发布里带了生成器：

```
/opt/robot/daemon/current/bin/sounds show
sudo /opt/robot/daemon/current/bin/sounds ensure-bank --force
```

`sounds theremin` 试听*实时*合成器——特雷门琴演奏用的那种声音，由一次脚本化的手扫、以 ToF 自身的帧率驱动。`--out sweep.wav` 把它写出来而不是播放，这样没有机器人在面前你也能听到声音的变化。

### 鸭子合唱（chorale）

```
robotctl chorale
```

房间里的两只鸭子一起唱一首四声部的曲子；更多的鸭子加入它们发现已经在进行的合唱。一直运行到 Ctrl-C。`--off` 停下一只。

**默认关闭**——`robotd.toml` 里的 `[chorale] accept`，而且必须在每只应参加的鸭子上设置。合唱会动嘴和头，所以一只因为另一只鸭子走进来而开始做动作的鸭子，是在做没人要求的动作。关闭也意味着*隐形*：一只没有选择加入的鸭子什么也不发到空中，而不是礼貌地谢绝。

它的原理，按问题出现的顺序：

- **没有谁负责。** 两只鸭子看到同样的信标，id 较低的那只担任指挥，所以没有可输的选举，也没有必须到达的消息。
- **没有共享时钟。** 板子没有 NTP，也没有 RTC 协议，所以指挥的节拍计数器*就是*时基：它每个节拍在 BLE 广播里把一个字节加一，新值的到达就是强拍。跟随着在约 25 个节拍上平均相位，这把无线电的抖动收进一个合奏所需的 ±20 ms 内。
- **声部是推演出来的，不是分配的。** 最低的鸭子唱低音。指挥广播名册，每个人在它上面重放同样的座次——这正是两只鸭子各自只能看到房间里不同子集时，不会唱同一句的原因。
- **加入不会改变任何人的声部。** 一只到场的鸭子拿走空闲的声部。一只离开的鸭子保留它的*座位*——它的那句只是没人唱了，就像有人中途退场的合唱团——因为在中途重新排座是唯一值得避免的事。

一旦声部确定，读数就报出它，所以一只鸭子最终唱了什么会留在滚动缓冲里：

```
listening for other ducks — Ctrl-C to stop
  singing tenor    with 3 voices
  tenor    bar   12  beat  45.2  3 voices
```

指挥为每次演出选曲子，而一次演出会*结束*——最后一个音之后大家都回去聆听、重新安定，喘口气后再唱别的。`robotctl chorale --piece 2` 固定这只机器人在**它指挥时**所选的东西（跟随者唱信标点名的，所以要在它身上设置）。

### 用鸭子演奏（ToF 特雷门琴）

```
robotctl theremin
```

头上的深度传感器变成一件乐器：喙前的手就是音高——越近越高——嘴随音符张开，音域顶端张到最大。一直运行到 Ctrl-C，并在退出的路上把乐器放下。`--off` 放下一件某个客户端留下没放下的乐器。

**默认关闭**——`robotd.toml` 里的 `[theremin] enabled`，每只鸭子各自设置，如上面的合唱。`robotctl configure` 是设置它的办法，并会提议让 `robotd` 重启来生效；在那之前 `robotctl theremin` 会拒绝并点名这个键。没有别的东西随它一起关掉：`tofd` 照常运行，所以下面的深度网格在一只从未奏过一个音符的鸭子上也能用。

一个没有花活的显式模式：它开启期间，可弹奏带内最近的返回就是手。把鸭子指向开阔空间，它就沉默；指向 40 厘米外的一面墙，它就弹一个稳定的音。它坐着、站着或走着都能演奏——嘴不属于任何策略。

读数最后一列是**传感器对那一帧说了什么**，而它正是每个「它为什么不弹了」的答案：

```
  0.34 m    438.1 Hz   60% ██████    14 usable · 255:38 4*:9 5*:5 1:12
```

有多少个区域携带机器人相信的状态，然后按 ST 状态码计数、在可信的那些上打 `*`。音符前的一个 `~` 表示它是一个桥接传感器掉线的*保持*音符，而不是此刻刚测得的东西。

那一列存在，是因为它在一分钟内就会发现的一个 bug：ST 把 5 和 9 记录为「range valid」，而一个只相信这两者的构建**在约 30 厘米处开始看不见手**——超过那里，一只移动的手会以 4 或 13 返回（*consistency failed*，sigma 过高），携带着一个对音高来说完全够用的距离。如果够得着的地方太短，把码加到 `robotd.toml` 的 `[theremin] statuses`；如果它对着空气弹幽灵音符，就移除一些。`hold_ms` 是防抖：它骑过一个闪烁的区域。

注意 `robotctl monitor` 的 ToF 网格比特雷门琴更严格——它把 5/9 之外的一切标为 `x`、*无法测量*。一个满是 `x` 的网格不意味着传感器坏了；它意味着它对自己确实拥有的数字持悲观态度。

### ToF 传感器（`tofd`）

来自头传感器的 8×8 深度矩阵。`robotctl monitor`，然后 **`t`**：

```
┌ tof VL53L8CX · 15 Hz · 8×8 · 48/64 ranged · 0.12–3.54 m ─────────────┐
│ 0.12 0.15    x 1.44 1.86    · 2.70 3.12                              │
└ · nothing in range · x could not measure · near→far ── seq 412 · 6 ms ┘
```

距离以米计，从近暖色到远冷色着色。这两个记号要紧：`·` 是*测得、范围内没有东西*——自由空间，这是信息——而 `x` 是*无法测量*，对外面有什么什么都没说。一个把两者都显示成空白的网格会藏起这个差别。

这是传感器自己的坐标系，不是机器人的：在运动学就位之前没有重投影，这也让这个块成为检查一个安装角度的正确位置。

`tofd` 拥有传感器，没有别的进程读总线。它是一个普通服务——`sudo systemctl stop tofd` 是安全的，没有东西依赖它，而 `monitor` 会说「no depth stream」然后继续。它区分三种情形，因为它们需要不同的修法：

| 该块说 | 含义 |
| --- | --- |
| `connecting to tofd…` / `no depth stream` | 守护进程没在运行 |
| `no sensor: …` | `tofd` 起来了；总线上没有应答（大多数鸭子） |
| `waiting for the first frame…` | 一个传感器在测距；它的第一次扫描约 66 ms 后到 |

要手工看总线上有什么，或不用终端 UI 观察帧：

```
sudo i2cdetect -y -r 3
journalctl -u tofd -b
```

传感器与编解码器共享 I²C 总线，所以 `setup-board.sh` 的 audio 小节已经给总线本身做好了配置；ToF 步骤只添加稳定的 `/dev/i2c-pihat` 名字。两代传感器都支持——VL53L5CX 和 VL53L8CX 在板上可互换，守护进程从一个 ID 读取里选择驱动。

#### 头部 IMU（`head_imu.stream`）

来自头部 IMU 的陀螺仪、加速度和一个姿态。哪块芯片、哪个守护进程提供这个流，取决于板子：

- **zero3**：`tofd` 用 Madgwick 融合读取 HAT 的 BMI088，**默认关闭**。100 Hz 时约花一个核心的 4%。`tofd --imu` 在不碰文件的情况下读一个会话，`--imu-hz` 用速率换开销。
- **beta**：`robotd` 以 60 Hz 读取面部的 LSM6DSV16X，**默认开启**。芯片自己做姿态融合并批量采样，所以代价是每秒 15 次突发读取，CPU 上无融合。

`robotd.toml` 里的 `[head_imu] enabled` 覆盖任一默认值；`robotctl configure` 会提议重启读取它的那个守护进程。问另一个守护进程，答案会点出正确的那一个；它关闭时，答案点出的是键，而不是一个未装传感器给出的沉默。

这些都不触及深度：无论怎样 ToF 都在测距，所以上面的网格在一只 IMU 从未打开过的鸭子上也能用。

### 面部 LED

```
robotctl led list
robotctl led set rgb:red on
robotctl led set all off
robotctl led blink cam:red
robotctl led trigger green1:green heartbeat
robotctl flashlight on --color cyan
robotctl flashlight toggle
```

beta 板的面部有十二个 LED（`face:<place>:<colour>`，在一个 GPIO 扩展器上），`led` 通过内核的 LED class 切换它们——名字带或不带 `face:` 前缀，或 `all`。没有面部板的板子会说明。每个 LED 有一个拥有它的守护进程，它显示什么见 [`architecture.md` §3.2](../design/architecture.md#32-the-face-leds)；`led set` 只覆盖到该守护进程下一次有变化要显示为止，所以这是台架工具。`flashlight` 询问拥有那个 LED 的 `robotd`，这正是手柄 Home 键做的事。非默认的 500/500 ms 闪烁速率（`--on-ms`、`--off-ms`）需要 `sudo`。

### 摄像头（`mediad`）

来自头部摄像头的一帧，显示在终端里。`robotctl monitor`，然后 **`c`**：

```
camera 1280×720 · mount 90° · answered in 41 ms                    0.5 s ago
```

画面按每个字符单元两个像素绘制，无论安装角多少都方向正确。它很小——这台机器人转四分之一圈之后画面是竖的，约 16 像素宽——而这对一个数字说不出来的东西足够了：头真正指向哪、房间光照是否足以检测到任何东西、镜头是否花了或有根拇指挡在上面。

**该块关闭时什么都不抓取。** 一帧是 1.84 MiB，`mediad` 只因有人要它才从采集分支拷贝，所以一个带着关闭的块一直开着的 monitor 不花摄像头任何开销。打开时它每秒问两次，而 `answered in 41 ms` 是摄像头自己的存活信号：请求等待*下一次*采集，所以一个健康的 30 fps 摄像头约一个帧周期内应答，一个停了的则等超时并失败。关闭块会忘掉画面，所以重新打开显示的是新的一帧，而不是十分钟前的房间。

| 该块说 | 含义 |
| --- | --- |
| `asking mediad for a frame…` | 第一次请求还在途中 |
| `no picture: connection refused` | `mediad` 没在运行 |
| `no picture: no frame arrived within the capture timeout` | `mediad` 起来了；摄像头没在生产画面 |

要全尺寸画面而不是缩略图：

```
robotctl frame --output /tmp/frame.uyvy
```

一帧原始 UYVY，几何和安装角打印到 stderr。`mediad` 在 `:8080` 上服务的控制台在 `GET /frame` 提供同一帧的 PNG，已经转正。

### WiFi（`configd`）

```
robotctl net status
```

```
robotctl net scan
```

```
sudo robotctl net connect <ssid> --psk <passphrase>
```

```
sudo robotctl net connect <ssid> --psk-stdin
```

```
sudo robotctl net forget <ssid>
```

`--psk-stdin` 把口令挡在 `ps` 之外，后者会在命令存续期间把 `--psk` 参数显示给机器上的每个用户。在共享的机器上优先用它。

加入一个网络**会让机器人断开它当前的网络**，所以一个走 WiFi 的 ssh 会话会掉线。那是操作在生效。扫描要几秒——它等待无线电扫完，而不是返回上一次扫描的结果。

### Hugging Face 账户（`updaterd`）

让机器人登录，是将来让你从它自己的网络之外访问它的前提。目前还没有别的需要它。

```
sudo robotctl account login
```

那会打印一个码。在任何设备上打开 <https://hf.co/oauth/device>，输入这个码，命令就完成：

```
Open https://hf.co/oauth/device and enter this code:

    A6MY-0314

Waiting for approval…
Signed in as PierreRouanet.
```

```
robotctl account status
```

```
sudo robotctl account logout
```

等待时按 Ctrl-C 不损失任何东西——机器人继续轮询，`account status` 说明是否被批准。码五分钟有效；过了就再跑一次 `login`。

一台已经登录的机器人会拒绝，并报出账户名：

```
sudo robotctl account login --force
```

同一个标志也放弃一个仍在等待批准的码——就是有人开了登录然后走开的情形。旧码停止工作；之后再批准它什么都不做。

令牌三十天有效，机器人自己续期。关掉超过这么久的机器人回来后需要再次 `login`，`account status` 会原话说明。

### 身份与电源（`configd`）

```
robotctl system info
```

```
robotctl system pin
```

```
sudo robotctl system set-name <name>
```

```
sudo robotctl system set-pin <six-digits>
```

```
sudo robotctl system reboot
```

开箱时机器人自称 `duck-` 加四个由它自身序列号派生的字符，所以从同一镜像刷出来的两块板子在手机的蓝牙列表里仍显得不同。改名几秒内在蓝牙上生效——无需重启——但手机要重新扫描才能看到。

PIN 是手机通过蓝牙认证用的。出厂默认是 `000000`，它认证任何读过这个仓库的人。

### 更新（`updaterd`）

```
robotctl update status
```

```
robotctl update check daemon
```

```
sudo robotctl update apply daemon
```

```
sudo robotctl update rollback daemon
```

```
robotctl update log
```

```
robotctl update show
```

```
robotctl update watch
```

`log` 列出尝试，每行一次，新的在前；第一列是运行号。`show` 取其中一个号——或不取，即最近一次——打印那次运行做的所有事，然后是同一时间窗的日志：

```
run 42 · daemon · 2025-08-27 13:06:40 UTC
  applied 0.1.3 → 0.1.4
  asked for latest, from github.com/pollen-robotics/microduck, onto 0.1.3
  requested by uid=1000 gid=1000 pid=2317

  13:06:41      +1s  manifest     0.1.4 · 184.2 MB · sha256 3f9a1c2b… · signed by release.pub · rev 88efc03
  13:06:41           downloading
  13:07:58   +1m17s  note         downloaded 184.2 MB to /opt/robot/daemon/staging/0.1.4/dl/…
  13:08:02      +4s  note         hash matches; signature verifies against release.pub
  13:08:20     +18s  pre-hook
  13:10:12   +1m52s  hook         hooks/preinstall
                                 │ onnxruntime 1.20.1 already present
                                 │ gstreamer: h264 encode ok
  13:10:12           swapping     0.1.3 → 0.1.4
  13:10:14      +1s  unit         robotd: restart
  13:10:23      +8s  health       the robot reported healthy
  13:10:24           ended        applied 0.1.3 → 0.1.4

  ── journal · 2025-08-27 13:06:40 to 2025-08-27 13:11:24 UTC ──
```

时间是 UTC，下面的日志也是。`+` 列是自上一行以来的间隔，这就是你找到那两分钟的办法。

读日志需要 `robot` 组没有的权限，所以除非你是 root，后半段会是空的。发生时它会打印 `journalctl` 那行；在它前面加 `sudo` 是修法。`--no-journal` 不尝试就打印那行，`--json` 只给转录。

组件是 `daemon`——一个覆盖所有二进制的组件。`apply daemon` 安装稳定通道提供的版本；分支构建和候选发布需要 [`cheatsheet-dev.md`](cheatsheet-dev.md)。

### 不下载地切换

切换到板子上已经解包好的某个版本。不涉及网络：

```
sudo robotctl update select daemon 0.1.4
```

```
sudo robotctl update rollback daemon
```

```
sudo robotctl update reset-to-golden daemon
```

`select` 和 `rollback` 在本地发布之间切换；两者都需要网络之外的某个解包好的前身。`reset-to-golden` 是回到一个已知完好构建的锚，由 provisioning 设置。它拒绝、并说明原因，当没有配置 golden 或 `current` 已经是 golden 时——如果守护进程在 golden 自身上都失败，回滚就不是答案，日志才是：

```
journalctl -b -u robotd -u updaterd -u btd -u configd
```

### 机器人可能已经这么做了

每次启动三分钟后，一个定时器问这个发布是否把它的守护进程带起来了，没有就回退到 golden。所以一台自己重启了、且运行着比你安装的更旧发布的机器人，很可能已经自救过。它做了什么：

```
robotctl update log
```

该条目读作一次回滚，原因里点出失败的守护进程。要看决策被做出而不是其结果：

```
journalctl -b -u robot-boot-check
```

```
sudo robot-boot-check --dry-run
```

它只行动一次。第一次仍在记录中时，第二次救援会被拒绝——`updaterd` 下次启动时清掉它，所以被拒绝意味着守护进程在 golden 上也没起来，答案是日志而不是再重启一次。过了它，如果你已经读过日志并做了决定：

```
sudo robot-rescue --force --reboot
```

### 三件容易搞错的事

**`rollback` 需要一个前身，但一次更新会创建一个。** 一块刚 provisioning 的板子恰好有一个发布，所以那时的 `rollback` 没有更旧的可回、并会说明。自动回滚*不*受影响：应用一个发布会把它解包在当前那个旁边，只有到那时才移动 `current`，所以到健康门控运行时已经有两个，而你来自的那个发布就是目标。`rollback_target` 选日志尚未记录为坏的、`current` 之下的最高已安装版本——所以只有一份发布的板子从它第一次更新的那一刻起就受到完全保护。

唯一真正不受保护的安装是 bootstrap 本身，按定义它之前什么都没有。`golden` 能覆盖它，而它在 1.0.0 存在之前被刻意留空——所以 `reset-to-golden` 会诚实地报告没有配置，而不是做令人意外的事。

**`version` 显示每个组件的实时发布，而不是发布仓库。** 无论解包了多少个，它都不会列出两个版本。直接问仓库：

```
ls -l /opt/robot/daemon/releases/ /opt/robot/daemon/current
```

**`apply --version` 需要该发布仍存在于上游；`select` 不需要。** 携带已知坏构建的发布会从 GitHub 删除，所以 `apply --version 0.1.3` 故意失败，而 `select 0.1.3` 在一块已经解包它的板子上仍可用。这种不对称是刻意的：新板子无法获得一个坏发布，而有一份坏发布的板子保留它的逃生舱。

### 无网络安装

侧载、工厂安装，或救回一台 `updaterd` 太旧、无法接受那个修复「太旧」的发布的板子。见 [`install-dev.md`](install-dev.md)——它是 `updaterd install --from`，而 `--force` 变体有些条件值得在用之前读一读。

### 日志

```
robotctl logs robotd
```

```
robotctl logs robotd -n 300 -b -1
```

`-b -1` 是这次启动之前的那次。无需 `sudo`：`configd` 读日志。它接受 `robotd`、`btd`、`configd`、`updaterd`、`padd`、`mediad`、`tofd`、`bluetooth` 或 `NetworkManager`。

要跟踪或搜索，用 `journalctl`：

```
sudo journalctl -u configd -b --no-pager | tail -40
```

```
journalctl -u btd -f
```

换成 `robotd` 或 `updaterd`。`-f` 跟踪；`-b` 仅本次启动。

启动行携带版本、git 修订版以及进程从哪个发布目录启动，在 `warn` 级别，所以它在任何日志级别下都存活。

更新历史刻意与日志分开——每条在 `/var/lib/robot/updater/` 下 `fsync`——这样它能挺过一台日志易失的机器人：

```
robotctl update log
```

最近二十次运行在那里也保留完整转录，在 `runs/` 下，按发生顺序写：

```
robotctl update show 42
```

两者都活得比交换、回滚和一次断电久，而这块板子上的日志不行——`/var/log` 是 zram。如果坏的是 `robotctl` 本身，这些文件是换行分隔的 JSON，用 `cat` 就能读：

```
sudo cat /var/lib/robot/updater/runs/000042.jsonl
```

### Tab 补全

`install.sh` 把它设在 `/usr/share/bash-completion/completions/`，作为一个向二进制询问它自身补全项的加载器——于是它们跟随已安装的发布，而不是在更新新增命令后过期。它放在那里而不是 `/etc/bash_completion.d/`，是为了在你第一次敲 `robotctl<TAB>` 时被读取，而不是每次登录都读；更新会移走旧副本。对于它没覆盖的 shell，或一个你直接跑出 `target/` 的构建：

```
eval "$(robotctl completions bash)"
```

`zsh`、`fish`、`elvish` 和 `powershell` 可替换 `bash`。
