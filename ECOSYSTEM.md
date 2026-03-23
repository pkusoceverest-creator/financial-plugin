# 金融科技生态框架文档

> 本文档从 REFERENCE.md 拆分
> 版本: 2.0
> 最后更新: 2026-03-23

---

## 概述

本文档整理了与 financial-plugin 相关的开源框架，作为未来集成的参考。

| 框架 | 来源 | 核心能力 |
|------|------|---------|
| **FinRL** | AI4Finance | 强化学习交易策略 |
| **Qlib** | Microsoft | AI 量化投资平台 |
| **Open Deep Research** | LangChain | 深度研究代理 |
| **RD-Agent** | Microsoft | 自动化研发框架 |

---

## 目录

- 二十一、FinRL - 金融强化学习框架
- 二十二、Qlib - Microsoft AI 量化投资平台
- 二十三、Open Deep Research - LangChain 开源深度研究代理
- 二十四、RD-Agent - Microsoft 自动化研发框架

---

**修改后**:
```markdown
## Step 1: 获取财务数据

使用 OpenBB REST API:
- get_historical_financials: 获取历史财务数据
- get_valuation_multiples: 获取估值倍数
```

---

## 二十一、FinRL - 金融强化学习框架

> 来源: https://github.com/AI4Finance-Foundation/FinRL

### 21.1 项目概述

**FinRL** 是首个开源的金融强化学习框架，由 AI4Finance Foundation 维护。

**核心定位**: 自动化量化交易策略开发

**三层架构**:
```
┌─────────────────────────────────────┐
│         Applications (应用层)        │
│  股票交易 / 加密货币 / 投资组合配置    │
├─────────────────────────────────────┤
│           Agents (算法层)            │
│  A2C / DDPG / PPO / TD3 / SAC       │
├─────────────────────────────────────┤
│       Market Environments (环境层)   │
│  Gym-style 市场模拟环境              │
└─────────────────────────────────────┘
```

### 21.2 开发路线图

| 阶段 | 用户 | 项目 | 说明 |
|------|------|------|------|
| 0.0 | 入门者 | FinRL-Meta | Gym-style 市场环境 |
| 1.0 | 开发者 | **FinRL** | 端到端自动流水线 |
| 2.0 | 研究者 | ElegantRL | 高级算法库 |
| 3.0 | 机构 | FinRL-X | 生产级部署 |

### 21.3 支持的数据源

| 数据源 | 类型 | 频率 | 限制 |
|--------|------|------|------|
| **YahooFinance** | 美股 | 1min | 2000/小时 |
| **Alpaca** | 美股/ETF | 1min | 账户限制 |
| **Binance** | 加密货币 | 1s | API 限制 |
| **Akshare** | A股 | 1day | 账户限制 |
| **Tushare** | A股 | 1min | 账户限制 |
| **WRDS** | 美股 | 1ms | 5次/请求 |

### 21.4 核心工作流程

```
数据下载 → 特征工程 → 模型训练 → 回测评估 → 实盘交易
```

**三步使用**:

```bash
# Step 1: 数据下载与预处理
python examples/FinRL_StockTrading_2026_1_data.py
# 输出: train_data.csv, trade_data.csv
# 包含: OHLCV + 技术指标 (MACD, RSI, etc.) + VIX + 湍流指数

# Step 2: 训练 DRL Agents
python examples/FinRL_StockTrading_2026_2_train.py
# 训练 5 种算法: A2C, DDPG, PPO, TD3, SAC
# 模型保存: trained_models/

# Step 3: 回测
python examples/FinRL_StockTrading_2026_3_Backtest.py
# 对比基准: MVO (均值方差优化) + DJIA 指数
```

### 21.5 支持的 DRL 算法

| 算法 | 类型 | 适用场景 |
|------|------|---------|
| **A2C** | On-policy | 简单任务 |
| **PPO** | On-policy | 稳定训练 |
| **DDPG** | Off-policy | 连续动作 |
| **TD3** | Off-policy | 改进 DDPG |
| **SAC** | Off-policy | 自动探索 |

### 21.6 文件结构

```
FinRL/
├── finrl/
│   ├── applications/          # 应用场景
│   │   ├── stock_trading/     # 股票交易
│   │   ├── cryptocurrency_trading/  # 加密货币
│   │   ├── portfolio_allocation/    # 投资组合
│   │   └── high_frequency_trading/  # 高频交易
│   ├── agents/                # DRL 算法
│   │   ├── stablebaseline3/   # SB3 实现
│   │   ├── elegantrl/         # ElegantRL 实现
│   │   └── rllib/             # RLlib 实现
│   └── meta/                  # 市场环境
│       ├── env_stock_trading/
│       ├── data_processors/
│       └── preprocessor/
├── examples/                  # 示例脚本
├── train.py                   # 训练入口
├── test.py                    # 测试入口
└── trade.py                   # 交易入口
```

### 21.7 技术指标

**默认特征**:
- MACD (移动平均收敛发散)
- Bollinger Bands (布林带)
- RSI_30 (相对强弱指数)
- DX_30 (方向性指数)
- Close_30_SMA (30日简单移动平均)
- Close_60_SMA (60日简单移动平均)

**可扩展**: 用户可自定义新特征

### 21.8 与 financial-plugin 的集成建议

| 集成点 | 说明 |
|--------|------|
| **数据层** | FinRL 可使用 OpenBB 作为数据源 |
| **特征层** | OpenBB 的技术指标可注入 FinRL |
| **策略层** | FinRL 训练的策略可作为 financial-plugin 的决策支持 |

**潜在 Skill**: `rl-trading-strategy`
- 输入: 股票池、时间范围、风险偏好
- 过程: FinRL 训练 + 回测
- 输出: 最优交易策略 + 回测报告

### 21.9 安装

```bash
# 克隆仓库
git clone https://github.com/AI4Finance-Foundation/FinRL.git
cd FinRL

# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate

# 安装
pip install -e .
```

### 21.10 相关项目

| 项目 | 说明 |
|------|------|
| **FinRL-Meta** | 市场环境库 |
| **ElegantRL** | 轻量级 DRL 框架 |
| **FinGPT** | 金融大语言模型 |
| **FinRL-X** | 生产级交易平台 |

### 21.11 学术影响力

| 论文 | 会议/期刊 | 引用数 |
|------|---------|--------|
| Practical deep RL for stock trading | NeurIPS 2018 | 303 |
| FinRL: DRL library for automated trading | NeurIPS 2020 | 275 |
| Ensemble strategy for stock trading | ICAIF 2020 | 426 |
| FinRL-Meta: Market environments | NeurIPS 2022 | 136 |

---

## 二十二、Qlib - Microsoft AI 量化投资平台

> 来源: https://github.com/microsoft/qlib

### 22.1 项目概述

**Qlib** 是微软开源的 AI 导向量化投资平台，覆盖量化投资全链路。

**核心定位**: 从想法探索到生产实现的一站式量化平台

**支持的 ML 范式**:
- 监督学习 (Supervised Learning)
- 市场动态建模 (Market Dynamics Modeling)
- 强化学习 (Reinforcement Learning)

### 22.2 四层架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Interface Layer (接口层)                  │
│  Analyser: 信号分析、组合分析、执行分析                        │
├─────────────────────────────────────────────────────────────┤
│                    Workflow Layer (工作流层)                 │
│  Info Extractor → Forecast Model → Decision Generator        │
│                   ↓                                          │
│              Execution Env (交易执行)                         │
├─────────────────────────────────────────────────────────────┤
│                 Learning Framework Layer (学习框架层)         │
│  Supervised Learning | Reinforcement Learning | Meta Learning│
├─────────────────────────────────────────────────────────────┤
│                 Infrastructure Layer (基础设施层)            │
│  DataServer (高性能数据服务) | Trainer (训练控制)             │
└─────────────────────────────────────────────────────────────┘
```

### 22.3 核心功能

| 模块 | 功能 |
|------|------|
| **Data** | 高性能数据服务器、多源数据支持 |
| **Model** | 20+ SOTA 量化模型 (LightGBM, Transformer, GATs...) |
| **Strategy** | 交易策略生成与优化 |
| **Execution** | 订单执行模拟 |
| **Analysis** | 回测分析、风险分析 |

### 22.4 量化投资全链路覆盖

```
数据处理 → 模型训练 → 回测评估 → 组合优化 → 订单执行
    ↓           ↓          ↓          ↓          ↓
 Data      Forecast    Backtest   Portfolio   Execution
 Server     Model      Analysis   Optimizer   Simulator
```

### 22.5 最新特性: RD-Agent

**RD-Agent** 是基于 LLM 的自动化量化研发代理：

| 功能 | 说明 |
|------|------|
| **因子挖掘** | 从报告/数据自动挖掘因子 |
| **模型优化** | 自动搜索最优模型配置 |
| **多代理协作** | 数据中心化因子+模型联合优化 |

**Demo 场景**:
- Quant Factor Mining
- Quant Factor Mining from reports
- Quant Model Optimization

### 22.6 量化模型库

| 模型 | 类型 | 论文/来源 |
|------|------|---------|
| **LightGBM** | 梯度提升树 | Microsoft |
| **Transformer** | 注意力机制 | NeurIPS 2021 |
| **GATs** | 图注意力网络 | ICLR |
| **HIST** | 行业感知模型 | CIKM |
| **TCTS** | 时序模型 | AAAI |
| **DoubleEnsemble** | 集成学习 | NeurIPS |
| **ADD** | 概念漂移适应 | KDD |
| **TRA** | 时序路由适配器 | ICML |

### 22.7 支持的 Python 版本

| Python | pip 安装 | 源码安装 | 绑图 |
|--------|---------|---------|------|
| 3.8 | ✅ | ✅ | ✅ |
| 3.9 | ✅ | ✅ | ✅ |
| 3.10 | ✅ | ✅ | ✅ |
| 3.11 | ✅ | ✅ | ✅ |
| 3.12 | ✅ | ✅ | ✅ |

### 22.8 安装与快速开始

```bash
# 安装
pip install pyqlib

# 或从源码安装
git clone https://github.com/microsoft/qlib.git && cd qlib
pip install .

# 准备数据 (社区数据源)
wget https://github.com/chenditc/investment_data/releases/latest/download/qlib_bin.tar.gz
mkdir -p ~/.qlib/qlib_data/cn_data
tar -zxvf qlib_bin.tar.gz -C ~/.qlib/qlib_data/cn_data --strip-components=1

# 运行示例
qrun examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
```

### 22.9 数据支持

| 数据类型 | 频率 | 说明 |
|---------|------|------|
| 日线数据 | 1day | 默认提供 |
| 分钟数据 | 1min | 高频交易 |
| Point-in-Time | - | 避免未来数据泄露 |
| Orderbook | tick | 高频数据 |

### 22.10 与 FinRL 对比

| 维度 | Qlib | FinRL |
|------|------|-------|
| **开发方** | Microsoft | AI4Finance |
| **核心范式** | 监督学习 + RL | 强化学习 |
| **模型数量** | 20+ SOTA 模型 | 5 种 DRL 算法 |
| **数据支持** | 内置数据服务器 | 外部数据源 |
| **生产支持** | ✅ 在线服务 | ⚠️ 有限 |
| **LLM 集成** | ✅ RD-Agent | ❌ 无 |

### 22.11 与 financial-plugin 集成建议

| 集成点 | 说明 |
|--------|------|
| **数据层** | Qlib DataServer 可作为 OpenBB 补充 |
| **模型层** | Qlib 的预训练模型可用于信号生成 |
| **策略层** | Qlib 的交易策略可作为决策支持 |
| **RD-Agent** | 可用于自动化因子挖掘 |

### 22.12 学术影响力

| 论文 | 会议 | 引用 |
|------|------|------|
| Qlib: AI-oriented Quant Platform | arXiv | 高 |
| R&D-Agent-Quant | arXiv 2025 | 新 |

---

## 二十三、Open Deep Research - LangChain 开源深度研究代理

> 来源: https://github.com/langchain-ai/open_deep_research

### 23.1 项目概述

**Open Deep Research** 是 LangChain 开源的深度研究代理框架，支持多种模型提供商、搜索工具和 MCP 服务器。

**核心定位**: 可配置、完全开源的深度研究 Agent

**性能**: Deep Research Bench 排行榜 #6 (RACE Score: 0.4344)

### 23.2 核心特性

| 特性 | 说明 |
|------|------|
| **多模型支持** | OpenAI, Anthropic, OpenRouter, Ollama 本地模型 |
| **多搜索工具** | Tavily (默认), MCP 兼容, 原生 Web 搜索 |
| **结构化输出** | 支持工具调用和结构化输出 |
| **可配置** | 通过 LangGraph Studio UI 配置 |

### 23.3 架构设计

```
用户问题
    │
    ↓
┌─────────────────────────────────────────────────────────┐
│                   LangGraph 工作流                       │
├─────────────────────────────────────────────────────────┤
│  1. Summarization (gpt-4.1-mini)                        │
│     └── 搜索 API 结果摘要                                │
│                                                         │
│  2. Research (gpt-4.1)                                  │
│     └── 驱动搜索代理                                    │
│                                                         │
│  3. Compression (gpt-4.1)                               │
│     └── 压缩研究发现                                    │
│                                                         │
│  4. Final Report (gpt-4.1)                              │
│     └── 撰写最终报告                                    │
└─────────────────────────────────────────────────────────┘
    │
    ↓
最终研究报告
```

### 23.4 四个 LLM 任务

| 任务 | 默认模型 | 功能 |
|------|---------|------|
| **Summarization** | gpt-4.1-mini | 搜索结果摘要 |
| **Research** | gpt-4.1 | 驱动搜索代理 |
| **Compression** | gpt-4.1 | 压缩研究发现 |
| **Final Report** | gpt-4.1 | 撰写最终报告 |

### 23.5 支持的搜索工具

| 工具 | 说明 |
|------|------|
| **Tavily** | 默认搜索 API |
| **MCP Servers** | Model Context Protocol 兼容 |
| **Anthropic 原生** | Anthropic Web Search |
| **OpenAI 原生** | OpenAI Web Search |

### 23.6 安装与运行

```bash
# 克隆仓库
git clone https://github.com/langchain-ai/open_deep_research.git
cd open_deep_research

# 创建虚拟环境
uv venv
source .venv/bin/activate

# 安装依赖
uv sync

# 配置环境变量
cp .env.example .env

# 启动 LangGraph 服务器
uvx --refresh --from "langgraph-cli[inmem]" --with-editable . --python 3.11 langgraph dev --allow-blocking
```

### 23.7 访问方式

| 接口 | 地址 |
|------|------|
| **API** | http://127.0.0.1:2024 |
| **Studio UI** | https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024 |
| **API Docs** | http://127.0.0.1:2024/docs |

### 23.8 Deep Research Bench 评测

**评测基准**: 100 个博士级研究任务 (50 英文 + 50 中文)

| 评测指标 | 说明 |
|---------|------|
| **RACE Score** | LLM-as-judge (Gemini) 评估 |
| **覆盖领域** | 22 个领域 (科技、商业金融等) |

**性能结果**:

| 模型配置 | 成本 | Tokens | RACE Score |
|---------|------|--------|------------|
| GPT-5 | - | 204M | **0.4943** |
| Claude Sonnet 4 | $187 | 139M | 0.4401 |
| Defaults (gpt-4.1) | $46 | 58M | 0.4309 |

### 23.9 两种遗留实现

| 方案 | 特点 |
|------|------|
| **Plan-and-Execute** | 人机协作规划、迭代优化、质量导向 |
| **Supervisor-Researcher** | 多代理并行、速度优化、MCP 支持 |

### 23.10 部署选项

| 平台 | 说明 |
|------|------|
| **LangGraph Platform** | 官方云部署 |
| **Open Agent Platform (OAP)** | 非技术用户 UI |

**OAP Demo**: https://oap.langchain.com

### 23.11 相关资源

| 资源 | 链接 |
|------|------|
| **免费课程** | LangChain Academy Deep Research |
| **课程仓库** | deep_research_from_scratch |
| **博客** | LangChain Blog |
| **视频** | YouTube 概览 |

### 23.12 与 financial-plugin 集成建议

| 集成点 | 说明 |
|--------|------|
| **研究报告** | Open Deep Research 可用于生成金融研究报告 |
| **MCP 兼容** | 可与 OpenBB MCP 集成获取金融数据 |
| **配置复用** | LangGraph 工作流设计可参考 |

---

## 二十四、RD-Agent - Microsoft 自动化研发框架

> 来源: https://github.com/microsoft/RD-Agent

### 24.1 项目概述

**RD-Agent** 是微软开源的自动化研发框架，通过 LLM 驱动数据与模型的自动化研发。

**核心理念**: "让 AI 驱动数据驱动的 AI"

**核心框架**: R (Researcher) + D (Developer)
- **R**: 提出新想法、新假设
- **D**: 实现想法、验证假设

### 24.2 核心成就

| 成就 | 说明 |
|------|------|
| **MLE-Bench 第一** | 75 个 Kaggle 竞赛评测，最高性能 ML Agent |
| **NeurIPS 2025** | R&D-Agent-Quant 论文被接收 |
| **量化交易突破** | $10 成本，ARR 提升 2×，因子减少 70% |

### 24.3 MLE-Bench 评测结果

| Agent | Low (%) | Medium (%) | High (%) | All (%) |
|-------|---------|------------|----------|---------|
| **R&D-Agent o3(R)+GPT-4.1(D)** | 51.52 | 19.3 | 26.67 | **30.22** |
| R&D-Agent o1-preview | 48.18 | 8.95 | 18.67 | 22.4 |
| AIDE o1-preview | 34.3 | 8.8 | 10.0 | 16.9 |

### 24.4 应用场景

| 场景 | 功能 | Demo |
|------|------|------|
| **Automatic Quant Factory** | 因子挖掘 + 模型优化 | ✅ |
| **Data Mining Agent** | 迭代式数据/模型提案 | ✅ |
| **Research Copilot** | 自动阅读论文/报告并实现 | ✅ |
| **Kaggle Agent** | 自动模型调优 + 特征工程 | 开发中 |

### 24.5 RD-Agent(Q) - 量化金融专用

**核心特性**:
- 数据中心化多代理框架
- 因子-模型联合优化
- 与 Qlib 深度集成

**性能表现**:
| 指标 | 结果 |
|------|------|
| **成本** | <$10 |
| **ARR 提升** | 2× 基准因子库 |
| **因子数量** | 减少 70%+ |
| **资源效率** | 优于深度时序模型 |

### 24.6 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    R&D-Agent 框架                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   ┌─────────────┐         ┌─────────────┐              │
│   │      R      │  ───→  │      D      │              │
│   │ (Research)  │  想法   │ (Developer) │              │
│   │             │  ←───  │             │              │
│   └─────────────┘  反馈   └─────────────┘              │
│         │                     │                        │
│         ↓                     ↓                        │
│   提出新假设              实现验证                      │
│   阅读论文                构建数据                      │
│   分析报告                训练模型                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 24.7 支持的 LLM 后端

| 后端 | 配置方式 |
|------|---------|
| **LiteLLM** (默认) | 统一多模型接口 |
| OpenAI | 直接 API |
| Azure OpenAI | 企业部署 |
| DeepSeek | 成本优化 |
| SiliconFlow | Embedding 支持 |

### 24.8 安装与配置

```bash
# 创建环境
conda create -n rdagent python=3.10
conda activate rdagent

# 从 PyPI 安装
pip install rdagent

# 或从源码安装
git clone https://github.com/microsoft/RD-Agent
cd RD-Agent
make dev

# 健康检查
rdagent health_check --no-check-env
```

### 24.9 运行命令

| 场景 | 命令 |
|------|------|
| **量化交易 (因子+模型)** | `rdagent fin_quant` |
| **量化交易 (因子)** | `rdagent fin_factor` |
| **量化交易 (模型)** | `rdagent fin_model` |
| **从报告提取因子** | `rdagent fin_factor_report --report-f` |

### 24.10 与 Qlib 集成

RD-Agent 与 Qlib 深度集成：

| 集成点 | 说明 |
|--------|------|
| **数据源** | 使用 Qlib 数据服务器 |
| **因子库** | 生成 Qlib 兼容因子 |
| **模型训练** | 使用 Qlib 训练框架 |
| **回测** | 使用 Qlib 回测引擎 |

### 24.11 Web UI

**Live Demo**: https://rdagent.azurewebsites.net/

**功能**:
- 实时交互
- 追踪查看
- 多场景演示

### 24.12 与 financial-plugin 集成建议

| 集成点 | 说明 |
|--------|------|
| **因子挖掘** | RD-Agent 可自动生成金融因子 |
| **模型优化** | 自动调优预测模型 |
| **报告分析** | 从金融报告提取信号 |
| **研究辅助** | 自动阅读研究论文并实现 |

### 24.13 技术报告与论文

| 资源 | 链接 |
|------|------|
| **技术报告** | arxiv.org/abs/2505.14738 |
| **R&D-Agent-Quant** | arxiv.org/abs/2505.15155 |
| **文档** | rdagent.readthedocs.io |

---

*文档版本: 1.7*  
