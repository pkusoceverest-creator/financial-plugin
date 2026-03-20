# Financial Analysis Plugin - 项目设计

## 版本信息

- 版本: 2.0.0
- 状态: 规划中
- 方案: Claude Bundle (纯数据，无运行时代码)
- 创建日期: 2026-03-20
- 更新日期: 2026-03-20

---

## 1. 项目概述

### 1.1 目标

借鉴 Anthropic Financial Services Plugins (GitHub: anthropics/financial-services-plugins, 6,433 stars)，使用 Claude Bundle 格式开发财务分析 Plugin。

### 1.2 关键发现

**Anthropic 插件是纯数据格式**:
- 0 个 TypeScript 文件
- 0 个 JavaScript 文件
- 0 个 Python 文件
- 全部是 Markdown + JSON (368 KB)

**这意味着**: 不需要编写任何运行代码，只需编写 SKILL.md 知识库。

---

## 2. 架构设计

### 2.1 Claude Bundle 目录结构

```
financial-plugin/
├── .claude-plugin/
│   └── plugin.json              # Bundle 清单
│
├── commands/                     # 命令定义
│   ├── comps.md                  # 可比公司分析
│   ├── dcf.md                    # DCF 估值
│   ├── lbo.md                    # LBO 模型
│   ├── merger-model.md            # 并购模型
│   ├── 3-statement-model.md      # 三表模型
│   ├── earnings.md               # 财报分析
│   ├── one-pager.md              # 一页概况
│   ├── cim.md                    # 投资备忘录
│   ├── teaser.md                 # 交易概要
│   └── screen.md                 # 股票筛选
│
├── skills/                       # 知识库 (参考 Anthropic)
│   ├── comps-analysis/
│   │   └── SKILL.md             # 可比公司分析
│   ├── dcf-model/
│   │   └── SKILL.md             # DCF 估值
│   ├── lbo-model/
│   │   └── SKILL.md             # LBO 模型
│   ├── merger-model/
│   │   └── SKILL.md             # 并购模型
│   ├── three-statement-model/
│   │   └── SKILL.md             # 三表模型
│   ├── earnings/
│   │   └── SKILL.md             # 财报分析
│   └── one-pager/
│       └── SKILL.md             # 一页概况
│
├── hooks/
│   └── hooks.json               # 事件钩子
│
├── config.json                   # 配置
│
└── README.md                     # 说明文档
```

### 2.2 Anthropic 架构映射

| Anthropic 组件 | 本项目组件 |
|----------------|------------|
| `.claude-plugin/plugin.json` | `.claude-plugin/plugin.json` |
| `commands/*.md` | `commands/*.md` |
| `skills/*/SKILL.md` | `skills/*/SKILL.md` |
| `hooks/hooks.json` | `hooks/hooks.json` |
| MCP 连接器 | 无 (纯 Skill) |

---

## 3. 核心设计原则

### 3.1 纯数据实现

- ❌ 无运行时代码
- ❌ 不复用 us-stock-financials
- ✅ 纯 SKILL.md 知识库
- ✅ 纯 commands/*.md 命令定义

### 3.2 SKILL.md 内容来源

**完全参考 Anthropic**:
1. 复制 Anthropic 的 SKILL.md 内容
2. 适配 OpenClaw 触发词
3. 添加中文说明

---

## 4. 触发词配置

| 命令 | 触发词 |
|------|--------|
| comps | comps, 可比公司分析 |
| dcf | dcf, DCF估值, 现金流估值 |
| lbo | lbo, LBO模型, 杠杆收购 |
| merger-model | merger, 并购, 收购模型 |
| 3-statement-model | 三表模型, 财务模型 |
| earnings | 财报, 财报分析, earnings |
| one-pager | 一页概况, one-pager |
| cim | CIM, 投资备忘录 |
| teaser | teaser, 交易概要 |
| screen | screen, 筛选股票 |

---

## 5. 详细实施方案

### Phase 1: 基础框架 (1天)

**目标**: 创建 Plugin 目录结构和基础配置

| 任务 | 说明 |
|------|------|
| 创建目录结构 | 创建 financial-plugin/ 目录 |
| .claude-plugin/plugin.json | Bundle 清单文件 |
| config.json | 配置文件 |
| README.md | 说明文档 |

**产出**:
```
financial-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
├── skills/
├── hooks/
├── config.json
└── README.md
```

---

### Phase 2: 核心 Skills (8天)

#### Phase 2a: Comps (2天)

**目标**: 创建可比公司分析功能

**任务**:
1. commands/comps.md
   - 从 Anthropic 复制命令格式
   - 适配中文触发词
   
2. skills/comps-analysis/SKILL.md
   - 复制 Anthropic 的 comps-analysis SKILL.md
   - 保留英文工作流
   - 添加中文说明

**产出**:
- 知道如何进行可比公司分析
- 知道如何构建 Excel 输出
- 知道数据来源优先级

---

#### Phase 2b: DCF (2天)

**目标**: 创建 DCF 估值功能

**任务**:
1. commands/dcf.md
2. skills/dcf-model/SKILL.md

**内容重点**:
- WACC 计算
- 终值计算 (Gordon Growth / Exit Multiple)
- 敏感性分析表 (5x5)
- 股权价值桥接

---

#### Phase 2c: LBO (2天)

**目标**: 创建 LBO 模型功能

**任务**:
1. commands/lbo.md
2. skills/lbo-model/SKILL.md

**内容重点**:
- 交易结构 (债务/股权比例)
- 退出分析 (Multiple / IRR)
- 敏感性分析

---

#### Phase 2d: Merger (2天)

**目标**: 创建并购模型功能

**任务**:
1. commands/merger-model.md
2. skills/merger-model/SKILL.md

**内容重点**:
- 协同效应计算
- 交易估值方法
- 合并报表

---

### Phase 3: 扩展 Skills (3天)

#### Phase 3a: 3-Statement Model (1天)

**任务**:
1. commands/3-statement-model.md
2. skills/three-statement-model/SKILL.md

**内容**: 损益表、资产负债表、现金流量表联动

---

#### Phase 3b: Earnings (1天)

**任务**:
1. commands/earnings.md
2. skills/earnings/SKILL.md

**内容**: 财报解读、关键指标、趋势分析

---

#### Phase 3c: One-Pager (1天)

**任务**:
1. commands/one-pager.md
2. skills/one-pager/SKILL.md

**内容**: 公司单页简介模板

---

### Phase 4: 完善 (2天)

| 任务 | 说明 |
|------|------|
| cim | commands/cim.md + skills/cim/SKILL.md |
| teaser | commands/teaser.md + skills/teaser/SKILL.md |
| screen | commands/screen.md + skills/screen/SKILL.md |
| hooks.json | 事件钩子配置 |
| 文档完善 | 检查完整性 |

---

## 6. SKILL.md 内容模板

```markdown
---
name: comps-analysis
description: |
  Build institutional-grade comparable company analyses.
  
trigger:
  - "comps"
  - "可比公司分析"
---

# Comparable Company Analysis

## Overview
...

## Data Source Priority
1. MCP Servers
2. User-Provided Data
3. Web Search

## Workflow

### Step 1: Data Retrieval
...

### Step 2: Peer Selection
...

### Step 3: Build Excel
...
```

---

## 7. commands 命令模板

```markdown
---
description: "Build a comparable company analysis"
argument-hint: "[company name or ticker]"
---

# Comps Command

## Workflow

### Step 1: ...
```

---

## 8. 部署

### 本地开发
```bash
openclaw plugins install -l ./financial-plugin
openclaw plugins enable financial-analysis
openclaw gateway restart
```

### Claude Bundle 安装
```bash
openclaw plugins install ./financial-plugin
```

---

## 9. 迭代扩展 (无限)

扩展方式: 在 commands/ 和 skills/ 添加新目录即可。

示例扩展:
- buyer-list, deal-tracker, process-letter
- morning-note, initiate, thesis
- dd-checklist, ic-memo

---

## 10. 参考

- Anthropic Financial Services Plugins: https://github.com/anthropics/financial-services-plugins
- OpenClaw Plugin Bundles: https://docs.openclaw.ai/plugins/bundles

---

## 11. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-03-20 | 初始设计 (Native Plugin) |
| 2.0.0 | 2026-03-20 | 修正为 Claude Bundle 格式 |

---

*最后更新: 2026-03-20 22:52 (Asia/Shanghai)*