# 配对一个游戏手柄

每只手柄一次。之后，`padd.service` 从启动起就驱动任何连接上的手柄——没什么要启动的，也没有东西会随你的 ssh 会话死掉。局域网另一台机器上的手柄可以改走 UDP 驾驶机器人——[`udp-pad.md`](udp-pad.md)。

## 让手柄进入配对模式

在 **Xbox** 控制器上这是两下按，而第二下是容易出错的那个：

1. **短**按 Xbox 键开机。别按住那个键——按住它会关掉控制器。
2. 按顶边、USB-C 口旁边的小 **Sync** 键，直到 Xbox 灯**快速闪烁**。慢闪意味着它开机了但没在配对。

在 **DualSense** 上：同时按住 Create 和 PS，直到灯条闪烁。

在 **Pro Controller** 上——那些无品牌的 Switch 风格手柄，bluetoothctl 把它们列为 `Pro Controller`——按住顶边、USB-C 口旁边的小 **Sync** 键，直到玩家灯来回扫动。它是经典蓝牙（BR/EDR）手柄，Xbox 的是 LE，机器人配两者的顺序相反；`pad pair` 自己选对的，而如果你不得不手工做，[手工配对 Pro Controller](#手工配对-pro-controller) 说那个顺序是什么。

## 配对它

```bash
sudo robotctl pad pair
```

```
looking for a gamepad in pairing mode — on an Xbox pad, press the small Sync button on the
top edge (not the Xbox button, which switches it off)
paired  Xbox Wireless Controller 78:86:2E:BB:13:28
padd is driving from it now.
```

不需要 MAC 地址：机器人找一只配对模式里的手柄并拿走它找到的那只。手柄既被*信任*也被配对，这正是没人登录时它能在重启后自己重连的原因。

如果两只都在配对模式，它拒绝而不是猜，并打印它们的地址。报出一个名字也是配对一个机器人不认作游戏手柄的硬件的办法：

```bash
sudo robotctl pad pair 78:86:2E:BB:13:28
```

**第二只手柄不需要遗忘。** 一只已配对的手柄在范围内、在每次扫描里，所以机器人偏好配对模式里的那只；之后两只都保持配对，`padd` 驱动连接上的那只。代价是：在配对模式里没新东西时重跑，会等完整个搜索窗口才报告你已有的那只——如果你只是修复信任，用 `--timeout 5`。

### 用 NFC 标签

在一台插着 NFC 读卡器（USB 上的 CLRC663 板，`/dev/ttyACM0`）的机器人上，一个携带手柄地址的标签不用输入任何东西就能做同样的事：让手柄进入配对模式，把标签贴到读卡器上。机器人读到标签时会嘎嘎叫，手柄配对好时会问候——一个不同的声音，常常是双「wak-wak」；一次拒绝没有第二声。`nfcd.service` 总在运行，读卡器插上后三十秒内捡起它。

- **一只已连接的手柄胜出。** 如果一只在驾驶，一次触碰什么都不做，所以一个擦过机器人的标签绝不会把配对交给别人的手柄。
- **每次触碰一次尝试。** 如果没配上，抬起标签再碰。
- 标签持有一个带地址的文本或 URI 记录——`98:B6:E9:28:06:09`，用 `-` 或没有分隔符也行。从一台插着读卡器的笔记本，winnie 的工具写一个：`ntag write --text 98:B6:E9:28:06:09`。

它做了什么在 `journalctl -u nfcd -b` 里。

## 检查它

```bash
robotctl pad status
```

```
pad     Xbox Wireless Controller 78:86:2E:BB:13:28  connected
padd    active — driving whatever pad connects
```

两行，因为它们分开失败：一只连着的手柄带一个死掉的驱动，看起来和一台正常机器人无视你一模一样。

Pro Controller 还带一个六轴 IMU，内核把它暴露为驱动设备旁边的第二个输入设备。`robotctl monitor` 用 `p` 打开手柄块，把它显示为一个只有手柄有时才出现的面板：一个像真手柄那样摆姿的线框手柄，俯仰和横滚来自重力、偏航只来自陀螺仪（它漂移——手柄上没有任何东西观测朝向），以及陀螺仪的静止偏置，仿品需要先学会它画面才会停止自己转动。把手柄放下半秒，面板就说 `settled`。monitor 没开时这个流不花任何东西：只有有人订阅那个 tap 时 `padd` 才读 IMU 节点——或当它用它摆头时，见下。

同一个姿态能驱动机器人的头。`sudo robotctl configure`，小节 *Controller-IMU head control*，`enabled`：头 + 移动模式（方向键右）然后把手柄的倾斜交给头，同时摇杆继续驾驶，再按一次方向键右就重居到手柄当前姿态——[速查表](cheatsheet.md#gamepad-configd) 有整个映射。monitor 里的画面和头用同一个滤波器，所以画出的手柄指向哪、头就去哪。

`paired but NOT trusted` 是值得知道的状态。它现在能用、重启后不重连，因为批准……

## 遗忘一只

```bash
sudo robotctl pad forget 78:86:2E:BB:13:28
```

这移除**配对中机器人那一半**，那是一个机器人能移除的全部。手柄保留它自己的那一半，所以再配对它需要它回到配对模式——否则它带着这台机器人已没有的密钥到来，配对被拒绝。

一只 Xbox 手柄持**一个**主机配对，而一次半完成的尝试让它持有一个这块板子已没有的密钥——它以一块坏板子完全一样的方式失败。如果配对一直失败，把手柄先配到一台笔记本一次并在那里移除它；那消耗并释放它的配对槽，而只让它进配对模式并不可靠地做到这点。

## 当手柄完全配不上对时

在 aic8800 无线电上，`btd` 正在广播时手柄无法形成**新的**配对。用 `--pause-btd-on-pair` 重新 provision 这样一块板子：

```bash
./scripts/provision-board.sh --pause-btd-on-pair pierre@192.168.1.42
```

那在 `/var/lib/robot/weird-ble` 留一个标记、别的什么都不改。在一块有那个标记的板子上，`sudo robotctl pad pair` 自己处理剩下的——它停 `btd`、给适配器做电源循环、配对、再启动 `btd`。它边走边说明。现有配对不受这些任何影响，所以一只配好对的手柄在一切运行时连接并驾驶。

有些 unit 即使在 `btd` 暂停时，也根本无法在 BlueZ 默认的 `Privacy = off` 下配对。那些要 `--weird-ble`，它隐含暂停并设 `Privacy = device`。

**先试暂停。** 在一块只需要暂停的板子上，`Privacy = device` 产生的失败比完全没有标志更糟：手柄配对后抖动 `Encryption Change: PIN or Key Missing (0x06)`，从不创建输入设备。如果你看到那个，去掉 `--weird-ble`、保留暂停——[`install-dev.md`](install-dev.md#三种配置以及如何分辨你是哪一种) 有表格和在两者之间移动板子的命令。

在这样的板子上手工配对需要同样的两步：

```bash
sudo systemctl stop btd
```

```bash
sudo bluetoothctl power off && sudo bluetoothctl power on
```

配对，然后：

```bash
sudo systemctl start btd
```

电源循环不是可选的。停掉 `btd` 会留下它的广播和它的配对代理给控制器的 IO 能力，手柄仍拒绝配对——一次重启有同样效果，这正是它被发现的方式。绝不要改用 `systemctl restart bluetooth`：在这块板子上那会直到重启前完全没有适配器。

`--weird-ble` 在 [`install-dev.md`](install-dev.md) 里是默认，因为约一半这类板子需要它、而没有什么可测量的东西说哪些。但一块不需要它的板子不该带着它——那个标志每次配对都花一次 `btd` 停止和一次适配器电源循环——所以那一页也说了怎么检查和怎么去掉它。

两者都是对 aic8800 无线电的绕行，不是设计的属性。无线电走它们就走。

### 经典手柄与 bluetoothd 的 CPU

一只 Pro Controller 在每个包里流式传输 IMU 采样，每秒约 200 个包，无论有没有人碰它。在 BlueZ 默认 `UserspaceHID=true` 下，每个都经 uhid 由 bluetoothd 中继，那在手柄闲置时花掉 graphite 上核心的 16%。`scripts/setup-board.sh` 在每块板上把 `/etc/bluetooth/input.conf` 设为 `UserspaceHID=false`，把通道交给内核的 `hidp`、把 bluetoothd 移出数据路径——之后测得 0.0%，同样的驱动、同样的输入节点、同样的配对。它在下次启动时生效。像 Xbox 这样的 LE 手柄不受该设置影响：HID over GATT 从不经过 `input.conf`。一块在此之前 provision 的板子重跑脚本得到它：

```bash
sudo sh scripts/setup-board.sh && sudo reboot
```

## 手工配对 Pro Controller

只在 `pad pair` 不可用时。顺序要紧，且与 Xbox 的**相反**：

```bash
bluetoothctl pair 98:B6:E9:28:06:09
```

```bash
bluetoothctl connect 98:B6:E9:28:06:09
```

```bash
bluetoothctl trust 98:B6:E9:28:06:09
```

## 当它驾驶时掉线

`robotctl monitor` 里的 `p` 块是来自 `padd` 正从中驾驶的手柄的原始 evdev 流——每份报告，由内核打时间戳：

```
┌ pad Xbox Wireless Controller · /dev/input/event5 · 78:86:2e:bb:13:28 ─────────────┐
│ cadence  124/s while driving · last 8 ms ago · worst 84 ms · over 100 ms 0 · …    │
│ gap ms                                       ▁▁▁▁▂▁▁▁▁▃▁▁▁▁▁▁▂▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁ │
│ X     ····│██··   -8734 Y     ····│····       0 Z     ·········       0 …         │
│ held     BTN_START                                                                │
└ 4213 reports · gap ≤100 ms ───────────────────────────────── reports intact ──────┘
```

动摇杆、条跟着动。轨迹要展示的是上面那行：每份报告一根条，所以一次卡顿是一个尖峰，而一次已恢复的仍留在屏幕上。满高是 100 ms——一个驾驶者开始感到它的点——超过 500 ms 时 `robotd` 已把速度清零。

**机器人上没有别的东西能显示这个。** `padd` 以 50 Hz 重发最后的摇杆值，所以一条已停止投递的无线电在每处下游看起来仍是一个活驾驶者：`robot.state` 携带新鲜意图、deadman 从不触发、机器人继续走在一条没人下达的命令上。当这一块拉平时，它上面那块里的 `asked` 列看起来仍完美。

一只静止的手柄什么都不发，所以沉默只在你在驾驶时才是证据。该块在五秒过后说 `the sticks are still` 而不是指责链路，并分开计数那些时段。

然后，要一个跨时间窗的结论而不是实时画面，从本仓库的克隆把测量拷到板上：

```bash
scp scripts/pad-link-test.sh microduck@<board>:/tmp/
```

已经发生了什么，来自 `padd` 的日志——不需要手柄、立即回答：

```bash
sudo sh /tmp/pad-link-test.sh --history
```

要现在测链路，手柄开着、`padd` 运行。**全程两分钟保持摇杆在动**：静止的手柄什么都不发，而沉默读起来和一条卡死的链路一模一样。

```bash
sudo sh /tmp/pad-link-test.sh
```

它计数掉线，以及手柄连接时输入报告之间的间隔。一个超过 500 ms 的间隔就是机器人在停——`robotd` 在那里把速度清零。每次掉线后面跟内核的原因：`0x08` 是监管超时，意味着距离或干扰，`0x13` 意味着有人把手柄关了。

放下手柄不是卡顿，也不计为一次——但那是测量学不到东西的时间，所以报告说这个窗口里你实际驾驶了多少，并拒绝评判一条它几乎没看到的链路。

机器人看着你的时候走远，就是你找范围的办法。

## 这块板子和那块跑的是同一个栈吗

一只在一台机器人上卡顿、在它的孪生上不卡的手柄，通常不是手柄。两块相隔几周构建的板子跑不同的内核、不同的 BlueZ、不同的控制器固件，手柄也在不同手柄固件上——而这些没有任何一样在 `pad status` 里可见。

从本仓库的克隆把报告拷到每块板上：

```bash
scp scripts/pad-stack-report.sh microduck@<board>:/tmp/
```

```bash
sudo sh /tmp/pad-stack-report.sh
```

它打印整个栈并把同样的文本存到 `/tmp/pad-stack-<host>-<when>.log`：内核、BlueZ、适配器的 HCI 版本、是哪块无线电以及内核在启动时为它加载的固件、什么在承载 HID、配对持有哪些密钥、当前使用的传输、以及手柄自己的固件修订版。它无需 root 运行，对需要 root 的三样东西说 `unreadable`。

比较两块板子，各问它只取必须匹配的值：

```bash
ssh microduck@<board-a> sudo sh /tmp/pad-stack-report.sh --fingerprint > /tmp/a.fp
```

```bash
ssh microduck@<board-b> sudo sh /tmp/pad-stack-report.sh --fingerprint > /tmp/b.fp
```

```bash
diff /tmp/a.fp /tmp/b.fp
```

没输出意味着同一个栈。指纹不带时间戳和地址，所以 `diff` 打印的任何东西都是真实差异。

读其余之前先读两行。`transport` 在迄今试过的每只手柄上都是 `LE`，而一块说 `BR/EDR` 的板子正把手柄放进内核的经典 HID 路径而不是 BlueZ 的——不同的驱动、不同的按键编号。`input` 是 SDL 和 `gilrs` 哈希成映射 GUID 的 `Bus`/`Vendor`/`Product`/`Version` 四元组：两块在那里不同的板子有不同的轴和按键映射，无论别的多匹配。

---

驾驶——控制和速度上限——在[速查表](cheatsheet.md#gamepad-configd)；从笔记本通过转发的套接字跑 `padd` 在[开发速查表](cheatsheet-dev.md#从笔记本电脑--手里拿着手柄驾驶)。
