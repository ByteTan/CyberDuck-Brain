# 在开发板上安装

把一块板子从零带到一台你能推送分支的机器人。

开发板信任团队开发密钥，所以它会安装团队里任何人构建的任何东西。客户机器人设置不同，会刻意拒绝那些构建——这里的一切都假设是开发板，绝不是客户机器人。

开发构建什么都不放松：同样的签名和哈希校验、同样的健康门控、同样的自动回滚。唯一的区别是哪个密钥签的名，而那正是把这些构建挡在客户机器人之外的原因——它们双重拒绝开发密钥。`allow_dev_keys = false`，而一个受信任密钥只有在其文件名以 `.dev.pub` 结尾时才算是开发密钥。下面设置的两半正是为了翻转那个。

## 刷板子

用 [Armbian imager](https://www.armbian.com/radxa-zero-3/)。选 **Radxa Zero 3**，然后 **Armbian 26.2.1 Minimal**。

写之前，填好 imager 的 profile——wifi 网络和密码，以及你想要的用户名和密码。在那里做省得之后用串口控制台：板子在第一次启动时加入你的网络，并立即可通过 ssh 访问。

然后加你的 ssh 密钥，这样 provisioning 在重启板子后能重连：

```bash
ssh-copy-id microduck@192.168.1.42
```

## 你需要什么

- 板子的 **IP 地址**。这个镜像上的 mDNS 不可靠，所以 `.local` 名字看心情解析。`duckctl ip` 通过蓝牙问机器人，不需要你自己的网络、也不需要读 DHCP 租约；如果板子还没在广播，你路由器的租约表是后备。
- **ssh 密钥访问**，来自上一步。provisioning 会重启板子并自己重连，而一个密码提示挺不过那次。
- 一个 **GitHub token**，当这个仓库还是私有的时候：没有它，它的发布资产不可达。一旦公开，token 就是可选的，只买一个更高的 API 速率上限（`docs/design/updater-design.md` §6.1）。
- 一个**本仓库的克隆**。它需要的开发密钥提交在 `deploy/dev-key/team.dev.pub`，所以不用向任何人要。

## 安装

从你自己机器上的一个克隆，两条命令：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
./scripts/provision-board.sh --pause-btd-on-pair --name <MY_COOL_ROBOT_NAME> microduck@192.168.1.42
```

那会发你的开发密钥、启动 provisioning、等过重启、流式传输日志，并以 `robotctl health` 结束。

### 为什么那条命令里有 `--pause-btd-on-pair`

在 aic8800 无线电上，`btd` 正在广播时手柄无法形成**新的**配对。那个标志留下一个标记，让 `robotctl pad pair` 在配对窗口期间停掉 `btd` 并给适配器做电源循环，然后再启动它。现有配对不受影响——一个已配对的手柄在整个栈都起来时连接并驾驶——所以代价是一个守护进程在一次配对期间下线。

它在这里是默认的，因为一块需要它、却在 provisioning 时没带它的板子，会表现为一只配不上对的手柄，而你最先追查的每个看似合理的原因都在别处。

### 三种配置，以及如何分辨你是哪一种

有两个独立的故障，所以有两个标志。**配一只手柄并读失败信息**，然后选：

| 你看到什么 | 板子想要什么 |
|---|---|
| 手柄配对并驾驶 | 什么都不用——不带标志 provision |
| 手柄配不上对；最后的 SMP 步骤从不完成 | `--pause-btd-on-pair` |
| 即使 `btd` 暂停也配不上对 | `--weird-ble`（隐含暂停，并加 `Privacy = device`） |
| 手柄配对成功，然后**抖动**——`PIN or Key Missing (0x06)`、没有输入设备 | `--weird-ble` 对这块板子是错的：去掉它，保留暂停 |

最后一行是要盯着的。在一块只需要暂停的板子上，`Privacy = device` 会产生一个立刻失效的配对，这比一只明显配不上对的手柄更难诊断——在 `50:37:CD:16:1D:90` 上测得：`off` 加暂停能配对并保持，而 `device` 在 45 秒里抖了 46 次。所以 `--weird-ble` **不再**是默认。

把一块板子从 `--weird-ble` 移到仅暂停，保留标记：

```bash
sudo sed -i 's/^Privacy = device/Privacy = off/' /etc/bluetooth/main.conf && sudo reboot
```

反过来，用 provisioning 留在板上的脚本副本：

```bash
sudo DUCK_WEIRD_BLE=1 /usr/local/sbin/robot-setup-board && sudo reboot
```

要检查一块板子两者都不需要，去掉两者并配一只手柄：

```bash
sudo rm /var/lib/robot/weird-ble
sudo sed -i '/^Privacy = /d' /etc/bluetooth/main.conf && sudo reboot
```

任何对 `Privacy` 的更改之后都要重新配对：它改变了存储密钥所对照的地址，所以现有配对停止匹配、并以 `PIN or Key Missing` 抖动，直到被重新建立。

这一切都是对 aic8800 无线电的绕行，不是设计的属性；无线电走它就走。[`pair-a-gamepad.md`](pair-a-gamepad.md) 有细节。

它是一个**观察者**，不是干活的——provisioning 安装一个启动时续跑的系统 unit，所以无论你是否还在看，板子都会完成。Ctrl-C 不损失任何东西，你可以把日志捡回来：

```bash
ssh -t microduck@192.168.1.42 'sudo tail -f /var/lib/robot/provision.log'
```

`--ref BRANCH` 从一个分支 provision：它的脚本跑 bring-up，它的守护进程构建装在最上面。`golden` 保持稳定发布——它是启动恢复网的后备，而把一个分支构建当 golden 会给一个坏分支一个坏后备——而 `current` 是那个分支。

如果那个构建无法安装，或装上了然后被健康门控回滚，provisioning **失败**。一块在要求分支时悄悄跑着稳定发布的开发板，是最难调试的失败：一切看起来都装上了，而被测代码不在那里。在 provisioning 前给 CI 一两分钟，如果它停了就用 `gh run list --branch BRANCH` 检查。`--name` 在分支构建之前应用，所以一块在这里失败的板子仍有它的名字。

其他有用标志：`--name Ducky` 给机器人起名，而不是留下它从自身序列号派生的 `duck-7f3a`（`robotctl system set-name` 之后能改，所以这只省一条命令）、`--local` 发这个克隆的 `provision.sh` 而不是去抓取它（这是不先合并就测试 provisioning 脚本改动的办法）、`--no-dev-key` 做一块只接受发布的板子。

## 检查它生效了

```bash
robotctl health
```

只有密钥真的装上了，板子才算开发板，而那是你能检查的、而不是你得记住的：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

`1` 意味着是。`0` 意味着密钥没落地，而之后 `--ref` 会被拒绝，报一个读起来像发布损坏的错误。那正是这个检查要提早抓住的失败。

然后真正的测试——把一个分支放上去：

```bash
sudo robotctl update apply --ref main daemon
```

## 重刷后 ssh 拒绝连接时

重刷会重新生成板子的主机密钥，所以你上次用的地址现在展示的是另一个。`StrictHostKeyChecking=accept-new` 覆盖不了它：主机不是新的，它的密钥才是。对此原始 ssh 错误是一大段关于可能攻击的文字。

```bash
./scripts/provision-board.sh microduck@192.168.1.42 --forget-host-key
```

这比听起来更要紧，因为 DHCP 租约会被复用——上周是某块板子的地址，今天是另一块板子，带着不同的密钥。

## 板子换地址回来时

provisioning 中途的 wifi 切换可能让板子落在一个与你给的不同租约上。`provision-board.sh` 会去找：它等 ssh 的同时也通过蓝牙问机器人最终落在哪个地址，并采纳答案。

```
  bluetooth: the robot reports 192.168.1.57, and 192.168.1.42 was its old lease.
```

什么都不用做——运行剩余部分都指向那里。

三件事让它不工作，而它会说明是哪个：

- 它需要 `cargo` 和这个克隆，因为 `duckctl` 是一个示例而不是已安装的二进制。
- 它只能在 `btd` 运行后问，而在一块首次 provision 的板子上，那是 phase 2 开始后几分钟。
- 机器人报告它的 **wifi** 地址，所以一块你通过以太网访问的板子不被覆盖。

`--no-ble` 关掉它。一台被给了自己配对 PIN 的机器人需要在 `DUCK_PIN` 里放它。

## 让一块现有板子接受开发构建

给一块用别的方式 provision 的板子，或一块在你拿到密钥之前设置的板子。两半都需要：只有一半会留下仍拒绝分支构建的板子。

简单的办法是用密钥重跑安装器，它一步完成验证并翻转标志：

```bash
sudo DUCK_TOKEN="$DUCK_TOKEN" DUCK_DEV_KEY=/tmp/team.dev.pub sh /tmp/install.sh
```

无论源文件叫什么，它都把密钥装为 `team.dev.pub`。那个名字要紧：`.dev.` 中缀是把一个密钥归为开发密钥的依据，而以任何其他名字落地的密钥会被当成**发布**密钥信任。

手工做，如果你宁愿看每一步：

```bash
sudo cp team.dev.pub /etc/robot/trusted_keys/team.dev.pub
```

```bash
sudo sed -i 's/^allow_dev_keys.*/allow_dev_keys        = true/' /etc/robot/updater.toml
```

```bash
sudo systemctl restart updaterd
```

## 手工处理 token

`scripts/install.sh` 在给定 `DUCK_TOKEN` 时会替你写这个。这些步骤是给一块用别的方式 provision 的板子——而且只有在这个仓库私有时，或在一块频繁抓取、想要 token 买来的更高速率上限的板子上才需要。

`updaterd` 从它自己的环境读 `GITHUB_TOKEN`，所以在你的 shell 里导出它到不了守护进程——它需要一个 systemd drop-in。

```bash
sudo mkdir -p /etc/systemd/system/updaterd.service.d
```

在下一块里替换成你自己的 token——这是这里唯一的占位符：

```bash
sudo tee /etc/systemd/system/updaterd.service.d/token.conf > /dev/null <<'EOF'
[Service]
Environment=GITHUB_TOKEN=ghp_replace_with_your_token
EOF
```

drop-in 默认全局可读，而这一份装着一个凭据：

```bash
sudo chmod 600 /etc/systemd/system/updaterd.service.d/token.conf
```

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl restart updaterd
```

token 在*开发者*的板子上没问题。token 在客户机器人上不行——镜像里的一个全舰队凭据不重刷就无法轮换——这就是为什么答案是公开仓库而不是随附 token（`docs/design/updater-design.md` §6.1）。一块没有 token 的板子仍能从本地目录或一次开发推送安装。

## 无网络安装

在一块已有发布的板子上，用普通方式装一个本地目录——通过守护进程，带健康门控和自动回滚：

```bash
sudo robotctl update apply daemon --from /media/usb/release
```

那也是 `scripts/dev-push.sh` 最终做的；[`dev-push.md`](dev-push.md) 是笔记本到板子的路径。

这一节其余部分是**裸板**情形：工厂或离线安装，在有一个守护进程可问之前。因此是 `updaterd` 而不是 `robotctl`，而 `updaterd` 刻意不在 `PATH` 上：

```bash
sudo /opt/robot/daemon/current/bin/updaterd install --from /media/usb/release
```

目录装着一个发布是什么：`<version>.manifest.json`、它的 `.minisig`、工件以及工件的 `.minisig`。签名、哈希和兼容性检查得和下载时一模一样——`--from` 改变的是字节从哪来，而不是信任什么。

一旦有发布在线，那条命令就拒绝运行，因为它强制关闭 `on_apply` 和健康门控，而给一台正常工作的机器人这么做会悄悄禁用自动回滚。有一种情形还是需要它，而 `robotctl update apply` 帮不了——一块其已安装 `updaterd` 太旧、无法接受那个*修复*「太旧」的发布的板子。它每次都把新发布滚回去，而跑着那个门控的二进制正是被替换的那个。停掉机器人并明说：

```bash
sudo systemctl stop robotd
```

```bash
sudo /opt/robot/daemon/current/bin/updaterd install --from /media/usb/release --force
```

`--force` 本身在 `robotd` 仍在应答时也会被拒绝，因为反对意见关乎一台*正常工作的*机器人失去安全网。它只为那一次安装放弃自动回滚、别的什么都不放弃——签名、哈希和兼容性仍被检查，而如果发布表现异常，`sudo robotctl update rollback daemon` 是恢复路径。

## 深入

[`deploy/README.md`](../../deploy/README.md) 是所有这一切实际做什么的参考：信任链、什么落到哪里、其他入口（没有克隆的板上、逐步手工）、日志去哪以及什么能挺过重启。
