# OpenBB 技术实现文档

> 本文档从 REFERENCE.md 拆分
> 版本: 2.0
> 最后更新: 2026-03-23

---

## 目录

- 十五、OpenBB REST API 详细架构
- 十六、OpenBB 接口方式对比
- 十七、REST API Server 工作原理
- 十八、TET Pipeline 详解
- 十九、Provider 配置
- 二十、Skills 适配说明

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

