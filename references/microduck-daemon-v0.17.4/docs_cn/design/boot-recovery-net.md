# 启动恢复网

当机器人启动的那个发布起不来它的守护进程时，什么把它放回已知完好的发布。

这一页拥有启动截止期及其谓词、它监视的 unit 的成员规则、`robot-rescue`、`golden` 符号链接，以及循环守卫。[`restart-order.md`](restart-order.md) 拥有它身处其中的序列——包括 `updaterd` 启动时从哪读面包屑——而 [`updater-design.md`](updater-design.md) 拥有仓库、健康门控和启动计数器。

## 它补上的缺口

`updaterd` 在更新期间从不重启自己，因为它是执行更新的进程。三层缩小那代价，且三层都跑在一块完全起来的板上：

- `self_test_updaterd` 在提交前跑发布自己的 `updaterd --self-test`，所以一个错误架构的二进制、一个缺失库或一个被拒的 `updater.toml` 在旧发布仍在线时就让更新失败；
- 延迟重启在更新应答五秒后把 `updaterd` 和 `btd` 放到新二进制上；
- 启动对账从后继一侧检查那些重启发生了。

一个在运行板上能启动的二进制仍可能在**启动时**失败，那里网络没起、外设没枚举、unit 顺序不同。而对账跑在*`updaterd` 启动时*，所以它预设一个能启动的 `updaterd`。

最后一点正是这个东西存在的全部原因。`BootCounter::record_boot` 从 `Engine::recover_on_start` 跑、在 `updaterd` 里，计数 `updaterd` **启动**而非字面启动——所以一个不启动的 `updaterd` 不计数，而永不砖保证经那一个可能成为牺牲品的进程闭合。留在那里，一块板子磁盘上有一个好发布、一个武装好要回退到它的试运行，却没有任何运行中的东西能对任一者行动。

启动计数器处理一个*不同*的失败：一个启动、运行、却从不报健康的发布。这个处理一个其守护进程根本不起来的发布。

## 部件

| | |
|---|---|
| `robot-boot-check.timer` | 每次启动 180 秒后触发 |
| `robot-boot-check.service` | oneshot：问发布起来没有，没有就交接 |
| `scripts/robot-boot-check` | 谓词。装在 `/usr/local/sbin/robot-boot-check` |
| `scripts/robot-rescue` | 把 `current` 换成 golden、记录它、重启。`/usr/local/sbin/robot-rescue` |
| `<install_dir>/golden` | 指向带持久保证的发布的符号链接，由 `updaterd` 发布 |
| `<state_dir>/rescued` | 面包屑：上次尝试，以及循环守卫 |

两个脚本都是 `/bin/sh`，都由 `scripts/install.sh` 和 `hooks/postinstall` **拷**进 `/usr/local/sbin`——与那两个拷而非经 `current` 读的 unit 文件是同一个决定、同一原因。一个作为发布负载里二进制发布的救援，恰会被它存在要挺过的那些发布弄坏：错误架构、缺失共享库、启动时恐慌。一个经 `current` 读的救援，会把恢复路由经正被恢复的东西。

## 触发：一个截止期，不是 `OnFailure=`

`OnFailure=` 看起来是自然搭配——systemd 恰好在一个 unit 耗尽重启、进入 `failed` 时触发它，无轮询、无驻留看门狗。它不搭配*这些* unit，它们被配置成不到达那个状态：

| unit | 策略 | `RestartSec` |
|---|---|---|
| `btd` | `Restart=always` | 5 s |
| `padd` | `Restart=always` | 5 s |
| `configd` | `Restart=always` | 2 s |
| `robotd` | `Restart=always` | 2 s |
| `updaterd` | `Restart=on-failure` | — |

`Restart=always` 在每种情形都是刻意的、在 unit 文件里论证。没有 unit 覆盖 `StartLimitIntervalSec`/`StartLimitBurst`，所以 systemd 十秒五次启动的默认适用——而一个每 5 s 重启的 `btd` 从不到达它。它无限崩溃循环、从不进入 `failed`，那是 `OnFailure=` 等的状态。让 `OnFailure=` 触发意味着重调那些 unit 好让守护进程*放弃*：为了启用救援情形而降级正常情形。

一个截止期反转问题、无需改 unit、且抓住永远的-重启者。`OnBootSec=180` 因为板子比看起来慢——`hci0` 直到上电后约 73 秒才存在，其中 `bluetooth.service` 有 26 秒堵在 `dbus` 后面——且守护进程等它们的硬件而非退出，所以一次慢启动不是一次失败启动。

### 谓词

两个条件，都是 systemd 试着跑一个守护进程却没法让它继续跑的正证据。任一成员上任一个：

- `ActiveState` 是 `failed`；
- `NRestarts` 是 3 或更多。

其他一切都别动，而那个不对称是重点。一个操作员停掉的 unit 是 `inactive`、无重启，绝不能以发布为代价——启动对账留一个停掉的 unit 停着时做的是同一个调用。一个……

## 救援

`robot-rescue` 把 `current` 换成 golden、记录它、重启。它命名的 golden 是最后写它时是 golden 的发布，即一个已知完好的发布。一个配置了却*未安装*的 golden 根本不发布链接，因为一个悬空的会告诉救援它有一个目标、而它没有。

**Golden，不是上一个：** 当坏掉的是恢复路径本身，上一个可能也是坏的。Golden 是唯一带持久保证的发布，`prune` 永不移除它。

当没有 golden 链接、当 golden 指名一个未安装的发布、当 `current` 已经是 golden 时，它谢绝并说明是哪个。最后那个最要紧：守护进程那时在带持久保证的发布上失败，所以这不是发布故障，交换除了多一次重启什么都改不了——那正是一次硬件故障变成重启循环的方式。

交换把一个临时符号链接 rename 到 `current` 上，匹配 `Store::link_to`，因为 `ln -sfn` 会 unlink 再重建、留下一个守护进程无法 exec 的窗口。`rename(2)` 不跟随符号链接、但 `mv` 的路径解析会，所以一个普通 `mv tmp current` 把链接移*进* `releases/<version>`、留下 `current` 未动；GNU 把修法拼作 `-T`、BSD 拼作 `-h`。

`--reboot` 是选择加入。每个 unit 经 `current` exec，所以在它们重启前交换什么都不做，而一台在守护进程死时站着的机器人是一台会倒的机器人——扶着它的人决定。启动检查传标志；控制台前的人改看到打印的命令。

## 循环守卫，以及什么打开它

两件独立的事阻止「交换并重启」自己发生两次：

1. 当 `current` 已是 golden 时的拒绝，它覆盖普通情形，因为一次成功救援恰好留下那个状态；
2. 面包屑，它覆盖第一个漏掉的情形——一次更新把 `current` 移离 golden 又失败了。

`Engine::record_rescue` 在下次 `updaterd` 启动时清面包屑，先把它拷进更新日志。所以守卫恰好在板子证明它能再跑它的更新守护进程时打开、不能时保持关：一台带着干净日志静静坐着的机器人，胜过一台永远重启的。`robot-rescue --force` 是手工绕过它的办法。

那个清除跑在启动计数器**之前**，顺序是承重的。一个武装的试运行会回退到 `previous`；救援已走得更远、到 golden。留着武装，`record_boot` 会推进试运行、最终把 `current` 移*离*救援选的发布——恢复网和永不砖保证互相抵消，相隔一次启动。

面包屑是 `key=value` 而非 JSON，尽管 `updaterd` 解析它：

```text
at=1786453421
install_dir=/opt/robot/daemon
from=1.2.0
to=1.0.0
because=boot check: robotd.service (failed, 7 restarts)
```

写入者是一个跑在东西已经出错板上的 shell 脚本，而在 `sh` 里正确引 JSON 是产出没人读得了的记录的办法。`journal::Breadcrumb` 因同原因宽容——每个字段可选——因为一个解析不了的面包屑是一个永远打不开的守卫。`install_dir` 是条目归属到一个组件、而无需第二个地方就得和 `updater.toml` 就名字 `daemon` 一致的办法。

## 证据

一次无声回滚是「在我板上它能用」的一天发生的方式，所以交换留三条痕迹：

- **日志**，来自两个脚本，有 `logger` 时经它、stderr 总是；
- **更新日志**，作为 `Outcome::RolledBack`、原因里点名失败 unit——`robotctl update log` 显示的，且它永久，不像被清的面包屑；
- **`robotctl version`**，间接但有用：一块跑着比已安装更旧发布的板子正是它报告的。

## 什么没被测试

决定是：`xtask/tests/rescue.rs` 对一棵临时树问两个脚本每个问题，用一个 stub `systemctl` 提供 unit 状态并记录是否要了重启，而 `scripts/board-test.sh` 在真 ARM64 Linux 上覆盖安装器那一半。

**systemd 没有。** 定时器是否在 `OnBootSec=180` 触发、`Conflicts=shutdown.target` 是否让 oneshot 在下行路上停着、`NRestarts` 是否在崩溃循环 unit 上按谓词假设的那样读——都未验证。除真 systemd——`systemd-nspawn`、或一个它作 pid 1 的特权容器——没别的能答任何一个，而 [`install-path-gap.md`](../project/install-path-gap.md) 是那件事论证住的地方。这是它的第二个论证：恢复网是当其他一切都不工作时必须工作的代码，也是最不适配测试的代码。

在那之前这个机制在一块台架板上验证、或根本不验证，而做它的办法是给一块板子一个其 `robotd` 起不来的发布。

它没有任何一个在硬件上跑过，那里有趣的路径需要一块其发布起不来的板子。
