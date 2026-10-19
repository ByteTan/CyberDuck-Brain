# 手工逐步安装

把 `scripts/provision-board.sh` 做的事拆成单独的命令。当某一步需要单独测试时用它；只想要一块能用的板子时用 `provision-board.sh`。

## 把文件传上去

从你机器上的一个克隆。传到 `~`，**不是** `/tmp`——中间有一次重启，`/tmp` 挺不过去。

```bash
scp scripts/setup-board.sh scripts/migrate-network.sh pierre@192.168.1.42:~/
```

```bash
scp scripts/install.sh deploy/dev-key/team.dev.pub pierre@192.168.1.42:~/
```

## 重启之前

先建 `robot` 组，这样重启后成员身份就是生效的，而不需要再来一次：

```bash
sudo groupadd --system robot
```

```bash
sudo usermod -aG robot "$USER"
```

板子 bring-up——设备树覆盖、内核控制台离开电机 UART、getty 屏蔽、`Privacy = device`、onnxruntime：

```bash
sudo sh ~/setup-board.sh
```

网络——netplan 转 NetworkManager：

```bash
sudo sh ~/migrate-network.sh
```

```bash
sudo reboot
```

这次重启不是可选的：一个设备树覆盖和一个网络栈无法在运行中的内核下替换。

## 重启之后

两个都再跑一次。它们是幂等的，而第二次 `migrate-network.sh` 正是让 wifi 后备退场的那次运行，否则在任何 wifi 较慢的启动上它都会把这块板子倒回 netplan：

```bash
sudo sh ~/setup-board.sh
```

```bash
sudo sh ~/migrate-network.sh
```

然后是守护进程。`install.sh` 从环境读取设置，`sudo -E` 是把它们带过去的方式：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
export DUCK_REF=main
```

```bash
export DUCK_DEV_KEY=$HOME/team.dev.pub
```

```bash
sudo -E sh ~/install.sh
```

对一块只该接受发布的板子，去掉 `DUCK_DEV_KEY`。把 `DUCK_REF` 设成一个分支，则安装该分支最后一次构建的版本。

上面的 `setup-board.sh` 运行带上 `DUCK_WEIRD_BLE=1` 就是 `--weird-ble`：给一块蓝牙根本无法配对手柄的板子。见 [`pair-a-gamepad.md`](pair-a-gamepad.md)。

`DUCK_NO_START=1` 安装发布、unit、用户和组，但**什么都不启用**——连下一次启动也不。它还会停止并禁用上一次安装留下的任何运行中的守护进程，所以无论卡是新的还是旧的，状态都相同。用于把板级故障与守护进程分开：重启进一块我们什么都没在跑的板子、测试，然后逐个把它们拉起来。

```bash
sudo -E DUCK_NO_START=1 sh ~/install.sh
```

```bash
sudo reboot
```

这次重启是它的一部分，不是整洁问题。发布自己的 `hooks/postinstall` 会在 `install.sh` 能停掉它们之前启用并启动每个守护进程，所以无论旋钮说什么，它们都在那次启动上跑过了——而一个守护进程死掉时不会撤销它推给子系统的东西。`btd` 留下 `Pairable` 设置、一个广播实例，以及它的默认配对代理给适配器的 IO 能力。

把它变回一台能用的机器人：

```bash
sudo systemctl enable --now updaterd robotd configd btd padd
```

想要个名字的话，给它起一个：

```bash
robotctl system set-name duck-01
```

## GStreamer，给一块要推流的板子

Provisioning 默认做这个——`provision-board.sh` 上的 `--no-gstreamer`（这里即 `DUCK_GSTREAMER=0`）跳过它。两种方式都无需重启，所以它可以在一块跳过它的板子上随时手工运行。

```bash
scp scripts/setup-gstreamer.sh pierre@192.168.1.42:~/
```

```bash
sudo sh ~/setup-gstreamer.sh
```

它打印这块板子能编码什么——注册了哪些 `webrtc*` 元素、能触达哪个 H.264 编码器，以及运行中的内核是否暴露了 VPU。在 Zero 3W 上它报告 `/dev/mpp_service` 而没有 `v4l2h264enc`，这正是一块 Rockchip BSP 内核应有的样子：VPU 经 MPP 而非 V4L2 访问，报告会点出那两个证明它能编码的 Radxa deb。任何内核变更后都重跑它；那是改变答案的事件：

```bash
sudo /usr/local/sbin/robot-setup-gstreamer
```

加 `--dev` 还安装头文件，用于在板上构建 `gst-plugin-webrtc`，或构建一个 aarch64 sysroot 来交叉编译。

## 检查它

```bash
robotctl health
```

```bash
robotctl version
```

`robotd` 在一块舵机未供电的台架板上报告 unhealthy，是诚实的答案，不是失败的安装。

然后是游戏手柄——[`pair-a-gamepad.md`](pair-a-gamepad.md)：

```bash
sudo robotctl pad pair
```
