# Financial Plugin 实施指南

> 版本: 1.0  
> 创建日期: 2026-03-23  
> 预计工期: 10-14 天

---

## 一、项目概述

### 1.1 目标

构建 OpenClaw Native Plugin，实现金融分析自动化。

### 1.2 核心组件

| 组件 | 数量 | 说明 |
|------|------|------|
| **Tools** | 11 个 | 数据代理，调用 OpenBB REST API |
| **Skills** | 11 个 | 业务逻辑，参考 Anthropic FSI |
| **配置文件** | 3 个 | package.json, openclaw.plugin.json, tsconfig.json |

### 1.3 技术栈

| 层级 | 技术 |
|------|------|
| **插件框架** | OpenClaw Native Plugin (TypeScript) |
| **数据源** | OpenBB REST API |
| **Skills 来源** | Anthropic Financial Services Plugin |
| **输出生成** | Python (openpyxl, python-pptx) |

---

## 二、目录结构

```
~/.openclaw/extensions/financial-core/
├── package.json                    # npm 配置
├── openclaw.plugin.json            # OpenClaw 插件配置
├── tsconfig.json                   # TypeScript 配置
├── index.ts                        # 入口文件
│
├── src/
│   ├── config.ts                   # 配置常量
│   ├── types.ts                    # 类型定义
│   └── tools/                      # 11 个 Tools
│       ├── get_historical_financials.ts
│       ├── get_market_data.ts
│       ├── get_valuation_multiples.ts
│       ├── get_economic_data.ts
│       ├── get_news.ts
│       ├── get_industry_peers.ts
│       ├── get_ownership_data.ts
│       ├── get_calendar_events.ts
│       ├── get_options_data.ts
│       ├── get_crypto_data.ts
│       └── get_fixed_income_data.ts
│
└── skills/                         # 11 个 Skills
    ├── comps-analysis/
    │   └── SKILL.md
    ├── dcf-model/
    │   ├── SKILL.md
    │   ├── TROUBLESHOOTING.md
    │   ├── requirements.txt
    │   └── scripts/
    │       └── validate_dcf.py
    ├── lbo-model/
    │   └── SKILL.md
    ├── 3-statement-model/
    │   ├── SKILL.md
    │   └── references/
    │       ├── formatting.md
    │       ├── formulas.md
    │       └── sec-filings.md
    ├── competitive-analysis/
    │   ├── SKILL.md
    │   └── references/
    │       ├── frameworks.md
    │       └── schemas.md
    ├── deck-refresh/
    │   └── SKILL.md
    ├── ib-check-deck/
    │   ├── SKILL.md
    │   ├── references/
    │   │   ├── ib-terminology.md
    │   │   └── report-format.md
    │   └── scripts/
    │       └── extract_numbers.py
    ├── ppt-template-creator/
    │   └── SKILL.md
    ├── audit-xls/
    │   └── SKILL.md
    ├── clean-data-xls/
    │   └── SKILL.md
    └── skill-creator/
        ├── SKILL.md
        ├── LICENSE.txt
        ├── references/
        │   ├── output-patterns.md
        │   └── workflows.md
        └── scripts/
            ├── init_skill.py
            ├── package_skill.py
            └── quick_validate.py
```

---

## 三、实施阶段

### Phase 1: 基础环境 (Day 1-2)

#### 3.1.1 安装 OpenBB

```bash
# 安装 OpenBB
pip install openbb[all]

# 启动 REST API Server
openbb-api --port 6900

# 验证
curl http://localhost:6900/api/v1/equity/price/historical?symbol=AAPL
```

#### 3.1.2 创建项目骨架

```bash
# 创建目录
mkdir -p ~/.openclaw/extensions/financial-core
cd ~/.openclaw/extensions/financial-core

# 初始化 npm 项目
npm init -y

# 安装依赖
npm install typescript @types/node --save-dev
```

#### 3.1.3 创建配置文件

**package.json**:
```json
{
  "name": "financial-core",
  "version": "1.0.0",
  "description": "OpenClaw Financial Core Plugin",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "test": "jest"
  },
  "dependencies": {},
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0"
  }
}
```

**tsconfig.json**:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*", "index.ts"]
}
```

**openclaw.plugin.json**:
```json
{
  "id": "financial-core",
  "name": "Financial Core Plugin",
  "version": "1.0.0",
  "description": "Financial analysis tools and skills",
  "tools": ["./dist/tools/*.js"],
  "skills": ["./skills/*/SKILL.md"]
}
```

---

### Phase 2: Tools 实现 (Day 2-3)

#### 3.2.1 配置文件 (src/config.ts)

```typescript
// src/config.ts
export const OPENBB_API = process.env.OPENBB_API_URL || "http://localhost:6900";

export const API_ENDPOINTS = {
  // 股票数据
  equity: {
    fundamental: {
      balance: "/api/v1/equity/fundamental/balance",
      income: "/api/v1/equity/fundamental/income",
      cash: "/api/v1/equity/fundamental/cash",
      metrics: "/api/v1/equity/fundamental/metrics",
      transcript: "/api/v1/equity/fundamental/transcript"
    },
    price: {
      historical: "/api/v1/equity/price/historical",
      quote: "/api/v1/equity/price/quote"
    },
    calendar: "/api/v1/equity/calendar",
    ownership: "/api/v1/equity/ownership",
    compare: "/api/v1/equity/compare/groups"
  },
  // 宏观数据
  economic: {
    indicators: "/api/v1/economic/indicators"
  },
  // 新闻
  news: {
    company: "/api/v1/news/company"
  },
  // 衍生品
  derivatives: {
    options: "/api/v1/derivatives/options"
  },
  // 加密货币
  crypto: {
    historical: "/api/v1/crypto/price/historical"
  },
  // 固定收益
  fixedincome: {
    treasury_rates: "/api/v1/fixedincome/treasury_rates"
  }
} as const;
```

#### 3.2.2 类型定义 (src/types.ts)

```typescript
// src/types.ts

export interface ToolParameter {
  type: string;
  description?: string;
  enum?: string[];
  default?: any;
}

export interface ToolDefinition {
  name: string;
  description: string;
  parameters: {
    type: "object";
    properties: Record<string, ToolParameter>;
    required: string[];
  };
}

export interface OpenBBResponse<T> {
  results: T[];
  provider: string;
  warnings: string[] | null;
  extra: Record<string, any>;
}

// 财务数据类型
export interface BalanceSheet {
  date: string;
  total_assets: number;
  total_liabilities: number;
  total_equity: number;
  current_assets: number;
  current_liabilities: number;
  cash: number;
  inventory: number;
  accounts_receivable: number;
  [key: string]: number | string;
}

export interface IncomeStatement {
  date: string;
  revenue: number;
  gross_profit: number;
  operating_income: number;
  net_income: number;
  eps: number;
  [key: string]: number | string;
}

export interface CashFlowStatement {
  date: string;
  operating_cash_flow: number;
  investing_cash_flow: number;
  financing_cash_flow: number;
  free_cash_flow: number;
  capex: number;
  [key: string]: number | string;
}

export interface MarketData {
  date: string;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  adjusted_close?: number;
}

export interface ValuationMetrics {
  date: string;
  pe_ratio: number;
  pb_ratio: number;
  ps_ratio: number;
  ev_ebitda: number;
  dividend_yield: number;
  [key: string]: number | string;
}
```

#### 3.2.3 Tool 实现模板

```typescript
// src/tools/tool_template.ts
import { OPENBB_API, API_ENDPOINTS } from "../config";
import type { ToolDefinition, OpenBBResponse } from "../types";

/**
 * Tool 工厂函数
 */
function createTool<T>(config: {
  name: string;
  description: string;
  endpoint: string;
  params: Record<string, any>;
  buildUrl: (params: any) => string;
}): ToolDefinition & { execute: (params: any) => Promise<OpenBBResponse<T>> } {
  
  const tool: ToolDefinition & { execute: (params: any) => Promise<OpenBBResponse<T>> } = {
    name: config.name,
    description: config.description,
    parameters: {
      type: "object",
      properties: config.params,
      required: Object.keys(config.params).filter(k => !config.params[k].default)
    },
    
    async execute(params: any): Promise<OpenBBResponse<T>> {
      const url = `${OPENBB_API}${config.buildUrl(params)}`;
      
      const response = await fetch(url, {
        method: "GET",
        headers: {
          "Accept": "application/json"
        }
      });
      
      if (!response.ok) {
        throw new Error(`OpenBB API error: ${response.status} ${response.statusText}`);
      }
      
      return response.json();
    }
  };
  
  return tool;
}

export { createTool };
```

#### 3.2.4 11 个 Tools 完整代码

**1. get_historical_financials.ts**
```typescript
// src/tools/get_historical_financials.ts
import { createTool } from "./tool_template";
import type { BalanceSheet, IncomeStatement, CashFlowStatement } from "../types";

const STATEMENT_TYPES = {
  balance: "balance",
  income: "income",
  cash: "cash"
} as const;

export const getHistoricalFinancials = createTool<BalanceSheet | IncomeStatement | CashFlowStatement>({
  name: "get_historical_financials",
  description: "获取历史财务数据 (资产负债表/利润表/现金流量表)",
  endpoint: "/api/v1/equity/fundamental/{statement}",
  params: {
    ticker: {
      type: "string",
      description: "股票代码 (如: AAPL, MSFT)"
    },
    statement: {
      type: "string",
      enum: ["balance", "income", "cash"],
      description: "报表类型: balance(资产负债表), income(利润表), cash(现金流量表)"
    },
    years: {
      type: "number",
      description: "获取年数",
      default: 3
    },
    provider: {
      type: "string",
      description: "数据提供商",
      default: "fmp"
    }
  },
  buildUrl: (params) => {
    const statement = params.statement || "balance";
    return `/api/v1/equity/fundamental/${statement}?symbol=${params.ticker}&provider=${params.provider || "fmp"}`;
  }
});
```

**2. get_market_data.ts**
```typescript
// src/tools/get_market_data.ts
import { createTool } from "./tool_template";
import type { MarketData } from "../types";

export const getMarketData = createTool<MarketData>({
  name: "get_market_data",
  description: "获取历史股价数据 (开/高/低/收/量)",
  endpoint: "/api/v1/equity/price/historical",
  params: {
    ticker: {
      type: "string",
      description: "股票代码"
    },
    start_date: {
      type: "string",
      description: "开始日期 (YYYY-MM-DD)"
    },
    end_date: {
      type: "string",
      description: "结束日期 (YYYY-MM-DD)"
    },
    provider: {
      type: "string",
      default: "yfinance"
    }
  },
  buildUrl: (params) => {
    let url = `/api/v1/equity/price/historical?symbol=${params.ticker}&provider=${params.provider || "yfinance"}`;
    if (params.start_date) url += `&start_date=${params.start_date}`;
    if (params.end_date) url += `&end_date=${params.end_date}`;
    return url;
  }
});
```

**3. get_valuation_multiples.ts**
```typescript
// src/tools/get_valuation_multiples.ts
import { createTool } from "./tool_template";
import type { ValuationMetrics } from "../types";

export const getValuationMultiples = createTool<ValuationMetrics>({
  name: "get_valuation_multiples",
  description: "获取估值倍数 (P/E, P/B, P/S, EV/EBITDA 等)",
  endpoint: "/api/v1/equity/fundamental/metrics",
  params: {
    ticker: {
      type: "string",
      description: "股票代码"
    },
    provider: {
      type: "string",
      default: "fmp"
    }
  },
  buildUrl: (params) => {
    return `/api/v1/equity/fundamental/metrics?symbol=${params.ticker}&provider=${params.provider || "fmp"}`;
  }
});
```

**4. get_economic_data.ts**
```typescript
// src/tools/get_economic_data.ts
import { createTool } from "./tool_template";

export const getEconomicData = createTool<any>({
  name: "get_economic_data",
  description: "获取宏观经济数据 (GDP, 失业率, 通胀率等)",
  endpoint: "/api/v1/economic/indicators",
  params: {
    symbol: {
      type: "string",
      description: "指标符号 (如: GDP, UNRATE, CPIAUCSL)"
    },
    start_date: {
      type: "string",
      description: "开始日期"
    },
    end_date: {
      type: "string",
      description: "结束日期"
    },
    provider: {
      type: "string",
      default: "fred"
    }
  },
  buildUrl: (params) => {
    let url = `/api/v1/economic/indicators?symbol=${params.symbol}&provider=${params.provider || "fred"}`;
    if (params.start_date) url += `&start_date=${params.start_date}`;
    if (params.end_date) url += `&end_date=${params.end_date}`;
    return url;
  }
});
```

**5. get_news.ts**
```typescript
// src/tools/get_news.ts
import { createTool } from "./tool_template";

interface NewsItem {
  date: string;
  title: string;
  summary: string;
  source: string;
  url: string;
}

export const getNews = createTool<NewsItem>({
  name: "get_news",
  description: "获取公司相关新闻",
  endpoint: "/api/v1/news/company",
  params: {
    ticker: {
      type: "string",
      description: "股票代码"
    },
    limit: {
      type: "number",
      description: "返回新闻数量",
      default: 10
    },
    provider: {
      type: "string",
      default: "finnhub"
    }
  },
  buildUrl: (params) => {
    return `/api/v1/news/company?symbol=${params.ticker}&limit=${params.limit || 10}&provider=${params.provider || "finnhub"}`;
  }
});
```

**6. get_industry_peers.ts**
```typescript
// src/tools/get_industry_peers.ts
import { createTool } from "./tool_template";

interface PeerCompany {
  symbol: string;
  name: string;
  industry: string;
  sector: string;
}

export const getIndustryPeers = createTool<PeerCompany>({
  name: "get_industry_peers",
  description: "获取同行业公司列表",
  endpoint: "/api/v1/equity/compare/groups",
  params: {
    ticker: {
      type: "string",
      description: "股票代码"
    },
    provider: {
      type: "string",
      default: "finviz"
    }
  },
  buildUrl: (params) => {
    return `/api/v1/equity/compare/groups?symbol=${params.ticker}&provider=${params.provider || "finviz"}`;
  }
});
```

**7. get_ownership_data.ts**
```typescript
// src/tools/get_ownership_data.ts
import { createTool } from "./tool_template";

interface OwnershipData {
  holder: string;
  shares: number;
  percentage: number;
  type: string;
}

export const getOwnershipData = createTool<OwnershipData>({
  name: "get_ownership_data",
  description: "获取股东持股数据",
  endpoint: "/api/v1/equity/ownership",
  params: {
    ticker: {
      type: "string",
      description: "股票代码"
    },
    provider: {
      type: "string",
      default: "fmp"
    }
  },
  buildUrl: (params) => {
    return `/api/v1/equity/ownership?symbol=${params.ticker}&provider=${params.provider || "fmp"}`;
  }
});
```

**8. get_calendar_events.ts**
```typescript
// src/tools/get_calendar_events.ts
import { createTool } from "./tool_template";

interface CalendarEvent {
  date: string;
  event_type: string;
  description: string;
}

export const getCalendarEvents = createTool<CalendarEvent>({
  name: "get_calendar_events",
  description: "获取财报日历和重要事件",
  endpoint: "/api/v1/equity/calendar",
  params: {
    ticker: {
      type: "string",
      description: "股票代码"
    },
    start_date: {
      type: "string",
      description: "开始日期"
    },
    end_date: {
      type: "string",
      description: "结束日期"
    }
  },
  buildUrl: (params) => {
    let url = `/api/v1/equity/calendar?symbol=${params.ticker}`;
    if (params.start_date) url += `&start_date=${params.start_date}`;
    if (params.end_date) url += `&end_date=${params.end_date}`;
    return url;
  }
});
```

**9. get_options_data.ts**
```typescript
// src/tools/get_options_data.ts
import { createTool } from "./tool_template";

interface OptionsData {
  strike: number;
  expiry: string;
  call_price: number;
  put_price: number;
  implied_volatility: number;
}

export const getOptionsData = createTool<OptionsData>({
  name: "get_options_data",
  description: "获取期权数据",
  endpoint: "/api/v1/derivatives/options",
  params: {
    ticker: {
      type: "string",
      description: "股票代码"
    },
    expiry: {
      type: "string",
      description: "到期日期"
    },
    provider: {
      type: "string",
      default: "cboe"
    }
  },
  buildUrl: (params) => {
    let url = `/api/v1/derivatives/options?symbol=${params.ticker}&provider=${params.provider || "cboe"}`;
    if (params.expiry) url += `&expiry=${params.expiry}`;
    return url;
  }
});
```

**10. get_crypto_data.ts**
```typescript
// src/tools/get_crypto_data.ts
import { createTool } from "./tool_template";

interface CryptoData {
  date: string;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
}

export const getCryptoData = createTool<CryptoData>({
  name: "get_crypto_data",
  description: "获取加密货币历史价格",
  endpoint: "/api/v1/crypto/price/historical",
  params: {
    symbol: {
      type: "string",
      description: "加密货币符号 (如: BTC, ETH)"
    },
    start_date: {
      type: "string",
      description: "开始日期"
    },
    end_date: {
      type: "string",
      description: "结束日期"
    },
    provider: {
      type: "string",
      default: "binance"
    }
  },
  buildUrl: (params) => {
    let url = `/api/v1/crypto/price/historical?symbol=${params.symbol}&provider=${params.provider || "binance"}`;
    if (params.start_date) url += `&start_date=${params.start_date}`;
    if (params.end_date) url += `&end_date=${params.end_date}`;
    return url;
  }
});
```

**11. get_fixed_income_data.ts**
```typescript
// src/tools/get_fixed_income_data.ts
import { createTool } from "./tool_template";

interface FixedIncomeData {
  date: string;
  rate: number;
  maturity: string;
}

export const getFixedIncomeData = createTool<FixedIncomeData>({
  name: "get_fixed_income_data",
  description: "获取债券和利率数据",
  endpoint: "/api/v1/fixedincome/treasury_rates",
  params: {
    maturity: {
      type: "string",
      description: "期限 (如: 3m, 6m, 1y, 2y, 5y, 10y, 30y)"
    },
    start_date: {
      type: "string",
      description: "开始日期"
    },
    end_date: {
      type: "string",
      description: "结束日期"
    },
    provider: {
      type: "string",
      default: "fred"
    }
  },
  buildUrl: (params) => {
    let url = `/api/v1/fixedincome/treasury_rates?provider=${params.provider || "fred"}`;
    if (params.maturity) url += `&maturity=${params.maturity}`;
    if (params.start_date) url += `&start_date=${params.start_date}`;
    if (params.end_date) url += `&end_date=${params.end_date}`;
    return url;
  }
});
```

#### 3.2.5 入口文件 (index.ts)

```typescript
// index.ts
export { getHistoricalFinancials } from "./src/tools/get_historical_financials";
export { getMarketData } from "./src/tools/get_market_data";
export { getValuationMultiples } from "./src/tools/get_valuation_multiples";
export { getEconomicData } from "./src/tools/get_economic_data";
export { getNews } from "./src/tools/get_news";
export { getIndustryPeers } from "./src/tools/get_industry_peers";
export { getOwnershipData } from "./src/tools/get_ownership_data";
export { getCalendarEvents } from "./src/tools/get_calendar_events";
export { getOptionsData } from "./src/tools/get_options_data";
export { getCryptoData } from "./src/tools/get_crypto_data";
export { getFixedIncomeData } from "./src/tools/get_fixed_income_data";

// 导出所有 Tools
export const tools = [
  getHistoricalFinancials,
  getMarketData,
  getValuationMultiples,
  getEconomicData,
  getNews,
  getIndustryPeers,
  getOwnershipData,
  getCalendarEvents,
  getOptionsData,
  getCryptoData,
  getFixedIncomeData
];
```

---

### Phase 3: Skills 实现 (Day 3-8)

#### 3.3.1 下载 Anthropic FSI 源码

```bash
# 克隆仓库
git clone https://github.com/anthropics/financial-services-plugins.git
cd financial-services-plugins/financial-analysis/skills

# 查看目录结构
ls -la
# comps-analysis  dcf-model  lbo-model  3-statement-model  ...
```

#### 3.3.2 复制 Skills 到项目

```bash
# 创建目标目录
mkdir -p ~/.openclaw/extensions/financial-core/skills

# 复制 11 个 Core Skills
SKILLS=(
  "comps-analysis"
  "dcf-model"
  "lbo-model"
  "3-statement-model"
  "competitive-analysis"
  "deck-refresh"
  "ib-check-deck"
  "ppt-template-creator"
  "audit-xls"
  "clean-data-xls"
  "skill-creator"
)

for skill in "${SKILLS[@]}"; do
  cp -r "$skill" ~/.openclaw/extensions/financial-core/skills/
done
```

#### 3.3.3 修改 SKILL.md 数据源引用

**修改脚本 (Python)**:

```python
# migrate_skills.py
import os
import re

SKILLS_DIR = "/root/.openclaw/extensions/financial-core/skills"

# MCP 数据源到 OpenBB Tool 的映射
MCP_TO_OPENBB = {
    # S&P Kensho MCP
    r"S&P Kensho MCP": "get_historical_financials Tool",
    r"S&P.*MCP": "get_historical_financials Tool",
    
    # FactSet MCP
    r"FactSet MCP": "get_valuation_multiples Tool",
    r"FactSet.*MCP": "get_valuation_multiples Tool",
    
    # Daloopa MCP
    r"Daloopa MCP": "get_historical_financials Tool",
    r"Daloopa.*MCP": "get_historical_financials Tool",
    
    # Morningstar MCP
    r"Morningstar MCP": "get_valuation_multiples Tool",
    
    # Moody's MCP
    r"Moody'?s MCP": "get_fixed_income_data Tool",
    
    # LSEG MCP
    r"LSEG MCP": "get_fixed_income_data Tool",
    
    # MT Newswires MCP
    r"MT Newswires MCP": "get_news Tool",
    
    # Aiera MCP
    r"Aiera MCP": "get_news Tool (财报转录)",
    
    # Chronograph MCP
    r"Chronograph MCP": "get_market_data Tool",
    
    # PitchBook MCP (无替代)
    r"PitchBook MCP": "[注意] PitchBook 数据源暂无替代",
    
    # 通用 MCP 引用
    r"MCP Server": "OpenBB REST API",
    r"MCP data": "OpenBB data",
    r"mcp_client\.call\(": "fetch(OPENBB_API + ",
}

def migrate_skill(skill_path):
    """迁移单个 Skill"""
    skill_md = os.path.join(skill_path, "SKILL.md")
    
    if not os.path.exists(skill_md):
        print(f"跳过: {skill_path} (无 SKILL.md)")
        return
    
    with open(skill_md, 'r', encoding='utf-8') as f:
        content = f.read()
    
    original = content
    
    # 应用映射
    for pattern, replacement in MCP_TO_OPENBB.items():
        content = re.sub(pattern, replacement, content)
    
    # 替换 MCP 调用语法
    content = re.sub(
        r'mcp_client\.call\("([^"]+)"',
        r'fetch(OPENBB_API + "/api/v1/\1"',
        content
    )
    
    if content != original:
        with open(skill_md, 'w', encoding='utf-8') as f:
            f.write(content)
        print(f"已更新: {skill_md}")
    else:
        print(f"无需更新: {skill_md}")

def main():
    for skill_name in os.listdir(SKILLS_DIR):
        skill_path = os.path.join(SKILLS_DIR, skill_name)
        if os.path.isdir(skill_path):
            migrate_skill(skill_path)

if __name__ == "__main__":
    main()
```

#### 3.3.4 验证 Skills 目录结构

```bash
# 检查每个 Skill 的文件
for skill in ~/.openclaw/extensions/financial-core/skills/*/; do
  echo "=== $(basename $skill) ==="
  find "$skill" -type f | head -10
done
```

---

### Phase 4: 测试 (Day 8-10)

#### 3.4.1 Tools 单元测试

```typescript
// tests/tools.test.ts
import { getHistoricalFinancials } from "../src/tools/get_historical_financials";
import { getMarketData } from "../src/tools/get_market_data";

describe("Financial Tools", () => {
  
  beforeAll(() => {
    // 确保 OpenBB API 运行
    process.env.OPENBB_API_URL = "http://localhost:6900";
  });
  
  test("get_historical_financials should return balance sheet", async () => {
    const result = await getHistoricalFinancials.execute({
      ticker: "AAPL",
      statement: "balance"
    });
    
    expect(result).toBeDefined();
    expect(result.results).toBeInstanceOf(Array);
  });
  
  test("get_market_data should return price history", async () => {
    const result = await getMarketData.execute({
      ticker: "AAPL",
      start_date: "2024-01-01",
      end_date: "2024-12-31"
    });
    
    expect(result).toBeDefined();
    expect(result.results).toBeInstanceOf(Array);
  });
  
});
```

#### 3.4.2 Skills 触发测试

```bash
# 启动 OpenClaw Gateway
openclaw gateway start

# 测试 Skills 触发
curl -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "帮我做 AAPL 的可比公司分析"}'

# 预期: 触发 comps-analysis Skill
```

#### 3.4.3 端到端测试

```bash
# 测试完整流程
# 1. 获取财务数据
# 2. 执行 DCF 分析
# 3. 生成 Excel 报告

curl -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "对 MSFT 进行 DCF 估值分析"}'
```

---

### Phase 5: 文档与发布 (Day 10-12)

#### 3.5.1 编写 README

```markdown
# Financial Core Plugin

OpenClaw 金融分析核心插件。

## 安装

\`\`\`bash
openclaw plugin install financial-core
\`\`\`

## 使用

### 可比公司分析
> 帮我做 AAPL 的可比公司分析

### DCF 估值
> 对 MSFT 进行 DCF 估值分析

### 三表建模
> 为 GOOGL 构建三表财务模型

## 前置要求

- OpenBB REST API Server (端口 6900)
- Python 3.10+ (用于报告生成)

\`\`\`bash
pip install openbb[all]
openbb-api --port 6900
\`\`\`

## Skills 列表

| Skill | 说明 |
|-------|------|
| comps-analysis | 可比公司分析 |
| dcf-model | DCF 估值模型 |
| lbo-model | LBO 杠杆收购模型 |
| 3-statement-model | 三表财务模型 |
| competitive-analysis | 竞争分析 |
| deck-refresh | PPT 数据更新 |
| ib-check-deck | 投行 Deck 检查 |
| ppt-template-creator | PPT 模板创建 |
| audit-xls | Excel 审计 |
| clean-data-xls | 数据清洗 |
| skill-creator | 创建新 Skill |
```

#### 3.5.2 发布到 Git

```bash
cd ~/.openclaw/extensions/financial-core

git init
git add .
git commit -m "feat: initial release of financial-core plugin

- 11 Data Tools (OpenBB REST API)
- 11 Core Skills (Anthropic FSI adapted)
- Complete documentation"

git remote add origin https://github.com/YOUR_USERNAME/financial-core.git
git push -u origin main
```

---

## 四、环境要求

### 4.1 系统要求

| 组件 | 版本 | 说明 |
|------|------|------|
| **Node.js** | 18+ | 运行 OpenClaw |
| **Python** | 3.10+ | 运行 OpenBB |
| **TypeScript** | 5.0+ | 编译插件 |
| **OpenBB** | latest | 数据源 |

### 4.2 环境变量

```bash
# .env
OPENBB_API_URL=http://localhost:6900

# 可选: API Keys
FMP_API_KEY=your_key
ALPHAVANTAGE_API_KEY=your_key
POLYGON_API_KEY=your_key
```

### 4.3 Python 依赖

```txt
# requirements.txt
openbb[all]
openpyxl
python-pptx
python-docx
weasyprint
```

---

## 五、检查清单

### Phase 1 完成标准

- [ ] OpenBB REST API 正常运行
- [ ] 项目骨架创建完成
- [ ] 配置文件就绪

### Phase 2 完成标准

- [ ] 11 个 Tools 编译通过
- [ ] 每个 Tool 可独立调用
- [ ] 单元测试通过

### Phase 3 完成标准

- [ ] 11 个 Skills 复制完成
- [ ] 数据源引用已修改
- [ ] references/scripts 目录完整

### Phase 4 完成标准

- [ ] Tools 调用测试通过
- [ ] Skills 自动触发正常
- [ ] 端到端流程验证

### Phase 5 完成标准

- [ ] 文档完善
- [ ] Git 仓库就绪
- [ ] 可发布状态

---

## 六、常见问题

### Q1: OpenBB API 连接失败

```bash
# 检查服务状态
curl http://localhost:6900/health

# 重启服务
openbb-api --port 6900
```

### Q2: Skills 不触发

检查 SKILL.md 的 `description` 字段是否包含正确的关键词。

### Q3: Excel 输出失败

确保 Python 环境已安装 openpyxl:
```bash
pip install openpyxl
```

---

## 七、参考资源

| 资源 | 链接 |
|------|------|
| **Anthropic FSI** | https://github.com/anthropics/financial-services-plugins |
| **OpenBB 文档** | https://docs.openbb.co |
| **OpenClaw 文档** | https://docs.openclaw.ai |
| **设计文档** | DESIGN.md |
| **参考文档** | REFERENCE.md |

---

*文档版本: 1.0*  
*最后更新: 2026-03-23*