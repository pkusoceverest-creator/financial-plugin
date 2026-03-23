# Financial Plugin 项目设计方案

> 基于 OpenClaw Native Plugin 架构  
> 参考: Anthropic Financial Services Plugin  
> 版本: 5.0 (OpenBB API + Anthropic FSI Skills)  
> 最后更新: 2026-03-23

---

## 一、项目概述

### 1.1 项目定位

构建面向金融服务的 OpenClaw Native Plugin，实现投资分析、财务建模、估值等工作的自动化。

**核心目标**:
- 将 AI 打造为金融专业人员的专业助手
- 实现数据获取 → 分析建模 → 报告生成的完整工作流
- 遵循 OpenClaw Best Practice: **Skills 自动触发，无需手动 Command**

### 1.2 核心设计决策

#### 决策 1: 数据层选择 OpenBB REST API

| 对比项 | OpenBB MCP | OpenBB REST API | 决策 |
|--------|-----------|-----------------|------|
| 协议复杂度 | 高 (新协议) | 低 (标准 HTTP) | ✅ API |
| Tool 控制力 | 低 (自动暴露) | 高 (完全自定义) | ✅ API |
| 调试便利性 | 低 | 高 (curl/Postman) | ✅ API |
| OpenClaw 兼容 | 需适配 | 原生支持 | ✅ API |

**启动命令**: `openbb-api --port 6900`

#### 决策 2: SKILL.md 完全参考 Anthropic FSI

**原则**: 复制 Anthropic FSI 的 SKILL.md 内容，仅修改数据源引用。

| 修改项 | Anthropic FSI | financial-plugin |
|--------|--------------|------------------|
| 数据源 | MCP Server | OpenBB REST API |
| 调用方式 | `mcp_client.call(...)` | `fetch(OPENBB_API + ...)` |
| SKILL 内容 | **完全保留** | **完全保留** |

#### 决策 3: 仅保留 Skills 自动触发

**不实现 Commands**，用户通过自然语言交互，AI 自动匹配 Skill。

### 1.3 架构概览

```
用户自然语言请求
        │
        ↓
Skills 层 (业务逻辑，参考 Anthropic FSI)
        │
        ↓
Tools 层 (数据代理，调用 OpenBB REST API)
        │
        ↓
OpenBB REST API Server (openbb-api --port 6900)
        │
        ↓
100+ 数据源 (yfinance, SEC, FRED, FMP...)
```

---

## 二、架构设计

### 2.1 分层架构 (Core + Add-on)

```
┌─────────────────────────────────────────────────────────────┐
│                  Add-on Plugins (可选)                      │
│  investment-banking | equity-research | private-equity ... │
├─────────────────────────────────────────────────────────────┤
│                  financial-core (必装)                      │
│  • 11 Data Tools                                           │
│  • 11 Core Skills                                          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 与 Anthropic FSI 对比

| 组件 | Anthropic FSI | financial-plugin |
|------|--------------|------------------|
| **数据层** | 11 MCP Servers | OpenBB REST API |
| **Tools** | MCP 自动暴露 | 11 个自定义 Tools |
| **Skills** | 11 Core + 34 Add-on | 11 Core (完全复制) |
| **Commands** | 8 个 | 不实现 |

---

## 三、11 Tools 定义

### 3.1 设计原则

**Tool = 纯数据代理** (参考 Anthropic FSI 的 MCP 层设计)

- Tool 只负责调用 OpenBB REST API 获取数据
- 数据处理、计算、格式化全部在 SKILL 层完成
- Tool 返回标准化 JSON，不做业务逻辑

### 3.2 Tools 与 OpenBB API 映射

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

### 3.3 与 Anthropic FSI MCP 对应关系

| Anthropic FSI MCP | financial-plugin Tool | 数据覆盖 |
|-------------------|----------------------|---------|
| Daloopa + FactSet | `get_historical_financials` | ✅ SEC EDGAR |
| FactSet + S&P | `get_valuation_multiples` | ✅ 部分 |
| Morningstar | `get_valuation_multiples` | ✅ 部分 |
| Moody's | `get_fixed_income_data` | ⚠️ FRED 有限 |
| MT Newswires | `get_news` | ⚠️ Finnhub 免费 |
| Aiera | `get_earnings_transcript` | ⚠️ FMP 替代 |
| LSEG | `get_fixed_income_data` | ⚠️ FRED 有限 |
| PitchBook | - | ❌ 无替代 |
| Chronograph | `get_market_data` | ✅ |
| Egnyte | - | ❌ 非数据源 |

### 3.4 Aiera MCP 替代方案

**Aiera 功能**: 财报电话会议转录 (管理层讨论 + Q&A)、音频录音、实时监控

**Anthropic FSI 中使用 Aiera 的 Skills**:
- equity-research Add-on: earnings-analysis, earnings-preview

**替代方案**: OpenBB + FMP

```typescript
// Tool: get_earnings_transcript
api.registerTool({
  name: "get_earnings_transcript",
  description: "获取财报电话会议转录",
  parameters: { ticker: "string", year: "number" },
  async execute(params) {
    return fetch(`${OPENBB_API}/api/v1/equity/fundamental/transcript?symbol=${params.ticker}&year=${params.year}&provider=fmp`);
  }
});
```

**覆盖对比**:

| 功能 | Aiera | OpenBB + FMP |
|------|-------|--------------|
| 文字转录 | ✅ | ✅ |
| 音频 | ✅ | ❌ |
| 实时监控 | ✅ | ❌ |

---

## 四、11 Skills 定义

### 4.1 设计原则

**Skills 内容完全参考 Anthropic FSI**，仅修改数据源调用方式。

**不修改的内容**:
- 所有公式定义
- 所有格式规范
- 所有计算步骤
- 所有输出模板
- 所有质量检查清单
- **references 目录下的所有 .md 文件**
- **scripts 目录下的所有脚本**

### 4.2 11 Core Skills 列表

| 序号 | Skill | 说明 |
|------|-------|------|
| 1 | **comps-analysis** | 可比公司分析 - 运营指标、估值倍数、统计基准 |
| 2 | **dcf-model** | DCF 估值模型 - 现金流预测、WACC、敏感性分析 |
| 3 | **lbo-model** | LBO 杠杆收购模型 - 债务结构、IRR 计算 |
| 4 | **3-statement-model** | 三表财务模型 - BS/IS/CF 联动 |
| 5 | **competitive-analysis** | 竞争分析 - 行业格局、竞争对手对比 |
| 6 | **deck-refresh** | PPT 更新 - 数据刷新、格式保持 |
| 7 | **ib-check-deck** | 投行 Deck 检查 - 质量控制清单 |
| 8 | **ppt-template-creator** | PPT 模板创建 - 自定义模板 |
| 9 | **audit-xls** | Excel 审计 - 公式检查、错误检测 |
| 10 | **clean-data-xls** | 数据清洗 - 格式标准化 |
| 11 | **skill-creator** | 创建新 Skill - Skill 编写指南 |

### 4.3 SKILL.md 修改规范

**修改前 (Anthropic FSI)**:
```markdown
## Step 1: 获取财务数据

使用 MCP 数据源:
- S&P Kensho MCP: 获取历史财务数据
- FactSet MCP: 获取估值倍数
```

**修改后 (financial-plugin)**:
```markdown
## Step 1: 获取财务数据

使用 OpenBB REST API:
- get_historical_financials: 获取历史财务数据
- get_valuation_multiples: 获取估值倍数
```

### 4.4 Skill 目录结构

每个 Skill 可能包含:

| 目录/文件 | 说明 |
|----------|------|
| SKILL.md | 主技能文件 (必需) |
| references/ | 参考材料 (.md 文件) |
| scripts/ | 辅助脚本 (.py 文件) |

**references 目录内容**:

| Skill | 文件 |
|-------|------|
| 3-statement-model | formatting.md, formulas.md, sec-filings.md |
| competitive-analysis | frameworks.md, schemas.md |
| ib-check-deck | ib-terminology.md, report-format.md |
| skill-creator | output-patterns.md, workflows.md |

**scripts 目录内容**:

| Skill | 文件 |
|-------|------|
| dcf-model | validate_dcf.py |
| ib-check-deck | extract_numbers.py |
| skill-creator | init_skill.py, package_skill.py, quick_validate.py |

---

## 五、目录结构

### 5.1 设计文档目录

```
memory/projects/financial-plugin/
├── DESIGN.md           # 本文档
├── REFERENCE.md        # 参考材料 (Anthropic FSI 完整结构、OpenBB 详情)
├── RULES.md            # 开发规范
└── backup_2026-03-23/  # 备份
```

### 5.2 源代码目录

```
~/.openclaw/extensions/financial-core/
├── package.json
├── openclaw.plugin.json
├── index.ts
│
├── src/
│   ├── config.ts
│   ├── types.ts
│   └── tools/
│       ├── get_historical_financials.ts
│       ├── get_market_data.ts
│       └── ... (11 Tools)
│
└── skills/
    ├── comps-analysis/
    │   └── SKILL.md
    ├── dcf-model/
    │   ├── SKILL.md
    │   ├── TROUBLESHOOTING.md
    │   ├── requirements.txt
    │   └── scripts/
    │       └── validate_dcf.py
    ├── 3-statement-model/
    │   ├── SKILL.md
    │   └── references/
    │       ├── formatting.md
    │       ├── formulas.md
    │       └── sec-filings.md
    └── ... (其他 Skills)
```

---

## 六、实施计划

### Phase 1: 基础环境 (1 天)

- [ ] 安装 OpenBB 并验证 REST API
- [ ] 创建 financial-core 插件骨架
- [ ] 实现 11 个 Tools (薄封装)

### Phase 2: Skills 实现 (5 天)

- [ ] 从 Anthropic FSI 复制 11 个 SKILL.md
- [ ] 复制 references 和 scripts 目录
- [ ] 修改数据源引用 (MCP → OpenBB API)
- [ ] 创建示例文件

### Phase 3: 测试与文档 (2 天)

- [ ] Skills 自动触发测试
- [ ] Tools 调用测试
- [ ] 文档完善

**总计**: 8 天

---

## 七、风险与缓解

| 风险 | 缓解措施 |
|------|---------|
| OpenBB API 端点变化 | 版本锁定，定期测试 |
| 数据源 API 限制 | 多 Provider 备份 |
| 部分数据源无替代 | 明确告知用户限制 |

---

## 附录: Anthropic FSI 数据源覆盖

| 数据源 | OpenBB 替代 | 覆盖率 |
|--------|------------|--------|
| Daloopa | SEC + FMP | ✅ 90% |
| Morningstar | 部分 | ⚠️ 60% |
| S&P Global | 部分 | ⚠️ 50% |
| FactSet | FMP | ✅ 80% |
| Moody's | FRED | ⚠️ 40% |
| MT Newswires | Finnhub | ⚠️ 60% |
| Aiera | FMP | ⚠️ 70% |
| LSEG | FRED | ⚠️ 40% |
| PitchBook | 无 | ❌ 0% |
| Chronograph | yfinance | ✅ 90% |

**总体覆盖**: 约 70% 的 Anthropic FSI 数据能力可通过 OpenBB 免费/低成本实现。