<div align="center">

# ISE-Trace

**An agentic data pipeline, smarter and broader.**

*Intent → Simulate → Execute：一个三阶段范式，为训练 OS Agent 合成多样、多轮、且真实执行落地的数据。*

[English](README_en.md) · 简体中文

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](#)

</div>

- 📄 **论文:** [arXiv:2606.11520](https://arxiv.org/abs/2606.11520)
- 🛠️ **流水线（总入口）:** https://github.com/Valiere01/ISE-Trace
- 🧩 **Stage 1 — 意图构造:** https://github.com/NairongZheng/intent_creator
- ⚙️ **Stage 2+3 — 模拟 + 执行:** https://github.com/NairongZheng/openclaw_gen_data
- 🤗 **数据集:** https://huggingface.co/datasets/valiere/ISETrace

---

## 概述

训练一个能干活的 OS Agent，需要的数据要同时具备三个属性：**结构化的用户意图**、**多轮任务委派**、**真实落地的工具执行**——而现有数据集几乎都缺这些。多数合成流程从 API 目录出发反推任务（任务分布因此镜像工具空间，而非用户真正想要什么），是单轮的，并且用模拟代替真实的工具调用。

**ISE**（**I**ntent → **S**imulate → **E**xecute）是一个三阶段合成范式，一次性解决这三个 gap：

| 阶段 | 名称 | 做什么 |
|:----:|------|--------|
| **I** | 4D 意图构造 | 在 `Persona × Domain × Task × Complexity` 上采样结构化意图，让训练分布由「用户需求的组合」塑造，而不是由 API 是否存在决定。 |
| **S** | 多轮模拟 | 通过一个 **role-locked 用户模拟器** 驱动用户–Agent 交互，四条行为约束用于抑制 *role drift*（角色漂移）与 *state hallucination*（状态幻觉）。 |
| **E** | 执行落地 | 把**每一个**工具调用都放进一个隔离的真实 OS workspace 里跑，得到真实的「失败–恢复」动态，而非模拟出来的响应。 |

最终得到的语料 **ISETrace** 包含 **23,132** 条多轮轨迹（平均 **8.12** 轮用户回合 / **68.24** 轮总对话回合，每条轨迹 **29.26** 次工具调用），由 **43,956** 条去重后的结构化意图构建而成。

> 在 ISETrace 上微调 Qwen3-8B，把 **ClawEval** pass@1 从 **19.3 提升到 37.7**（+18.4 个点，1.95×），同时超过 GPT-4o 零样本参考和 4× 规模的 Qwen3-32B base。

<div align="center">
  <img src="assets/pipeline.png" width="920" alt="ISE 三阶段范式总览">
  <br>
  <em>ISE 范式总览：每个阶段把先前工作的典型失败模式（上，✗）与 ISE 的做法（下，✓）对照。</em>
</div>

---

## 整条流水线

ISE-Trace 是总入口（umbrella）项目。整条流水线按阶段拆成两个子仓库：

```
   intent_creator              openclaw_gen_data
  +-------------------+        +-------------------+        +-----------+
  | [1] Intent        | intents| [2] Simulate      |        |           |
  |                   | .jsonl | [3] Execute       |        |  ISETrace |
  | Persona x Domain  |------->| role-locked sim   |------->|  23,132   |
  | x Task x Complex  |        | + real OS exec    |        |  轨迹     |
  +-------------------+        +-------------------+        +-----------+
        Stage I                   Stage S + E                  output
```

> **[1] Intent** — `intent_creator`：在 `Persona x Domain x Task x Complexity` 上采样 4D 结构化意图。
> **[2] Simulate** + **[3] Execute** — `openclaw_gen_data`：role-locked 多轮模拟，每个工具调用在真实 OS 上隔离执行。
> 产出 **ISETrace**：23,132 条多轮、执行落地的轨迹。

| 仓库 | 角色 | 链接 |
|------|------|------|
| **`intent_creator`** | **Stage 1。** 基于 Domain 的 4D 意图生成。采样 `Persona × Domain × Task × Complexity`，并由 LLM 把每个结构化元组渲染成一条自然语言用户意图。 | [github.com/NairongZheng/intent_creator](https://github.com/NairongZheng/intent_creator) |
| **`openclaw_gen_data`** | **Stage 2 + 3。** 驱动本地 [OpenClaw](https://github.com/openclaw/openclaw) Agent 进行多轮、role-locked 的用户模拟；把每个工具调用放到真实 OS 上、在隔离的 worker workspace 中执行；归档完整 session 并转换成可训练的 *OpenAI format*。 | [github.com/NairongZheng/openclaw_gen_data](https://github.com/NairongZheng/openclaw_gen_data) |

---

## 核心特性

- **Intent-first，而非 tool-first。** `Persona × Domain × Task × Complexity` 的采样空间超过 10¹¹ 种组合——长尾与跨域意图是被「采样」出来的，而不是从一张 API 列表反推出来的。
- **Role-locked 多轮模拟。** 四条行为原则——*perspective lock*（视角锁定）、*register matching*（语域匹配）、*incremental advancement*（增量推进）、*responsive conditioning*（响应式条件化）——让用户模拟器不会漂移成 assistant 口吻、也不会凭空臆想执行状态。发布语料中实测的角色漂移率：**0.02%**。
- **真实执行，真实失败。** 每个 `exec` 都调用真实 shell、产生真实的 stdout/stderr/退出码；文件操作落到真实文件系统。轨迹里带有真实的凭证失败、非零退出和恢复过程——而不是模型的自我报告。
- **规模化的 workspace 隔离。** 每个 worker 在从共享快照模板恢复出的隔离 workspace 中运行，把存储从 O(N) 降到 O(1) per worker。支持并发与断点续跑（resume）。
- **可量化的多样性。** 随语料一并给出全栈多样性度量——embedding（Vendi）、词汇（Distinct-N）、结构（工具调用拓扑）。

---

<div align="center">
  <img src="assets/landscape.png" width="760" alt="ISETrace 在并发 Agent 数据格局中的位置">
  <br>
  <em>ISETrace 在并发 Agent 数据格局中的位置：横轴=平均对话回合，纵轴=轨迹数（对数）；气泡面积=每条轨迹工具调用数，色相=环境落地方式。</em>
</div>

## ISETrace 速览

| 统计量 | 数值 |
|--------|----:|
| 结构化意图（去重） | 43,956 |
| 完整轨迹 | 23,132 |
| 平均用户回合 / 轨迹 | 8.12（中位 8，最大 23） |
| 含 6–10 个用户回合的轨迹占比 | 91.1% |
| 平均总对话回合 | 68.24（最大 565） |
| 平均工具调用 / 轨迹 | 29.26 |
| 平均独立工具数 / 轨迹 | 4.69 |
| 平均 Domains × Tasks × Tools | 2.35 × 4.40 × 3.18 |
| 复杂度（complex / medium / simple） | 50% / 40% / 10% |
| 全池 Vendi Score（mpnet-base-v2, cosine, q=1） | 61.57 |

轨迹覆盖 **10 个功能域**（Intelligence-Core、Code-Runtime、File-IO、Source-Chain、Automation-Flow、Web-Extraction、Comms-Social、Multimedia、Agent-Memory、System-Infrastructure），跨 47 个行业的 **965** 个不同 persona。

---

## 快速开始

两个阶段独立运行——Stage 1 产出一份 `intents.jsonl`，由 Stage 2+3 消费。每个子仓库都有自己更详细的 README；下面只展示它们如何衔接。

### Stage 1 — 生成意图（`intent_creator`）

```bash
git clone https://github.com/NairongZheng/intent_creator
cd intent_creator
pip install -r requirements.txt

# 配置 LLM endpoint
cp .env.example .env          # 填入 OPENAI_API_KEY 和 OPENAI_BASE_URL

# 生成意图（异步，推荐）
python main.py --count 1000 --async --max-concurrent 50 --output intents_out
```

这会把结构化意图写到 `intents_out/intents.jsonl`，每行包含一条 `natural_language_intent` 以及它采样出的 domain / task / persona / complexity。

### Stage 2 + 3 — 生成并落地轨迹（`openclaw_gen_data`）

```bash
git clone https://github.com/NairongZheng/openclaw_gen_data
cd openclaw_gen_data
pip install -r requirements.txt
cp config/config.yaml.example config/config.yaml

# 初始化 agent workers（捕获真实的 runtime 工具 + system prompt）
python scripts/init_agents.py --num-agents 4 --force-recreate --refresh-tools

# 用 Stage 1 的意图驱动多轮模拟 + 真实执行
INTENTS_FILE=../intent_creator/intents_out/intents.jsonl \
python scripts/run_generation.py --concurrent 4
```

输出：
- 原始 session → `output/sessions/`
- 可训练的 OpenAI format → `output/middle_format/`（目录名沿用 `middle_format`，内容即 OpenAI messages 格式）

> **前置条件：** Stage 2+3 需要本机有可用的 [OpenClaw](https://github.com/openclaw/openclaw)（或其 Docker 部署）作为执行底座。搜索 provider 配置与容器部署见 `openclaw_gen_data` 的 README。该范式与具体 Agent 系统无关——任何支持「真实工具执行 + workspace 隔离」的平台都可作为执行底座。

---

## 实验结果

在 **ClawEval** 上评测（pass@1，%，在每个 run 都被打过分的 114 个 *T* 类 Agent 工具使用任务的共同集合上）：

| 系统 | pass@1 | Completion | Robustness |
|------|:------:|:----------:|:----------:|
| Qwen3-8B（base, 0-shot） | 19.3 | 0.367 | 0.925 |
| Qwen3-32B（base, 0-shot） | 30.7 | 0.446 | 0.947 |
| GPT-4o（0-shot） | 25.4 | 0.442 | 0.965 |
| **SFT-ISETrace（8B, ours）** | **37.7** | **0.533** | 0.959 |

微调后的 8B 模型超过 GPT-4o 参考（+12.3 点）与 4× 规模的 Qwen3-32B base（+7.0 点）。增益主要来自任务完成度，而在扰动工具输出上的鲁棒性仍维持高位。一个把轨迹截断成单轮用户回合的消融把 pass@1 拉到 28.1（−9.6 点），说明多轮模拟贡献了增益中相当大的一部分。

---

<div align="center">
  <img src="assets/cross_corpus_turns.png" width="640" alt="跨语料轨迹深度对比">
  <br>
  <em>跨 14 个 Agent 语料的轨迹深度（平均总回合）：ISETrace 以 68.24 回合大幅领先，为第二名的 2.68×。</em>
</div>

<div align="center">
  <img src="assets/coverage.png" width="820" alt="ISETrace 覆盖度分析">
  <br>
  <em>覆盖度分析。左：5,000 条意图的 t-SNE 投影，10 个域充分重叠而非孤立成簇；右：Vendi 随样本量从 40.7 增长到 61.6（全池），接近但未饱和。</em>
</div>

## 仓库结构

```
ISE-Trace/                    ← 本仓库（总入口 / umbrella）
├── README.md                 #  中文（本文件）
├── README_en.md              #  English
└── LICENSE

intent_creator/               ← Stage 1（独立仓库）
├── main.py                   #  入口
├── domains/                  #  10 个功能域定义
├── libraries/                #  全局 persona / tool / skill 池
├── src/{builders,generators,libraries,models,validators}/
└── config.yaml

openclaw_gen_data/            ← Stage 2 + 3（独立仓库）
├── scripts/run_generation.py #  主入口
├── scripts/init_agents.py    #  worker 初始化 + runtime probe
├── src/                      #  agent runtime、session parser、converter ...
├── data_examples/            #  示例 intents、session、OpenAI format
└── docs/                     #  架构、运行模式、部署
```

---

## 引用

如果你使用了 ISE-Trace、ISE 范式或 ISETrace 数据集，请引用：

```bibtex
@misc{isetrace2026,
  title         = {From Intent to Trajectory: Execution-Grounded Multi-Turn Data Synthesis for OS Agents},
  author        = {Valiere01},
  year          = {2026},
  eprint        = {2606.11520},
  archivePrefix = {arXiv},
  primaryClass  = {cs.CL},
  url           = {https://arxiv.org/abs/2606.11520}
}
```

---

## 许可

本项目代码以 [MIT License](LICENSE) 发布。ISETrace 数据集单独分发，其使用条款见数据集卡片。
