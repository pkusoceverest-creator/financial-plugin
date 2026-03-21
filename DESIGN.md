# Financial Plugin 项目设计方案

> 基于 OpenClaw Native Plugin 架构
> 参考: Anthropic Financial Services Plugin (11 MCP → 11 Tools)
> 设计日期: 2026-03-21
> 版本: 4.0 (Core + Add-on 分层设计)

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
│  • 4 Core Skills (comps, dcf, lbo, merger)                                 │
│  • 核心建模能力                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

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
| **Core** | financial-core | ✅ | 11 Tools + 4 Skills |
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

## 四、11 MCP → 11 Tools 映射

| 序号 | MCP (Anthropic) | 对应 Tool | 功能 |
|------|-----------------|-----------|------|
| 1 | Daloopa | `get_historical_financials` | 历史财务数据 |
| 2 | Morningstar | `get_morningstar_data` | 晨星分析数据 |
| 3 | S&P Global | `get_sp_global_data` | Capital IQ 数据 |
| 4 | FactSet | `get_factset_data` | FactSet 数据 |
| 5 | Moody's | `get_moodys_data` | 债券/信用数据 |
| 6 | MT Newswires | `get_news` | 实时新闻 |
| 7 | Aiera | `get_earnings_call` | 财报电话会 |
| 8 | LSEG | `get_lseg_data` | 债券/FX/宏观 |
| 9 | PitchBook | `get_pitchbook_data` | PE/VC 数据 |
| 10 | Chronograph | `get_chronograph_data` | 历史数据 |
| 11 | Egnyte | `get_document` | 文档管理 |

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
  "name": "@openclaw/financial-plugin",
  "version": "1.0.1",
  "type": "module",
  "main": "index.ts",
  "openclaw": {
    "extensions": ["./index.ts"],
    "skills": ["comps-analysis", "dcf-model", "lbo-model", "merger-model"],
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
  "id": "financial-plugin",
  "name": "Financial Plugin",
  "description": "Financial analysis: Comps, DCF, LBO, Merger with 11 data sources",
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
    { "id": "merger-model", "path": "skills/merger-model/SKILL.md" }
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
  id: "financial-plugin",
  name: "Financial Plugin",
  description: "Financial analysis with 11 data sources - SKILL auto-trigger",
  
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

## 十、与 v3.1 对比

| 维度 | v3.1 (旧) | v4.0 (当前) |
|------|----------|-------------|
| **架构** | 单层 | **Core + Add-on** |
| **11 Tools** | 集中 | **集中在 Core** |
| **Add-ons** | 无 | **4 个可选** |
| **扩展性** | 有限 | **灵活** |
| **复用** | 无 | **Add-ons 复用 Core** |

---

## 十一、实施计划

### Phase 1: Core Plugin
- [ ] financial-core 目录结构
- [ ] 11 Tools 实现
- [ ] 4 Core Skills (comps-analysis, dcf-model, lbo-model, merger-model)

### Phase 2: Add-ons (可选)
- [ ] investment-banking
- [ ] equity-research
- [ ] private-equity
- [ ] wealth-management

### Phase 3: 测试
- [ ] Core 安装测试
- [ ] Add-on 安装测试
- [ ] SKILL 自动触发测试

---

*文档版本: 4.0*
*最后更新: 2026-03-21*
*特点: SKILL 自动触发，符合 OpenClaw Best Practice*