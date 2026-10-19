# `duckctl` —— 每条命令

从笔记本电脑与机器人对话，无需网络、无需 ssh。它是手机 App 的替身，也是访问一台从未见过 wifi 网络的机器人的方式。

今天它是靠蓝牙 LE 到达那里的，而这个名字刻意不说这一点：`mediad` 给了机器人第二种传输、能触达另一组方法，所以这个工具以它对话的对象命名，而不是以它当前使用的无线电命名。在 BLE 还是唯一答案时，它叫 `duck-btctl`。

**绝不在机器人上。** 发布里没有任何东西依赖它——随发布的工具是 `robotctl`，[`cheatsheet.md`](cheatsheet.md) 有它的命令，其中大多数在下面都有一个 `duckctl` 等价物。

## 获取它

在任何装了 [`uv`](https://docs.astral.sh/uv/) 的机器上——Linux、macOS 或 Windows：

```bash
uvx duckctl --name <robot-name> info
```

那运行最新发布，下载一次并缓存；什么都不编译。要让 PATH 上有个普通的 `duckctl`，用 `uv tool install duckctl`（`uv tool upgrade duckctl` 让它往前走）。它是最新 `daemon-v*` 发布的构建，所以可能落后于 `main`；[`ci-setup.md`](../project/ci-setup.md#duckctl-on-pypi) 有它是如何发布的。

用 Homebrew，在 macOS 或 Linux 上：

```bash
brew install pierre-rouanet/tools/duckctl
```

`brew upgrade duckctl` 把它移到每个新发布。

要一个分支最后构建的版本——含 `main`——在 Linux 或 macOS 上：

```bash
curl -fsSL https://raw.githubusercontent.com/pollen-robotics/microduck/main/scripts/install-duckctl.sh | sh
```

它把 `main` 最后构建的装到 `~/.local/bin`。要另一个分支的构建，报出它：

```bash
curl -fsSL https://raw.githubusercontent.com/pollen-robotics/microduck/main/scripts/install-duckctl.sh | sh -s -- udp-pad
```

在 Windows 上，用 PowerShell：

```powershell
irm https://github.com/pollen-robotics/microduck/releases/download/daemon-dev-main/duckctl-windows-x86_64.zip -OutFile duckctl.zip
```

```powershell
Expand-Archive duckctl.zip -DestinationPath .
```

每次分支推送都在 `daemon-dev-<branch>` 预发布上发布这些构建，就在守护进程旁边。一个在浏览器里下载而不是用脚本下载的 macOS 二进制会被隔离；用 `xattr -d com.apple.quarantine duckctl` 清除它。

有 Rust 工具链的话，从本仓库的一个克隆运行它：

```bash
cargo run -q -p duckctl -- --name <robot-name> info
```

或安装一次，代价是一个不再跟随分支的快照：

```bash
cargo install --path duckctl
```

没有克隆，从 GitHub 同样做——`main`，或加 `--branch <name>` 要另一个：

```bash
cargo install --locked --git https://github.com/pollen-robotics/microduck duckctl
```

构建不需要系统头文件：libdbus 编译在内。然后：

```bash
duckctl --name <robot-name> info
```

下面每条命令都以安装后的形式写出。要改成从克隆运行，在它前面加 `cargo run -q -p duckctl --`。

这个工具过去把自己装成 `btctl`。如果 `which btctl` 还找到一个，那是你安装它那会儿的构建，永远不会再变了：

```bash
cargo uninstall btd --bin btctl
```

## 找到机器人

```bash
duckctl scan
```

```
1 robot(s) advertising the duck service:
  aa:bb:cc:dd:ee:ff duck-c51b — 192.168.1.42, 1 service(s)  ← DUCK_ROBOT

7 other device(s) in range, not listed. …
```

只列机器人，无线电范围内的其他一切只计数、不列出。`--verbose` 展开那个列表，当你要的机器人不在第一个里时值得读它。

每台机器人都广播它的 IPv4 地址，所以这也是 ssh 要用的地址的来源。不建立连接、不需要 PIN。行上的 `no address` 意味着机器人不在网络上；完全没有地址的一行意味着一个机器人还没广播地址的更早发布，而 `duckctl wifi status` 仍会报告它。

SSID 不在列表里——一个广播放不下。`duckctl wifi status` 有它，连同信号和两个地址。

只要地址本身：

```bash
ssh microduck@$(duckctl ip)
```

`ip` 只打印地址、别的什么都没有，所以它可以被替换。它读广播，所以不连接任何东西、不需要 PIN、大约一秒——而且答案不是过期的：`btd` 每五秒重读一次地址，移动时重新广播。

或者跳过替换：

```bash
duckctl ssh
```

```bash
duckctl ssh -- sudo robotctl pad pair
```

`ssh` 像 `ip` 那样找到地址，然后变成 `ssh`，所以提示符、终端和退出状态都是 ssh 自己的。账户是 `--user`，否则是环境里的 `DUCK_BOARD_USER`——[`dev-push.sh`](dev-push.md) 读的那个变量，所以一台为推送设置好的笔记本也为这个设置好了——否则是 `microduck`。`--` 之后的词在机器人上运行，而不是打开一个 shell。

文件走同样的路：

```bash
duckctl scp report.md :/tmp/
```

```bash
duckctl scp :/var/log/robotd.log .
```

以 `:` 开头的路径在机器人上——`scp` 自己的 `host:path` 省掉了 host，因为 host 正是这个帮你找的部分。其他一切都原样到达 `scp`，`-r` 和它其余标志都在内，进度条和退出状态是 `scp` 自己的。账户像 `ssh` 那样解析。

这个工具自己的标志在前，在路径之前：

```bash
duckctl --name ducky scp -r logs/ :/tmp/
```

一个到处都没有 `:` 的拷贝会在扫描前被拒绝，因为它是一个本地到本地的拷贝、没有机器人参与，而 `scp` 输出里没有任何东西会这么说。一个真的叫 `:foo` 的本地文件是 `./:foo`。

一台配对到这台机器的机器人常常停止向它广播服务，那时 `ip` 改为连接并询问 `net.status`。那更慢、需要 PIN，而且总是有答案。`--verbose` 说明发生的是两者中的哪个。

一台没有网络地址的机器人会被告诉该怎么办，而不是报告为空，因为修法必须走无线电——`net.connect` 被设计成在 WebRTC 上拒绝。

一台从没被改过名的机器人自称 `duck-` 加四个由序列号派生的字符，所以是 `duck-c51b`。一台同时以两个名字报告的机器人的任一半——macOS 显示 `radxa-zero3 [duck-c51b]`——都能当 `--name` 用。

一台先扫成 `duck-c51b`、一次连接后只扫成 `radxa-zero3` 的机器人，是在一个把名字给了广播却没给适配器的发布上，而客户端缓存了适配器的。更新机器人。那不清除客户端已缓存的，所以也清掉它：Linux 上 `bluetoothctl remove <mac>`，或在 macOS 蓝牙设置里忘掉机器人。

完全没有名字——没有 `--name`、没有 `DUCK_ROBOT`——找到的第一台胜出。有名字时，两台机器人都应答它是错误而不是选择：

```
2 robots answer to "radxa-zero3": radxa-zero3, radxa-zero3
```

这发生在一块 bootloader 留下空序列号的板子上，因为那样它以主机名命名，而从同一个镜像刷出来的每块板子都有同一个。从机器人本身上给一个改名，然后用新名字：

```bash
robotctl system set-name ducky
```

## 控制台

机器人服务一个页面，推流它的摄像头并驾驶它：

```bash
duckctl open
```

找到机器人，然后在浏览器里打开 `http://<address>:8080/`。`--print` 改为给 URL，给没有浏览器的机器或脚本；`--port` 给一台以非默认 `mediad --web-port` 启动的机器人。

无需安装、无需服务——`mediad` 内嵌页面，所以一台跑着那个守护进程的机器人就是一台有控制台的机器人。

上面有什么：摄像头，旁边是链路的码率、帧率、丢包和往返时间；两个手柄和 `W`/`A`/`S`/`D` 与 `Q`/`E` 键驾驶，以手柄的 0.3 m/s 和 1.5 rad/s；在画面上拖拽来注视一个点；enable、init、relax、stop 和 shutdown；语音库作为一个菜单，以及**技能作为一个由机器人填写的菜单**——一台机器人有哪些技能是配置，所以页面问 `robot.policies` 而不是提供一个它猜的列表；还有 2 Hz 的状态流，在 `robot.health` 旁边，那里会点名一个发烫的舵机、一块亏电的电池或一个跑慢的循环。

`stop` 把页面正在发送的意图清零。**它不是紧急停止**——这个系统里没有任何东西能从浏览器切断舵机电源——因此按钮就是一个普通按钮。

原始 JSON 框、日志，以及一个 WebRTC 对端被拒绝的两个调用，在底部的抽屉里。它们证明路由表在被查询，而不是驾驶机器人。

涉及两个端口，而只有这个是你敲的：页面自己用被服务的主机访问 8443 上的信令服务器。如果页面加载了、然后说它的信令端口没应答，机器人是起来的，而你和 8443 之间有东西不是——通常是防火墙。

摄像头和驾驶控制在 WebRTC 上、别的什么都没有，所以一台没有网络地址的机器人没有控制台。先通过无线电把它加入一个——下面的 `duckctl wifi connect` 不需要你自己的网络。

## 总是同一台机器人

把名字放进环境，而不是每条命令行都带：

```bash
export DUCK_ROBOT=duck-c51b
```

把那行放进 `~/.zshrc` 保留它。下面每条命令就都不需要 `--name` 了：

```bash
duckctl info
```

`DUCK_PIN` 对 `--pin` 做同样的事，一台有自己 PIN 的机器人每条命令都需要它：

```bash
export DUCK_PIN=418299
```

针对另一台机器人的一条命令，`--name` 仍胜出：

```bash
duckctl --name duck-ffff info
```

要为一条命令忽略默认——一个上面有别人机器人的台架——把它设成空：

```bash
DUCK_ROBOT= duckctl scan
```

`scan` 标出 `DUCK_ROBOT` 指名的机器人并把它列在前面，而每条去找它的命令在开始扫描前都会说明。

## 身份

```bash
duckctl --name <robot-name> info
```

名字、序列号和开机时长。

```bash
duckctl --name <robot-name> quack
```

机器人用它自己的声音嘎嘎叫——每个声音都由序列号生成——所以应答的鸭子正是名字到达的那只。一台静音的机器人会拒绝并说明，而不是保持沉默。

```bash
duckctl --name <robot-name> name <new-name>
```

最多 24 个字符。几秒内生效、无需重启，但 Mac 继续服务它之前学到的名字，所以 `scan` 和 macOS 蓝牙设置都滞后。之后每条命令都用新名字。

改名不跟随 `DUCK_ROBOT`。工具事后会说明；变量得手工改，否则之后每条命令都去找一个不再应答的名字。

```bash
duckctl --name <robot-name> reboot
```

## WiFi

```bash
duckctl --name <robot-name> wifi status
```

SSID、信号和地址。

```bash
duckctl --name <robot-name> wifi scan
```

要几秒——机器人扫无线电，而不是返回上一次扫描。

```bash
duckctl --name <robot-name> wifi connect <ssid> --psk <passphrase>
```

开放网络省略 `--psk`。加入会让机器人断开它当前的网络，所以一个走 wifi 的 ssh 会话会掉线；那是命令在生效。它可能最多 45 秒才应答。

```bash
duckctl --name <robot-name> wifi forget <ssid>
```

## 它还好吗

```bash
duckctl --name <robot-name> health
```

控制循环是否健康。

```bash
duckctl --name <robot-name> reboot-motors
```

从硬件错误中的舵机回来的路——过载、过热、电击——否则它会一直锁着扭矩直到拔电池。`health` 点名关节。加 id 只重启那些：

```bash
duckctl --name <robot-name> reboot-motors 3 11
```

**先给每个关节断扭矩**，所以扶着机器人或让它躺着。之后保持松软，增益在下一次写入时恢复——按手柄的 Start，或 `duckctl call robot.init`，让它重新站起来。

```bash
duckctl --name <robot-name> status
```

版本握手和更新状态。

## 它在跑哪个发布

```bash
duckctl --name <robot-name> version
```

API 版本、发布，以及构建它的 git 修订版。`revision` 为 `null` 意味着发布是在某人的笔记本上构建的，而不是 CI。

## 日志

```bash
duckctl --name <robot-name> logs robotd
```

该守护进程日志的最后 40 行，本次启动。要更多，以及本次启动之前的那次：

```bash
duckctl --name <robot-name> logs robotd -n 200
```

```bash
duckctl --name <robot-name> logs btd --boot -1
```

可读的 unit：`updaterd`、`robotd`、`configd`、`btd`、`padd`、`mediad`、`tofd`，外加 `bluetooth` 和 `NetworkManager`。`.service` 后缀可选，其他任何东西都会带着那个列表被拒绝。

行去 stdout、其他一切去 stderr，所以 `logs robotd -n 200 | grep -i panic` 能用。一个长尾巴会按无线电能承载的量裁剪，旧行在前，并带一条说明。

一个跨重启的尾巴会说在哪：

```
2026-09-09T12:27:20+00:00 systemd[1]: Starting robotd.service - Robot control daemon...
-- new robotd process, pid 3227 --
2026-09-09T12:27:21+00:00 robotd[3227]: control loop running joints=15 hz=50.0 driving=true
```

这在更新后要紧，那时四十行携带两个不同构建的输出。`-- … --` 里的任何东西来自机器人而不是日志。

没有 `-f`、没有 `--since`、没有搜索。要那些，ssh 进去：

```bash
ssh microduck@$(duckctl --name <robot-name> ip)
```

```bash
journalctl -u robotd -f
```

## 更新

与 `robotctl update` 相同的词，所以在机器人上学到的命令在这里能用。它们每个都接受 `--component <name>` 并默认 `daemon`，那是机器人今天仅有的组件。

```bash
duckctl --name <robot-name> update check
```

```bash
duckctl --name <robot-name> update status
```

```bash
duckctl --name <robot-name> update versions
```

```bash
duckctl --name <robot-name> update log --limit 20
```

安装要几分钟，边走边打印进度行：

```bash
duckctl --name <robot-name> update apply
```

```
· daemon: preflight
· daemon: downloading 12%
· daemon: downloading 47%
· daemon: verifying
· daemon: swapping
· daemon: health_gate
{
  "outcome": "applied",
  "from": "0.5.1",
  "to": "0.6.0"
}
note: the robot restarts its daemons now, and `btd` about five seconds after this reply — so this
connection drops. That is the update working. Reconnect and run `duckctl update status`:
`last_attempt` carries the outcome of what just ran.
```

apply 之后连接掉线是更新在生效，不是失败。重连并读 `update status`。

一个分支构建、一个确切版本，或 staging 候选：

```bash
duckctl --name <robot-name> update apply --ref my-branch
```

```bash
duckctl --name <robot-name> update apply --version 0.5.1
```

```bash
duckctl --name <robot-name> update apply --staging
```

`--dry-run` 验证一切并在交换前停下。`--ref` 和 `--version` 是二选一；两个都要会被拒绝。

回去——上一个发布，或从 `update versions` 报出的一个：

```bash
duckctl --name <robot-name> update rollback
```

```bash
duckctl --name <robot-name> update select 0.5.1
```

两者都像 apply 一样被门控，所以一个起不来的会被回退。两者都不丢弃任何东西。

别人启动的、或机器人自己触发的一次更新的进度：

```bash
duckctl --name <robot-name> update watch
```

它打印进行中的更新走到哪了，然后是接下来的一切。它从不收到应答，所以用 Ctrl-C 结束。

## Hugging Face 账户

```
duckctl account login
```

打印一个码并打开 <https://hf.co/oauth/device>，在那里输入它——Hugging Face 的设备页不接受 URL 里的码，所以打开省的是导航、不是输入。**机器人负责等待**，所以这个工具一打印出码就断开——在它打开的浏览器里批准，或从任何其他设备批准，然后：

```
duckctl account status
```

```
duckctl account logout
```

这是唯一在一台从未见过网络的机器人上也能用的事：没有 wifi 就没有控制台、没有 LAN，剩下的是蓝牙。通过 BLE 登录是设置向导跑的同一个流程。

```
duckctl account login --no-open
```

要码和 URL 而不要浏览器——当输出不是终端时你自动得到的也是它，所以脚本不会启动任何东西。

一台已登录的机器人会拒绝并报出账户名，一台仍在等码被批准的也是。`--force` 替换任一者——被放弃的码停止工作。

## 策略与技能

每个槽位跑什么，以及这台机器人有哪些技能：

```bash
duckctl policy list
```

`skills` 数组是「这台机器人能被要求什么」的答案。它们是配置，所以在机器人之间不同、也没有可假设的列表——给一个按钮之前先读它。

跑一个：

```bash
duckctl do roulade
```

**机器人必须在驾驶。** 先按手柄的 Start，否则它会应答说明。它不需要手柄*输入*：deadman 自己把 twist 清零，所以一台没人操纵的机器人站定并做那件事。

实时换它走路的策略：

```bash
duckctl policy load walk /opt/robot/policies/current/alpha_walking.onnx
```

把那个槽位放回：

```bash
duckctl policy reset walk
```

一次一个槽位，因为线调用只取一个——重置全部七个是机器人上的 `robotctl policy reset`。路径在*机器人*上，且必须绝对。

从这里加载**挺过重启**，和机器人上的 `robotctl policy load` 完全一样：守护进程在交换前把槽位写进 `robotd.toml`。`duckctl policy reset <slot>` 是撤销。

从配置文件重读每个槽位，给别的什么改过它的时候：

```bash
duckctl policy reload
```

机器人带着扭矩开启回到家姿势、加载、再次驱动——几秒钟，超时留了余量。一个不是 `obs[1,61] -> actions[1,14]` 的文件在任何东西改变前被拒绝，一个最终仍失败的加载保留原来在跑的策略。

### 从 Hub

为这台机器人还发布了什么，以及官方策略集是否动过：

```bash
duckctl policy search microduck
```

```bash
duckctl policy check
```

安装最新的官方策略集——或 `--version v1` 回去：

```bash
duckctl policy update
```

下载别人的、而不运行它：

```bash
duckctl policy fetch RemiFabre/microduck-flamingo-cycle
```

应答点名它落在的路径。`load` 取一个路径、从不取 `org/repo`：在这里抓取和加载是两个调用，而 `robotctl` 用一个字符串拼写两者。

### 给它一个名字

`fetch` 下载一个文件；这让它成为 `robot do` 应答的东西：

```bash
duckctl policy skill polite-bow --path /var/lib/robot/policies/fffiloni/microduck-polite-bow-b1d864/main/policy.onnx --duration 4
```

```bash
duckctl do polite-bow
```

一个调用——机器人写它的配置并重载，所以什么都不重启。`--duration` 第一次必填、之后保留，所以改一个字段就是发一个字段：

```bash
duckctl policy skill polite-bow --command 1,0,0
```

`--command` 是运行期间喂给网络的 twist，除非策略把它的 twist 读成别的东西时为全零——flamingo 的是 `[flag, side, 0]`。`--unwind` 和 `--unwind-s` 是一个保持到另有指令的策略被带回来的方式。

这台机器人能被要求做什么，以及每个背后的时序：

```bash
duckctl policy skills
```

那个答案里的 `built_in` 点名 `ground_pick` 和 `sit_toggle`，`robot do` 也接受它们，但它们由机器人自己驱动、不是表项。

```bash
duckctl policy unskill polite-bow
```

本机器人这个发布自带的技能会在你做那个之后回来，因为移除条目只是移除了覆盖。

陌生人发布的东西没有任何人验证过。让它试起来安全的是下载前的 manifest 门、加载时的形状门、关节钳制和跌倒反射——不是描述。第一次把它放在支架上。

## 哪个按键跑哪个技能

```bash
duckctl pad bindings
```

```bash
duckctl pad bind x polite-bow
```

```bash
duckctl pad reset x
```

**什么都不重启、别的什么都不需要**——`padd` 在一秒内重读文件。名字先对照这台机器人的技能检查，所以打错字会作为一个点名真实技能的拒绝回来，而不是变成一个按下时什么都不做的按键。

列表标出两件客户端自己算不出的东西：`overridden`，这样某人的改动无需知道默认值就可见，和 `error`，给一个绑到这台机器人已没有的技能的按键——得到它的现实方式是移除一个技能，而不是打错字。

`""` 关掉一个按键，这和 `pad reset` 把它放回机器人出厂的东西是两回事。六个按键可绑定：`a`、`b`、`x`、`y`、`lb`、`rb`——而 `lb`/`rb` 是**肩键**，因为模拟扳机是嘴巴和嘎嘎声。

配对一个手柄是另一个命名空间、另一个守护进程——`pad.pair` 和 `pad.forget` 是 `configd` 的，用 `call` 到达。这两个是 `robotd` 的，因为检查一个技能名需要技能列表。

## 这台机器上的手柄，走 UDP

```bash
duckctl udp-pad
```

把被按下按键的手柄（或 `--pad N`）的状态发给一台 `[netpad] enabled` 为真的鸭子。地址像 `ip` 找到它的方式那样找到，所以 `--name` 选机器人；改给一个（`duckctl udp-pad 192.168.1.42`）就完全不涉及蓝牙。`--hz`（10 到 100，默认 30）是它每秒最多发送的。每只鸭子一个，从一台笔记本驾驶多只鸭子。它是局域网工具，[`udp-pad.md`](udp-pad.md) 有设置、多只鸭子和线格式。

## 其他一切 —— `call`

```bash
duckctl --name <robot-name> call <method> '<json-params>'
```

参数默认 `{}`。这些通过蓝牙可达但没有自己的包装器，写的时候前面不带 `duckctl --name <robot-name>`：

| | |
|---|---|
| `call system.services` | 哪些守护进程起来了，每个跑哪个发布。 |
| `call pad.status` | 手柄配对了吗，连接了吗？ |
| `call pad.pair '{"timeout_seconds":30}'` | 配对一只按在配对模式里的手柄。 |
| `call pad.forget '{"mac":"<address>"}'` | 丢掉一个配对。 |

`call` 等 60 秒要答案。上面的更新命令改等*沉默*——三分钟里什么都没有到——这就是为什么它们才是跑更新的方式，而不是 `call update.apply`。

## 全局选项

- `--name <robot-name>` —— 哪台机器人。没有它，用 `DUCK_ROBOT`；还没有，找到的第一台胜出。总给它是值得的：它跳过一个慢速后备层，那个层会试 Mac 上每个已连接的外设，耳塞也算。
- `--pin <six-digits>` —— 默认 `DUCK_PIN`，然后 `000000`。机器人上的 `robotctl system pin` 显示真的那个。
- `--verbose` —— 打印发送和接收的每一行，并让 `scan` 列出每个设备而不是只列机器人。什么东西挂起时要加的第一样。

## 它打印什么

应答以漂亮的 JSON 去 stdout，其他一切——进度、诊断、无线电看到什么——去 stderr。`logs` 是例外，把它的行按行打印，因为转义 JSON 里的日志尾巴没法读；它来的拒绝仍按 JSON 打印。所以 `duckctl ... info > reply.json` 把两者分开，而机器人来的一个 JSON-RPC 错误仍以非零退出。进度行以 `·` 开头、每行一条，所以 `update apply > outcome.json` 把它们留在屏幕上、把结果留在文件里。

一条命令是一个连接：它找到机器人、必要时配对、证明 PIN、问、然后断开。

每条命令在一段沉默后放弃，而不是在一个固定总量后，所以一次慢更新从不会被切断，而一台停止应答的机器人在几秒内被报告。一条*掉线*的链路立即被报告而不是等完，apply 之后它会说明：是重启把连接带下来的。一次进行中的更新能挺过两者：机器人拉取，所以它没人看着也继续，之后的 `update status` 说它走得如何。

## 什么被拒绝

遥操作（`robot.move`、`robot.head`、`robot.look`、`robot.pose`、`robot.mouth`）、紧急停止（`robot.stop`）、放开关节（`robot.relax`）、高速遥测（`robot.subscribe`）、那两条必须由人有意为之的更新命令（`update.pin`、`update.resetToGolden`）以及配对 PIN（`system.pairingPin`、`system.setPairingPin`）被 `btd` 自己拒绝，从不到达守护进程。它们以错误码 14「not available over Bluetooth」回来。

`robot.do`、`robot.init`、`robot.enable` 和 `robot.rebootMotors` **不在**那个列表里，尽管四个都动机器人：遥操作是每秒五十次小更新的流，那是 20 字节通知预算载不动的，而这里每一个都是一次请求。`robot.relax` 是那个因速率之外的理由保持被拒绝的——它唯一的结果是一台刚才还撑着自己的机器人现在不撑了。

那是一条安全边界而不是缺失的功能，每个拒绝的原因就在 `btd/src/route.rs` 里挨着它——[`app-path-design.md`](../design/app-path-design.md) §3.1 是设计。那些命令在机器人上是 `robotctl`。

## 当它找不到机器人时

```bash
duckctl --verbose scan
```

一个空列表——连一副耳塞都没有——指向 Mac 而不是机器人：蓝牙关了，或终端从没被授予蓝牙权限。

一个缺了机器人的列表指向机器人。它在一次扫描响应里广播名字、那本身可能被错过，所以一台以无名无服务报告的设备是一台可信的机器人；`--name` 照样连一个。如果 macOS 显示机器人为已配对、但连接或第一次读挂起，配对是半完成的：

```bash
sudo pkill bluetoothd
```

在 macOS 蓝牙设置里忘掉机器人做同样的事。在机器人本身上，`journalctl -u btd -b` 说 GATT 应用到底注册了没有。
