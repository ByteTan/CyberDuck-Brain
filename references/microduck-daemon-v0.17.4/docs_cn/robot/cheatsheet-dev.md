# 速查表 —— 开发板

只在由 [`install-dev.md`](install-dev.md) 搭建的板子上才有意义的命令。机器人日常需要的一切都在 [`cheatsheet.md`](cheatsheet.md)。

## 开发通道

安装某分支最后一次在 CI 上构建的版本：

```
sudo robotctl update apply --ref <branch> daemon
```

```
sudo robotctl update apply --ref main daemon
```

`--version` 改为钉死一个确切的发布。给出其中一个，除非你确实是想「回到稳定版」。

**不带 `--ref` 的 `apply daemon` 安装最新的*稳定*发布，这在开发板上通常是降级。** 它不是「安装最新的东西」；它是「安装稳定通道提供的版本」。一个分支刚合并之后，那个稳定发布仍比你一直在测的每样东西都旧——而如果它早于一个如今在板子上有 unit 文件的守护进程，它的 `ExecStart` 会指向旧发布里没有的二进制，重启失败，更新回滚。那是门控在工作，但导致它的那条命令看起来像是最明显的那条。

标签 `daemon-dev-<branch>` 随分支移动，所以没有版本号可抄。*里面的*版本对每次构建保持唯一——`0.1.0-dev.42.c719ec8`——所以同一分支的两次构建永远不会混淆。`--ref main` 是一块板子在不离开开发通道的情况下回到主线的办法；一个普通的 `apply daemon` 会离开它，因为预发布排序在它的发布之下，而且没有单独的退出步骤。

合并不会立即发布：CI 必须先构建 `main`，`--ref main` 才会解析到它。

```
gh run list --branch main
```

## 候选发布

`release.yml` 发布到 staging、还没人晋升的东西——金丝雀机器人在晋升之前应该运行的：

```
sudo robotctl update apply --staging daemon
```

```
sudo robotctl update apply --staging --version 0.3.0 daemon
```

候选版像任何发布一样用发布密钥签名，并携带它将来晋升时所用的版本号。让它在没有标志时不可达的，是它被标记为预发布，而普通的 `apply` 跳过它们，这样没有机器人会漂到一个没人验证过的构建上。`--staging` 是这个过滤器的唯一选择加入方式，它只作用于那一条命令，之后不留任何开着的东西。

## 更新之后 —— 会咬人的部分

- **`robotd`、`configd` 和 `padd` 在更新期间重启。`updaterd` 和 `btd` 在它应答 5 秒后重启**——第一个不能在更新中途重启自己，第二个可能正在携带那个应答。所以一个 `btd` 修复几秒后就生效，无需手工步骤。重连，它就在那里了。
- **如果那两个重启之一没发生，下一次 `updaterd` 启动会修好它。** 除了 `updaterd` 自己，它报告不一致而不是重启自己。对那一个再跑一次 apply：它应答 `already_current`，点名没在运行它的守护进程，并安排重启。`sudo systemctl restart updaterd` 手工做同样的事。
- **一块跑着比 0.4.0 更旧 `updaterd` 的板子没有这些**，它让两者都留在旧二进制上，直到你重启它们。一次更新就能修好它，只有那之后的更新才这样表现。
- **如果你要的版本是板子已经有的，`robotctl update apply` 报告 `already_current` 且什么都不装**——但它不再是无害的。它检查哪些守护进程在运行那个发布，并重启没有的，在 `stale` 里点名它们。所以当一个修复看起来缺失时，*它*正是该去用的命令：要么它修好，要么 `stale` 为空、那个修复从来没进过那个发布。

症状是一个确实装上了、又确实没起作用的修复。问每个守护进程在跑哪个发布：

```
robotctl health
```

`units` 块为每个守护进程打印一行，连同它的进程从哪个发布启动，并在它与已安装版本不一致时给出点明重启的警告。`build unknown` 意味着该守护进程在运行、但没发布身份——很可能是旧构建；重启它它就会应答。`restarting` 意味着它退出了、systemd 正在把它带回来：更新后几秒内是常态，而如果一直那样，就是一个根本无法启动的守护进程。

如果一个守护进程确实过时了，重启它——这应该没必要，所以值得读日志看它为什么是：

```
sudo systemctl restart configd
```

`updaterd` 是那个从不自我修复的：

```
sudo systemctl restart updaterd
```

不需要编辑板子的 `updater.toml`——重启集合来自发布所带的 unit。`../design/restart-order.md` 是完整的逐步序列。

## 从笔记本电脑 —— 在这构建、装到板上

完全跳过 CI：在你的机器上构建并通过 ssh 安装，约一分钟。

```bash
scripts/dev-push.sh microduck@<board>
```

结果是一次普通的门控更新，所以上面的重启陷阱仍然适用。[`dev-push.md`](dev-push.md) 有设置、容器构建、`--dry-run`、……

先停掉机器人上的那个，否则两个进程会争抢摇杆：

```bash
sudo systemctl stop padd
```

转发套接字并让它保持打开：

```bash
ssh -L /tmp/robotd.sock:/run/robotd.sock microduck@192.168.1.42
```

然后从这个克隆里，在另一个终端：

```bash
cargo run -p padd -- --socket /tmp/robotd.sock
```

`systemctl start padd` 把机器人自己的那个放回。摇杆映射及其限幅不是标志：它们是 `robotd.toml` 里的 `[pad_drive]`、`[pad_axes]`、`[pad_head]`、`[pad_body]` 和 `[pad_roller]`，用 `sudo robotctl configure` 设置、一秒内生效。`--max-head`（弧度，每个头关节）和 `--deadzone` 保留，且在给出时压过文件，用于不改文件就试一个值：

```bash
sudo -u padd /opt/robot/daemon/current/bin/padd --deadzone 0.15
```

## 从笔记本电脑 —— `duckctl`

通过蓝牙 LE 访问机器人，无需网络、无需 ssh：[`duckctl.md`](duckctl.md) 有每条命令。
