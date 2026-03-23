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

### 14.1 Skill 目录结构总览

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

### 14.2 references 目录内容

| Skill | 文件 | 内容 |
|-------|------|------|
| **3-statement-model** | formatting.md | 格式规范 |
| | formulas.md | 公式定义 |
| | sec-filings.md | SEC 文件说明 |
| **competitive-analysis** | frameworks.md | 分析框架 |
| | schemas.md | 数据结构 |
| **ib-check-deck** | ib-terminology.md | 投行术语 |
| | report-format.md | 报告格式 |
| **skill-creator** | output-patterns.md | 输出模式 |
| | workflows.md | 工作流 |

### 14.3 scripts 目录内容

| Skill | 文件 | 用途 |
|-------|------|------|
| **dcf-model** | validate_dcf.py | DCF 模型验证 |
| **ib-check-deck** | extract_numbers.py | 数据提取 |
| **skill-creator** | init_skill.py | 初始化 Skill |
| | package_skill.py | 打包 Skill |
| | quick_validate.py | 快速验证 |

---

## 十五、OpenBB REST API 详细架构

### 15.1 启动与配置

```bash
# 安装
pip install openbb[all]

# 启动 REST API Server
openbb-api --port 6900

# 默认配置
# Host: 127.0.0.1
# Port: 6900
```

### 15.2 API 端点结构

```
/api/v1/
├── equity/                    # 股票数据
│   ├── price/
│   │   ├── historical         # 历史价格
│   │   └── quote              # 实时报价
│   ├── fundamental/
│   │   ├── balance            # 资产负债表
│   │   ├── income             # 利润表
│   │   ├── cash               # 现金流量表
│   │   ├── metrics            # 财务指标
│   │   ├── ratios             # 财务比率
│   │   └── transcript         # 财报转录
│   ├── calendar/              # 财报日历
│   ├── ownership/             # 股东数据
│   └── compare/               # 同行对比
│
├── economic/                  # 宏观经济
│   └── indicators/            # 经济指标
│
├── news/                      # 新闻
│   └── company/               # 公司新闻
│
├── fixedincome/               # 固定收益
│   ├── treasury_rates         # 国债利率
│   └── corporate/             # 公司债
│
├── derivatives/               # 衍生品
│   └── options/               # 期权数据
│
└── crypto/                    # 加密货币
    └── price/historical       # 历史价格
```

### 15.3 免费数据源

| Provider | 数据类型 | 说明 |
|----------|---------|------|
| **yfinance** | 股价、基本面 | Yahoo Finance，最常用 |
| **sec** | 财务报表 | SEC EDGAR 官方 |
| **fred** | 宏观经济 | 美联储数据库 |
| **oecd** | 国际经济 | OECD 统计 |
| **cboe** | 期权、波动率 | CBOE 交易所 |

### 15.4 需注册的数据源

| Provider | 数据类型 | 注册地址 |
|----------|---------|---------|
| **alpha_vantage** | 股价、技术指标 | alphavantage.co |
| **polygon** | 股价、期权 | polygon.io |
| **fmp** | 股价、基本面 | financialmodelingprep.com |
| **finnhub** | 新闻、财报 | finnhub.io |

### 15.5 TET 数据管道

OpenBB 内置 Transform-Extract-Transform 管道：

```
请求 → Transform(参数转换) → Extract(数据提取) → Transform(数据标准化) → 返回
```

**自动处理**:
- 字段名标准化 (lower_snake_case)
- 类型强制转换
- NaN/null 统一处理
- 跨 Provider 字段映射

---

## 十六、OpenBB 接口方式对比

### 16.1 五种接口方式

| 方式 | 启动命令 | 协议 | 适用场景 |
|------|---------|------|---------|
| **Python SDK** | `from openbb import obb` | Python 对象 | Jupyter、脚本 |
| **REST API** | `openbb-api` | HTTP | Web 应用、跨语言 |
| **MCP Server** | `openbb-mcp` | MCP | AI Agent (Claude) |
| **CLI** | `openbb-cli` | 终端交互 | 快速查询 |
| **Desktop** | GUI 应用 | 图形界面 | 非开发者 |

### 16.2 详细对比

#### Python SDK

```python
from openbb import obb

# 直接调用，无需启动服务
data = obb.equity.price.historical("AAPL", provider="yfinance")
df = data.to_dataframe()
```

**特点**:
- ✅ 无需额外服务
- ✅ 直接返回 Python 对象
- ✅ 适合 Jupyter/数据分析
- ❌ 仅限 Python 环境

#### REST API Server

```bash
# 启动服务
openbb-api --port 6900

# HTTP 调用
curl 'http://localhost:6900/api/v1/equity/price/historical?symbol=AAPL'
```

**特点**:
- ✅ 标准 HTTP 接口
- ✅ 任何语言可调用
- ✅ 适合 Web 应用、微服务架构
- ✅ **OpenClaw Tool 封装推荐**
- ⚠️ 需要独立进程

#### MCP Server

```bash
# 启动 MCP 服务
openbb-mcp --port 8000

# AI Agent 通过 MCP 协议调用
```

**特点**:
- ✅ MCP 协议
- ✅ AI Agent 原生支持 (Claude/Cursor/Copilot)
- ✅ 自动工具发现
- ⚠️ 协议较新，调试工具少
- ⚠️ Tool 定义不可控

#### CLI

```bash
# 启动交互式终端
openbb

# 进入后:
# ~/equity/price/historical -t AAPL
```

**特点**:
- ✅ 终端交互
- ✅ 类似 Bloomberg 终端体验
- ❌ 非程序化接口

#### Desktop App

**特点**:
- ✅ 图形界面
- ✅ 无需编程
- ❌ 非程序化接口

### 16.3 OpenClaw 选择建议

| 接口方式 | 是否适合 | 原因 |
|---------|---------|------|
| **Python SDK** | ❌ 不适合 | OpenClaw Tool 是 TypeScript |
| **REST API** | ✅ **最适合** | HTTP 跨语言，调试方便，Tool 可控 |
| **MCP Server** | ⚠️ 可选 | 协议复杂，Tool 控制弱 |
| **CLI** | ❌ 不适合 | 非程序化接口 |
| **Desktop** | ❌ 不适合 | 非程序化接口 |

---

## 十七、REST API Server 工作原理

### 17.1 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                   OpenBB REST API Server                     │
│                        (代理层)                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   HTTP 请求                                                 │
│   GET /api/v1/equity/price/historical?symbol=AAPL          │
│                              │                              │
│                              ↓                              │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              FastAPI Application                     │  │
│   │  • 自动生成 OpenAPI 文档                             │  │
│   │  • 参数验证 (Pydantic)                              │  │
│   │  • 路由分发                                          │  │
│   └─────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ↓                              │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              OpenBB Core (Router 层)                 │  │
│   │  /equity → openbb-equity router                     │  │
│   │  /economic → openbb-economy router                  │  │
│   │  /news → openbb-news router                         │  │
│   └─────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ↓                              │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              TET Pipeline (Fetcher 层)               │  │
│   │  Transform Query → Extract Data → Transform Data     │  │
│   └─────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ↓                              │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              Data Providers                          │  │
│   │  Yahoo Finance, Polygon, FRED, SEC EDGAR...         │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 17.2 启动与配置

```bash
# 安装
pip install openbb[all]

# 启动 REST API Server
openbb-api

# 默认配置
# Host: 127.0.0.1
# Port: 6900

# 自定义配置
openbb-api --host 0.0.0.0 --port 8000 --reload

# 排除某些端点
openbb-api --exclude '["/api/v1/crypto/*"]'
```

### 17.3 工作流程

```
Step 1: 接收 HTTP 请求
        │
        ↓
Step 2: FastAPI 解析路由和参数
        │
        ↓
Step 3: 路由到对应的 OpenBB Router
        │
        ↓
Step 4: TET Pipeline 处理
        │
        ├── Transform Query: 参数标准化、映射到 Provider 格式
        │
        ├── Extract Data: 调用 Provider API 获取原始数据
        │
        └── Transform Data: 字段标准化、类型转换
        │
        ↓
Step 5: 返回标准化 JSON 响应
```

### 17.4 请求示例

```bash
# 获取历史股价
curl -X 'GET' \
  'http://localhost:6900/api/v1/equity/price/historical?symbol=AAPL&provider=yfinance' \
  -H 'accept: application/json'

# 获取资产负债表
curl -X 'GET' \
  'http://localhost:6900/api/v1/equity/fundamental/balance?symbol=AAPL&provider=fmp' \
  -H 'accept: application/json'

# 获取宏观经济指标
curl -X 'GET' \
  'http://localhost:6900/api/v1/economic/indicators?symbol=GDP&provider=fred' \
  -H 'accept: application/json'
```

### 17.5 响应格式

```json
{
  "results": [
    {
      "date": "2024-01-02",
      "open": 185.0,
      "high": 186.5,
      "low": 184.2,
      "close": 185.8,
      "volume": 50000000
    }
  ],
  "provider": "yfinance",
  "warnings": null,
  "extra": {
    "metadata": {}
  }
}
```

---

## 十八、TET Pipeline 详解

### 18.1 三阶段说明

| 阶段 | 名称 | 职责 |
|------|------|------|
| **Stage 1** | Transform Query | 参数验证、映射、默认值 |
| **Stage 2** | Extract Data | 调用 Provider API、处理分页 |
| **Stage 3** | Transform Data | 字段标准化、类型转换、null 处理 |

### 18.2 Stage 1: Transform Query

**问题**: 不同 Provider 参数名不同

| 参数 | Polygon | Intrinio | FMP |
|------|---------|----------|-----|
| symbol | `stocksTicker` | `identifier` | `symbol` |
| start_date | `from` | `start_date` | `from` |
| interval | `multiplier + timespan` | `interval_size` | `timeframe` |

**解决**: QueryParams 模型自动映射

```python
# 用户统一调用
obb.equity.price.historical("AAPL", provider="polygon")

# OpenBB 自动转换为 Polygon 参数
# stocksTicker=AAPL, multiplier=1, timespan=day
```

### 18.3 Stage 2: Extract Data

**职责**:
- 调用 Provider API
- 处理分页 (某些 Provider 单次返回有限)
- 合并多次请求结果

### 18.4 Stage 3: Transform Data

**处理内容**:

| 处理项 | 说明 |
|--------|------|
| 字段映射 | Provider 字段名 → 标准字段名 |
| 类型转换 | 确保数字是数字，日期是日期 |
| null 处理 | NaN, "", "None" → null |
| 命名规范 | 所有字段 → lower_snake_case |

---

## 十九、Provider 配置

### 19.1 环境变量配置

```bash
# 免费数据源 (无需配置)
# yfinance, sec, fred, oecd, cboe 等

# 需注册的数据源
export POLYGON_API_KEY="your_key"
export FMP_API_KEY="your_key"
export ALPHAVANTAGE_API_KEY="your_key"
export FINNHUB_API_KEY="your_key"
```

### 19.2 配置文件方式

`~/.openbb_platform/user_settings.json`:

```json
{
  "preferences": {
    "default_provider": "yfinance"
  },
  "credentials": {
    "polygon_api_key": "your_key",
    "fmp_api_key": "your_key"
  }
}
```

### 19.3 OpenBB Hub 配置

```bash
# 登录 OpenBB Hub
pip install openbb-hub
obb.account.login(email="your@email.com")

# API Keys 在网页端配置，自动同步
```

---

## 二十、skills 适配说明

### 20.1 需要修改的文件

| 文件类型 | 是否需要修改 | 原因 |
|---------|-------------|------|
| **SKILL.md** | ✅ **需要** | 包含数据源调用步骤 |
| **references/*.md** | ❌ **不需要** | 格式规范，与数据源无关 |
| **scripts/*.py** | ❌ **不需要** | 处理本地文件，不涉及 API |

### 20.2 SKILL.md 修改示例

**修改前**:
```markdown
## Step 1: 获取财务数据

使用 MCP 数据源:
- S&P Kensho MCP: 获取历史财务数据
- FactSet MCP: 获取估值倍数
```

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
*最后更新: 2026-03-23*