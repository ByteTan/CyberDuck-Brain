# 在这构建，装到板上

从改一行到看着机器人跑它的循环，不 push、不跑 CI、不打标签。从本仓库克隆里的一条命令就为板子构建、给结果签名并通过 ssh 安装——增量构建约一分钟，而 push 加一次 CI 运行要几分钟。

`scripts/dev-push.sh` 就是那条命令。下面的一切都是它的一个标志。

## 第一次推送之前，一次性

三件事，之后再也不用了。

**板子必须是开发板。** 工件用团队开发密钥签名，所以客户机器人会像拒绝 `--ref` 一样拒绝它。[`install-dev.md`](install-dev.md) 是让一块板子成为开发板的办法。

**开发签名密钥**放在 `~/.duck-keys/team.dev.key`——CI 给分支构建签名用的密钥的私钥一半，团队成员手里有。如果你的在别处，设 `DUCK_DEV_SECRET_KEY`。

**一套能为板子构建的工具链。** 要么安装交叉编译器：

```bash
cargo install cargo-zigbuild --locked
```

```bash
brew install zig
```

要么什么都不装，给下面的每个 `dev-push.sh` 传 `--docker`，那只需要一个运行中的 Docker 守护进程。那条路启动更慢，是你在完全没有板子之前该走的路——见[在容器里构建](#在容器里构建)。

## 循环

报出机器人名字，让推送找到它：

```bash
scripts/dev-push.sh --name duck-c51b
```

或每个 shell 一次：

```bash
export DUCK_ROBOT=duck-c51b
```

```bash
scripts/dev-push.sh
```

名字是 `duckctl scan` 列出的、`robotctl system set-name` 设置的那个——和那个工具读取的同一个 `DUCK_ROBOT`（[`duckctl.md`](duckctl.md)）。它的地址通过蓝牙询问——机器人自己的 `net.status` 用地址作答——然后被缓存，所以只有一次够不到缓存地址的推送才会回去找无线电。这就是为什么换一次 DHCP 租约、一次重刷或换一个网络都不花力气跟上。

ssh 用户是 `microduck`。如果你的不是：

```bash
export DUCK_BOARD_USER=pierre
```

一个地址仍然有效，并完全跳过无线电：

```bash
scripts/dev-push.sh microduck@192.168.1.42
```

```bash
export DUCK_BOARD=microduck@192.168.1.42
```

它交叉编译 workspace、打出一个发布相同的工件、用开发密钥签名、拷到板子上的 `~/duck-sideload`，并通过 `robotctl update apply --from` 在那里应用它。然后它等待守护进程报告新发布：

```
==> building 0.5.1-dev.local.1763400000.g7fc1444 for the board (zigbuild)
==> packaging
==> signing with /Users/you/.duck-keys/team.dev.key
==> copying to microduck@192.168.1.42:/home/microduck/duck-sideload
==> applying on microduck@192.168.1.42
==> 0.5.1-dev.local.1763400000.g7fc1444 is live on microduck@192.168.1.42
==> checking every daemon is running it
    current -> 0.5.1-dev.local.1763400000.g7fc1444
    [ok] robotd
    [ok] configd
    [ok] padd
    [ok] updaterd
    [ok] btd
    [ok] mediad
    [ok] tofd
==> every daemon on microduck@192.168.1.42 is running 0.5.1-dev.local.1763400000.g7fc1444
```

`updaterd` 和 `btd` 在 apply 应答五秒后重启，所以那两行要过一会儿才到。一个 `[--] padd published nothing` 不是失败——一个停着的或刻意禁用的守护进程看起来也是这样，而一块没有摄像头的板子上的 `mediad` 看起来也是这样。

一个 `[--] tofd published nothing` 意味着一个 `tofd` 还没发布过身份的更早发布——再推一次就修好——而不是传感器缺失：无论是否装了传感器，`tofd` 都运行。

在板上，那个版本就是 `robotctl version` 报告的：

```bash
robotctl version
```

同一棵脏树的两次推送永远不会冲突——版本携带推送的时间戳，而不只是提交。这里预期树就是脏的。

**这是一次普通更新。** 签名、工件哈希、兼容性、健康门控和自动回滚全都运行：一个起不来的构建会被回退，板子回到它之前跑的东西。有意回退也是那条普通命令：

```bash
sudo robotctl update rollback daemon
```

## 观察你刚推送的东西

从第二个终端、在推送之前，这样重启就会出现在它里面：

```bash
ssh microduck@192.168.1.42 'journalctl -f -u robotd -u configd -u btd -u padd'
```

单独的 `-u updaterd` 是更新本身——每个阶段、健康门控，以及它安排的重启：

```bash
ssh microduck@192.168.1.42 'journalctl -f -u updaterd'
```

一个守护进程里的 panic 会带着完整回溯落到那里：这些二进制没有剥除任何东西，所以栈帧有名字。

```bash
ssh microduck@192.168.1.42 robotctl health
```

控制循环是否起来，以及每个守护进程在跑哪个发布。[`cheatsheet.md`](cheatsheet.md) 有其余可以问机器人的东西。

### 不止 `info`

每个 unit 设 `RUST_LOG=info`。要看一个守护进程的 `debug!` 行，用一个 drop-in 覆盖它——在板上，这里是 `robotd`：

```bash
sudo mkdir -p /etc/systemd/system/robotd.service.d
```

```bash
sudo tee /etc/systemd/system/robotd.service.d/log.conf > /dev/null <<'EOF'
[Service]
Environment=RUST_LOG=debug
EOF
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart robotd
```

drop-in 住在 `/etc`，所以它比之后每次推送都活得久。用完把它摘掉：

```bash
sudo rm /etc/systemd/system/robotd.service.d/log.conf
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart robotd
```

## 不安装地验证

```bash
scripts/dev-push.sh --dry-run microduck@192.168.1.42
```

构建、签名、拷贝，然后做真正 apply 除交换外的一切：签名、哈希、兼容性、解包，以及检查没有任何已安装 unit 指向这个构建不含的二进制。它在 `current` 移动之前停下，所以板子继续跑它之前的东西，没有守护进程重启。

## 改为在容器里构建

```bash
scripts/dev-push.sh --docker microduck@192.168.1.42
```

无需 zig、无需 `cargo-zigbuild`，也无需一块板子来拷 libudev。在 Apple Silicon Mac 上容器的目标就是主机，所以什么都不用交叉编译；在 x86 笔记本上它在仿真下运行，脚本会说明。同样的工件，`--dry-run` 和 `--bootstrap` 同样可用。

两种模式保留各自的 `target/` 目录，所以在两者之间切换要花一次完整重建。默认更快，也是 CI 用的。

## 重装板上已有的东西

推送把工件留在板上的 `~/duck-sideload`，所以它无需构建任何东西就能再次安装：

```bash
sudo robotctl update apply daemon --from ~/duck-sideload
```

同一条命令接受任何装着一个发布的目录——比如一根 USB 盘。每次推送替换该目录而不是往里加。

## 没有机器人地为板子编译

要检查一个改动能为目标构建——C 链接的 crate、仅 Linux 的路径——而手边没有板子：

```bash
cargo board --bins
```

`cargo board` 是 `cargo zigbuild` 配上板子的目标和 glibc 下限，定义在 `.cargo/config.toml`。它需要与默认推送相同的 zig 工具链，而 `padd` 需要第一次推送从板子上抓来的 libudev 副本。`--docker` 两者都不需要。

## 第一次推给低于 0.5.0 的板子

`apply --from` 需要 API 版本 7，它首次随 0.5.0 发布。跑任何更早版本的板子有一个无法被要求使用它的 `updaterd`，它会拒绝调用，而不是悄悄改成从它配置的源安装。用无门控的方式交付那一次发布：

```bash
scripts/dev-push.sh --bootstrap microduck@192.168.1.42
```

那会停掉 `robotd`，并为那一次安装放弃健康门控。之后每次推送都是普通命令。

## 当它不工作时

**`no dev signing key at ...`** —— 板子像验证任何发布一样验证这个工件，所以它必须被签名。从团队成员那里拿 `team.dev.key`，或把 `DUCK_DEV_SECRET_KEY` 指向它。

**`cargo-zigbuild is not installed`** —— 安装它和 zig，或用 `--docker`。

**`no libudev.so.1 on <board>`** —— 第一次推送从板子上拷那个库来链接 `padd`。一块还没起来的板子提供不了它；`--docker` 不需要它。

**重刷板子后，关于 libudev 的链接错误** —— 副本被缓存了。丢掉它，下次推送抓一个新的：

```bash
rm -rf ~/.cache/duck-cross/aarch64
```

**`apply failed (exit 2)`，且 `robotctl` 和 `updaterd` 报告 API 不匹配** —— 板子已安装的发布早于 `apply --from`。用一次 `--bootstrap`。

**`preflight check failed: SideloadDir: ... is not there for updaterd`** —— 发布在 `/tmp` 或 `/var/tmp` 下，这发生在 `DUCK_SIDELOAD_DIR` 或一个手写的 `--from` 指向那里时。`updaterd.service` 设了 `PrivateTmp=yes`，所以守护进程有自己的 `/tmp` 和 `/var/tmp`，两者都不是你的 shell 拷进去的那个。任何其他路径都行；默认的 `~/duck-sideload` 就是。在一块发布早于那个检查的板子上，同样的错误读作 `no manifest for version <version> in <dir>` —— 而那个目录的 `ls` 恰好列出了那个 manifest。

**`verification failed: signature did not verify against any of N usable trusted key(s)`** —— 读起来像发布损坏，通常意味着板子不是开发板：开发密钥从没落地，或 `allow_dev_keys` 关着，二者之一都会把那个密钥排除在可用集之外。在板上：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

`0` 意味着密钥缺失；[`install-dev.md`](install-dev.md) 有修法的两半。

**`could not reach <name> over Bluetooth`** —— 机器人必须正在广播且在范围内，名字路径才能解析出地址。

```bash
duckctl scan
```

什么都没列出，是一台关机、超出范围或已连接到手机的机器人。改给地址，无线电就不掺和了：

```bash
scripts/dev-push.sh microduck@192.168.1.42
```

**`<name> answered over Bluetooth but has no wifi address`** —— 它起来了但不在网络上，所以没有可 ssh 的东西。通过同一个无线电加入一个：

```bash
duckctl --name duck-c51b wifi connect <ssid> --psk <passphrase>
```

**`still <address>, which ssh could not reach`** —— 地址从没变，所以 ssh 不高兴的是别的。重刷过的板子是常见的那种；见下面的主机密钥。

**重刷后 ssh 拒绝连接** —— 板子重新生成了主机密钥。

```bash
./scripts/provision-board.sh microduck@192.168.1.42 --forget-host-key
```

**`the release is live but not everything is running it`** —— 交换发生了、健康门控通过了，但一个守护进程仍在旧二进制上，读作一个没起作用的修复。脚本点名是哪些。在板上：

```bash
robotctl health
```

`units` 块点名每个守护进程在跑哪个发布。

```bash
journalctl -u updaterd -b | tail
```

找 `restart scheduled`，或没有它的原因。

```bash
sudo systemctl restart updaterd
```

这本不该必要，所以值得读日志看它为什么是。[`cheatsheet-dev.md`](cheatsheet-dev.md) 有完整的重启陷阱，[`../design/restart-order.md`](../design/restart-order.md) 是逐步序列。

## 设置

| | |
|---|---|
| `DUCK_ROBOT` | 机器人，按名字。它的地址通过蓝牙找到并缓存。 |
| `DUCK_BOARD_USER` | 板上的 ssh 用户，用于名字路径。默认 `microduck`。`duckctl ssh` 和 `duckctl scp` 也读它。 |
| `DUCK_PIN` | 机器人的配对 PIN，如果不是出厂 `000000`。由 `duckctl` 读取。 |
| `DUCK_BOARD_CACHE` | 解析出的地址缓存到哪。默认 `~/.cache/duck/boards`。 |
| `DUCK_BOARD` | 板子，按地址，代替参数。`microduck@192.168.1.42`。 |
| `DUCK_DEV_SECRET_KEY` | 开发签名密钥。默认 `~/.duck-keys/team.dev.key`。 |
| `DUCK_SIDELOAD_DIR` | 工件在板上的落点。默认那里的 `~/duck-sideload`。绝不要在 `/tmp` 或 `/var/tmp` 下：`updaterd` 有两者的私有副本，会读那些。 |
| `DUCK_CROSS_SYSROOT` | 缓存的 libudev 副本。默认 `~/.cache/duck-cross/aarch64`。 |

## 这刻意不做什么

不做发布为来源证明做的任何事。版本携带时间戳而不是标签，工件用客户机器人拒绝的密钥签名，什么都不发布——所以没有别人能装你刚跑过的东西。发一个发布仍是打标签和 `release.yml`（[`../../README.md`](../../README.md)）。
