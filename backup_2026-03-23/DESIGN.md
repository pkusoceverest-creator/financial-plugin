# Financial Plugin 项目设计方案

> 基于 OpenClaw Native Plugin 架构
> 参考: Anthropic Financial Services Plugin (11 MCP → 11 Tools)
> 设计日期: 2026-03-21
> 版本: 5.0 (OpenBB API + Anthropic FSI Skills)
> 最后更新: 2026-03-23

---

## 〇、v5.0 核心设计决策

### 决策 1: 数据层选择 OpenBB REST API

| 对比项 | OpenBB MCP | OpenBB REST API | 决策 |
|--------|-----------|-----------------|------|
| 协议复杂度 | 高 (新协议) | 低 (标准 HTTP) | ✅ API |
| Tool 控制力 | 低 (自动暴露) | 高 (完全自定义) | ✅ API |
| 调试便利性 | 低 | 高 (curl/Postman) | ✅ API |
| OpenClaw 兼容 | 需适配 | 原生支持 | ✅ API |

**启动命令**: `openbb-api --port 6900`

### 决策 2: SKILL.md 完全参考 Anthropic FSI

**原则**: 逐字复制 Anthropic FSI 的 SKILL.md 内容，仅修改数据源引用。

| 修改项 | Anthropic FSI | financial-plugin |
|--------|--------------|------------------|
| 数据源 | MCP Server (Daloopa, FactSet...) | OpenBB REST API |
| 调用方式 | `mcp_client.call("equity.price...")` | `fetch(OPENBB_API + "/api/v1/...")` |
| SKILL 内容 | **完全保留** | **完全保留** |
| 公式/格式 | **完全保留** | **完全保留** |

---

## 一、系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Gateway                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      用户自然语言输入                                │   │
│   │    "帮我做 AAPL 的可比公司分析"  "分析微软的 DCF 估值"               │   │
│   │    "看看这个并购交易的回报"  "A 公司 LBO 估值"                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ↓                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   SKILL 自动触发层                                  │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │   │
│   │  │  comps-     │  │   dcf-     │  │   lbo-      │  │  merger-   │  │   │
│   │  │  analysis   │  │   model    │  │   model    │  │  model    │  │   │
│   │  │  SKILL      │  │  SKILL     │  │  SKILL      │  │  SKILL     │  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘  │   │
│   │       ↑               ↑              ↑               ↑            │   │
│   │       │               │              │               │            │   │
│   │   trigger:         trigger:       trigger:         trigger:        │   │
│   │   "comps"          "DCF"          "LBO"           "M&A"        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ↓                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         AI 执行层                                    │   │
│   │   ┌──────────────────────────────────────────────────────────────┐  │   │
│   │   │  1. 理解用户需求                                                │  │   │
│   │   │  2. 根据 SKILL 指导执行分析                                     │  │   │
│   │   │  3. 需要数据时自动调用 Data Tools                              │  │   │
│   │   └──────────────────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ↓                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      11 Data Tools 层                                │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │   │
│   │  │ get_     │ │ get_     │ │ get_     │ │ get_     │ │ get_     │   │   │
│   │  │ hist-    │ │ morning- │ │ sp_      │ │ factset  │ │ moodys   │   │   │
│   │  │ fin.     │ │ star     │ │ global   │ │ _data    │ │ _data    │   │   │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │   │
│   │  │ get_     │ │ get_     │ │ get_     │ │ get_     │ │ get_     │   │   │
│   │  │ news     │ │ earnings │ │ lseg     │ │ pitchbook│ │ chronog- │   │   │
│   │  │          │ │ _call    │ │ _data    │ │ _data    │ │ raph     │   │   │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │   │
│   │                         ↑                                           │   │
│   │                         │ (还有一个: get_document)                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ↓                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       输出层                                         │   │
│   │              Excel / PDF / PPT 结果返回                             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、Core + Add-on 分层架构

### 2.1 分层设计模式 (参考 Anthropic)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Add-on Plugins (可选)                               │
├─────────────────────┬─────────────────────┬─────────────────┬────────────────┤
│  investment-      │  equity-           │  private-      │  wealth-      │
│  banking          │  research          │  equity        │  management   │
│  • CIM, Teaser    │  • 财报更新        │  • DD, IC memo │  • 客户会议    │
│  • 并购模型        │  • 首次覆盖报告     │  • 投资筛选    │  • 组合再平衡  │
└─────────────────────┴─────────────────────┴────────────────┴────────────────┘
                              ↑ 共享依赖 Core Plugin
┌─────────────────────────────────────────────────────────────────────────────┐
│                    financial-core (Core - 必装)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  • 11 Data Tools (统一管理)                                                 │
│  • 11 Core Skills (与 Anthropic 保持一致)                                   │
│  • 核心建模能力                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Add-on 与 Core 部署目录关系

Add-on 与 Core 部署在**同级目录**，不需要嵌套：

```
~/.openclaw/extensions/
├── financial-core/              # Core (必装)
│   ├── package.json
│   ├── openclaw.plugin.json
│   ├── index.ts
│   └── skills/                  # 11 Core Skills
│
├── investment-banking/          # Add-on (同级目录)
│   ├── package.json
│   ├── openclaw.plugin.json
│   └── skills/                  # 9 Skills
│
├── private-equity/              # Add-on (同级目录)
│   └── skills/                  # 10 Skills
│
├── equity-research/             # Add-on (同级目录)
│   └── skills/                  # 9 Skills
│
└── wealth-management/           # Add-on (同级目录)
    └── skills/                  # 6 Skills
```

**每个插件独立**:
- `package.json`: 每个插件独立
- `openclaw.plugin.json`: 每个插件独立清单
- `index.ts`: 每个插件独立入口
- Add-ons 通过 OpenClaw 框架引用 Core 的 Tools (非目录嵌套)

**原因**:
1. OpenClaw 按插件 ID 加载，不关心目录嵌套
2. Add-ons 独立安装: `openclaw plugins install ./investment-banking`
3. 便于管理: 各插件独立版本控制、测试、发布

### 2.2 分层原则

| 原则 | 说明 |
|------|------|
| **Core First** | 必须先安装 `financial-core` |
| **Connectors 集中化** | 所有 11 Tools 只在 Core 中定义，Add-ons 继承复用 |
| **可扩展** | 新增功能只需创建新 Add-on，无需修改 Core |
| **SKILL 自动触发** | 各层均支持上下文自动匹配 |

### 2.3 插件结构

| 类型 | 插件名 | 必装 | 包含 |
|------|--------|------|------|
| **Core** | financial-core | ✅ | 11 Tools + 11 Skills |
| **Add-on** | investment-banking | ❌ | IB 扩展 Skills |
| **Add-on** | equity-research | ❌ | ER 扩展 Skills |
| **Add-on** | private-equity | ❌ | PE 扩展 Skills |
| **Add-on** | wealth-management | ❌ | WM 扩展 Skills |

---

## 三、项目定位

构建一个面向金融服务的 OpenClaw Native Plugin，让 AI 能够完成投资分析、财务建模、估值等工作流。

### 核心目标

- 将 AI 打造为金融专业人员 (投资银行、股权研究、私募股权、财富管理) 的专业助手
- 实现从数据获取 → 分析建模 → 报告生成的完整工作流自动化
- 遵循 OpenClaw Best Practice: **SKILL 自动触发，无需手动 Command**

---

## 四、11 OpenBB Tools 定义

### 4.1 Tools 设计原则

**Tool = 纯数据代理** (参考 Anthropic FSI 的 MCP 层设计)

- Tool 只负责调用 OpenBB REST API 获取数据
- 数据处理、计算、格式化全部在 SKILL 层完成
- Tool 返回标准化 JSON，不做业务逻辑

### 4.2 Tools 与 OpenBB API 映射

| 序号 | Tool 名称 | OpenBB API 端点 | 功能 |
|------|-----------|----------------|------|
| 1 | `get_historical_financials` | `/api/v1/equity/fundamental/{statement}` | 财务三表 |
| 2 | `get_market_data` | `/api/v1/equity/price/historical` | 历史股价 |
| 3 | `get_valuation_multiples` | `/api/v1/equity/fundamental/metrics` | 估值倍数 |
| 4 | `get_economic_data` | `/api/v1/economic/indicators` | 宏观数据 |
| 5 | `get_news` | `/api/v1/news/company` | 公司新闻 |
| 6 | `get_industry_peers` | `/api/v1/equity/compare/groups` | 同行公司 |
| 7 | `get_ownership_data` | `/api/v1/equity/ownership` | 股东数据 |
| 8 | `get_calendar_events` | `/api/v1/equity/calendar` | 财报日历 |
| 9 | `get_options_data` | `/api/v1/derivatives/options` | 期权数据 |
| 10 | `get_crypto_data` | `/api/v1/crypto/price/historical` | 加密货币 |
| 11 | `get_fixed_income_data` | `/api/v1/fixedincome/...` | 债券数据 |

### 4.3 与 Anthropic FSI MCP 的对应关系

| Anthropic FSI MCP | financial-plugin Tool | 数据覆盖 |
|-------------------|----------------------|---------|
| Daloopa + FactSet | `get_historical_financials` | ✅ SEC EDGAR |
| FactSet + S&P | `get_valuation_multiples` | ✅ 部分 (可自算) |
| Morningstar | `get_valuation_multiples` | ✅ 部分 |
| Moody's | `get_fixed_income_data` | ⚠️ FRED 有限 |
| MT Newswires | `get_news` | ⚠️ Finnhub 免费 |
| Aiera | `get_earnings_transcript` | ⚠️ FMP 替代 |
| LSEG | `get_fixed_income_data` | ⚠️ FRED 有限 |
| PitchBook | - | ❌ 无替代 |
| Chronograph | `get_market_data` | ✅ |
| Egnyte | - | ❌ 非数据源 |

### 4.4 Aiera MCP 替代方案

**Aiera 提供什么**:
- 财报电话会议转录 (管理层讨论 + Q&A)
- 音频录音
- 实时监控

**Anthropic FSI 中使用 Aiera 的 Skills**:

| 插件 | Skill | 使用程度 |
|------|-------|---------|
| equity-research (Add-on) | earnings-analysis | ✅ 必需 |
| equity-research (Add-on) | earnings-preview | ✅ 必需 |
| financial-analysis (Core) | competitive-analysis | ⚠️ 可选 |
| financial-analysis (Core) | dcf-model | ⚠️ 提及 |

**替代方案: OpenBB + FMP**

```typescript
// 新增 Tool: get_earnings_transcript
api.registerTool({
  name: "get_earnings_transcript",
  description: "获取财报电话会议转录",
  parameters: {
    ticker: { type: "string", description: "股票代码" },
    year: { type: "number", description: "年份" }
  },
  async execute(params) {
    return fetch(
      `${OPENBB_API}/api/v1/equity/fundamental/transcript?symbol=${params.ticker}&year=${params.year}&provider=fmp`
    );
  }
});
```

**替代方案对比**:

| 功能 | Aiera | OpenBB + FMP | Seeking Alpha | Finnhub |
|------|-------|--------------|---------------|---------|
| **文字转录** | ✅ | ✅ | ✅ | ✅ |
| **音频** | ✅ | ❌ | ✅ | ❌ |
| **实时监控** | ✅ | ❌ | ❌ | ⚠️ |
| **AI 分析** | ✅ | 需自建 | ❌ | ❌ |
| **费用** | 付费 | 免费/付费层 | 免费 | 免费/付费层 |
| **OpenBB 集成** | ❌ | ✅ | ⚠️ 部分 | ❌ |

**结论**: 
- **Core Skills (11 个)** 不强制依赖财报转录
- 如需 equity-research Add-on，可用 `get_earnings_transcript` (OpenBB + FMP) 替代 Aiera
- 缺失功能: 音频录音、实时监控、AI 自动分析

---

## 五、工作流程设计 (Best Practice)

### 3.1 核心原则

遵循 OpenClaw Best Practice:
- **SKILL 自动触发**: 根据上下文自动匹配，不需要手动命令
- **Tool 自动调用**: AI 判断需要数据时自动调用对应 Tool
- **自然语言交互**: 用户用自然语言表达需求，无需记住命令

### 3.2 工作流程

```
┌─────────────────────────────────────────────────────────────────┐
│  用户自然语言输入                                                │
│  "帮我做 AAPL 的可比公司分析"                                    │
│  "分析微软的 DCF 估值"                                           │
│  "看看这个并购交易的回报"                                         │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  OpenClaw 自动匹配 SKILL                                        │
│  (根据 trigger 关键词自动加载相关 SKILL)                         │
│                                                              │
│   "comps", "comparable", "peer"     → comps-analysis SKILL    │
│   "DCF", "折现", "估值"              → dcf-model SKILL        │
│   "LBO", "杠杆收购"                  → lbo-model SKILL        │
│   "merger", "并购", "收购"           → merger-model SKILL    │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  SKILL.md (分析步骤)                                            │
│  • Trigger: 定义触发关键词                                       │
│  • Section 1-11: 完整分析步骤和公式规范                          │
│  • Quality Checklist: 质量检查清单                             │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  AI 执行分析                                                     │
│                                                              │
│   Step 1: 理解用户需求                                          │
│   Step 2: 需要数据时，自动调用对应的 Data Tool                  │
│   Step 3: 根据 SKILL.md 指导完成计算                            │
│   Step 4: 生成结果                                              │
│                                                              │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              按需自动调用 Data Tools                    │  │
│   │  get_historical_financials (需要财务数据时)              │  │
│   │  get_market_data (需要行情数据时)                       │  │
│   │  get_news (需要新闻时)                                  │  │
│   │  ... (11 Tools 按需调用)                               │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                              │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  返回结果 (Excel/PDF)                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 对比: 单层 vs 分层

| 模式 | 架构 | 扩展性 | 复用 |
|------|------|--------|------|
| **旧版 (v3.1)** | 单层 | 有限 | 无 |
| **新版 (v4.0)** | Core + Add-on | 灵活 | Add-ons 复用 Core |

---

## 六、文件结构

### 4.1 目录结构

```
financial-plugin/
├── package.json                      # npm 元数据 + openclaw 配置
├── openclaw.plugin.json              # 插件清单
├── index.ts                          # 入口点 (定义 11 Tools)
├── tsconfig.json                     # TypeScript 配置
│
├── skills/                           # 知识库 (自动触发)
│   ├── comps-analysis/
│   │   └── SKILL.md                  # Trigger: comps, comparable, peer
│   ├── dcf-model/
│   │   └── SKILL.md                  # Trigger: dcf, 折现, 估值
│   ├── lbo-model/
│   │   └── SKILL.md                  # Trigger: lbo, 杠杆收购
│   └── merger-model/
│       └── SKILL.md                  # Trigger: merger, 并购, 收购
│
└── src/
    └── tools/                         # 11 Data Tools
        ├── get_historical_financials.ts
        ├── get_morningstar_data.ts
        ├── get_sp_global_data.ts
        ├── get_factset_data.ts
        ├── get_moodys_data.ts
        ├── get_news.ts
        ├── get_earnings_call.ts
        ├── get_lseg_data.ts
        ├── get_pitchbook_data.ts
        ├── get_chronograph_data.ts
        └── get_document.ts
```

**注意**: 移除了 commands/ 目录，采用 SKILL 自动触发模式。

---

## 七、SKILL.md Trigger 设计

### 5.1 comps-analysis/SKILL.md

```yaml
---
name: comps-analysis
description: |
  Build institutional-grade comparable company analyses...

trigger: |
  - "comps", "comparable", "peer analysis"
  - "可比公司", "同行分析"
  - "估值倍数", "trading multiples"
  - User asks for company valuation comparison

not-ideal-for: |
  - Private companies without public peers
  - Pre-revenue startups
---
```

### 5.2 dcf-model/SKILL.md

```yaml
---
name: dcf-model
description: |
  Build DCF valuation model...

trigger: |
  - "DCF", "dcf valuation", "discounted cash flow"
  - "现金流折现", "估值"
  - "intrinsic value"
---
```

### 5.3 lbo-model/SKILL.md

```yaml
---
name: lbo-model
description: |
  Build LBO model...

trigger: |
  - "LBO", "leveraged buyout"
  - "杠杆收购", "私有化"
  - "buyback"
---
```

### 5.4 merger-model/SKILL.md

```yaml
---
name: merger-model
description: |
  Build merger/acquisition analysis...

trigger: |
  - "M&A", "merger", "acquisition"
  - "并购", "收购", "交易"
  - "synergy", "协同效应"
---
```

---

## 八、核心文件规范

### 6.1 package.json

```json
{
  "name": "@openclaw/financial-core",
  "version": "1.0.1",
  "type": "module",
  "main": "index.ts",
  "openclaw": {
    "extensions": ["./index.ts"],
    "skills": [
      "comps-analysis",
      "dcf-model",
      "lbo-model",
      "merger-model",
      "3-statement-model",
      "competitive-analysis",
      "deck-refresh",
      "ib-check-deck",
      "ppt-template-creator",
      "audit-xls",
      "clean-data-xls"
    ],
    "tools": [
      "get_historical_financials",
      "get_morningstar_data",
      "get_sp_global_data",
      "get_factset_data",
      "get_moodys_data",
      "get_news",
      "get_earnings_call",
      "get_lseg_data",
      "get_pitchbook_data",
      "get_chronograph_data",
      "get_document"
    ]
  },
  "dependencies": {
    "openclaw": "workspace:*",
    "@sinclair/typebox": "^0.32.0"
  }
}
```

### 6.2 openclaw.plugin.json

```json
{
  "id": "financial-core",
  "name": "Financial Core",
  "description": "Core financial analysis with 11 data sources - Comps, DCF, LBO, Merger models",
  "version": "1.0.1",
  "configSchema": {
    "type": "object",
    "properties": {
      "defaultDataSource": {
        "type": "string",
        "default": "sec-edgar"
      }
    }
  },
  "skills": [
    { "id": "comps-analysis", "path": "skills/comps-analysis/SKILL.md" },
    { "id": "dcf-model", "path": "skills/dcf-model/SKILL.md" },
    { "id": "lbo-model", "path": "skills/lbo-model/SKILL.md" },
    { "id": "merger-model", "path": "skills/merger-model/SKILL.md" },
    { "id": "3-statement-model", "path": "skills/3-statement-model/SKILL.md" },
    { "id": "competitive-analysis", "path": "skills/competitive-analysis/SKILL.md" },
    { "id": "deck-refresh", "path": "skills/deck-refresh/SKILL.md" },
    { "id": "ib-check-deck", "path": "skills/ib-check-deck/SKILL.md" },
    { "id": "ppt-template-creator", "path": "skills/ppt-template-creator/SKILL.md" },
    { "id": "audit-xls", "path": "skills/audit-xls/SKILL.md" },
    { "id": "clean-data-xls", "path": "skills/clean-data-xls/SKILL.md" }
  ],
  "tools": [
    "get_historical_financials",
    "get_morningstar_data",
    "get_sp_global_data",
    "get_factset_data",
    "get_moodys_data",
    "get_news",
    "get_earnings_call",
    "get_lseg_data",
    "get_pitchbook_data",
    "get_chronograph_data",
    "get_document"
  ]
}
```

---

## 九、入口点定义 (index.ts)

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/core";
import { Type } from "@sinclair/typebox";

export default definePluginEntry({
  id: "financial-core",
  name: "Financial Core",
  description: "Core financial analysis with 11 data sources - SKILL auto-trigger",
  
  register(api) {
    // 注册 11 Data Tools (按需自动调用)
    api.registerTool({
      name: "get_historical_financials",
      description: "Fetch historical financial data",
      parameters: Type.Object({
        ticker: Type.String(),
        years: Type.Optional(Type.Number())
      }),
      async execute(id, params) {
        return { content: [{ type: "text", text: "financial data" }] };
      }
    });

    // ... 其他 10 个 Tools 类似注册

    // 无需注册 Command，依赖 SKILL 自动触发
  }
});
```

---

## 十、Skills 设计 (参考 Anthropic FSI)

### 10.1 核心原则

**Skills 内容完全参考 Anthropic FSI**，仅修改数据源调用方式：

```
Anthropic FSI                           financial-plugin
─────────────────                       ─────────────────
MCP Server 调用                    →    OpenBB REST API 调用
mcp_client.call("equity.price...") →    fetch(OPENBB_API + "/api/v1/equity/price...")
SKILL.md 内容                       →    完全保留
公式/格式规范                        →    完全保留
```

**设计决策**: 仅保留 Skills 自动触发，不实现 Commands。用户通过自然语言交互，AI 自动匹配 Skill。

### 10.2 11 Core Skills (来自 Anthropic FSI)

| Skill | 来源 | SKILL.md |
|-------|------|----------|
| **comps-analysis** | Anthropic FSI | 可比公司分析 - 运营指标、估值倍数、统计基准 |
| **dcf-model** | Anthropic FSI | DCF 估值模型 - 现金流预测、WACC、敏感性分析 |
| **lbo-model** | Anthropic FSI | LBO 杠杆收购模型 - 债务结构、IRR 计算 |
| **3-statement-model** | Anthropic FSI | 三表财务模型 - BS/IS/CF 联动 |
| **competitive-analysis** | Anthropic FSI | 竞争分析 - 行业格局、竞争对手对比 |
| **deck-refresh** | Anthropic FSI | PPT 更新 - 数据刷新、格式保持 |
| **ib-check-deck** | Anthropic FSI | 投行 Deck 检查 - 质量控制清单 |
| **ppt-template-creator** | Anthropic FSI | PPT 模板创建 - 自定义模板 |
| **audit-xls** | Anthropic FSI | Excel 审计 - 公式检查、错误检测 |
| **clean-data-xls** | Anthropic FSI | 数据清洗 - 格式标准化 |
| **skill-creator** | Anthropic FSI | 创建新 Skill - Skill 编写指南 |

### 10.3 SKILL.md 修改规范

**原则**: 最小化修改，仅改数据源引用。

#### 修改前 (Anthropic FSI)

```markdown
## Step 1: 获取财务数据

使用 MCP 数据源:
- S&P Kensho MCP: 获取历史财务数据
- FactSet MCP: 获取估值倍数
- Daloopa MCP: 获取 KPI 数据
```

#### 修改后 (financial-plugin)

```markdown
## Step 1: 获取财务数据

使用 OpenBB REST API:
- get_historical_financials: 获取历史财务数据
- get_valuation_multiples: 获取估值倍数
- get_market_data: 获取市场数据
```

#### 不修改的内容

- 所有公式定义
- 所有格式规范
- 所有计算步骤
- 所有输出模板
- 所有质量检查清单
- **references 目录下的所有 .md 文件**
- **scripts 目录下的所有脚本**

### 10.5 Anthropic FSI Skill 完整目录结构

**重要**: 实现时需完整复制每个 skill 的所有文件和目录。

#### Skill 目录结构总览

| Skill | SKILL.md | references/ | scripts/ | 其他文件 |
|-------|----------|-------------|----------|---------|
| **comps-analysis** | ✅ | ❌ | ❌ | - |
| **dcf-model** | ✅ | ❌ | ✅ | TROUBLESHOOTING.md, requirements.txt |
| **lbo-model** | ✅ | ❌ | ❌ | - |
| **3-statement-model** | ✅ | ✅ | ❌ | - |
| **competitive-analysis** | ✅ | ✅ | ❌ | - |
| **deck-refresh** | ✅ | ❌ | ❌ | - |
| **ib-check-deck** | ✅ | ✅ | ✅ | - |
| **ppt-template-creator** | ✅ | ❌ | ❌ | - |
| **audit-xls** | ✅ | ❌ | ❌ | - |
| **clean-data-xls** | ✅ | ❌ | ❌ | - |
| **skill-creator** | ✅ | ✅ | ✅ | LICENSE.txt |

#### references 目录内容

| Skill | references 文件 | 内容 |
|-------|----------------|------|
| **3-statement-model** | formatting.md, formulas.md, sec-filings.md | 格式规范、公式定义、SEC 文件说明 |
| **competitive-analysis** | frameworks.md, schemas.md | 分析框架、数据结构 |
| **ib-check-deck** | ib-terminology.md, report-format.md | 投行术语、报告格式 |
| **skill-creator** | output-patterns.md, workflows.md | 输出模式、工作流 |

#### scripts 目录内容

| Skill | scripts 文件 | 用途 |
|-------|-------------|------|
| **dcf-model** | validate_dcf.py | DCF 模型验证 |
| **ib-check-deck** | extract_numbers.py | 数据提取 |
| **skill-creator** | init_skill.py, package_skill.py, quick_validate.py | Skill 创建工具 |

### 10.6 financial-plugin Skill 目录结构

```
skills/
├── comps-analysis/
│   └── SKILL.md              # 完全复制 Anthropic FSI
│
├── dcf-model/
│   ├── SKILL.md
│   ├── TROUBLESHOOTING.md    # 复制
│   ├── requirements.txt      # 复制
│   └── scripts/
│       └── validate_dcf.py   # 复制
│
├── lbo-model/
│   └── SKILL.md
│
├── 3-statement-model/
│   ├── SKILL.md
│   └── references/
│       ├── formatting.md     # 复制
│       ├── formulas.md       # 复制
│       └── sec-filings.md    # 复制
│
├── competitive-analysis/
│   ├── SKILL.md
│   └── references/
│       ├── frameworks.md     # 复制
│       └── schemas.md        # 复制
│
├── deck-refresh/
│   └── SKILL.md
│
├── ib-check-deck/
│   ├── SKILL.md
│   ├── references/
│   │   ├── ib-terminology.md # 复制
│   │   └── report-format.md  # 复制
│   └── scripts/
│       └── extract_numbers.py # 复制
│
├── ppt-template-creator/
│   └── SKILL.md
│
├── audit-xls/
│   └── SKILL.md
│
├── clean-data-xls/
│   └── SKILL.md
│
└── skill-creator/
    ├── SKILL.md
    ├── LICENSE.txt
    ├── references/
    │   ├── output-patterns.md # 复制
    │   └── workflows.md       # 复制
    └── scripts/
        ├── init_skill.py      # 复制
        ├── package_skill.py   # 复制
        └── quick_validate.py  # 复制
```
---

## 十一、与 v4.0 对比

| 维度 | v3.1 (旧) | v4.0 (当前) |
|------|----------|-------------|
| **架构** | 单层 | **Core + Add-on** |
| **11 Tools** | 集中 | **集中在 Core** |
| **Add-ons** | 无 | **4 个可选** |
| **扩展性** | 有限 | **灵活** |
| **复用** | 无 | **Add-ons 复用 Core** |

---

## 十一、实施计划

### Phase 1: Core Plugin (financial-core)
- [ ] 11 Data Tools 实现
- [ ] 11 Core Skills 实现:
  - [ ] comps-analysis (可比公司分析)
  - [ ] dcf-model (DCF 估值模型)
  - [ ] lbo-model (LBO 杠杆收购模型)
  - [ ] 3-statement-model (三表财务模型)
  - [ ] competitive-analysis (竞争分析)
  - [ ] deck-refresh (PPT 更新)
  - [ ] ib-check-deck (投行 deck 检查)
  - [ ] ppt-template-creator (PPT 模板创建)
  - [ ] audit-xls (Excel 审计)
  - [ ] clean-data-xls (数据清洗)
  - [ ] skill-creator (创建新 Skill)

### Phase 2: Add-ons
| Add-on | Skills 数量 | 包含内容 |
|--------|------------|---------|
| investment-banking | 9 | CIM, Teaser, Pitch Deck, Merger Model, Buyer List, Deal Tracker... |
| private-equity | 10 | Deal Sourcing, DD Checklist, IC Memo, Returns Analysis, Unit Economics... |
| equity-research | 9 | Earnings Analysis, Initiating Coverage, Model Update, Thesis Tracker... |
| wealth-management | 6 | Client Review, Financial Plan, Portfolio Rebalance, Tax Loss Harvesting... |

### Phase 3: 测试
- [ ] Core 安装测试
- [ ] Add-on 安装测试
- [ ] SKILL 自动触发测试

---

### 附录: 45 Skills 总览

| 层级 | Skills 数量 |
|------|-------------|
| Core (financial-core) | 11 |
| investment-banking (Add-on) | 9 |
| private-equity (Add-on) | 10 |
| equity-research (Add-on) | 9 |
| wealth-management (Add-on) | 6 |
| **总计** | **45** |

---

## 十二、目录结构建议

### 12.1 三类目录

| 类型 | 路径 | 用途 |
|------|------|------|
| **设计文档** | `~/.openclaw/workspace/memory/projects/financial-plugin/` | DESIGN.md, REFERENCE.md, RULES.md |
| **源代码 (开发)** | `~/.openclaw/workspace/projects/financial-plugin/` | Core + Add-ons 源码 |
| **部署 (运行时)** | `~/.openclaw/extensions/` | 安装后的插件 |

### 12.2 源代码目录结构

```
~/.openclaw/workspace/projects/financial-plugin/
├── financial-core/              # Core (必装)
│   ├── package.json
│   ├── openclaw.plugin.json
│   ├── index.ts
│   ├── skills/                  # 11 Core Skills
│   └── src/tools/               # 11 Data Tools
│
├── investment-banking/          # Add-on
│   ├── package.json
│   ├── openclaw.plugin.json
│   └── skills/                  # 9 Skills
│
├── private-equity/              # Add-on
│   └── skills/                  # 10 Skills
│
├── equity-research/             # Add-on
│   └── skills/                  # 9 Skills
│
└── wealth-management/           # Add-on
    └── skills/                  # 6 Skills
```

### 12.3 部署目录结构

```
~/.openclaw/extensions/
├── financial-core/              # Core
├── investment-banking/
├── private-equity/
├── equity-research/
└── wealth-management/
```

### 12.4 安装命令

```bash
# 链接开发 (推荐开发时)
openclaw plugins install -l ~/.openclaw/workspace/projects/financial-plugin/financial-core
openclaw plugins install -l ~/.openclaw/workspace/projects/financial-plugin/investment-banking
openclaw plugins install -l ~/.openclaw/workspace/projects/financial-plugin/private-equity
openclaw plugins install -l ~/.openclaw/workspace/projects/financial-plugin/equity-research
openclaw plugins install -l ~/.openclaw/workspace/projects/financial-plugin/wealth-management

# 或复制安装 (生产环境)
openclaw plugins install ~/.openclaw/workspace/projects/financial-plugin/financial-core
```

---

## 十三、未来优化建议（Future Enhancements）

### 13.1 量化交易 Add-on（基于 FinRL）

**评估日期**: 2026-03-22  
**来源**: [FinRL](https://github.com/AI4Finance-Foundation/FinRL)  
**优先级**: 低（Phase 3 之后）

#### FinRL 概述

FinRL 是一个开源金融强化学习框架，专注于：
- 深度强化学习交易策略（A2C, PPO, SAC, TD3, DDPG）
- 自动化股票交易
- 投资组合优化
- 交易回测

#### 与 financial-plugin 的关系

| 维度 | FinRL | financial-plugin Core |
|------|-------|----------------------|
| 定位 | 量化交易/策略训练 | 财务分析/估值建模 |
| 核心功能 | DRL 训练、回测 | 数据获取、分析 |
| 数据源 | 13 个（OHLCV + 技术指标） | 100+（OpenBB） |
| 财务数据 | ❌ 无 | ✅ 完整 |
| 估值建模 | ❌ 无 | ✅ 完整 |

#### 结论

**FinRL 不适合作为 Core 功能**，原因：
1. 无财务报表数据（资产负债表、利润表等）
2. 无估值数据（EV, P/E, DCF 等）
3. 数据仅用于强化学习训练，不适用于机构级分析

#### 建议方案

作为 **独立的 Add-on** 在 Phase 3 之后考虑：

```
financial-plugin/
├── financial-core/              # Core (OpenBB 驱动)
│   └── 11 Core Skills
│
├── investment-banking/          # Add-on
├── private-equity/              # Add-on
├── equity-research/             # Add-on
├── wealth-management/           # Add-on
│   └── portfolio-optimization/  # 可用 FinRL 增强
│
└── quant-trading/               # 🆕 未来新 Add-on
    ├── algorithmic-trading/     # 算法交易
    ├── portfolio-optimization/  # 组合优化（FinRL）
    └── risk-management/         # 风险管理
```

#### 潜在 Skills

| Skill | 功能 | 数据来源 |
|-------|------|---------|
| **portfolio-optimization** | 强化学习资产配置 | FinRL + OpenBB |
| **algorithmic-trading** | 自动化交易策略 | FinRL Agents |
| **risk-parity** | 风险平价模型 | FinRL + 自有数据 |
| **tactical-asset-allocation** | 战术资产配置 | DRL 预测 |

#### 技术实现

```python
# quant-trading/index.ts
import { definePluginEntry } from "openclaw/plugin-sdk/core";
import finrl from "finrl";  # FinRL Python 绑定

export default definePluginEntry({
  id: "quant-trading",
  name: "Quant Trading",
  description: "DRL-based quantitative trading strategies",
  
  register(api) {
    // 组合优化 Tool
    api.registerTool({
      name: "optimize_portfolio",
      description: "Optimize portfolio using FinRL",
      async execute(params) {
        // 调用 FinRL 训练好的 agent
        const result = await finrl.portfolio.optimize(params);
        return result;
      }
    });
  }
});
```

#### 依赖关系

```
quant-trading (Add-on)
    ├── 依赖: financial-core (获取历史价格)
    ├── 依赖: wealth-management (客户组合数据)
    └── 依赖: FinRL (强化学习框架)
```

#### 参考资源

- **GitHub**: https://github.com/AI4Finance-Foundation/FinRL
- **文档**: https://finrl.readthedocs.io/
- **论文**: FinRL: Deep reinforcement learning framework (ICAIF 2021)
- **相关项目**: 
  - FinRL-Meta: 市场环境与基准
  - ElegantRL: DRL 算法库
  - FinGPT: 金融大语言模型

---

## 十四、技术架构方案（Technology Architecture）

### 14.1 方案对比与选择

| 方案 | 数据源 | 费用 | AI 集成 | 维护成本 | 推荐度 |
|------|--------|------|---------|----------|--------|
| 11 MCP | 专业级 | ~$50,000/年 | 需开发 | 低 | ⭐⭐ |
| finagg | 3 个 | 免费 | 需封装 | 中 | ⭐⭐⭐ |
| **OpenBB** | **100+** | **免费** | **原生 MCP** | **低** | **⭐⭐⭐⭐⭐** |
| 自研 API | 自定义 | 免费 | 需开发 | 高 | ⭐⭐ |

**决策**: 采用 **OpenBB** 作为 financial-core 的数据层

---

### 14.2 OpenBB 架构设计

#### 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Gateway                            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    financial-core (纯 OpenBB)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              OpenBB Platform (100+ 数据源)                  │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │   │
│  │  │   SEC   │ │  FRED   │ │  Yahoo  │ │  Alpha  │          │   │
│  │  │ EDGAR   │ │         │ │ Finance │ │Vantage  │          │   │
│  │  │(via OB) │ │         │ │         │ │         │          │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │   │
│  │  │Polygon  │ │Finnhub  │ │  BEA    │ │  ...    │          │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │   │
│  │                                                             │   │
│  │  统一接口: obb.equity.* / obb.economic.* / obb.news.*       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              OpenClaw Tool Layer (11 Tools)                 │   │
│  │                                                             │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  get_historical_financials                          │   │   │
│  │  │  └── OpenBB: obb.equity.fundamental.*               │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  get_market_data                                    │   │   │
│  │  │  └── OpenBB: obb.equity.price.*                     │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  get_economic_data                                  │   │   │
│  │  │  └── OpenBB: obb.economic.*                         │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  │  ... (共 11 个)                                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Data Processing Layer                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │ DataFrame   │→│ Transformer │→│ Normalizer  │         │   │
│  │  │ (OpenBB)    │  │ (自建)      │  │ (US GAAP)   │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Export Layer (自建)                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │ Excel       │  │ JSON        │  │ Markdown    │         │   │
│  │  │ (openpyxl)  │  │ (标准格式)   │  │ (报告)      │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              11 Core Skills                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 14.3 11 Tools 与 OpenBB 映射

| Tool | OpenBB API | 数据源 | 输出 |
|------|-----------|--------|------|
| **get_historical_financials** | `obb.equity.fundamental.income/balance/cash` | SEC EDGAR (via OpenBB) | 三表数据 |
| **get_market_data** | `obb.equity.price.historical` | Yahoo Finance, Polygon | 股价、市值 |
| **get_valuation_data** | `obb.equity.fundamental.metrics` | OpenBB 计算 | 估值倍数 |
| **get_economic_data** | `obb.economic.indicators/gdp` | FRED, BEA | 宏观数据 |
| **get_news** | `obb.news.company` | Finnhub, NewsAPI | 新闻 |
| **get_industry_data** | `obb.equity.compare.groups` | OpenBB | 行业对比 |
| **get_ownership_data** | `obb.equity.ownership` | OpenBB | 股东数据 |
| **get_calendar** | `obb.equity.calendar` | OpenBB | 财报日历 |
| **get_options_data** | `obb.derivatives.options` | Polygon | 期权 |
| **get_crypto_data** | `obb.crypto.price` | CCXT, Binance | 加密货币 |
| **get_futures_data** | `obb.derivatives.futures` | Yahoo Finance | 期货 |

---

### 14.4 项目结构

```
financial-core/
├── package.json                      # npm 配置
├── openclaw.plugin.json              # 插件清单
├── requirements.txt                  # Python 依赖
│   └── openbb[all]>=4.0.0
├── index.ts                          # 入口点
├── src/
│   ├── tools/                        # 11 Tools（纯 OpenBB）
│   │   ├── get_historical_financials.ts
│   │   ├── get_market_data.ts
│   │   ├── get_valuation_data.ts
│   │   ├── get_economic_data.ts
│   │   ├── get_news.ts
│   │   ├── get_industry_data.ts
│   │   ├── get_ownership_data.ts
│   │   ├── get_calendar.ts
│   │   ├── get_options_data.ts
│   │   ├── get_crypto_data.ts
│   │   └── get_futures_data.ts
│   ├── adapters/                     # OpenBB 适配
│   │   ├── openbb_client.py          # Python 桥接
│   │   └── openbb_types.ts           # 类型定义
│   ├── transformers/                 # 数据转换
│   │   ├── data_transformer.ts
│   │   └── us_gaap_mapper.ts
│   ├── exporters/                    # 导出层
│   │   ├── excel_exporter.ts
│   │   ├── json_exporter.ts
│   │   └── markdown_exporter.ts
│   ├── validators/                   # 验证层
│   │   ├── us_gaap_validator.ts
│   │   └── data_quality_checker.ts
│   ├── calculators/                  # 计算层
│   │   ├── financial_ratios.ts
│   │   ├── valuation_multiples.ts
│   │   └── statistics.ts
│   └── types/                        # 类型定义
│       └── index.ts
├── skills/                           # 11 Core Skills
│   ├── comps-analysis/
│   │   └── SKILL.md
│   ├── dcf-model/
│   ├── lbo-model/
│   ├── merger-model/
│   ├── 3-statement-model/
│   ├── competitive-analysis/
│   ├── deck-refresh/
│   ├── ib-check-deck/
│   ├── ppt-template-creator/
│   ├── audit-xls/
│   └── clean-data-xls/
└── tests/
    └── ...
```

---

### 14.5 Excel 导出实现

**参考 Anthropic FSI 规范，自建实现（非 AI 生成）：**

```typescript
// src/exporters/excel_exporter.ts
import * as XLSX from 'xlsx';
import { FinancialData } from '../types';

export class ExcelExporter {
  /**
   * 导出可比公司分析 Excel
   * 按照 Anthropic FSI 规范实现
   */
  exportComps(data: FinancialData[]): Buffer {
    const wb = XLSX.utils.book_new();
    
    // Sheet 1: 封面
    const coverWs = this.createCoverSheet(data);
    XLSX.utils.book_append_sheet(wb, coverWs, 'Cover');
    
    // Sheet 2: 运营指标
    const opsWs = this.createOperatingSheet(data);
    XLSX.utils.book_append_sheet(wb, opsWs, 'Operating');
    
    // Sheet 3: 估值倍数
    const valWs = this.createValuationSheet(data);
    XLSX.utils.book_append_sheet(wb, valWs, 'Valuation');
    
    // Sheet 4: 统计数据
    const statsWs = this.createStatisticsSheet(data);
    XLSX.utils.book_append_sheet(wb, statsWs, 'Statistics');
    
    return XLSX.write(wb, { type: 'buffer', bookType: 'xlsx' });
  }
  
  /**
   * 格式规范（Anthropic FSI 标准）
   * - Section headers: Dark blue (#1F4E79)
   * - Column headers: Light blue (#D9E1F2)
   * - Statistics: Light grey (#F2F2F2)
   * - Formulas: 写入公式而非硬编码值
   */
  private createOperatingSheet(data: FinancialData[]): XLSX.WorkSheet {
    const headers = ['Company', 'Revenue', 'Growth', 'Gross Margin', 'EBITDA', 'EBITDA Margin'];
    
    // 写入公式而非值
    const rows = data.map((d, i) => [
      d.ticker,
      d.revenue,
      `=C${i+2}/C${i+1}-1`,  // Growth formula
      `=E${i+2}/C${i+2}`,    // Gross Margin formula
      d.ebitda,
      `=G${i+2}/C${i+2}`     // EBITDA Margin formula
    ]);
    
    // 添加统计行 (MAX, 75%, MEDIAN, 25%, MIN)
    const stats = this.calculateStatistics(rows);
    rows.push(...stats);
    
    return XLSX.utils.aoa_to_sheet([headers, ...rows]);
  }
}
```

---

### 14.6 实施计划

| Phase | 任务 | 时间 | 依赖 |
|-------|------|------|------|
| Phase 0 | OpenBB 环境搭建、API 调研 | 2 天 | - |
| Phase 1 | 自建 Excel 导出器 | 3 天 | Phase 0 |
| Phase 2 | 自建 US GAAP 转换器 + 验证器 | 3 天 | Phase 0 |
| Phase 3 | 实现 11 个 Tools | 5 天 | Phase 1-2 |
| Phase 4 | 自建财务比率计算 | 2 天 | Phase 2 |
| Phase 5 | 测试 6 个 Skill | 3 天 | Phase 3-4 |
| **总计** | | **18 天** | |

---

### 14.7 关键决策

| 决策项 | 选择 | 原因 |
|--------|------|------|
| 数据源 | OpenBB | 100+ 免费数据源，原生 MCP 支持 |
| Excel 导出 | 自建代码库 | 比 AI 实时生成更可靠、可测试 |
| US GAAP 映射 | 自建 | 完全可控，符合机构级标准 |
| 复用 us-stock-financials | ❌ 否 | 统一架构，纯 OpenBB |

---

*文档版本: 4.5*  
*最后更新: 2026-03-23*  
*特点: 新增基于 OpenBB 的技术架构方案*