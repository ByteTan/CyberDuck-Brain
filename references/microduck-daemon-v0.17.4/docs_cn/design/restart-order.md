# 什么会重启，以及在什么时候

每一条移动 `current` 符号链接的路径，以及启动时的权威顺序。之所以写下这些，是因为
“哪个守护进程重启了、在哪一步”曾由三份不同的文档给出三种不同的答案，而答案决定了
一次版本偏差（skew）如何被诊断。

这里的每一条都读自代码，每一步都点明它所归属的函数。凡是别处某个叙述性注释与本页
不一致，以注释为 bug。

## 1. 八个守护进程，以及那个不是守护进程的 unit

一个 release 会发布八个守护进程——`robotd`、`configd`、`btd`、`padd`、`netpadd`、`mediad`、
`tofd`、`updaterd`——每个都以 `/opt/robot/daemon/current/bin/` 下的一条路径作为 `ExecStart`，
所以符号链接一动，每个就都过时了，而每个要么被更新重启，要么在更新之后被重启。

它还会发布 `robot-boot-check.service`，它**不是**这八个之一，更新也绝不能重启它：它负责
询问本次启动的 release 是否起来了，如果没有就交接给 `robot-rescue`，所以在更新中途运行它，
会把一次回滚检查指向那些正当处于重启中的守护进程。它不带 `[Install]` 段，而正是这一点让
它被排除在外——见 §1.1。

| unit | 更新中途重启 | 应答后约 5 秒重启 |
|---|---|---|
| `robotd` | 是 | — |
| `configd` | 是 | — |
| `padd` | 是 | — |
| `netpadd` | 是 | — |
| `mediad` | 是 | — |
| `tofd` | 是 | — |
| `updaterd` | **从不**——它就是执行更新的那个进程 | 是 |
| `btd` | **从不**——它可能是更新经由的传输层 | 是 |

两半是同一个决定，而一个测试（`every_unit_held_back_is_restarted_once_the_answer_is_out`）
断言 `engine.rs` 里的两个列表是同一集合：`NEVER_RESTART` 与 `RESTART_AFTER_REPLYING`。

没有任何东西被扣下直到重启。在 `RESTART_AFTER_REPLYING` 存在之前如此，而这是关于本系统
最常见的一条过时说法。

又因为“安排了重启”并不等于“确实发生了重启”，下一次 `updaterd` 启动会把每个 unit 与当前
激活的 release 对照，并重启任何过时的——§5。

### 1.1 什么才算更新会启动的 unit

一个带有 `[Install]` 段的 `systemd/*.service` 文件。没有这个段的由别的东西触发——一个 timer，
或另一个把它拉进来的 unit——所以它的生命周期不由更新来驱动，而 `engine.rs::has_install_section`
就是决定这一点的地方。

定成规则而不是一列名字，是因为已经有应用它的两个地方意见不一：`hooks/postinstall` 拒绝
`enable --now` 一个没有 `[Install]` 的 unit，并白纸黑字说明——*“对它 `enable --now` 会在安装
它的更新中途运行一次回滚检查，而那些守护进程正当处于重启中”*——而引擎却读取每一个
`*.service`，片刻之后照样把它重启。以该段为判据后，两者由构造保证一致，下一个同类 unit
不再需要任何人记住。

一个无法读取的 unit 文件算作需要重启的。这更可能是权限问题，而不是一个刻意无触发器的
unit，而响亮地失败一次重启，胜过静默地跳过一次。

### 集合如何计算

两个集合，相隔一个函数。`engine.rs::units_shipped` 是 release 提供的全部：

```
(the release's own systemd/*.service stems)  ∪  (on_apply.units from /etc/robot/updater.toml)
```

经排序并去重。`engine.rs::units_to_restart` 是它减去 `NEVER_RESTART`：

```
units_shipped(…)  −  {updaterd, btd}
```

更新重启 `units_to_restart`；§5 的启动检查读取 `units_shipped`，因为更新碰不得的那两个 unit，
恰恰就是它存在是为了照看的那两个。

在今天的 release 上，`units_to_restart` 恰好是：

```
configd, mediad, netpadd, padd, robotd, tofd
```

就按这个顺序——字母序，因此在每块板子、每次测试里顺序都相同。`padd` 与 `netpadd` 恰好只有
一个会运行，由 `ExecCondition=` 在 `[netpad] enabled` 上选择——`netpadd/systemd/netpadd.service`
说明了为何如此而非 `Conflicts=`——所以两者之间的重启顺序无关紧要。为了让 `mediad` 和 `tofd`
进入那个列表，`deploy/updater.toml` 里没有添加任何东西，也无需添加：两者都发布带 `[Install]`
段的 unit，这就是全部规则。

从 release 而非从板子推导它的两个后果：

- `padd` 会被重启，尽管 `deploy/updater.toml` 从未提及它。那份配置列表是**追加式**的，
  不是权威式的；它只需列出 release *不*发布的 unit。
- 一块 `updater.toml` 早于某个守护进程的板子依然会重启那个守护进程。那份文件属于运维者，
  而 `install.sh` 会保留它——`configd` 曾一度因此而不被重启（`../project/install-path-gap.md`
  §4）。

每个 unit 都由它自己单独的 `systemctl restart <unit>` 调用重启，从不批处理——一条
`systemctl restart a b` 在任一 unit 未知时会整体失败，*而不*重启其中存在的那一个。systemd
报告为 `LoadState=not-found` 的 unit 会带警告跳过；存在却重启失败的 unit **会使更新失败并
将其回滚**。

## 2. `robotctl update apply` / BLE 上的 `update.apply`

`engine.rs::apply` → `apply_inner` → `stage_and_swap` → `post_swap`。第 8 步之前没有任何东西会
触及活动路径。

| # | 步骤 | 会重启什么吗？ |
|---|---|---|
| 1 | 预检：时钟、机器人已停止、无活动会话 | 否 |
| 2 | 获取 manifest，校验其签名，channel / pin / 降级 / 兼容性检查 | 否 |
| 3 | 再次预检，这次是磁盘空间（要求来自 manifest） | 否 |
| 4 | 下载到 `releases/.staging-<ver>/dl/` | 否 |
| 5 | 校验 sha256，再校验 artifact 签名 | 否 |
| 6 | 解压到 `releases/.staging-<ver>/root/`，写入 `.updater-manifest.json` | 否 |
| 7 | **`hooks/preinstall`**（cwd = 暂存树）——如果低于下限则安装 ONNX Runtime，然后为 `mediad`
     的栈运行 release 的 `scripts/setup-gstreamer.sh` | 否 |

**hook 脚本来自即将到来的 release；运行它的代码则不是。** `updaterd` 在更新中途从不重启
自身（§1），所以本表每一步都由那个*已经在运行的* `updaterd`——即*上一个* release 的——执行。
因此对 hook 的运行、记录或限界方式所做的改动，要到一次 apply 之后、即 `updaterd` 已重启到
携带该改动的 release 之后的第一次 apply 才生效。这曾耗费了一轮令人困惑的“hook 跑了却什么都没
记录”；`cat /run/updaterd/identity.json` 能告诉你即将运行你的 hook 的是哪个构建。

| 8 | 将暂存树 `rename` 为 `releases/<ver>/` | 否 |
| 9 | 武装 boot counter（`pending.json`），在交换*之前* | 否 |
| 10 | 交换 `current` → `releases/<ver>` | 否 |
| 11 | **`hooks/postinstall`**（cwd = `releases/<ver>`） | **启动新发布的 unit** |
| 12 | `on_apply` —— 对 §1 的每个 unit 逐一 `systemctl restart` | **configd, mediad, netpadd, padd,
robotd, tofd** |
| 13 | `releases/<ver>/bin/updaterd --self-test` | 否 |
| 14 | 健康门：通过 socket 轮询 `robotd`，每 500 ms 一次，最多 30 s | 否 |
| 15 | 确认 boot counter，清理旧 release | 否 |
| 16 | 释放更新锁，安排延迟重启 | **安排 updaterd, btd** |
| 17 | 应答发给 `robotctl` / app | — |
| 18 | 第 16 步之后约 5 秒 | **updaterd, btd** |

从第 11 步起的任何失败都会回滚：把 `current` 换回去，针对上一个 release 重新运行
`on_apply`，确认 boot counter，并记入日志。第 1–10 步失败时旧 release 仍处于活动状态，
无需撤销。

**第 2 步也可能就此结束，且未必什么都不重启。** 如果 manifest 指名的就是已安装的 release，
操作在此停止并报告 `already_current`——不下载、不交换。在应答之前，它会针对那个 release 的
unit 做 §5 的读取，把任何运行着别的东西的 unit 列进 `stale`，并像第 16–18 步安排延迟那对
一样安排它们。所以对一台已是最新的机器人执行 `apply` 是惰性的，而对其守护进程与已安装版本
不一致的机器人执行 `apply` 则不是。对一块板子已激活的 release 执行 `select` 也一样，理由相同。

### 第 11 步详解——`hooks/postinstall`

这个 hook 打包在已签名的 artifact 内，以 release 目录作为 cwd 运行。按顺序：

1. 把每个 `systemd/sysusers.d/*.conf` `install` 到 `/usr/lib/sysusers.d/`，然后运行
   `systemd-sysusers`。先账号后 unit，因为一个指名了缺失 `User=` 的 unit 会启动失败，
   看起来像坏掉的守护进程。
2. 把每个 `systemd/*.service` `install` 到 `/etc/systemd/system/`，覆盖。**全部**，
   包括 `updaterd.service` 和 `btd.service`——这个 hook 没有排除列表，也不需要。
3. `systemctl daemon-reload`。
4. 对每个带 `[Install]` 段的 unit 执行 `systemctl enable --now`（§1.1）；没有的则安装后
   不动，而 `.timer` 在 enable 时不带 `--now`。

第 4 步正是 §1 的排除项仍然成立的原因：`--now` 意味着*启动*，而启动一个已在运行的 unit
是空操作。它不会重启 `updaterd` 或 `btd`。它确实会做的是启动一个板子从未有过的 unit——这正是
该 hook 的全部意义，也意味着**一个首次引入的守护进程会在引入它的 release 上启动两次**：
一次在这里，然后第 12 步再一次。

hook 内的每一次失败都是警告，唯独一个 unit 或 sysusers 文件的 `install` 失败会以非零退出，
从而使更新回滚。一个不肯启动的服务被刻意视为非致命：`btd` 在一块无线电尚未出现的板子上
失败是合情合理的。

### 第 13 步——自检

`updaterd` 在更新中途不重启自身，所以一个无法启动的替换二进制，否则要到提交之后的下一次
启动才会被发现，而恢复逻辑就住在那个正在失败的进程里。取而代之的是用 `--self-test` 运行
*新*的 `bin/updaterd`：它加载 `/etc/robot/updater.toml`，构造引擎，并在触碰任何状态之前退出。
上限 10 秒。

失败是携带该二进制最后一行 stderr 的 `Error::SelfTest`，并回滚。没有 `bin/updaterd` 的
release（一个模型组件）平凡通过。除非 `on_apply` 是重启，否则完全跳过——这正是让它不挡
bootstrap 安装路的原因。

### 第 16–18 步——延迟重启

`engine.rs::schedule_deferred_restarts`，先是 `updaterd`，再是 `btd`：

```
systemd-run --on-active=5s --timer-property=AccuracySec=100ms -- systemctl restart <unit>
```

四个承重细节：

- **一个瞬时 unit，而非子进程。** `systemctl restart updaterd` 会杀掉 `updaterd` 的整个
  cgroup，而 `updaterd` 的子进程*就在*那个 cgroup 里——它会在重启自己父进程的中途被杀掉。
  一个 `systemd-run` timer 住在它之外。
- **先释放更新锁**（第 16 步，在 spawn 之前）。fork 会复制每一个打开的描述符，所以持锁
  spawn 会把副本交给子进程。
- **在 `Applied` 上，以及在带过时 unit 的 `already_current` 上**——`engine.rs::restarts_owed`
  决定是哪个，而且只有它决定。前者欠那固定的一对；后者欠 §2 发现的任何运行着错误 release
  的东西，通常是 `updaterd` 自己。回滚什么都不欠：它让驻留的 `updaterd` 已经与 `current`
  匹配，在那里重启只是折腾却无事可修。
- **每个 unit 都走 timer**，包括更新本会在第 12 步就地重启的那些。为 `configd` 和 `robotd`
  再开一条立即路径，能省五秒钟，却要搭上一个机制。
- **失败只记录，从不返回。** 一个成功的更新不会因为一次重启安排失败就被报告成失败；代价
  是一个守护进程以旧二进制运行到下一次启动。

timer 是在应答写出*之前*安排的（引擎运行在 `update.apply` 调用内部），所以那 5 秒从第 16 步
起算，而不是从客户端看到答案起算。客户端先看到它的结果，随后是一次断连——对 BLE 来说，
一次普通的重连。

### 重启后的 `updaterd` 接下来做什么

这是一次普通启动，所以它完整地走 §4 的启动序列：`clean_staging`、`record_boot`、§5 的对账
——没有发生的 `btd` 重启正是在那里被逮到——然后开始服务。有两件容易漏掉的事随之而来：

- **boot counter 数的是 `updaterd` 的启动次数，而非字面上的启动次数。** 一次成功的 apply 在
  第 15 步确认了自己的 trial，所以没有未完成的要推进——但一个由*更早*那次被打断的更新留下的
  武装 trial，会被这次重启推进。
- **周期性检查的时钟重置。** `INITIAL_CHECK_DELAY` 从进程启动起算为 60 秒，`check_interval`
  从那里起算。

## 3. 其它转换

`select`、`rollback` 和 `reset-to-golden` 共享 `engine.rs::transition_to`：武装、交换、
`on_apply`、健康门、失败时回退。它们**不**运行 hook，也**不**自检——没有提取任何 artifact，
所以没有新东西可安装或证明。

| | hooks | `on_apply` (§1) | 自检 | 延迟的 updaterd/btd 重启 |
|---|---|---|---|---|
| `update apply` | 是 | 是 | 是 | **是** |
| `update select <ver>` | 否 | 是 | 否 | **是** |
| `update rollback` | 否 | 是 | 否 | **否** |
| `update reset-to-golden` | 否 | 是 | 否 | **否** |
| 失败 apply 内的自动回滚 | 否 | 是 | 否 | **否** |
| 启动时的 boot-counter 回退 | 否 | 是 | 否 | **否** |

底部的两个 `否` 由构造保证正确：在失败的 apply 和 boot-counter 回退里，`updaterd` 与 `btd`
从未被重启，所以它们仍是属于被返回的那个 release 的二进制。

显式 `rollback` / `reset-to-golden` 的那个 `否` 是另一种情况，值得了解：如果被离开的 release
曾在某个更早的时间点成功应用过，那么 `updaterd` 与 `btd` *曾经*被重启到它上面，而一次显式
回退会把它们留在那里。`robotd`、`configd` 和 `padd` 移回；这两个则留在激活 release 的*前面*。

这会在下一次 `updaterd` 启动时自行解决，而不是在一次重启时：§5 做双向比较，所以运行着比
`current` 更新 release 的 `btd` 在同一测试下同样算过时，会被重启。`updaterd` 是唯一一个在那里
不自我修复的进程，这是设计使然。

因为 `select` 和各回退跳过 `hooks/postinstall`，它们也从不*移除* unit 文件。降级到一个早于
某守护进程的 release，会留下那个守护进程已安装的 unit，指向旧 release 并不包含的二进制；
该 unit 失败，而既然它在重启集合里，失败的重启又使转换失败（`../project/install-path-gap.md`）。
留下的 unit 文件还必须能与 `current` 现在指向的旧二进制配合：`padd.service` 只在激活的
release 发布 `netpadd` 时才问 `padd --should-run`，因为旧 `padd` 没有这个标志，而一个以 2
退出的条件是一次跳过、不是失败——否则一次回滚会让机器人没有任何 pad，而且无处说明。

## 4. 启动时

systemd 从 `multi-user.target` 启动各守护进程，而 `robot-boot-check.timer` 把恢复检查武装到
180 秒处。各守护进程之间**没有顺序**，除了 `padd` 和 `netpadd` 在 `robotd` 之后（建议性的——
各自在 socket 缺失时每 5 秒退出并重试一次）、以及 `btd` 在 `dbus`/`bluetooth` 之后。没有谁
等待 `updaterd`，而 `updaterd` 除了 `network-online.target`（是 `Wants` 而非 `Requires`）之外
不等待任何东西。

在这块板子上，`hci0` 要到上电后约 73 秒才存在，所以 `btd` 会为适配器重试而不是失败。

`updaterd` 自己的启动，按顺序（`main.rs::serve`）：

1. 以 `warn` 级别记录启动身份行——版本、修订、**`exe` 路径**、pid。`exe` 路径告诉你进程实际
   来自哪个 release 目录。
2. 加载 `/etc/robot/updater.toml` 和可信密钥。任一失败都是致命的。
3. 构造引擎。
4. `--self-test` 在此返回，在触碰任何状态之前。
5. `Engine::recover_on_start`，在 socket 开始服务**之前**，所以一台启动进坏 release 的机器人，
   在别的东西能请它做其它事情之前，就已经开始回退了：
   - 对每个组件 `clean_staging`：删除 `releases/.staging-*`。
   - `record_rescue`：如果 `<state_dir>/rescued` 存在，说明 boot 恢复网在无任何东西运行时把
     `current` 换成了 golden。把它作为一次回滚抄入更新日志，**清除该组件的 trial**，并删除
     该文件。特意放在 `record_boot` 之前：rescue 已经比 trial 走得更远，而一个仍武装着的
     trial 会在一两次启动之后把 `current` 从 golden 移回。删除文件也正是释放 rescue 的循环
     守卫。
   - `refresh_golden_links`：把每个组件配置的 golden 发布为紧挨着 `current` 的一个 `golden`
     符号链接，好让 rescue 用一次 `readlink`、无需解析器就能找到它。
   - `record_boot`：对 `pending.json` 里每个武装的 trial 递增 `boots`。
   - 对每个 `boots >= 2`（`MAX_BOOT_ATTEMPTS`）的 trial：询问 `robotd`，只有当答案既非
     healthy 也非 degraded 时才回退。这次询问**轮询，时长为该组件的 `health.timeout`**——
     `updater-design.md` §8 拥有其理由，而其后果属于这里：这一步可能在 socket 开始服务之前
     坐上那个超时时长，按随附配置为 30 秒。它只在发生回退的那次启动上这样做，且只在
     `robotd` 完全不应答时。
   - 回退意味着去往 `previous`，当 `previous` 缺失、磁盘上不存在、或自身被记录为已回滚时
     升级到 `golden`。这意味着交换符号链接、确认 trial、并重新运行 `on_apply`——**所以一次
     boot-counter 回退会重启 `configd`、`padd` 和 `robotd`。** 它不运行 hook，也不安排任何
     东西，所以 systemd 刚启动的 `updaterd` 和 `btd` 进程仍以被放弃的 release 继续运行。第 6 步
     随后逮到 `btd`，因为它在此之后运行，并与回退所激活的 release 比较；`updaterd` 被报告并
     留下，直到下一次启动前一直过时。
   - 把结果记入日志。这里的失败被记录，服务继续：拒绝服务会移除唯一能修复它的途径。
6. `Engine::reconcile_running_units` —— §5。
7. `--check-only` 在此返回。注意它*执行*了第 6 步的恢复和对账两者——它是运维工具，不是
   探针，它能够回退一个 release 并重启守护进程。
8. 服务于 `/run/updaterd.sock`。
9. 若设置了 `check_interval`，则启动调度器：启动后 60 秒第一次，之后每隔一个周期。
   `auto_apply` 决定在无客户端附着时它可以安装什么。

trial 只存在于一次 apply 在 §2 的第 9 步到第 15 步之间被打断时——一次断电、一次 `kill -9`、
或交换之后一次被取消的 RPC。一次已提交的更新不留下任何武装。而既然 `exhausted` 是
`boots >= 2` 而 `arm` 写入 `boots = 0`，那么中断后*第一次* `updaterd` 启动会记录“update still
on trial”，而*第二次*才回退。

**上面每一步都预设一个能启动的 `updaterd`。** 在启动时还有一件事，应对它全都跑不起来的情形：
`robot-boot-check.timer` 在 180 秒处触发，询问 `robotd`、`configd`、`btd` 和 `updaterd` 是否
起来了，如果没有就交接给一个把 `current` 换成 golden 并重启的 `/bin/sh` rescue。它被刻意放在
本页每一个进程之外——[`boot-recovery-net.md`](boot-recovery-net.md) 拥有它，包括为何它是一个
截止时间而非这些 unit 上的 `OnFailure=`。

## 5. 启动对账——那些重启真的发生了吗？

`systemd-run` 返回成功只意味着创建了一个瞬时 timer。它不意味着重启运行了，也不意味着新
二进制启动了；§2 的第 16 步有意吞掉了那些失败。这留下了一条静默路径，让机器人最终运行起
一个它并没有安装的 release，而它落在没有别的东西照看的两个 unit 上。
`updater/src/reconcile.rs` 堵上了它。

### 它读什么

每个守护进程在启动时发布自己的身份——`duck_ipc_proto::publish_identity`，由每个守护进程
开篇的 `log_startup_identity!` 宏调用：

```
/run/<service>/identity.json    { service, version, revision, built_at, exe, pid }
```

`exe` 字段来自 `/proc/self/exe`，所以它经由 `current` 符号链接解析，并指名进程实际从哪个
release 目录启动。`Identity::release()` 从中解析出 `releases/<ver>` 组件。

自我发布而非从外部读取：一个进程知道自己的版本和修订，无需特权就能读取自己的 `exe`，
而读取另一个用户的则需要 root。每个 unit 里的 `RuntimeDirectory=<service>` 创建由该 unit 的
`User=` 拥有的目录，熬过 `ProtectSystem=strict`，并在 unit 停止时**被 systemd 移除**——所以
一个已停止的守护进程无法留下一个声称仍在运行的身份。

### 它决定什么

逐组件，对 `units_shipped`（§1）里的每个 unit——所以 `updaterd` 和 `btd` 也包括在内，这正是
关键——把已发布的 release 与该组件的激活 release 比较。`verdict_for` 是纯函数；所有系统调用
都被挡在它外面。

| 判定 | 何时 | 动作 |
|---|---|---|
| `Current` | 已发布 release == 激活 | 什么都不做，静默 |
| `Restarted` | 二者不同 | `systemctl restart <unit>`，warn 级 |
| `ReportedOnly` | 二者不同**且该 unit 是 `updaterd`** | 只记录，从不行动 |
| `RestartFailed` | 尝试过重启且失败 | error 级记录 |
| `Unknown` | 没有身份文件 | 什么都不做 |

三个刻意的“不行动”：

- **`updaterd` 在这里从不重启自己。** 一个在哪个 release 激活上意见不一的继任者会重启、
  再次意见不一、然后循环——而这是在唯一拥有恢复的那个进程里，所以什么都不会留下来打破
  循环。只报告是安全的：进程刚刚启动，所以任何关于它的过时，都在这段代码运行之前就已
  决定。它改由另一侧修复——见下文。
- **一个停止的 unit 就让它停着。** 没有身份文件意味着要么停止、要么太旧而无法发布，两者
  在这里读取结果相同。启动它会在每次 `updaterd` 启动时推翻任何停止它的人。
- **一个什么都没发布的守护进程不算过时。** 因为旧而重启一台机器人的守护进程，是没人要求
  的决定；下一次更新会让它们变得可回答。

比较是方向无关的——任何差异都算过时——这正是它既能覆盖被留在 `current` *前面*的 release
（由显式回退造成），也不只覆盖落在后面的。

它在每次 `updaterd` 启动时运行，所以启动时它是一次空操作：一切都从同一个符号链接启动。
代价是每个 unit 一次文件读取。

### 同样的读取，来自 `apply`

`reconcile::stale_units` 是对同样的身份文件做 `verdict_for` 却省去行动，而 `apply` 和
`select` 在它们 already-current 的路径上调用它（§2）。它存在是因为上面的那个例外：一个过时的
`updaterd` 是本模块拒绝修复的唯一偏差，而此后唯一会看它的，是某个因为守护进程看似不对而
运行 `apply` 的人。过去它回答 `already_current` 且什么都不安排，读起来像是确认了无事可修。

有两个性质使它不至于成为 §5 所防范的那个循环。它在请求时触发，而不是每次启动时触发；而且
它通过 `systemd-run` 安排，而不是在被重启的进程内部驱动 `systemctl`。它既报告也行动：这些
unit 被列进 `already_current` 的结果，所以客户端——`robotctl` 或 app——能看到哪些守护进程
没有运行机器人已安装的 release。

两种过时判定在这里都算过时（`Restarted` 与 `ReportedOnly`）；两者的区别关乎谁可以执行
重启，而不关乎错在哪里。表格所说的其余部分不变：一个停止的 unit 和一个什么都没发布的
守护进程仍然不算过时。

## 6. 首次安装

`scripts/install.sh` 在一块裸板上，按顺序（`main`）：

1. 写入 `/etc/robot/updater.toml` 和可信密钥。
2. `bootstrap_first_release`：获取一个独立的 `updaterd` 二进制并运行
   `updaterd install [--from <dir>]`。那是普通的 `Engine::apply`，只是强制了两项设置，因为在
   没有 release 的板子上它们是事实而非策略：`on_apply = none`（unit 住在正被安装的 release
   内部）和 `health = none`（没有可探测的 `robotd`）。于是 §2 在跳过第 12、13、14 步的情况下
   运行——但**第 11 步仍会运行**，而 `hooks/postinstall` 正是安装 unit、并对每个带 `[Install]`
   段的执行 `enable --now` 的地方。守护进程首次在这里启动，从 hook 内部。第 16 步同样不跳过：
   bootstrap 安装把 `updaterd` 和 `btd` 的重启安排到 5 秒后，而那时 `install.sh` 仍在运行。
3. `install_units`：再次复制 unit、`daemon-reload`，然后按顺序 `enable --now` `updaterd`、
   `robotd`、`configd`、`btd`、`padd`、`mediad`——`configd` 在 `btd` 之前，因为 `btd` 要问
   `configd` 要配对 PIN，而 `mediad` 最后，因为它的 unit `After=` 其它三个中的三个。`btd`、
   `padd` 和 `mediad` 可能失败而不使安装失败：一台没有无线电、手柄或摄像头的机器人依然更新、
   依然行走。然后 `robot-boot-check.timer` 被 enable，*不带* `--now`。与 hook 冗余且无害；它
   也是早于该 hook 的 release 的路径。release 发布而本函数不认识的 unit 会被安装、报告并
   不动。`tofd` 和 `netpadd` 被列为已知但各自没有 `enable_unit`：hook 已提前一步 enable 了
   它们，且没有谁依赖它们，所以本函数无需对顺序表态。
4. `install_token_dropin`：写入 `GITHUB_TOKEN` drop-in、`daemon-reload`，以及
   `systemctl try-restart updaterd`——仅 `daemon-reload` 会让*正在运行*的 `updaterd` 拿不到
   token，那是每块板子都会遇到的情况。

`DUCK_FORCE_REINSTALL=1` 在第 2 步之前加入 `stop_for_reinstall`：按顺序 `systemctl stop` `padd`、
`netpadd`、`tofd`、`btd`、`configd`、`robotd`、`updaterd`。交换发生时无任何东西存活，其后
没有健康门。

## 7. 诊断一次偏差

两个版本号完全可能同时合理地不同，而具体是哪一对决定了诊断。

| 观察 | 含义 |
|---|---|
| 更新后几秒内，`updaterd` 或 `btd` 落后于已安装 release | 预期——延迟重启还没触发 |
| `btd`、`robotd`、`configd`、`padd`、`netpadd`、`mediad` 或 `tofd` 与 `current` 持续不一致 | 重启没生效*且* §5 没修好——所以要么 `updaterd` 此后没重启过，要么它的重启失败了（journal，error 级） |
| `updaterd` 持续落后于它 | 延迟重启从未落地，而 §5 修不了这一个。`robotctl update apply daemon` 修复它——它报告带 `updaterd` 在 `stale` 里的 `already_current` 并安排重启。`systemctl restart updaterd` 是相同的手工修复；两种方式 journal 里都有它为何没落地的原因 |
| 任一守护进程报告 `restarting` | 它退出了，systemd 正在把它带回来。更新后几秒内，那就是重启本身。持续如此，则该守护进程根本起不来——`mediad` 在摄像头排线拔掉时就是这个状态存在的原因——而 §5 不动它，因为一个没在运行的守护进程没有身份可比对。`journalctl -u <name> -b` 有它为何退出的原因 |
| 任一守护进程报告 `failed` | 它起不来，systemd 已停止重试。同样的 journal，且与 `stopped` 不同，没人是主动选择的 |
| 任一守护进程报告 `build unknown` | 它在运行，却没发布身份：一个早于该机制的构建，或 `exec` 与它第一行之间的那个瞬间。§5 不动它。一次更新就能让旧的变得可回答 |

去问机器人，而不是推断：

```bash
robotctl health
```

`units` 块为每个守护进程打印一行，标出它的进程从哪个 release 启动，并在它与已安装版本
不一致时警告。

手工方式，从守护进程发布的文件得到同样的答案：

```bash
cat /run/configd/identity.json; readlink /opt/robot/daemon/current
```
