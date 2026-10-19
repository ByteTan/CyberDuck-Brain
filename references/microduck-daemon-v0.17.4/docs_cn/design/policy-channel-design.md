# 策略通道——来自 Hub 的策略

状态：草稿 · 日期：2026-08-31 · 负责人：pierre

ONNX 策略从哪里来，某人如何试用一个不是自己训练的策略，以及“reset”会把什么放回去。

**迄今已构建**（2026-08-31）：§3、§4、§5、§9、§9.1 和 §9.2——§7 的全部，除了覆盖社区条目的
`check`。板子上的策略可以被试用、再撤销，无需编辑文件或重启任何东西；官方集合从 Hub 获取，
而非由 release 携带；一个重新训练的步态无需守护进程 release 就能到达机器人。

**尚未成立**：`policy check` 只覆盖官方集合——一个从移动分支获取的社区策略会记录它所来自的
提交，但还没有任何东西拿它与该分支比较、以说明它已经移动了。本页其余部分均已构建。

[`updater-design.md`](updater-design.md) 的姊妹篇，后者拥有更新引擎——组件、来源、签名、
健康门、回滚——以及 [`robotd-design.md`](robotd-design.md) §2.3，后者拥有一个策略如何被
校验并运行。本页拥有的是**通道**：哪个文件填进一个 slot、谁发布了它，以及改变它的命令。
某个机制属于那两页时，本页只写一句并指向它们。

## 1. 今天的问题

九个 `.onnx` 文件全都随守护进程 artifact 一起发布，`robotd` 从
`/opt/robot/daemon/current/policies/` 加载它们。三个后果，而 `policies/README.md` 从写下的
那天起就点明了它们：

- 一次步态重训需要一个守护进程 release；
- 一次守护进程修复会重新下载 6 MB 未变动的权重；
- 一个在笔记本上训练的策略，只能通过 CI 或整个守护进程的侧载才能到达鸭子。

由此引出两个诉求，而它们**不是同一个特性**：

1. 我们自己的策略独立于守护进程版本化；
2. 某人可以试用一个不是自己训练的策略，弄清楚它好不好，然后回到原状。

第一点是一个普通组件（更新器设计的 §5.5），无需新概念。第二点是本页的主要内容，因为它是
组件模型尚未覆盖的那一个。

## 2. 三种来源，一个 slot

一个 slot——`walk`、`stand`、`sitstand`、`ground_pick`、`kick_left`、`kick_right`、
`roulade`——恰好由三种来源之一填充：

| | 来自 | 溯源 | 已签名 | 自动更新 | reset 目标 |
|---|---|---|---|---|---|
| **official** | `policies` 组件 | manifest、semver | 是 | 是，按 `auto_apply` | **是** |
| **community** | 任何其它 HF 仓库 | 仓库 + 修订 + 提交 sha | 否 | 否——仅报告 | 否 |
| **local** | 板子上的一个路径 | 无 | 否 | 否——不可知 | 否 |

来源由 HF 组织决定：**`pollen-robotics/*` 是官方，其余一切都是社区**。一个常量，一处。
它不是配置键——一个能被告诉该信任哪个组织的机器人，其“官方”徽章毫无意义。

来源驱动行为，而不只是一个标签：

- **official** 是 `policy reset` 返回的目标，也是周期性检查唯一可能自行应用的对象；
- **community 和 local** 从不被自动应用，也从不作为 reset 目标，并在它们被展示的每一处
  都带着来源——`policy list`、`policy check`，以及 `robotctl monitor` 打印的策略名，后者
  今天对共享的不过是同一个 slot 的步态只说 `walk`。

**社区策略被刻意不要求签名校验。** 引擎安装的其它每个 artifact 都对照可信密钥校验，而这是
第一个例外。理由是策略不是二进制：`robotd` 在关节钳位、跌倒→瘫软反射和 intent deadman
（[`robotd-design.md`](robotd-design.md) §2.4）之后，握有总线的唯一写句柄，而
`obs[1,61] → actions[1,14]` 门在机器人站定时拒绝一个形状错误的图。那个沙箱才是边界，不是
签名。守护进程二进制没有这样的沙箱，这正是组件路径的规则不动的原因。

## 3. slot 是配置，所以 `load` 和 `reset` 是配置编辑

`policy.walk` 在 `robotd.toml` 里已经是 `Option<PathBuf>`，未设置已经意味着“解析到该模式的
默认值”（`robotd-params/src/lib.rs`，`PolicyParams::resolved`）。`robotctl configure` 已经
通过 `toml_edit` 精确设置*或移除*它所触碰的键，而不扰动任何注释。于是：

- `policy load <slot> <x>` 写入 `policy.<slot>`，然后即时重载；
- `policy reset <slot>` 移除该键，然后即时重载；
- `policy reset` 移除全部七个——“把它放回它来时的样子”。

**`robotd` 写键，而非调用者。** `robot.loadPolicy` 在排队交换之前先把 slot 记入
`robotd.toml`，所以上面的持久性是*方法*的性质，而非恰好调用它的命令的性质。起初是反过来的
——`robotctl` 写文件、守护进程只改自己的内存——


——这就在同一个名字下悄悄给了手机本节所拒绝的那种临时模式。写入者住在
`robotd_params::edit` 里，紧挨着它所校验的 schema，而每个写入者都在其整个
read-modify-write 期间对文件加锁：现在有四个进程在编辑它，包括守护进程。

这就是整个持久化模型，而且不是新机制。一次 load 能在重启后存活，这正是
[`updater-design.md`](updater-design.md) §5.7 已经要求的：“活动模型选择”是一项**用户偏好**，
保存在更新器从不覆盖的配置文件里，并在更新*和*回滚中都得以保留。

**一个临时性的“试用到重启”模式被考虑并拒绝了。** 它读起来像更安全的默认——一个坏步态
离消失只差一次断电——但它把第二个真相来源放到了配置文件旁边，用于说明机器人正在运行什么，
而且它让 `reset` 几乎失去意义，因为一次重启本来就已经是撤销。一个持久答案加一个词就能撤销，
更容易解释、也更容易确定。它原本防范的风险改由 §5 处理，那里才是它该在的地方。

## 4. 即时重载

`robot.setMode` 已经能在运行中的 50 Hz 循环下交换每一个 ONNX 会话，而一次策略 load 就是
范围更窄的同一条路径。流程（`robotd/src/main.rs`，模式切换处理器及其在 home 姿态的完成）：

```text
IPC call        → 校验这一侧能校验的，设置一个 intent，回答 accepted
loop            → 重新解析参数，在工作线程上开始加载控制器（load + 预热），
                  并判断这次改动是否触及正在驱动的那张网络
  if it does    → 以扭矩开启让机器人回 home；在 home 处原地换入，全新
  if it does not→ 在机器人所在的任何位置换入，携带运行状态跨越过去
on success      → 发布新的策略名，恢复
```

与一次模式切换有三处不同。

**范围。** 一次 load 覆盖 `policy_params` 的一个字段并重新解析，正如模式切换设置
`policy_params.mode` 并重新解析。重建控制器会重载全部七个会话，而非变化的那一个——在
工作线程上，因为交换现在可能发生在一台移动中的机器人之下，而循环不能为加载而停转。

**只有对正在驱动的那张网络的改动才动机器人。** 针对本设计的第一份现场报告：加载一个
walk，行走，坐下，`policy reset walk`——机器人却坡升到 home 并站了起来。这次 reset 对配置
是对的，对机器人是错的：驱动它的是 `sitstand`，而 reset 没碰 `sitstand`。所以循环把这次
改动与上一 tick 迈出的那张网络比较，依据那张网络的解析路径（当某个 skill 在驱动时是整个
skill 列表，因为 skill 是它的索引）。未变，则新控制器在机器人所在的任何位置被换入，接管
旧者的 seat、skill 和 filter 状态，所以下一 tick 是同一张网络、同一个位置。变了，则机器人
先回 home，正如每次改动过去所做的那样，若 load 尚未落地就在那里等待。一次 reload 总是算作
对正在驱动的任何东西的改动：它存在的意义，就是路径未变、字节已变的情形，所以路径无法
作答。

**一个已经为真的请求不做任何工作。** 重置一个没人覆盖过的 slot，或加载一个 slot 已经在
运行的文件，都被回答为一次什么都没排队的接受——与 `robot.setMode` 对“已经处于该模式”给出的
答案相同，理由也相同：调用者要的是一个状态，且已经拥有它。没有这个，一台未被触碰的机器人
上的 `policy reset` 会送它回家、重载七个网络、再回到它原本就在运行的东西，读起来像故障。

唯一永远不会“已经完成”的 slot，是带着错误的那一个。一个启动时失败的覆盖已在内存里被丢弃
（§5），所以它读作未被覆盖、运行默认值——与一个没人碰过的 slot 不可区分。重置它是错误和它
背后那行配置得以清除的方式，所以那个请求必须照常处理。

**失败绝不能让你失去原本的步态。** 今天，一个构建失败的控制器会留下 `policy_error`、
健康变为 unhealthy、机器人保持姿态——对一次模式切换是对的，它是对机器人如何配置的声明；
对一次试用是错的，试用的定义就是投机。试用一个 51-D 文件不该让机器人没有步态、直到有人
重启它。所以一次 load 在新控制器失败时**保留正在运行的控制器**，并把原因返回给调用者。
产出原因的那个门已经存在，且已经在任何东西上线前于 home 姿态运行；这里说的是如何处置它的
错误。

## 5. 一个坏覆盖绝不能门控守护进程更新

持久化有一个尖锐的棱角。一个在试用时能加载的社区或本地策略，可能在*下一次启动*失败——
文件被删除、库被清空、板子在配置恢复的情况下被重刷。`robotd` 随后会在每次启动都报告
**unhealthy**，而 `updaterd` 的健康门会为一个任何 release 都没有造成的原因回滚守护进程
release：一个由缺失文件造成的更新循环，而那个文件是任何更新都无法供应的。

修复它的规则：

- 一个加载失败的 **community 或 local** 覆盖回退到该 slot 的 **official** 默认值，并报告
  **degraded**，指名该文件；
- 一个加载失败的 **official** 策略保持 **unhealthy**。

这正是 `HealthResult::degraded` 存在所区分的。一个缺失的覆盖是板子的性质，而非被门控的
release 的性质——回退守护进程无法修复它，只会徒增 boot counter。一个坏掉的官方策略确实
是一个坏掉的包，回滚它才是正确的响应。

注意这**不会**静默修复配置。覆盖保持写入，`robotctl health` 说出哪个文件加载不了，所以
状态可见，`policy reset <slot>` 就是修复。一个在坏启动时悄悄重写配置的守护进程，比一个
说明原因的 degraded 机器人是更糟的意外。

## 6. 版本，以及检查更新的版本

一个作为“库中的一个文件”持有的社区策略没有版本、也没有可比对的对象，所以库为每个条目
保留一个小的 sidecar：仓库、修订、解析出的提交 sha、文件名、获取时间。正是那条记录让检查
成为可能，而读取它的大部分已经存在——`source/hf_hub.rs` 已经把 refs API 解析成
`RepoRefs { tags }`，`api/models/{repo}` 给出 `sha` 和 `lastModified`。

版本按发布者实际提供的东西渲染，而不是假装人人用 semver：

| 来源 | 显示为 |
|---|---|
| official | manifest 的 semver——`1.2.0`，像任何组件一样 |
| community，仓库有 `v*` 标签 | 该标签 |
| community，跟踪一个分支 | 短 sha + 日期 |
| local | `local`，且 `check` 报告 unknown |

**`policy check` 是唯一该看的地方。** 它在内部路由——官方 slot 问 `updaterd` 关于
`policies` 组件，社区 slot 问 Hub——因为“哪条命令能告诉我我的 walk 是否过时”有两个答案，
正是值得花一点路由去避免的那种困惑。

周期性检查（`check_interval`，6 小时）也覆盖社区策略，并**只报告而不应用**。这已经是
`auto_apply` 对待普通 release 的方式：可用性是要呈现的事实，安装是一个决定。它也使得 app
以后能说“你的 bouncy walk 有更新可用”，而无需新的管道。

## 7. 命令表面

```text
robotctl policy list                      slot · current · origin · version
robotctl policy check [slot]              源处是否有更新的东西
robotctl policy load <slot> <repo|name|path>
robotctl policy update <slot>             获取最新的并加载它
robotctl policy reset [slot]              回到 official；无 slot = 全部七个
robotctl policy search <query>            匹配查询的 Hub 模型
```

`load` 接受一个 HF 仓库（`org/name[@rev]`）、一个已在库中的名字，或**板子上的一个路径**。
路径形式是训练者的 `scp`-再试用的循环，也是整个特性在工作台上值得拥有的原因；它也是唯一
没有溯源的输入，这正是 `local` 在 §2 里是一等来源而非意外情况的原因。

没有单独的 `fetch`：`load` 在必要时下载，而库会累积，所以依次加载 A、B、再 A 各花费一次
下载。`search` 是对查询发起 `api/models?search=`，而在 Hub 上出现 `microduck` 标签之前，
搜索这个词就是现有的办法。它连同命中一起请求文件列表，这免费地给出了每个仓库预览片段的
名字，然后对每个有 manifest 的仓库读一个 `manifest.json` 来获取描述——
[`policy-manifest.md`](../policy-manifest.md) 拥有这两者及其运行的边界。

## 8. 线上协议，以及它的代价

`robotd` 上两个新方法：`robot.loadPolicy`（slot + 来源，像 `robot.setMode` 一样应答——接受、
带原因拒绝、或空操作）和 `robot.policies`（每个 slot 在运行什么，带来源和版本）。
`updaterd` 上一个新命名空间 `policy.*`，用于获取。

`API_VERSION` 从 17 → 18。`robotctl` 和 `btd` 与 `robotd` 在同一个 artifact 里发布，所以它们
一起动；一个钉在 v17 的客户端会收到指名该方法名的 `METHOD_NOT_FOUND`，这是设计好的偏差
行为，而非握手拒绝（[`duck-ipc-proto/src/lib.rs`](../../duck-ipc-proto/src/lib.rs)，
`API_VERSION`）。

**获取住在 `updaterd`，而不是 `robotctl`。** `robotctl/Cargo.toml` 声明了它会破坏的规则：
恢复路径上的一个支持工具不链接更新引擎的 http/tar/zstd/crypto 树，而且在结构上现在也
做不到。`updaterd` 已经有 HF URL 解析、一个 `reqwest` 客户端和 app 会想要的进度通知。代价是
诚实的——它是守护进程内部第一次无签名下载，而守护进程的不变式是没有任何无签名字节能到达
活动路径——所以 `policy.*` 命名空间**只能**写入库目录，绝不能写入组件的安装目录，这让那个
不变式在承重之处字面上成立。

库位于 `/var/lib/robot/policies/`，在所有 release 目录之外，按
[`updater-design.md`](updater-design.md) §5.7 规则 1。它为每个仓库保留两个修订——刚获取的
和它之前的那一个，即 §9.1 用于官方集合的规则——因为没有别的东西去整理它，而 `policy.fetch`
和它背后的命令都在两个无线传输层上被服务。一个机器人*正在使用*的修订，无论多旧都保留；
一台不应答的机器人什么都不清理，因为沉默意味着 `robotd` 宕机，而非一个都没在用。

## 9. 离开 artifact

`robotd` 从 `/opt/robot/policies/current` 读取它的策略，而非 release 内部。这部分已完成，且
正是关键所在：一个策略只有一个运行时来源，且没有 release 副本与其它任何东西之间的优先级
规则。

**填充那个目录的是 `scripts/seed-policies.sh`，而它会下载。** 由 `hooks/postinstall` 在每次
更新时运行、由 `install.sh` 在全新板子上运行，它把固定的集合从 Hub 获取到
`/opt/robot/policies/releases/seed-<pin>/`，并把 `current` 指向它——与更新器已经在交换的
`releases/` + `current` 符号链接布局相同（更新器设计的 §7.1）。

获取而非复制正是重点：这是 `setup-board.sh` 已用于 ONNX Runtime、`setup-gstreamer.sh` 已用于
获取而非复制正是重点：这是 `setup-board.sh` 已用于 ONNX Runtime、`setup-gstreamer.sh` 已用于插件的安排，它们是板子需要、而 release 没理由携带的另两样东西。鸭子检测器也沿着同样的路走出了 release——`scripts/seed-detector.sh`、`/opt/robot/detector/current`、`robotctl duck-detector check/update`、`[workspace.metadata.detector]` 里的一个 pin——用固定文件列表代替 manifest，因为它的两个文件有固定名字（`docs/project/npu-bringup.md`）。pin 活在 `[workspace.metadata.policies]` 里、也作为字面量活在脚本里，有一条测试断言两者一致——`setup-gstreamer.sh` 的陷阱，因为从 release 内部运行的脚本读不到 manifest。


三件它不做的事。它不重新下载已有的集合，所以 pin 未变的更新不碰网络——这很重要，因为
post-install hook 在 120 秒超时下运行，而超时的 hook 会回滚更新。它绝不安装部分下载：
一切都落到暂存目录，只有整个集合都到了，`current` 才动。而且当获取失败时，它不用更旧的
集合替换一个能用的集合——一个半发布的修订让板子留在它原有的步态上，pin 在下次更新时重试。

没有任何东西被签名，这是遵循 §2 而非违背它：这是任何人都能做的同一次下载，进入一个
`robotd` 从中形状校验一切的目录。形状门也正是逮住截断文件的地方，这就是这里不钉哈希、
以免每次重训都过期的原因。

一条规则让这成为 bootstrap 而非第二个家：**一个已经安装的集合绝不被替换，无论 pin 说什么**。
只有两种状态——装了点什么，或什么都没装——而只有第二种才获取。第一次安装之后的一切都属于
§9.1。

在某个板子证明较松的规则是错的之后，这条规则变得更严。它过去会替换更旧的 `seed-*`，理由是
守护进程更新仍是重新训练步态到达机器人的方式。`policy update` 现在才是，而旧规则成了陷阱：
一块被手工推进到 `v2` 的板子有 `current -> releases/seed-v2`，它匹配 seeder 认作自己的模式，
于是下一次无关的守护进程更新就会把 `v1` 放回去——作为一次二进制更新的静默副作用，回退了
某人所选择的步态。

**策略不再随守护进程回滚**，这值得说清楚，因为它过去是免费的。当它们住在 release 内部时，
回退 release 就回退了它们；现在 `current` 由 postinstall hook 重指，而回滚不运行 hook
（`post_swap` 只在 apply 路径上）。所以一个*因为其中某个策略是坏的*而被回滚的 release，会
让那个策略继续运行。有两条回头的路，第一条是普通的：一个向前的
`robotctl update apply daemon --version <older>` 会运行 hook 并重新 seed。若不行，release 所
替换的 seed 仍在板子上——seeder 正是为此而保留前一个——`current` 可以手工指向它。

这是有意的形态，而非退化：两样各有一行版本的东西不会一起回退。只有当 release 仍是策略的
唯一来源时它才尖锐，这也是不该让那种状态久留的另一个理由。

**一块首次安装时连不上 Hub 的板子没有步态**，这是被接受的形态而非疏忽。`robotd` 保持姿态并
报告 *degraded*，所以更新门通过、什么都不回滚；下次更新再获取。这与 `setup-board.sh` 已为
ONNX Runtime、`setup-gstreamer.sh` 已为插件达成的交易相同——一块板子需要一次网络，而这是
那时需要它的又一样东西，而非在每次守护进程 release 里永远携带六兆字节的理由。

seeder 绝不能做的是*失败*：post-install hook 的非零退出会回滚更新，所以一个网络问题会回退
一个与它毫无关系的 release。每条错误路径都以零退出，并在 stderr 上说明。

**整个集合一个组件，而非每个 slot 一个。** 更新器设计草拟了 `model-walk`、`model-jump` 等
（§5.5），而按 slot 的组件是它自己的机制最自然给出的。反对的理由是：九个文件由一次训练
运行作为*家族*产出，slot→文件的映射依赖模式（`walk` 在腿上模式是 `alpha_walking.onnx`、
在轮上模式是 `roller.onnx`，这晚于那一节），而九个组件意味着九个仓库、九份签名、九个配置块、
每次检查九次往返，以及一个九维偏差矩阵，其中没有任何东西记录某个 walk 和某个 stand 曾是
一起训练的。一个组件意味着模式切换什么都不下载，且集合按其构建方式版本化。

一个组件放弃的是回滚单个 slot，而这正是 §3 的按 slot 覆盖已经覆盖的——来自任何来源，这是
按 slot 组件也做不到的。

**一个缺失的策略是 degraded，而非 unhealthy**，按同一条 §5 规则：一块尚未获取集合的全新
供给板子没有步态，而这绝不能使此后每次守护进程更新的健康门失败。供给会安装 bootstrap
集合，在那里 `setup-board.sh` 已经安装 ONNX runtime、`setup-gstreamer.sh` 已经安装插件——
网络依赖落在存在网络依赖的地方，而在运行时一个策略恰好只有一个来源，release 副本与 Hub
副本之间没有优先级规则。

### 9.1 越过 pin

`robotctl policy check` 问 Hub 该集合的仓库提供哪些修订，对照板子上的那个。
`robotctl policy update` 安装一个——默认最新，或用 `--version v1` 回去——并告诉 `robotd`
重读它的 slot。

两者都由 `updaterd` 服务（§8），而三个细节正是为什么这不只是 seeder 的第二份副本：

**仓库来自集合，而非配置。** 每个已安装集合携带一个 `.source` 记录，指名它所来自的仓库和
修订，由安装它的任何东西写入。所以没有第二处去配置仓库、没有什么可漂移；由未来某个工具
安装的集合以同样方式回答同样的问题。

**“最新”指仓库自己的最新，而非 semver 排序。** 策略仓库没有义务使用 semver，而把
`bouncy-2` 与 `v10` 排序会是把一个猜测当作事实呈现。Hub 按最旧在前列出 refs，所以最新是它
的反序，而 `check` 打印整个列表而非只打印自己的判定——往回走和往前走同样关键。

**`robot.reloadPolicies` 存在，因为路径不会变。** 安装一个集合会在每个 slot 底下交换
`current`，所以每个仍解析到同一个字符串，`robot.loadPolicy` 会正确得出无事可做的结论。它
对路径是对的、对字节是错的，这正是需要单独方法的情形——而且是绝不被短路的那个，因为
“已经加载”正是它存在就是为了去怀疑的那个答案。

一次 reload 替换每张网络，所以当某张网络*在运动中*时它先回 home——每次扰动性改动都遵循的
规则。**一个坐着的机器人不在运动中**：对坐在工作台上的机器人执行 `policy update`，过去会
把它坡升到 home 并站起来，因为驱动的是 sitstand 网络，而 reload 把它算作被扰动。座位是一个
在恒定标志上的静态姿态，所以现在交换原地发生、座位跨越保留，正如对一张未在驱动的网络的
改动那样。

一次 reload 不改变配置，所以**你自己加载的 slot 熬得过它**。这不是细枝末节：reload 在每次
`policy update` 末尾运行，而一个悄悄丢弃某人步态实验的安装，会是在最糟的时刻才发现两者被
混为一谈。

一个不可达的 Hub 被报告，而非抛出。机器人无论如何都在走，而一个看到“could not reach the
Hub”挨着已安装内容的调用者，比一个只看到错误的调用者知道得更多。两者都不是一块什么都没
安装的板子：那不是网络故障，而通过同一个字段说它，会让一个健康的 Hub 看起来不可达。

安装会清理到当前集合和它所替换的那一个——各七兆字节，而一个比较两种步态的人会来回移动。
前一个被刻意保留：回滚不运行 hook，所以回退守护进程不回退它的策略，而把 `current` 指回
保留的集合，正是策略出错时的恢复手段。

### 9.2 别人的策略

`robotctl policy load walk RemiFabre/microduck-flamingo-cycle` 从任何 Hub 仓库获取一个策略到
`/var/lib/robot/policies/<org>/<name>/<revision>/`，并放入那个 slot。在所有 release 目录之外，
按更新器设计的 §5.7：某人选择的策略熬得过一次更新和一次回滚。

**路径是由答案构成的**，这让 `origin` 无需查找就能诚实：发布一个策略的组织是它住所的一个
组成部分，所以 `robot.policies` 对陌生人的报告 `community`、对我们的报告 `official`。它是
标签而非边界——任何能编辑 `robotd.toml` 的人都能随便命名目录，而能做到那点的人无论如何
都能运行任何东西。隔在一个坏策略与一台坏机器人之间的，是 §2 的沙箱。

**一个仓库携带一个 `policy.onnx`。** 这个约定在构建它之前就已存在于 Hub 上——每个已发布的
microduck 策略恰好有一个 `.onnx`，挨着一个 README 和一个 manifest——所以获取只取那唯一的
`.onnx`，并对带多个的仓库逐一指名而拒绝。选错意味着在真实机器人上运行错误的网络，这不是
可掷的硬币；`<repo>:<file>` 指明是哪个。

**而且它携带一个 `manifest.json`**，结果证明它是一个比本设计规定的任何东西都更丰富的约定：
`obs_len`、`action_len`、`model_api`、`robot.model`，外加 name、kind 和 description。读取它，
才能让一个策略在 800 KB 下载*之前*、在机器人被要求运行它之前被拒绝——“它的 manifest 说
观测宽度 51，而这台机器人构建 61”是 `robotd` 在加载时也会得出的同一个判定，只是它出现在
某人能据此行动的地方。这也是 `model_api` 终于有所作为的地方：更新器设计的 §5.5 规定了它，
而两侧都没用过它，现在一个需要更新守护进程的策略被拒绝，且以那为补救。

三条规则让它不成陷阱。manifest 是**不可信的**——一个陌生人对一个陌生人文件的描述——所以
它是拒绝的理由，绝不是信任的理由；一个撒谎的 manifest 会被形状门逮住，那里一直才是真正的
检查。**缺失不是证据**：一个没有 manifest、或省略了我们据以行动字段的仓库被接受，因为 Hub
的大部分并不遵循我们的任何约定，而对沉默拒绝会拒掉它的多数。而且它被对照检查的那些数字
发布在 `duck_ipc_proto` 里而非重复一份，`duck_control` 里有一条编译期断言保证两者一致——
与任何发布策略者之间的契约，该放在双方都看得见的地方。

**一个 slot 可以被关掉**，用 `none`——已经在用的字面 `[policy] <slot> = "none"`。除 `walk`
外的每个 slot，因为它是其它者回退的目标、所以不能为空；请求会被拒绝，而一个已经这么写的
配置会在启动时被修复并报告 degraded。不是事后补的：任何人会试用的第一个社区策略都会做它
自己的双脚站立，而 `will_stand` 在命令幅值为零时把机器人交给站立网络，而那恰恰是它双脚
站立时处于的状态。没有说 `stand none` 的办法，运行它就意味着编辑那份这整条命令存在就是
为了停止编辑的文件。

`robotctl policy search microduck` 列出外面的东西，标记每个命中的来源，下面带发布者的一行
描述，当仓库有片段时带一个链接。没有标签过滤：一个共享的名字是已发布策略的共同点，而标签
值得等到有东西可标时再加。

描述正是列表值得读的原因：每个命中都是某人的 `microduck-<something>`，而一个名字不足以在
一个火烈鸟循环和一段粗糙的走之间做选择。没有它，弄清一个策略做什么的唯一办法就是安装它。

### 9.3 集合描述它自己

策略仓库根部的 `manifest.json` 说明集合包含什么、每个策略是什么。它**只**住在 Hub 上——
本仓库里没有副本，是有意为之。这里的一份副本会是某个在 Hub 上版本化的东西的第二真相来源，
而针对它的测试钉住的是副本、而非机器人下载的那个东西，这比没有测试更糟：它通过的同时，
一块板子拿到的是别的东西。

两份列表过去是硬编码的，而两者加起来意味着集合里的第十个策略是一个守护进程 release 而非
一个标签：

- `scripts/seed-policies.sh` 知道要下载哪些文件。它现在从 manifest 读取，保留它知道的九个
  作为对早于 manifest 存在的修订的 fallback。
- `robotd-params` 知道哪些策略是一次性 skill、每个运行多久。它现在从 manifest 取，同样
  fallback。

**而 `policy.install` 随集合安装 manifest**，来自同样的两条规则。它起初从板子上已有的东西
取下载列表，这让第十个策略成为一个 seeder 认、`robotctl policy update` 却不认的标签——只能
靠守护进程 release 安装，而这正是整个通道存在就是为了阻止的。它还让新集合没有
`manifest.json`，于是 `robotd` 回退到它编译进去的三个 skill 名，集合声明的每个逐 skill 数字
被那条唯一职责就是在修订间移动的命令静默丢失。现在清单就是该修订自己的 manifest，文件被
写进集合、挨着它所描述的策略；板子自己的 `.onnx` 名仍是对没有 manifest 的修订的 fallback。
一个不是普通名字的 `file` 会被丢弃而非获取，在脚本和守护进程里都是——它是别人的文档，
而它选择了路径。

**逐字段契约是 [`../policy-manifest.md`](../policy-manifest.md)**——两个轴、每个字段、每个
在机器人上改变什么。这里不重复，本节只是该机制存在理由的推理。

值得重申的一件事，因为它是该机制的*目的*：`kind` 和 `command.encoding` 是两个轴，而一个
episodic 策略只有在一个常量命令上才成为一个 skill。一个 `phase` 或 `posture_flag` 条目为
守护进程驱动的某个 arm 贡献时机——这就是为什么一个周期更长的重训 ground pick 是一个标签、
而非一个守护进程 release。

**逐策略字段与单策略仓库所用的相同**，外加 `file`。这正是让对社区发布者的请求变成“加这些
字段”而非“采用我们的格式”的原因，也意味着一个读取器能理解两种形态。

## 10. Skill：机器人能被要求做的事

一个 slot 是机器人*默认*运行的东西。一个 **skill** 是它被要求时运行的东西——踢、roulade，
以及同形状的任何其它东西。

加一个过去要碰七个地方：`Skill` 枚举、`Net`、`PolicyPaths`、一个 `Policy` 字段及其 `has_*`、
一个带配置键和注册表条目的 `Slot`、控制循环里的一个分支，以及 padd 的按钮表。所以一个形状
与 roulade 完全相同的社区策略——零命令、四秒、选中它即触发——没有守护进程 release 就无法
添加。

它之所以是七个地方，是因为它本不必是一个。这两条 arm 是同一 arm：

```rust
} else if let Some((left, _)) = self.kick {
    let net = if left { Net::KickLeft } else { Net::KickRight };
    (net, Command::default(), label)
} else if self.roulade.is_some() {
    (Net::Roulade, Command::default(), "roulade")
```

踢和 roulade 在四个数字上不同——时长、动作缩放、增益比、以及按住按钮是否链接另一个——
而别无其它。所以它们是四个数字：

```toml
[[policy.skill]]
name = "polite-bow"
path = "/var/lib/robot/policies/fffiloni/microduck-polite-bow-b1d864/main/policy.onnx"
duration = 4.0
```

`robotctl policy add polite-bow <repo>` 写入它，从仓库的 manifest 取时长。缺失意味着内建的
三个，而条目按名合并，所以一块板子能在无配置、无迁移的情况下更新到它——而添加一个也不会
因遗漏而静默移除另一个。`"none"` 移除一个内建，与关掉一个策略 slot 的是同一个词。

### 10.1 谁来提供结尾

manifest 里的 `kind` 不是关于一个策略运行多久。它是关于**谁结束它**。

一个 *episodic* 策略自己回到安全姿态——`polite-bow` 四秒后又站好了——所以窗口可以干脆到期，
步态接管一台站直的机器人。一个 *perpetual* 策略则持续到被告知否则为止，而对它到期会把步态
交给一台单脚平衡的机器人。所以一个 skill 可以声明两半：

```toml
command  = [1.0, 1.0, 0.0]   # 它运行时的 twist——flamingo 读 [flag, side, 0]
unwind   = [0.0, 1.0, 0.0]   # 它在返回路上驱动什么
unwind_s = 3.0
```

那是守护进程提供该策略所没有的结尾，也是 sit toggle 已有的形态：锁存，然后在交回之前一次
定时升起，因为放下标志并不会瞬间让机器人站起。两者都默认为无，所以常见情形两者都不声明。

每个 skill 的 head 和 body 都被置零，无论阶段如何——迄今发布的每个一次性都声明它们未使用，
而用 `zero_command_padding` 训练的策略期望的正是如此。只有 twist 不同，且对大多数 skill 它
也是零。

**一个 skill 的 twist 由构造保证不平滑。** EMA 作用于*客户端*的命令在其进入的途中，而一个
skill 从不读那个——循环构建一个全新的命令块。这就是为什么一个读标志而非速度的策略作为
skill 不需要 `cmd_alpha`，而它挤在 walk slot 里时需要全局设置它。

### 10.2 什么留在守护进程里

`walk` 和 `stand` 是回退对，由命令幅值选择，而它们之下没有可交回的东西。`sitstand` 被锁存，
并由关机的坐下在内部驱动，而非只靠一个按钮——用 manifest 的话说，`scripted`。`ground_pick`
写入一个 phase 而非一个常量。四个中没有一个是一般化的一次性，而一个集合条目不得响应
`ground_pick` 或 `sit_toggle`——这两个名字背后的第二张网络会被喂一个它从未训练过的全零命令。
守卫既在编码上也在名字上：一个 `phase` 或 `posture_flag` 条目无论如何被命名都绝不作为 skill
加载。集合对这几条 arm *确实*能说的是它们的时机（§9.3）——pick 的周期和截断、升起和座位的
settle——因为那些是已训练网络的性质，而非构建的性质。

**一个运行中的 skill 在其整个时长里关闭跌倒反射**：limp-fall 预测器只在控制器不 `busy()`
时才被询问，而任何活动 skill 都让它 busy。当每个一次性都在一秒以内时，这无可争议。一个可
配置为持续十秒的 skill，是一台十秒没有跌倒反射的机器人，而这是要决定而非继承的。

### 10.3 按钮

`[pad]` 说明六个一次性按钮中哪个运行哪个 skill：

```toml
[pad]
x = "polite-bow"
```

`robotctl pad bindings` 显示它们，`pad bind` 改一个，`pad reset` 放回去。默认 A 坐下、B 捡起，
X、Y 和肩键空闲，而 `padd` 在一秒内重读文件——什么都不重启。

只有那六个——面按钮和肩键。`Start`、D-pad 的摇杆模式和按住 `Select` 不是 `robot.do` 调用，
而关掉机器人的那个按钮，正是那个不该能被一次配置编辑弄丢的绑定。

**在线上，`pad.bindings` 和 `pad.bind` 是 `robotd` 的**，而非 `configd` 的，后者拥有 `pad.*`
的其余部分。配对关乎无线电；绑定关乎一个按钮对机器人做什么，而回答它需要这台机器人的
skill 列表，好让一个名字被拒绝而非成为一个死按钮。全程按方法路由——`policy.*` 去
`updaterd`，而 `robot.loadPolicy` 对同一个概念去 `robotd`——所以这个分裂在机制上零成本，
只值得点明是因为命名空间暗示着相反。

一个绑定对照 `do_names` 检查，它是**不是** `policy.skills`：`ground_pick` 和 `sit_toggle` 有
cascade 里自己的 arm，而非配置条目，所以它们不在这份列表里，却是完全可以请求的。只对照
`skills` 校验，会拒绝 pad 发布绑定的五个按钮中的两个，一个测试逮到了这一点。

**skill 表也可在两个传输层上触达**，这闭合了路径：获取一个陌生人的策略，给它一个名字和
时长，用那个名字请求它，放到一个按钮上——没有一步需要在机器人上开终端。`robot.setSkill`
在一次调用内写配置并重载，而 `robot.skills` 单独报告 `built_in`，因为 `ground_pick` 和
`sit_toggle` 不是表条目，一个只读表的客户端会得出它们不存在的结论。

`padd` 仍对 skill *是什么*一无所知。它读哪个按钮按下、查它旁边的名字、发送那个名字；
`robotd` 决定机器人是否有这么个东西，并在没有时用它所拥有的列表应答。检查该在答案所在的
地方，这就是 `robotctl pad bind` 问机器人、并用真实列表拒绝一个拼写错误的原因。

### 10.4 从手机触达它

`robot.do`、`robot.policies`、`robot.loadPolicy` 和 `robot.reloadPolicies` 在 **BLE 和 WebRTC
两者**上都被服务。改变的不是其中任何一项的安全性——形状门、关节钳位和跌倒反射无论谁请求
都一样——而是现在有了一个客户端，而这正是这些拒绝中的每一个所说的它在等待的东西。

**一个 skill 不是 teleop。** `robot.do` 曾一度与 `robot.move` 及其同类一起被归在 BLE 的传输
论证下——20 字节的通知预算，和启动后最初约 73 秒不存在的链路。那个论证关乎一个*流*：每秒
五十个小更新。一个 skill 是一次请求，而且它完全不需要控制链路，因为 deadman 自己把 twist
置零，而一台没有东西驱动的机器人会站住并鞠躬。

**BLE 是最满足“观看条件”的传输层**，这里的大多数拒绝都以它为条件。它的无线电覆盖约十米，
所以按下按钮的人按构造与机器人同处一室——而 bond 是用 `encrypt_authenticated_write` 做了
PIN 检查的。WebRTC 以不同方式回答同样的条件：peer 正在看视频，这是已经允许 `robot.init`
和 `robot.shutdown` 的论证，而且它覆盖步态比覆盖站起更好，因为 peer 正看着那个步态即将
移动的东西。

值得记住的不对称与直觉相反。**BLE 是经过认证的；WebRTC 不是**——`remote-webrtc.md` 的 §4
意味着任何局域网 peer 都继承所开放的一切，而 BLE 携带来自十米内一个 PIN-bonded 调用者的
同一调用。那是让 §4 更锐利的理由，而不是从一个能向某人展示结果的传输层扣下该调用的理由。

**从手机选择的步态，是机器人启动进去的步态。** `robot.loadPolicy` 自己写 slot 键，所以 §3
的持久化模型属于*方法*，而非调用它的命令。它起初不是——`robotctl` 写文件、守护进程只改
自己的内存——这意味着同样的两个词因谁请求而含义不同，并给了远程调用者 §3 考虑过并
**拒绝**的那种临时“试用到重启”模式。撤销是同一个不带路径的调用，从同一部手机可达。

**`robot.policies` 携带 skill 列表**，那是让其余可用的那一块。一个客户端若不知道机器人有
一个鞠躬，就无法提供鞠躬，而哪些 skill 存在现在就是配置——无需编译进去。这些名字已经在
`robot.subscribe` 的确认里，但那是回答一个只问一次的问题的 50 Hz 流，而 BLE 刻意不路由它。

## 11. 官方集合目前是什么

记录在这里，因为既然这些文件不在了，它在本仓库里别无居所，也因为映射无法从 Hub 上的名字
恢复。从 `apirrone/microduck_runtime` 的 `5f3b314`（`roulade.onnx` 在 `7e4ab6d`，它首次出现的
地方）复制，解引用该仓库用来给特定训练运行以稳定名字的符号链接：

| 在集合中 | 上游 | 角色 |
| --- | --- | --- |
| `alpha_walking.onnx` | `BEST_alpha_walking_rough.onnx` | 走 / velstand（set v5 之前的默认 `walk`） |
| `velstand.onnx` | `pollen-robotics/microduck_rl` `velstand_best.onnx`（2026-09-14） | 走 + 零命令站立；set v5 起的默认 `walk`，`stand` 未设置 |
| `alpha_stand.onnx` | `BEST_alpha_stand_body_control.onnx` | 站立 + 身体姿态 |
| `alpha_sitstand.onnx` | `BEST_alpha_sitstand.onnx` | 坐 ↔ 站（posture flag） |
| `alpha_ground_pick.onnx` | `alpha_ground_pick.onnx` | 地面捡起（phase command） |
| `ball_kick_left.onnx` | `ball_kick_left.onnx` | 左腿踢 |
| `ball_kick_right.onnx` | `ball_kick_right.onnx` | 右腿踢 |
| `roller.onnx` | `BEST_roller.onnx` | 轮式模式移动 |
| `roller_crouch.onnx` | `BEST_roller_crounch.onnx` | 轮式模式蹲（ground-pick slot） |
| `roulade.onnx` | `roulade.onnx` | 前滚翻（Mjlab-Roulade-MicroDuck） |

（`roller_crouch` 也修掉了上游文件名的拼写错误。）

**名字是角色，而非训练运行**，而那个间接性正是重训是 pin bump、而非每台机器人上的配置
改动的原因：换掉哪个运行是“步行策略”，绝不能意味着编辑 `robotd.toml`。

**只有 61-D 家族。** 原型也发布一个 51-D 的——`3 gyro + 3 gravity + 42 joints + 3 command`，
旧式 `[vx, vy, vtheta]`——而 `robotd` 在加载时拒绝它，并指名两个宽度。那个检查把一个
wrong-policy 错误变成了一次诊断，而非一台以没人能解释的方式移动的机器人，而它现在也是
逮住截断下载的那个东西（§9）。

## 12. 命名

更新器说 `model`；`robotd` 说 `policy`。统一为 **policy**——组件是 `policies`，命名空间是
`policy.*`，命令是 `robotctl policy …`。`models/` 对真正是模型、而非控制策略的东西保留其
含义，比如 `pet_detect.onnx` 和鸭子检测器。

## 13. 已记录的决定

| | |
|---|---|
| load 和 reset 是配置编辑外加一次即时重载 | 机器人运行什么的一个真相来源；reset 无需新概念（§3） |
| 无临时试用模式 | 第二真相来源，且它让 `reset` 与一次重启冗余（§3） |
| 一次失败的 load 保留运行中的控制器 | 试用的定义就是投机；它绝不能让你失去原有步态（§4） |
| 一个已满足的请求不排队任何工作 | 否则未被触碰机器人上的 `reset` 是一个十秒的非事件（§4） |
| …除非是带错误的 slot | 一个已回退的 slot 看起来未被触碰，而清除它正是 reset 的用途（§4） |
| 一个失败的社区覆盖是 degraded，而非 unhealthy | 否则一份过时配置门控每次守护进程更新（§5） |
| `pollen-robotics/*` 是官方，硬编码 | 一个可配置的信任组织让徽章失去意义（§2） |
| 社区策略不做签名校验 | 安全层和形状门才是边界，而非密钥（§2） |
| 一个 `policies` 组件，而非每个 slot 一个 | 集合是作为家族训练的；按 slot 覆盖已经覆盖其余（§9） |
| 获取住在 `updaterd` | `robotctl` 不得链接 HTTP 栈（§8） |
| 策略住在 release 之外，由它 seed | 一个运行时来源，且没有会搞错的优先级规则（§9） |
| seeding 从不覆盖它没有安装的集合 | 交接无需标志：第一次真正安装就结束了它（§9） |
| 集合是下载的，不是发布的 | 与 ONNX Runtime 和插件相同；bump pin 就是发布一个步态（§9） |
| 一次失败的获取保留已安装的集合 | 一个半发布的修订不得降级一个能用的步态（§9） |
| pin 是最低值；`policy update` 越过它 | 一个指名更新集合文件的默认值，不得在落后板子上回滚守护进程；步态仍无需守护进程 release（§9.1） |
| 一个集合记录它所来自的仓库 | 一个写入者、一份副本，无需配置两次或漂移（§9.1） |
| Reload 是第三样东西，不是 reset-all | 它们从外部看起来一样，混为一谈会丢弃每个覆盖（§9.1） |
| 一次性 skill 是配置，而非代码 | 踢和 roulade 是同一 arm 的不同数字；一个社区的是第五个集合（§10） |
| `kind` 说谁结束一个策略，而非多久 | 一个 episodic 的会自己返回；一个 perpetual 的需要守护进程把它驱回；一个 scripted 的是 episodic 但可打断（§9.3、§10.1） |
| `command.encoding` 说守护进程喂它什么 | Constant → 一个 skill；`phase` → ground pick；`posture_flag` → 坐↔站。只有第一个可作为一次性加载（§9.3） |
| 集合携带它自己的时机 | pick 的周期和截断、升起和座位的 settle 是已训练网络的性质，而非构建里的字面量；`[policy]` 键仍优先（§9.3） |
| skill 不碰 walk、stand、sitstand 和 ground_pick | 回退对，一个内部驱动、一个 phase 驱动——没一个是一般化的一次性（§10.2） |
| 按钮是配置；Start 和 Select 不是 | 停下机器人的那个按钮正是那个不该能弄丢的（§10.3） |
| `robot.do` 不是 teleop，所以 BLE 可以携带它 | 一次请求，而非流；而且它无需控制链路，deadman 自己置零 twist（§10.4） |
| 加载一个策略在两个传输层上都被服务 | 每次拒绝都说它在等一个客户端；现在有一个了（§10.4） |
| 守护进程写 slot 键，而非调用者 | 否则同样的两个词因谁请求而含义不同（§3、§10.4） |
| 每个 `robotd.toml` 的写入者都加锁 | 四个进程编辑它，两个同时暂存就是一个写入者的一半被改名就位（§3） |
| 从 Hub 安装也被服务 | uid 门授权的是 `btd` 的凭据，而非手机的，正如它对 `update.apply` 已经做的（§10.4） |
| 一个集合的 manifest 随它安装 | 否则 `policy update` 丢掉集合声明的每个 skill，列表永远长不大（§9.3） |
| 库为每个仓库保留两个修订 | 没有别的东西整理它，而获取在两个无线传输层上都被服务（§8） |
| …但绝不是机器人正在用的那个，若它不应答则什么都不留 | 沉默是一个宕机的 `robotd`，而非一个都没用的（§8） |
| seeder 从不替换已安装的集合 | 否则一次守护进程更新静默回退用 `policy update` 选的步态（§9） |
| 来源是路径中的 org | 无需查找即诚实，且是标签而非边界（§9.2） |
| manifest 可以拒绝但绝不恩准 | 它是陌生人的声称；形状门才是检查（§9.2） |
| 一个带两个策略的仓库是拒绝 | 猜意味着在真实机器人上运行错误网络（§9.2） |
| seed 被清理到当前和前一个 | 每 release 7 MB 的无界增长；前一个是回滚后的手工恢复（§9） |
| 一个 `policy check`，内部路由 | 一个问题两条命令，正是值得避免的困惑（§6） |

## 14. 刻意推迟

- **一个社区策略的多个竞争版本。** 一个 slot 持有一个文件；库持有许多，但只有配置说哪个是
  活的。一个真正的 `(name, version)` 存储键在 [`updater-design.md`](updater-design.md) §17
  定价为约 14 个文件和一次 `API_VERSION` bump，而 A/B 情形由依次加载每个来服务。
- **一次加载整个集合。** 目前仅按 slot。若在训练运行家族间切换成为常规，集合才有意义，
  而把一次运行的 walk 与另一次的 stand 混在一起，是没人训练过的组合。
- **Hub 上的 `microduck` 标签。** 在有东西可标之前，搜这个词就够了。
- **给社区策略签名**，以及任何会要求它的精选组织方案。

## 15. 开放

- **一个运行中的 skill 在其整个时长里没有跌倒反射**（§10.2）。对半秒的踢没问题；一个配置
  为持续十秒的 skill 就是十秒没有它。
- **`robotctl configure` 列出 skill 表却无法编辑它。** 一个重复表不是带光标位置的键，所以
  编辑器改为指向 `robotctl policy`。教会 registry 重复段是其中最大的一块，却回报最少。
- **官方集合是否有一天成为一个更新器组件。** 投递已不需要：§9 直接从 Hub 获取固定集合，
  与板子其它前提条件的到达方式相同。所以这是一个关于组件会*增加*什么的问题——回滚、pin、
  golden、已知坏历史与周期性检查——对照一个与策略完全无关的代价。组件路径安装的每个
  artifact 都由安装守护进程二进制的同一份代码无条件做签名校验。以那种方式投递的官方集合
  因此必须被签名，不是因为策略需要签名，而是因为那条路径从未有过无签名模式，给它一个会把
  洞扩大到远超策略。若按集合回滚结果成为任何人会伸手的东西，值得重新审视；今天什么都不
  需要它。
- app 是否呈现其中任何东西，以及它需要 §7 的多少。这里的一切都可通过 `btd` 已经在转发的
  同一 socket 触达，所以答案是 UI 问题，而非协议问题。
- 每次 load 的七会话重建在板子上是否可测地糟糕（§4）。它对模式切换是现状，所以在任何工作
  之前这需要一次测量。
