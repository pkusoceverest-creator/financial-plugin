# Anthropic FSI 参考设计文档

> 来源: https://github.com/anthropics/financial-services-plugins
> 分析日期: 2026-03-21
> 文档版本: 2.0 (拆分版)

---

## 文档导航

| 文档 | 内容 | 行数 |
|------|------|------|
| **REFERENCE.md** | Anthropic FSI 架构设计 | ~600 行 |
| **TECHNICAL.md** | OpenBB REST API 技术实现 | ~500 行 |
| **ECOSYSTEM.md** | 第三方框架 (FinRL/Qlib/RD-Agent) | ~600 行 |

---

# Anthropic Financial Services Plugin - 参考设计文档

> 来源: https://github.com/anthropics/financial-services-plugins
> 分析日期: 2026-03-21

---

## 一、项目概述

将 Claude 打造为金融服务专业人员 (投资银行、股权研究、私募股权、财富管理) 的专业助手。

**核心规模**:
- 41 个 Skills
- 38 个 Commands
- 11 个 MCP Integrations

---

## 二、整体架构

### 2.1 分层设计模式

```
┌─────────────────────────────────────────────────────────────┐
│                    Add-on Plugins                             │
├─────────────────┬─────────────────┬──────────────────────────┤
│ investment-    │ equity-        │ private-   │ wealth-     │
│ banking        │ research       │ equity     │ management  │
└─────────────────┴────────────────┴────────────┴─────────────┘
           ↑              ↑              ↑           ↑
                    共享依赖 Core Plugin
┌─────────────────────────────────────────────────────────────┐
│            financial-analysis (Core - 必装)                 │
│  • 共享建模工具 (Comps, DCF, LBO, 3-Statement)             │
│  • 所有 Data Connectors (11 个 MCP)                         │
│  • 11 个 Skills + 8 个 Commands                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 设计原则

- **Core First**: 必须先安装 `financial-analysis`
- **Connectors 集中化**: 所有 MCP 配置只在 Core 中定义，Add-ons 继承复用
- **可扩展**: 新增功能只需创建新的 Add-on，无需修改 Core
- **Skill 驱动**: 分析逻辑封装在 SKILL.md
- **质量标准化**: 输出有明确 Checklist

### 2.3 插件依赖关系

```
financial-analysis (Core - 必装)
    │
    ├── investment-banking ─────┐
    ├── private-equity         │
    ├── equity-research        │──→ 依赖 Core 的 Tools + Skills
    └── wealth-management ─────┘
                                   
    (Partner-built 插件独立，无依赖)
```

### 2.4 各插件功能详情

| 插件 | 类型 | 核心功能 | 数据源 |
|------|------|----------|--------|
| **financial-analysis** | Core (必装) | Comps, DCF, LBO, 3-Statement, 竞品分析, 报告 QC | 11 MCP |
| **investment-banking** | Add-on | CIM 起草, Teaser, 并购模型 | 复用 Core |
| **private-equity** | Add-on | 尽职调查, IC memo, 投资筛选 | 复用 Core |
| **equity-research** | Add-on | 财报更新, 首次覆盖报告 | 复用 Core |
| **wealth-management** | Add-on | 客户会议准备, 投资组合再平衡 | 复用 Core |
| **Partner-built (LREG/SPGlobal)** | Partner | 债券定价, FX 分析, tearsheet | 独立 |

---

## 三、目录结构与文件规范

### 3.1 插件目录结构

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # 插件清单
├── .mcp.json                # MCP 服务器配置
├── commands/                # 用户显式触发的命令
│   ├── comps.md
│   ├── dcf.md
│   └── lbo.md
├── skills/                  # 领域知识 (自动加载)
│   ├── comps-analysis/
│   │   └── SKILL.md
│   ├── dcf-model/
│   │   └── SKILL.md
│   └── ...
└── hooks/                   # 事件钩子 (可选)
```

### 3.2 关键文件格式

**plugin.json (清单)**
```json
{
  "name": "financial-analysis",
  "version": "0.1.0",
  "description": "Core financial modeling and analysis tools: DCF, comps, LBO, 3-statement models, competitive analysis, and deck QC",
  "author": {
    "name": "Anthropic FSI"
  }
}
```

**.mcp.json (MCP 配置)**
```json
{
  "mcpServers": {
    "daloopa": { "type": "http", "url": "https://mcp.daloopa.com/server/mcp" },
    "morningstar": { "type": "http", "url": "https://mcp.morningstar.com/mcp" },
    "sp-global": { "type": "http", "url": "https://kfinance.kensho.com/integrations/mcp" },
    "factset": { "type": "http", "url": "https://mcp.factset.com/mcp" },
    "moodys": { "type": "http", "url": "https://api.moodys.com/genai-ready-data/m1/mcp" },
    "mtnewswire": { "type": "http", "url": "https://vast-mcp.blueskyapi.com/mtnewswires" },
    "aiera": { "type": "http", "url": "https://mcp-pub.aiera.com" },
    "lseg": { "type": "http", "url": "https://api.analytics.lseg.com/lfa/mcp" },
    "pitchbook": { "type": "http", "url": "https://premium.mcp.pitchbook.com/mcp" },
    "chronograph": { "type": "http", "url": "https://ai.chronograph.pe/mcp" },
    "egnyte": { "type": "http", "url": "https://mcp-server.egnyte.com/mcp" }
  }
}
```

---

## 四、Skill 与 Command 协作模式

### 4.1 Skill (自动触发)

```yaml
---
name: comps-analysis
description: |
  Build institutional-grade comparable company analyses with operating metrics, valuation multiples, and statistical benchmarking in Excel/spreadsheet format.
  
  **Perfect for:**
  - Public company valuation (M&A, investment analysis)
  - Benchmarking performance vs. industry peers
  - Pricing IPOs or funding rounds
  - Identifying valuation outliers (over/under-valued)
  - Supporting investment committee presentations
  - Creating sector overview reports

  **Not ideal for:**
  - Private companies without comparable public peers
  - Highly diversified conglomerates
  - Distressed/bankrupt companies
  - Pre-revenue startups
  - Companies with unique business models
---
```

**触发机制**: 
- Claude 根据对话上下文**自动加载**相关 Skill
- 无需用户显式指定
- 包含详细的步骤指南、格式规范、注意事项

### 4.2 Command (显式触发)

```yaml
---
description: Build a comparable company analysis with trading multiples
argument-hint: "[company name or ticker]"
---

# Comparable Company Analysis Command
```

**触发机制**:
- 用户输入 `/comps AAPL` 显式调用
- Command 调用对应 Skill 执行任务

### 4.3 协作流程

```
用户输入 /comps AAPL
       ↓
Command (comps.md) 被激活
       ↓
加载 Skill: comps-analysis
       ↓
执行完整工作流 → 生成 Excel 输出
```

---

## 五、Skill 内部结构深度解析

以 `comps-analysis` Skill 为例，核心结构：

### 5.1 数据源优先级 (CRITICAL)

```markdown
## ⚠️ CRITICAL: Data Source Priority (READ FIRST)

1. **FIRST: Check for MCP data sources** - If S&P Kensho MCP, FactSet MCP, or Daloopa MCP are available, use them exclusively for financial and trading information
2. **DO NOT use web search** if the above MCP data sources are available
3. **ONLY if MCPs are unavailable:** Then use Bloomberg Terminal, SEC EDGAR filings, or other institutional sources
4. **NEVER use web search as a primary data source** - it lacks the accuracy, audit trails, and reliability required for institutional-grade analysis

**Why this matters:** MCP sources provide verified, institutional-grade data with proper citations. Web search results can be outdated, inaccurate, or unreliable for financial analysis.
```

### 5.2 详细工作流步骤 (11 个 Section)

1. Document Structure & Setup - 文档结构与设置
2. Operating Statistics & Financial Metrics - 运营统计与财务指标
3. Valuation Multiples & Investment Metrics - 估值倍数与投资指标
4. Notes & Methodology Documentation - 备注与方法论文档
5. Choosing the Right Metrics - 选择正确的指标 (决策框架)
6. Best Practices & Quality Checks - 最佳实践与质量检查
7. Advanced Features - 高级功能
8. Workflow & Practical Tips - 工作流与实用技巧
9. Industry-Specific Additions - 行业特定补充
10. Red Flags & Warning Signs - 风险信号与警示
11. Formulas Reference Guide - 公式参考指南

### 5.3 质量标准 (Output Checklist)

```markdown
## Output Checklist

Before delivering a comp analysis, verify:
- [ ] All companies are truly comparable
- [ ] Data is from consistent time periods
- [ ] Units are clearly labeled (millions/billions)
- [ ] Formulas reference cells, not hardcoded values
- [ ] **All hard-coded input cells have comments with either: (1) exact data source with citation, OR (2) clear assumption with explanation**
- [ ] **Hyperlinks added where relevant** (SEC EDGAR filings, Bloomberg pages, research reports)
- [ ] Statistics include at least 5 metrics (Max, 75th, Med, 25th, Min)
- [ ] Notes section documents sources and methodology
- [ ] Visual formatting follows conventions (blue = input, black = formula)
- [ ] Sanity checks pass (margins logical, multiples reasonable)
- [ ] Date stamp is current ("As of [Date]")
- [ ] Formula auditing shows no errors (#DIV/0!, #REF!, #N/A)
```

### 5.4 格式化规范 (可选但详细)

**颜色约定**: 
- 蓝色 = 输入 (hardcoded data)
- 黑色 = 公式 (formulas)

**默认色板**:
- Section headers: 深蓝背景 (#1F4E79 或 #17365D) + 白色加粗文字
- Column headers: 浅蓝背景 (#D9E1F2) + 黑色加粗文字
- Statistics rows: 浅灰背景 (#F2F2F2)
- Data rows: 白色背景

**字体**: Times New Roman, 11pt data, 12pt headers

**统计指标**: 必须包含 quartile 分析 (Max, 75th Percentile, Median, 25th Percentile, Min)

---

## 六、MCP Integrations

### 6.1 11 个数据源

| 提供商 | 用途 | MCP URL |
|--------|------|---------|
| Daloopa | 历史财务数据 | https://mcp.daloopa.com/server/mcp |
| Morningstar | 晨星分析 | https://mcp.morningstar.com/mcp |
| S&P Global | S&P Capital IQ | https://kfinance.kensho.com/integrations/mcp |
| FactSet | FactSet 数据 | https://mcp.factset.com/mcp |
| Moody's | Moody's 分析 | https://api.moodys.com/genai-ready-data/m1/mcp |
| MT Newswires | 新闻wire | https://vast-mcp.blueskyapi.com/mtnewswires |
| Aiera | 财报电话会 | https://mcp-pub.aiera.com |
| LSEG | 债券/FX/宏观 | https://api.analytics.lseg.com/lfa/mcp |
| PitchBook | PE/VC 数据 | https://premium.mcp.pitchbook.com/mcp |
| Chronograph | 历史数据 | https://ai.chronograph.pe/mcp |
| Egnyte | 文档管理 | https://mcp-server.egnyte.com/mcp |

### 6.2 数据源使用优先级

```
1. FIRST: MCP 数据源 (S&P Kensho, FactSet, Daloopa)
          ↓
2. ONLY IF MCPs unavailable: Bloomberg, SEC EDGAR
          ↓
3. NEVER: Web Search (不满足机构级准确度要求)
```

---

## 七、关键设计模式总结

| 设计点 | 实现方式 | 价值 |
|--------|----------|------|
| **分层架构** | Core + Add-on | 复用 + 扩展性 |
| **文件驱动** | 纯 Markdown + JSON | 无需代码/基础设施 |
| **双触发机制** | Auto Skill + Manual Command | 灵活 + 可控 |
| **数据源集中** | Core 统一管理 MCP | 避免重复配置 |
| **质量内建** | 详细的 checklist 在 Skill 中 | 输出标准化 |
| **优先级策略** | MCP > Bloomberg > Web Search | 机构级准确度 |
| **可定制性** | 可 fork + PR 新增 | 生态扩展 |

---

## 八、工作流示例: Comps Analysis

### 8.1 Command 定义 (commands/comps.md)

```yaml
---
description: Build a comparable company analysis with trading multiples
argument-hint: "[company name or ticker]"
---

# Comparable Company Analysis Command

Build an institutional-grade comparable company analysis with operating metrics, valuation multiples, and statistical benchmarking.

## Workflow

### Step 1: Gather Company Information

If a company name or ticker is provided, use it. Otherwise ask:
- "What company would you like to analyze?"

### Step 2: Load Comps Analysis Skill

Use `skill: "comps-analysis"` to build the analysis:

1. **Clarify the analysis purpose**:
   - "What's the key question?" (valuation, efficiency, growth comparison)
   - "Who is the audience?" (IC, board, quick reference)
   - "Do you have a preferred format or template?"

2. **Identify peer group** (4-6 comparable companies):
   - Similar business model
   - Similar scale/market cap range
   - Same industry/sector
   - Geographic comparability

3. **Gather data** (prioritize MCP sources if available):
   - Operating metrics: Revenue, Growth, Gross Margin, EBITDA, EBITDA Margin
   - Valuation: Market Cap, Enterprise Value, EV/Revenue, EV/EBITDA, P/E
   - Additional metrics based on industry (Rule of 40 for SaaS, etc.)

4. **Build the analysis**:
   - Operating Statistics section with company data + statistics (Max, 75th, Median, 25th, Min)
   - Valuation Multiples section with same statistical summary
   - Notes & Methodology documentation

### Step 3: Create Excel Output

Generate Excel file with:
- Header block (analysis title, companies, date, units)
- Operating Statistics & Financial Metrics section
- Valuation Multiples section
- Statistical summary for each metric
- Notes section documenting sources and methodology

### Step 4: Deliver Output

Provide:
1. **Excel file** (.xlsx) - the comps analysis
2. **Summary** highlighting:
   - Peer group selection rationale
   - Key insights (who trades at premium/discount)
   - Median multiples for reference
```

### 8.2 Skill 结构 (skills/comps-analysis/SKILL.md) 要点

完整的 SKILL.md 包含:
1. **Trigger conditions** - 何时触发此 skill
2. **Data source priority rules** - 数据源优先级
3. **Step-by-step workflow** - 11 个详细步骤
4. **Quality checklist** - 输出质量检查清单
5. **Formatting standards** - 格式化规范
6. **Industry-specific variations** - 行业特定调整
7. **Common mistakes to avoid** - 常见错误避免

---

## 九、完整插件列表

### 9.1 Core Plugin

| 插件 | 类型 | 功能 | Connectors |
|------|------|------|------------|
| financial-analysis | Core (必装) | 构建 Comps, DCF, LBO, 3-Statement 模型 | 全部 11 个 MCP |

### 9.2 Add-on Plugins

| 插件 | 功能 | Connectors |
|------|------|------------|
| investment-banking | 起草 CIM, Teaser, 并购模型 | — |
| equity-research | 财报更新, 首次覆盖报告 | — |
| private-equity | 尽职调查, IC memo, 投资筛选 | — |
| wealth-management | 客户会议准备, 投资组合再平衡 | — |

### 9.3 Partner-built Plugins

| 插件 | 合作伙伴 | 功能 |
|------|----------|------|
| LREG | LSEG | 债券定价, FX 分析, 宏观仪表盘 |
| SPGlobal | S&P Global | 公司 tearsheet, 财报预览 |

---

## 十、安装与使用

### 10.1 安装命令

```bash
# Add the marketplace
claude plugin marketplace add anthropics/financial-services-plugins

# Install the core plugin first (required)
claude plugin install financial-analysis@financial-services-plugins

# Then add function-specific plugins as needed
claude plugin install investment-banking@financial-services-plugins
claude plugin install equity-research@financial-services-plugins
claude plugin install private-equity@financial-services-plugins
claude plugin install wealth-management@financial-services-plugins
```

### 10.2 可用 Commands

```
/comps [company]        # Comparable company analysis
/dcf [company]          # DCF valuation model
/earnings [company]    # Post-earnings update report
/one-pager [company]   # One-page company profile
/ic-memo [name]         # Investment committee memo
/source [criteria]      # Deal sourcing
/client-review [client] # Client meeting prep
```

---

## 十一、定制化扩展指南

### 11.1 Swap Connectors
编辑 `.mcp.json` 指向特定数据提供商和内部工具。

### 11.2 Add Firm Context
将术语、处理流程、格式标准放入 skill 文件，使 Claude 理解机构的工作方式。

### 11.3 Bring Your Templates
使用 `/ppt-template` 教授 Claude  firm's 品牌 PowerPoint 布局。

### 11.4 Adjust Workflows
修改 skill 指令以匹配团队实际工作方式。

### 11.5 Build New Plugins
按照上述结构创建新插件，贡献到社区。

---

## 十二、启示与收获

### 12.1 Core Plugin 模式
适合需要多场景复用的技能系统。核心功能集中管理，扩展功能通过 Add-on 实现，避免重复配置。

### 12.2 文件驱动架构
纯 Markdown + JSON 定义技能和工作流，降低使用门槛，非开发者也能修改和扩展。

### 12.3 双层触发机制
- **自动触发 (Skill)**: Agent 根据上下文自动加载相关领域知识
- **显式触发 (Command)**: 用户通过 `/command` 显式调用特定工作流
- 两者协作实现"智能辅助 + 精确控制"的平衡

### 12.4 数据源抽象层
MCP 作为统一接口标准，实现工具可替换。数据源配置集中在 Core Plugin，Add-ons 共享使用。

### 12.5 质量标准外显
将机构级专业标准编码到 Skill 文档中，包含详细的工作流步骤、格式规范、质量检查清单，确保输出的一致性和专业性。

### 12.6 优先级策略
明确数据源优先级，确保输出结果的准确性和可审计性。

---

## 十三、各插件 Skills 详解

### 13.1 总览

| 插件 | 类型 | Skills 数量 |
|------|------|-------------|
| financial-analysis | Core | 11 |
| investment-banking | Add-on | 9 |
| private-equity | Add-on | 10 |
| equity-research | Add-on | 9 |
| wealth-management | Add-on | 6 |
| **总计** | | **45** |

### 13.2 financial-analysis (Core - 必装)

| Skill | 功能 | 数据来源 | 输出 |
|-------|------|----------|------|
| **comps-analysis** | 可比公司分析 | MCP: Daloopa, FactSet, S&P | Excel (trading multiples) |
| **dcf-model** | DCF 估值模型 | MCP: Morningstar, S&P | Excel (DCF valuation) |
| **lbo-model** | LBO 杠杆收购模型 | MCP: FactSet, PitchBook | Excel (LBO model) |
| **3-statement-model** | 三表财务模型 | MCP: Daloopa, SEC | Excel (P&L, BS, CF) |
| **competitive-analysis** | 竞争分析 | MCP: FactSet, Morningstar | 分析报告 |
| **deck-refresh** | PPT 更新 | 本地模板 | PowerPoint |
| **ib-check-deck** | IB 投行 deck 检查 | 本地标准 | 质量检查报告 |
| **ppt-template-creator** | PPT 模板创建 | 本地品牌 | PowerPoint 模板 |
| **audit-xls** | Excel 审计 | Excel 文件 | 错误报告 |
| **clean-data-xls** | 数据清洗 | Excel 文件 | 清洗后 Excel |
| **skill-creator** | 创建新 Skill | — | SKILL.md |

### 13.3 investment-banking (Add-on)

| Skill | 功能 | 数据来源 | 输出 |
|-------|------|----------|------|
| **cim-builder** | 构建 CIM | Core Skills + MCP | Word (CIM 文档) |
| **teaser** | 交易概要 | Core Skills | Word (Teaser) |
| **process-letter** | 流程函 | Core Skills | Word (Process Letter) |
| **pitch-deck** | 投行 Pitch Deck | Core Skills | PowerPoint |
| **merger-model** | 并购模型 | Core Skills | Excel (M&A model) |
| **buyer-list** | 买家列表 | MCP: PitchBook, LSEG | Excel (buyer list) |
| **deal-tracker** | 交易追踪 | MCP: Chronograph | 交易状态追踪 |
| **strip-profile** | Strip 分析 | Core Skills | Excel (Strip profile) |
| **datapack-builder** | 数据包构建 | MCP: FactSet, S&P | 数据包 |

### 13.4 private-equity (Add-on)

| Skill | 功能 | 数据来源 | 输出 |
|-------|------|----------|------|
| **deal-sourcing** | deal 来源筛选 | MCP: PitchBook, LSEG | 潜在标的列表 |
| **deal-screening** | 交易筛选 | MCP: PitchBook | 筛选报告 |
| **dd-checklist** | 尽职调查清单 | — | 清单模板 |
| **dd-meeting-prep** | DD 会议准备 | Core Skills | 会议材料 |
| **ic-memo** | IC 投资委员会memo | Core Skills | Word (IC Memo) |
| **returns-analysis** | 回报分析 | MCP: PitchBook | Excel (returns) |
| **unit-economics** | 单位经济分析 | MCP: Daloopa | Excel 分析 |
| **portfolio-monitoring** | 投资组合监控 | MCP: PitchBook, LSEG | 监控仪表盘 |
| **value-creation-plan** | 价值创造计划 | Core Skills | 战略计划 |
| **ai-readiness** | AI 准备度评估 | — | 评估报告 |

### 13.5 equity-research (Add-on)

| Skill | 功能 | 数据来源 | 输出 |
|-------|------|----------|------|
| **earnings-analysis** | 财报分析 | MCP: Aiera, MT Newswires | 研究报告 |
| **earnings-preview** | 财报预览 | MCP: Aiera, FactSet | 预览报告 |
| **initiating-coverage** | 首次覆盖 | Core Skills | 首次覆盖报告 |
| **model-update** | 模型更新 | Core Skills | 更新的 Excel |
| **thesis-tracker** | 投资论点追踪 | — | 追踪仪表盘 |
| **catalyst-calendar** | 催化剂日历 | MCP: Aiera, MT Newswires | 日历事件 |
| **morning-note** | 晨会笔记 | MCP: Newswires | 晨会报告 |
| **idea-generation** | 投资想法生成 | MCP: PitchBook | 想法报告 |
| **sector-overview** | 行业概览 | MCP: Morningstar, S&P | 行业报告 |

### 13.6 wealth-management (Add-on)

| Skill | 功能 | 数据来源 | 输出 |
|-------|------|----------|------|
| **client-review** | 客户会议准备 | Core Skills | 会议准备材料 |
| **financial-plan** | 财务规划 | Core Skills | 财务计划 |
| **portfolio-rebalance** | 组合再平衡 | MCP: Morningstar | 再平衡建议 |
| **client-report** | 客户报告 | Core Skills | 报告文档 |
| **investment-proposal** | 投资提案 | Core Skills | 提案文档 |
| **tax-loss-harvesting** | 税损收割 | MCP: Morningstar | 税损机会 |

### 13.7 Partner-built Plugins

#### LREG (Partner)

| Skill | 功能 | 数据来源 | 输出 |
|-------|------|----------|------|
| **bond-pricing** | 债券定价 | MCP: LSEG | 价格分析 |
| **fx-carry** | FX 套利分析 | MCP: LSEG | FX 分析 |
| **macro-dashboard** | 宏观仪表盘 | MCP: LSEG | 宏观数据 |

#### S&P Global (Partner)

| Skill | 功能 | 数据来源 | 输出 |
|-------|------|----------|------|
| **tearsheet** | 公司概况 | MCP: S&P | tearsheet |
| **earnings-preview** | 财报预览 | MCP: S&P | 预览 |

### 13.8 数据流总结

```
输入 (用户需求)
    ↓
MCP Data Sources (11 个)
    ↓
Core Skills (分析逻辑)
    ↓
Excel / PowerPoint / Word (输出)
```

---

## 十四、Anthropic FSI Core Skill 完整目录结构
