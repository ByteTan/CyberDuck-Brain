# 策略清单（policy manifest），schema 2

microduck `.onnx` 旁边的 `manifest.json` 说了什么，以及机器人对每个字段做什么。一套词汇表覆盖两种形态：**单策略仓库**（Hub 上的 `<user>/microduck-<name>`，一个 `policy.onnx`，字段在顶层）和**官方策略集**（`pollen-robotics/microduck-policies`，十个文件，同样的字段在 `policies` 下每个条目重复一次）。一个读取器理解两者，而向发布者要东西时永远是「加一个字段」，从来不是「改用我们的格式」。

`pollen-robotics/microduck_rl` 里的 `uv run publish` 会从一个检查点或 ONNX 文件写出一个符合规范的单策略仓库。`robotctl policy load <slot> <repo>` 和 `robotctl policy add <name> <repo>` 读取它。`docs/design/policy-channel-design.md` §9 有背后的推理；这一页是契约。

## 两个轴

**`kind` 说明谁让策略结束。**

| `kind` | 含义 | 变成 |
| --- | --- | --- |
| `episodic` | 运行 `duration_s` 秒后自己回到安全姿态 | 一个技能，如果它的命令是常量 |
| `perpetual` | 一直运行直到另有指令——一种步态，或一个必须由人来结束的姿态 | 一个槽位的步态（`policy load`）；或者，带 `unwind_s`，一个带 `--hold` 的技能 |
| `scripted` | 是 episodic 但可中断：守护进程可以在中途改变它的命令 | 已记录；守护进程自己的手臂读取它的时序 |

**`command.encoding` 说明守护进程喂给它什么。**

| `encoding` | 要点 | 谁 |
| --- | --- | --- |
| 缺省或 `constant` | 该时间窗内一个固定的 twist，返回途中为 `idle` | 每个踢球、滚翻、迄今每个社区一次性动作 |
| `phase` | `[cos 2πφ, sin 2πφ, 0]`，φ 在 `period_s` 秒内从 0 递增，在 `end_phase` 交还 | 地面拾取；滚轮下蹲 |
| `posture_flag` | 一个槽位携带 `sit` 或 `stand` | 坐↔站 |

只有常量命令的 `episodic` 策略才能成为通用的一次性动作。一个 `phase` 或 `posture_flag` 策略会被 `policy add` 拒绝，它属于守护进程驱动的槽位（`policy load ground_pick …`、`policy load sitstand …`）。守护进程依据编码来把关，而不是依据名字。

## 字段

除了一个集合里的 `file`，其余全部可选。缺失的字段不是证据：机器人只会在某个「存在且错误」的声明上拒绝一个策略。

| 字段 | 类型 | 读取方 | 含义 |
| --- | --- | --- | --- |
| `schema_version` | int | 展示 | `2`。是 1 的超集；没有任何东西据此把关 |
| `model_api` | int | 拉取 | 策略所需的守护进程 API；若比守护进程的新则拒绝 |
| `obs_len` | int | 拉取 | `61`；若与机器人不一致则拒绝 |
| `action_len` | int | 拉取 | `14`；若不一致则拒绝 |
| `robot.model` | str | 拉取 | `microduck`；若是其他机器人则拒绝 |
| `robot.hw_rev`、`robot.servos`、`robot.control_hz` | | 展示 | `1`、`xl330`、`50` |
| `name` | str | 技能 | 客户端索要的名字；默认取文件的主干名 |
| `description` | str | 展示 | 一行、不可信——`policy search` 在每个命中下显示的内容，以及 `policy fetch` 读入时显示的内容 |
| `kind` | str | 技能、槽位 | 见上 |
| `duration_s` | float | 技能 | 它运行的秒数；对 phase 策略是 `period_s × end_phase` |
| `chain` | bool | 技能 | 按住按钮时，本次结束后再启动一次 |
| `action_scale` | float | 技能 | 它运行期间自己的输出缩放 |
| `unwind_s` | float | 技能、坐站 | 在交还前驱动 `command.idle` 的秒数；对坐↔站来说，是起身 |
| `ramp_s` | float | 坐站 | 座椅安定下来所需的秒数；关机的坐姿等这个时间加 1 秒，然后缓入休息姿态 |
| `mode` | str | 策略集 | `walk`（默认）或 `roller`；一个 phase 条目属于哪种模式的地面拾取 |
| `slot` | str | 展示 | 对 perpetual 步态：它对应的槽位（`walk`、`stand`、…），于是 `policy load <slot> <repo>` 就是安装命令 |
| `entry_pose` | str | 展示 | 策略预期从什么姿态开始，例如 `standing` |
| `command.encoding` | str | 技能 | 见上 |
| `command.idle` | [3] | 技能 | 表示「停止做这件事」的 twist |
| `command.period_s`、`command.end_phase` | float | 地面拾取 | 相位周期 |
| `command.sit`、`command.stand`、`command.slot` | | 展示 | 标志位的取值以及它挂在哪个槽位 |
| `command.twist`、`command.head`、`command.body` | 散文 | 展示 | 每个块对人意味着什么 |
| `training` | 对象 | 展示 | `task_id`、`repo`、`commit`、`branch`、`dirty`、`run`、`checkpoint`、`exported` |
| `eval` | 对象 | 展示 | 自由格式：检查了什么、在哪个仿真里、表现如何 |
| `policies[]` | 数组 | 策略集 | 每个文件一个条目，含 `file` 以及上面任何按策略而定的字段 |

循环 LSTM 导出要求 `model_api: 2`；现有的前馈导出仍是 API 1。张量契约、记忆生命周期与离线预演见[循环策略](recurrent-policies.md)。

## 单策略仓库

```json
{
  "schema_version": 2,
  "model_api": 1,
  "obs_len": 61,
  "action_len": 14,
  "robot": { "model": "microduck", "hw_rev": 1, "servos": "xl330", "control_hz": 50 },
  "name": "polite-bow",
  "kind": "episodic",
  "duration_s": 4.0,
  "chain": false,
  "entry_pose": "standing",
  "description": "Bows from a two-foot stand and comes back up.",
  "command": { "encoding": "constant", "idle": [0, 0, 0],
               "twist": "unused (zeros)", "head": "unused (zeros)", "body": "unused (zeros)" },
  "training": { "task_id": "Mjlab-PoliteBow-Flat-MicroDuck", "repo": "pollen-robotics/microduck_rl",
                "commit": "0bf9897", "branch": "bow", "dirty": false,
                "run": "pollen-robotics/mjlab_microduck/abc123", "checkpoint": 3000,
                "exported": "2026-09-02T14:05:00Z" }
}
```

该仓库恰好携带一个 `.onnx`，命名为 `policy.onnx`。`robotctl policy add polite-bow <user>/microduck-polite-bow` 读取 `duration_s`、`chain`、`action_scale`、`command.idle` 与 `unwind_s`，在 `obs_len`、`action_len`、`model_api`、`robot.model` 或非常量编码上拒绝，并写入技能。`RemiFabre/microduck-flamingo-cycle` 是一个已发布的 `perpetual` 示例：没有 `duration_s`，因此必须带 `--hold`，而它的 `command.idle` 就是 unwind 所驱动的内容。

## 官方策略集

同样的字段，在 `policies` 下每个条目重复一次，外加 `file`。线上副本是 `https://huggingface.co/pollen-robotics/microduck-policies/blob/main/manifest.json`；该集合的 `phase` 条目设置每种模式的地面拾取时序，它的 `scripted` 条目设置坐↔站的时序，这就是为什么一个周期更长、重新训练过的拾取动作只需打一个标签，而不需要发布一次守护进程。

每个 `file` 都是纯文件名——没有目录、没有前导点号。如果一个策略集的 manifest 里出现了带路径的名字，播种器和 `robotctl policy update` 都会跳过该条目。

manifest 会被安装**进策略集本身**，因此 `/opt/robot/policies/current/manifest.json` 就是 `robotd` 读取技能所用的文件。它同时也是下载清单：添加一个策略就是在这里加一条和一个标签，而一块开发板的首次播种以及之后的每次 `policy update`，都从它们正在安装的那个修订版里取这份清单。

## 预览视频，它不是一个字段

一个仓库可以携带一个**策略运行视频**，`robotctl policy search` 会把它链在描述旁边。它是路径而不是 manifest 字段，因为发布者们本来就这么做了，而[策略游乐场](https://huggingface.co/spaces/pollen-robotics/microduck-policy-playground)本来也这么读——要一个 `preview` 字段，等于让大家把路径已经表达过的东西再写一遍。

把它放在 **`media/preview.mp4`**。这是大多数已发布策略所用的位置，也是最先查找的位置。对于一个放了别的东西的仓库，完整顺序是：

1. `media/preview.mp4`
2. `preview.mp4`
3. 任意目录中文件名以 `preview` 开头的 `.mp4`、`.webm` 或 `.mov`
4. 任意的 `.mp4`、`.webm` 或 `.mov`
5. 任意的 `.gif`

同一步内平局时，取路径最浅者，再取名字最短者，再按字母序，这样一个每个变体各有一个视频的仓库——`10cm/preview.mp4` 旁边有 `100cm/preview.mp4`——每次搜索都得到同一个答案，而不是 Hub 碰巧先列出的那个。

第 4 步就是为什么一个仓库若唯一视频是某个观看器的录屏，就会显示它：发布者选择随仓库发布的视频胜过没有视频，而这里的任何东西都无法判断视频里是什么。一个宁愿什么都不显示的仓库，就应该不带任何视频，或把它留在 Hub 仓库之外。

搜索从 Hub 已经返回的文件列表里读取它，所以预览不产生额外的请求。而描述确实要为每个有 `manifest.json` 的仓库花一次 GET——有上限，超过几秒就丢弃而不是等待，因为一份策略列表与其迟来，不如平淡。

## 与 schema 1 的变化

Schema 1 是官方策略集最初的样子：`kind` 的取值为 `perpetual`、`episodic`、`scripted`，其中 `scripted` 意为「守护进程生成命令」，并应用于地面拾取。Schema 2 把这个含义移到 `command.encoding`，让 `scripted` 表示「可中断的 episodic」（坐↔站），并新增 `chain`、`ramp_s`、`mode`、`entry_pose`、`training`、`eval` 以及 `command.*` 时序字段。一个读取 schema-1 文件的守护进程会得到原型对一切所用的时序，以及同样三个技能；不会因版本而拒绝任何东西。
