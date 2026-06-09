     1|<div align="center">
     2|
     3|# ISE-Trace
     4|
     5|**An agentic data pipeline, smarter and broader.**
     6|
     7|*Intent → Simulate → Execute: a three-stage paradigm that synthesizes diverse, multi-turn, execution-grounded data for training OS agents.*
     8|
     9|English · [简体中文](README.md)
    10|
    11|[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
    12|[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](#)
    13|
    14|</div>
    15|
    16|> **Paper:** link (coming soon)
    17|>
    18|> **The intent creator:** https://github.com/NairongZheng/intent_creator
    19|>
    20|> **The agent loop gen data pipeline:** https://github.com/NairongZheng/openclaw_gen_data
    21|>
    22|> **The dataset:** huggingface link (coming soon)
    23|
    24|---
    25|
    26|## What is this
    27|
    28|Training a capable OS agent requires data that simultaneously captures **structured user intents**, **multi-turn task delegation**, and **grounded tool execution** — properties largely absent from existing datasets. Most synthesis pipelines start from an API catalog and back-derive tasks (so the task distribution mirrors the tool space rather than what users actually want), are single-turn, and simulate tool calls instead of running them.
    29|
    30|**ISE** (**I**ntent → **S**imulate → **E**xecute) is a three-stage synthesis paradigm that addresses these gaps jointly:
    31|
    32|| Stage | Name | What it does |
    33||:-----:|------|--------------|
    34|| **I** | 4D Intent Construction | Samples structured intents over `Persona × Domain × Task × Complexity`, so the training distribution is shaped by user-need composition rather than API availability. |
    35|| **S** | Multi-Turn Simulation | Drives user–agent interaction through a **role-locked user simulator** with four behavioral constraints that suppress *role drift* and *state hallucination*. |
    36|| **E** | Execution Grounding | Runs **every** tool call in a live, isolated OS workspace, yielding authentic failure–recovery dynamics instead of simulated responses. |
    37|
    38|The resulting corpus, **ISETrace**, contains **23,132** multi-turn trajectories (avg. **8.12** user turns / **68.24** total dialogue turns, **29.26** tool calls per trajectory) built from **43,956** unique structured intents.
    39|
    40|> Fine-tuning Qwen3-8B on ISETrace lifts **ClawEval** pass@1 from **19.3 → 37.7** (+18.4 pts, 1.95×), surpassing both a GPT-4o zero-shot reference and a 4×-larger Qwen3-32B base.

<div align="center">
  <img src="assets/pipeline.png" width="920" alt="ISE three-stage paradigm overview">
  <br>
  <em>The ISE paradigm at a glance. Each stage contrasts a typical failure mode of prior work (top, ✗) with what ISE contributes (bottom, ✓).</em>
</div>
    41|
    42|---
    43|
    44|## The Pipeline
    45|
    46|ISE-Trace is the umbrella project. The pipeline is split across two repositories, one per phase:
    47|
    48|```
    49|                 ┌─────────────────────────┐         ┌──────────────────────────────────┐
    50|   4D sampling   │     intent_creator      │ intents │        openclaw_gen_data         │  trajectories
    51|  ─────────────► │  Stage 1: 4D Intent     │ ──────► │  Stage 2+3: Multi-Turn Simulation│ ─────────────►  ISETrace
    52|                 │       Construction      │ .jsonl  │       + Execution Grounding      │   middle format
    53|                 └─────────────────────────┘         └──────────────────────────────────┘
    54|```
    55|
    56|| Repository | Role | Link |
    57||------------|------|------|
    58|| **`intent_creator`** | **Stage 1.** Domain-based 4D intent generation. Samples `Persona × Domain × Task × Complexity` and renders each structured tuple into a natural-language user intent via an LLM. | [github.com/NairongZheng/intent_creator](https://github.com/NairongZheng/intent_creator) |
    59|| **`openclaw_gen_data`** | **Stage 2 + 3.** Drives a local [OpenClaw](https://github.com/openclaw/openclaw) agent through multi-turn, role-locked user simulation; executes every tool call against a real OS in an isolated worker workspace; archives full sessions and converts them to training-ready *middle format*. | [github.com/NairongZheng/openclaw_gen_data](https://github.com/NairongZheng/openclaw_gen_data) |
    60|
    61|---
    62|
    63|## Key Features
    64|
    65|- **Intent-first, not tool-first.** A `Persona × Domain × Task × Complexity` sampling space exceeding 10¹¹ combinations — long-tail and cross-domain intents are sampled rather than back-derived from an API list.
    66|- **Role-locked multi-turn simulation.** Four behavioral principles — *perspective lock*, *register matching*, *incremental advancement*, *responsive conditioning* — keep the user simulator from drifting into assistant-style language or hallucinating execution state. Measured role-drift rate in the released corpus: **0.02%**.
    67|- **Real execution, real failures.** Every `exec` invokes an actual shell with real stdout/stderr/exit codes; file ops touch a real filesystem. Trajectories carry authentic credential failures, non-zero exits, and recovery — not model self-reports.
    68|- **Workspace isolation at scale.** Each worker runs in an isolated workspace restored from a shared snapshot template, reducing storage from O(N) to O(1) per worker. Supports concurrency and resume.
    69|- **Quantified diversity.** Full-stack diversity reported alongside the corpus — embedding (Vendi), lexical (Distinct-N), and structural (tool-call topology).
    70|
    71|---
    72|
    73|<div align="center">
  <img src="assets/landscape.png" width="760" alt="ISETrace in the concurrent agent-data landscape">
  <br>
  <em>ISETrace in the concurrent agent-data landscape. x-axis = avg. dialogue turns per trajectory; y-axis = #trajectories (log). Bubble area = tool calls per trajectory; hue = environment grounding.</em>
</div>

## ISETrace at a Glance
    74|
    75|| Statistic | Value |
    76||-----------|------:|
    77|| Structured intents (unique) | 43,956 |
    78|| Complete trajectories | 23,132 |
    79|| Avg. user turns / trajectory | 8.12 (med. 8, max 23) |
    80|| Trajectories with 6–10 user turns | 91.1% |
    81|| Avg. total dialogue turns | 68.24 (max 565) |
    82|| Avg. tool calls / trajectory | 29.26 |
    83|| Avg. unique tools / trajectory | 4.69 |
    84|| Domains × Tasks × Tools (avg.) | 2.35 × 4.40 × 3.18 |
    85|| Complexity (complex / medium / simple) | 50% / 40% / 10% |
    86|| Full-pool Vendi Score (mpnet-base-v2, cosine, q=1) | 61.57 |
    87|
    88|Trajectories span **10 functional domains** (Intelligence-Core, Code-Runtime, File-IO, Source-Chain, Automation-Flow, Web-Extraction, Comms-Social, Multimedia, Agent-Memory, System-Infrastructure) and **965** distinct personas across 47 industries.
    89|
    90|---
    91|
    92|## Quick Start
    93|
    94|The two stages run independently — Stage 1 produces an `intents.jsonl`, which Stage 2+3 consumes. Each repository has its own detailed README; the flow below shows how they connect.
    95|
    96|### Stage 1 — Generate intents (`intent_creator`)
    97|
    98|```bash
    99|git clone https://github.com/NairongZheng/intent_creator
   100|cd intent_creator
   101|pip install -r requirements.txt
   102|
   103|# Configure your LLM endpoint
   104|cp .env.example .env          # set OPENAI_API_KEY and OPENAI_BASE_URL
   105|
   106|# Generate intents (async, recommended)
   107|python main.py --count 1000 --async --max-concurrent 50 --output intents_out
   108|```
   109|
   110|This writes structured intents to `intents_out/intents.jsonl`, each line carrying a `natural_language_intent` plus its sampled domain / task / persona / complexity.
   111|
   112|### Stage 2 + 3 — Generate & ground trajectories (`openclaw_gen_data`)
   113|
   114|```bash
   115|git clone https://github.com/NairongZheng/openclaw_gen_data
   116|cd openclaw_gen_data
   117|pip install -r requirements.txt
   118|cp config/config.yaml.example config/config.yaml
   119|
   120|# Initialize agent workers (captures the real runtime tools + system prompt)
   121|python scripts/init_agents.py --num-agents 4 --force-recreate --refresh-tools
   122|
   123|# Drive multi-turn simulation + live execution over the intents from Stage 1
   124|INTENTS_FILE=../intent_creator/intents_out/intents.jsonl \
   125|python scripts/run_generation.py --concurrent 4
   126|```
   127|
   128|Output:
   129|- raw sessions → `output/sessions/`
   130|- training-ready middle format → `output/middle_format/`
   131|
   132|> **Prerequisite:** Stage 2+3 requires a working local [OpenClaw](https://github.com/openclaw/openclaw) installation (or its Docker setup) as the execution substrate. See the `openclaw_gen_data` README for container deployment and search-provider configuration. The paradigm is agent-system-agnostic — any platform supporting live tool execution and workspace isolation can serve as the substrate.
   133|
   134|---
   135|
   136|## Results
   137|
   138|Fine-tuning on ISETrace, evaluated on **ClawEval** (pass@1, %, on the common set of 114 *T*-family agent tool-use tasks scored under every run):
   139|
   140|| System | pass@1 | Completion | Robustness |
   141||--------|:------:|:----------:|:----------:|
   142|| Qwen3-8B (base, 0-shot) | 19.3 | 0.367 | 0.925 |
   143|| Qwen3-32B (base, 0-shot) | 30.7 | 0.446 | 0.947 |
   144|| GPT-4o (0-shot) | 25.4 | 0.442 | 0.965 |
   145|| **SFT-ISETrace (8B, ours)** | **37.7** | **0.533** | 0.959 |
   146|
   147|The fine-tuned 8B model surpasses the GPT-4o reference (+12.3 pts) and the 4×-larger Qwen3-32B base (+7.0 pts). The gain comes primarily from task completion while robustness on perturbed tool outputs holds high. An ablation truncating trajectories to a single user turn drops pass@1 to 28.1 (−9.6 pts), indicating multi-turn simulation contributes a substantial share of the gain.
   148|
   149|---
   150|
   151|<div align="center">
  <img src="assets/cross_corpus_turns.png" width="640" alt="Trajectory depth across agent corpora">
  <br>
  <em>Trajectory depth across fourteen agent corpora (avg. total turns). ISETrace leads by a wide margin at 68.24 turns, 2.68× the next corpus.</em>
</div>

<div align="center">
  <img src="assets/coverage.png" width="820" alt="ISETrace coverage analysis">
  <br>
  <em>Coverage analysis. Left: t-SNE of 5,000 intents — all 10 domains overlap rather than cluster. Right: Vendi grows from 40.7 to 61.6 (full pool), close to but not saturated.</em>
</div>

## Repository Structure
   152|
   153|```
   154|ISE-Trace/                    ← this repo (umbrella / entry point)
   155|├── README.md                 #  简体中文
   156|├── README_en.md              #  English (this file)
   157|└── LICENSE
   158|
   159|intent_creator/               ← Stage 1  (separate repo)
   160|├── main.py                   #  entry point
   161|├── domains/                  #  10 functional domain definitions
   162|├── libraries/                #  global persona / tool / skill pools
   163|├── src/{builders,generators,libraries,models,validators}/
   164|└── config.yaml
   165|
   166|openclaw_gen_data/            ← Stage 2 + 3  (separate repo)
   167|├── scripts/run_generation.py #  main entry point
   168|├── scripts/init_agents.py    #  worker init + runtime probe
   169|├── src/                      #  agent runtime, session parser, converter, ...
   170|├── data_examples/            #  sample intents, session, middle format
   171|└── docs/                     #  architecture, run-modes, deployment
   172|```
   173|
   174|---
   175|
   176|## Citation
   177|
   178|If you use ISE-Trace, the ISE paradigm, or the ISETrace dataset, please cite:
   179|
   180|```bibtex
   181|@misc{isetrace2026,
   182|  title        = {From Intent to Trajectory: Execution-Grounded Multi-Turn Data Synthesis for OS Agents},
   183|  author       = {Valiere01},
   184|  year         = {2026},
   185|  howpublished = {\url{https://github.com/Valiere01/ISE-Trace}},
   186|  note         = {Paper link forthcoming}
   187|}
   188|```
   189|
   190|---
   191|
   192|## License
   193|
   194|Code in this project is released under the [MIT License](LICENSE). The ISETrace dataset is distributed separately; see the dataset card for its terms.
   195|