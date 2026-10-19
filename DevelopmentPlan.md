# 🦆 CyberDuck-Brain 项目规划清单
> **项目愿景**：基于 vLLM 推理加速与 Agent 插件化架构，为 Open Duck Mini（桌面级双足仿生机器人）打造多模态具身智能大脑。
> **核心目标**：沉淀“大模型部署+异步调度+Agent编排+具身控制”的全栈工程能力，打造高壁垒的技术护城河，并探索商业化/开源影响力。
> **适用人群**：具备机械、图像算法、Python/C++、Docker/Celery 背景的资深工程师（业余时间推进）。

---

## 🗺️ 总体架构设计 (The "Cloud-Edge-Device" Pattern)

1. **云端/上位机 (The Brain)**：运行 vLLM (Qwen2-VL) 提供多模态推理，运行 Agent 状态机 (LangGraph/PydanticAI) 进行任务规划。
2. **边缘网关 (The Nervous System)**：基于 Celery + Redis 构建异步任务队列，处理高并发视觉请求与长耗时推理调度。
3. **端侧/实体 (The Body)**：运行 Open Duck Mini 底层控制 (C++/Python)，以及轻量级 CV 模型 (MediaPipe/YOLO) 实现低延迟本能反应。

---

## 📅 详细执行路线图 (Roadmap)

### Phase 1: 基础设施与“数字孪生” (纯软件基建)
**目标**：不碰硬件，在纯软件环境跑通 vLLM 推理与 Celery 异步调度闭环。
**时间安排**：第 1 - 3 周 (约 15-20 小时)

| 任务模块 | 具体行动 | 核心技术 / 第三方库 | Git 参考 / 文档链接 |
| :--- | :--- | :--- | :--- |
| **vLLM 部署** | 使用 Docker 部署 Qwen2-VL-7B-Instruct，暴露 OpenAI 兼容 API。配置显存优化参数。 | `vLLM`, `Docker`, `NVIDIA Container Toolkit` | [vllm-project/vllm](https://github.com/vllm-project/vllm) |
| **异步网关** | 搭建 Celery Worker，编写 FastAPI 接口接收图像/文本请求，推入 Redis 队列。 | `Celery`, `Redis`, `FastAPI`, `Pydantic` | [celery/celery](https://github.com/celery/celery) |
| **虚拟具身** | 编写一个 `VirtualDuck` Python 类，模拟接收动作指令并打印日志（模拟物理延迟）。 | `Python`, `asyncio` | 无 (自研) |
| **联调测试** | 发送一张桌面图片 -> FastAPI -> Celery -> vLLM 识别 -> 返回 JSON 动作指令 -> VirtualDuck 执行。 | `OpenAI Python SDK` | [openai-python](https://github.com/openai/openai-python) |

**🏆 阶段里程碑**：在终端看到大模型准确识别图片，并通过 Celery 成功将指令下发给虚拟鸭子。

---

### Phase 2: 躯壳唤醒与底层控制 (硬件组装)
**目标**：让物理世界的鸭子站起来，跑通官方策略，并开放底层控制 API。
**时间安排**：第 4 - 6 周 (约 20-30 小时，含等待物流时间)

| 任务模块 | 具体行动 | 核心技术 / 第三方库 | Git 参考 / 文档链接 |
| :--- | :--- | :--- | :--- |
| **硬件采购** | 采购 BOM (STS3215舵机x14, 树莓派Zero2W, BNO055 IMU, 电源管理等)。 | 淘宝/立创商城 | [Open_Duck_Mini BOM](https://github.com/apirrone/Open_Duck_Mini) |
| **结构制造** | 找淘宝代打 (PETG材料) 或自打 3D 结构件。用 SolidWorks 魔改一个顶部摄像头快拆支架。 | `SolidWorks`, `PETG`, `FDM` | [OnShape CAD](https://cad.onshape.com/documents/64074dfcfa379b37d8a47762) |
| **组装与标定** | 压入热熔螺母，组装机械结构。使用 Feetech 上位机统一舵机 ID 和波特率。 | `Feetech SDK`, `电烙铁` | [Tnkr 组装指南](https://tnkr.ai/explore/docs/open-duck-mini/open-duck-mini-v2) |
| **固件与Runtime** | 烧录 Raspberry Pi OS Lite，部署 `Open_Duck_Mini_Runtime`，跑通官方 RL 走路策略。 | `Raspberry Pi OS`, `Python` | [Open_Duck_Mini_Runtime](https://github.com/apirrone/Open_Duck_Mini_Runtime) |
| **API 封装** | 编写 C++/Python 中间件，将上层的 `(x,y,z)` 或 `动作Token` 转化为底层舵机控制指令。 | `C++`, `pyserial`, `gRPC/MQTT` | 自研 |

**🏆 阶段里程碑**：鸭子能在桌面上稳定站立并行走，且能通过自定义 API 接收外部指令。

---

### Phase 3: 赋予本能与视觉交互 (传统 CV 介入)
**目标**：实现极低延迟的“本能反应”，不依赖云端大模型，提升交互趣味性。
**时间安排**：第 7 - 9 周 (约 15 小时)

| 任务模块 | 具体行动 | 核心技术 / 第三方库 | Git 参考 / 文档链接 |
| :--- | :--- | :--- | :--- |
| **边缘端 CV** | 在树莓派/Jetson上部署轻量级人脸/手势检测模型，输出归一化坐标。 | `OpenCV`, `MediaPipe` / `YOLOv8-Fast` | [google-ai-edge/mediapipe](https://github.com/google-ai-edge/mediapipe) |
| **ONNX 加速** | 将 CV 模型导出为 ONNX 格式，使用 ONNX Runtime 在边缘端进行硬件加速推理。 | `ONNX`, `ONNX Runtime` | [microsoft/onnxruntime](https://github.com/microsoft/onnxruntime) |
| **本能控制** | 编写控制逻辑：检测到人脸 -> 计算偏移量 -> 控制鸭子颈部/身体舵机实现“眼神跟随”。 | `Python`, `PID控制算法` | 自研 |

**🏆 阶段里程碑**：您用手在鸭子面前晃动，鸭子能实时、流畅地转头盯着您的手（延迟 < 100ms）。

---

### Phase 4: 接入赛博大脑 (Agent 闭环编排)
**目标**：实现“自然语言 -> 视觉理解 -> 任务规划 -> 物理执行”的完整 Agent 闭环。
**时间安排**：第 10 - 13 周 (约 20 小时)

| 任务模块 | 具体行动 | 核心技术 / 第三方库 | Git 参考 / 文档链接 |
| :--- | :--- | :--- | :--- |
| **Agent 框架** | 引入 LangGraph 或 PydanticAI，构建 `Perception -> Planning -> Execution -> Verification` 状态机。 | `LangGraph` / `PydanticAI` | [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) |
| **Skill 插件化** | 借鉴 DeepSeek Harness，用 YAML 定义 `find_object`, `navigate_to`, `push_object` 等 Skill Manifest。 | `YAML`, `Pydantic` | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) |
| **闭环容错** | 执行动作后，再次拍照让 VLM 验证是否成功。若失败，LangGraph 触发重试或回退逻辑。 | `LangGraph Checkpointing` | LangGraph 官方文档 |
| **全链路联调** | 语音/文本输入“把红色方块推下桌子” -> 鸭子转头寻找 -> 规划路径 -> 执行 -> 验证。 | `FastAPI`, `WebSockets` | 自研 |

**🏆 阶段里程碑**：鸭子能理解复杂的自然语言指令，自主完成桌面物品的交互，并在失败时尝试纠正。

---

### Phase 5: 数据飞轮与垂直微调 (模型进化)
**目标**：收集专属数据，微调 VLM，提升鸭子在特定桌面场景下的“智商”。
**时间安排**：第 14 - 16 周 (约 15 小时)

| 任务模块 | 具体行动 | 核心技术 / 第三方库 | Git 参考 / 文档链接 |
| :--- | :--- | :--- | :--- |
| **数据采集** | 编写脚本，自动记录鸭子第一人称视角的“图像 + 用户指令 + 成功/失败状态”三元组。 | `Python`, `SQLite/JSON` | 自研 |
| **数据清洗** | 筛选高质量数据，构建符合 VLM 微调格式的 JSONL 数据集。 | `Pandas`, `JSON` | 自研 |
| **LoRA 微调** | 使用 LLaMA-Factory 对 Qwen2-VL 进行 LoRA 微调，使其更懂“桌面具身场景”和“动作 Token”。 | `LLaMA-Factory`, `LoRA`, `DeepSpeed` | [hiyouga/LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) |
| **模型替换** | 将微调后的权重合并，替换 Phase 1 中的 vLLM 基础模型，对比效果提升。 | `vLLM` | vLLM 官方文档 |

**🏆 阶段里程碑**：微调后的模型能更精准地识别桌面上的反光/异形零件，动作规划成功率提升 30% 以上。

---

### Phase 6: 产品化、开源与商业探索
**目标**：沉淀技术资产，扩大技术影响力，探索变现可能。
**时间安排**：第 17 周及以后 (持续进行)

| 任务模块 | 具体行动 | 核心技术 / 第三方库 | Git 参考 / 文档链接 |
| :--- | :--- | :--- | :--- |
| **一键部署** | 编写 `docker-compose.yml`，实现云端大脑和边缘端环境的一键拉起。 | `Docker Compose`, `Bash` | 自研 |
| **开源发布** | 整理代码，编写高质量的 README 和架构图，发布至 GitHub。 | `GitHub`, `Markdown` | 您的个人仓库 |
| **内容输出** | 录制极具视觉冲击力的 Demo 视频，发布至 B站/小红书/YouTube。撰写技术沉淀博客。 | `剪映/PR`, `知乎/掘金` | - |
| **商业探索** | 1. 提供“AI 鸭子软件刷机/定制行为包”服务。<br>2. 包装为“高校具身智能实验台”方案。<br>3. 接相关技术外包/咨询。 | 闲鱼/淘宝/微店 | - |

---

## 💡 核心心法与避坑指南 (必读)

1. **绝对不要手搓机械本体**：您的机械背景是优势，但业余时间画 CAD、搞减速机、调 PID 会耗尽热情。**买现成的开源臂/鸭子，把精力留在“大脑”和“视觉”上。**
2. **警惕 vLLM 的显存陷阱**：多模态模型处理高分辨率图像时显存暴增。在边缘端或消费级显卡部署时，务必限制 `--max-model-len` 并调整 `gpu_memory_utilization`。
3. **Agent 的“幻觉”控制**：大模型经常会输出物理上无法实现的动作。必须在 Agent 和底层控制之间，加一层**基于运动学正解的安全校验拦截器**（这是具身智能区别于纯软件 Agent 的核心壁垒）。
4. **保持“最小阻力”原则**：平时工作忙，不要强求每天推进。每周哪怕只花 2 小时跑通一个小脚本（比如让 vLLM 识别一张图），也要保持“正向反馈”。**有趣，是坚持的唯一理由。**

> *"AI 取代的是翻译需求为代码的初级工作，但无法取代设计高可用、可扩展、易维护的系统架构的能力。"*