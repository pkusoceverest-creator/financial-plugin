# Financial Analysis Plugin - 项目设计

**项目名称**: OpenClaw Financial Analysis Plugin  
**版本**: 1.0.0  
**状态**: 规划中  
**创建日期**: 2026-03-20  
**方案**: 选项 B - 核心版

---

## 1. 项目概述

### 1.1 目标

借鉴 Anthropic Financial Services Plugins (GitHub: anthropics/financial-services-plugins, 6,433 stars)，开发一个可在 OpenClaw 运行的财务分析 Plugin。

### 1.2 功能定位 (选项 B - 核心版)

**10 个命令 + 7 个 Skills**

| 命令 | 功能 | 优先级 |
|------|------|--------|
| comps | 可比公司分析 | P0 |
| dcf | DCF 估值 | P0 |
| lbo | LBO 模型 | P0 |
| merger-model | 并购模型 | P1 |
| 3-statement-model | 三表财务模型 | P1 |
| earnings | 财报分析 | P1 |
| one-pager | 一页概况 | P2 |
| cim | 投资备忘录 | P2 |
| teaser | 交易概要 | P2 |
| screen | 股票筛选 | P2 |

---

## 2. 架构设计

### 2.1 目录结构

```
financial-plugin/
├── openclaw.plugin.json           # 插件清单 (必需)
├── package.json                   # npm 包定义
├── SKILL.md                       # 插件主入口
├── config.json                    # 默认配置
│
├── skills/                        # Skills 目录
│   ├── comps/
│   │   ├── SKILL.md              # 可比公司分析
│   │   └── examples/
│   ├── dcf/
│   │   ├── SKILL.md              # DCF 估值
│   │   └── examples/
│   ├── lbo/
│   │   ├── SKILL.md              # LBO 模型
│   │   └── examples/
│   ├── earnings/
│   │   └── SKILL.md              # 财报分析
│   ├── merger/
│   │   └── SKILL.md              # 并购模型
│   ├── three-statement/
│   │   └── SKILL.md              # 三表模型
│   └── one-pager/
│       └── SKILL.md              # 一页概况
│
├── scripts/
│   ├── index.py                   # CLI 入口
│   ├── builders/                  # Excel 生成器
│   │   ├── comps_builder.py
│   │   ├── dcf_builder.py
│   │   ├── lbo_builder.py
│   │   ├── merger_builder.py
│   │   └── three_statement_builder.py
│   └── data/                      # 数据源
│       └── sec_xbrl.py            # SEC XBRL
│
└── tests/
    └── test_builders.py
```

### 2.2 Anthropic 架构映射

| Anthropic 组件 | OpenClaw 组件 |
|----------------|---------------|
| `.claude-plugin/plugin.json` | `openclaw.plugin.json` |
| `commands/*.md` | SKILL.md trigger 关键词 |
| `skills/*/SKILL.md` | `skills/*/SKILL.md` |
| MCP 连接器 | `scripts/data/` 模块 |
| Office JS | Python + openpyxl |

---

## 3. openclaw.plugin.json

```json
{
  "id": "financial-analysis",
  "name": "Financial Analysis",
  "description": "投资银行级财务分析工具：可比公司分析、DCF估值、LBO模型、并购模型",
  "version": "1.0.0",
  "configSchema": {
    "type": "object",
    "properties": {
      "secEmail": {
        "type": "string",
        "description": "SEC EDGAR API 邮箱"
      },
      "defaultCurrency": {
        "type": "string",
        "default": "USD"
      },
      "outputDir": {
        "type": "string",
        "default": "./output"
      }
    }
  },
  "skills": [
    "skills/comps",
    "skills/dcf",
    "skills/lbo",
    "skills/merger",
    "skills/earnings",
    "skills/three-statement",
    "skills/one-pager"
  ]
}
```

---

## 4. 触发词配置

### 4.1 主入口 SKILL.md

```markdown
---
name: financial-analysis
description: |
  投资银行级财务分析工具。
  支持：comps, dcf, lbo, merger-model, 3-statement-model, earnings, one-pager, cim, teaser, screen
  
trigger:
  - "comps"
  - "dcf"
  - "lbo"
  - "merger"
  - "并购"
  - "财报"
  - "估值"
  - "三表模型"
  - "一页概况"
  - "CIM"
  - "teaser"
  - "筛选股票"
  
metadata:
  openclaw:
    emoji: "📊"
    user-invokable: true
---
```

---

## 5. 开发计划

### Phase 1: 基础框架 (2天)

- [ ] 创建 Plugin 目录结构
- [ ] openclaw.plugin.json
- [ ] package.json
- [ ] 主 SKILL.md
- [ ] CLI 入口脚本

### Phase 2: 数据模块 (2天)

- [ ] 复用 us-stock-financials 数据提取
- [ ] sec_xbrl.py 模块
- [ ] 测试数据源

### Phase 3: 核心 Skills (6天)

#### Phase 3a: Comps (2天)
- [ ] comps-analysis SKILL.md
- [ ] comps_builder.py

#### Phase 3b: DCF (2天)
- [ ] dcf-model SKILL.md
- [ ] dcf_builder.py
- [ ] 敏感性分析表

#### Phase 3c: LBO (2天)
- [ ] lbo-model SKILL.md
- [ ] lbo_builder.py

### Phase 4: 扩展 Skills (4天)

- [ ] merger-model SKILL.md + builder
- [ ] 3-statement-model SKILL.md + builder
- [ ] earnings SKILL.md + 报告生成
- [ ] one-pager SKILL.md + 模板

### Phase 5: 文档与工具 (2天)

- [ ] cim SKILL.md
- [ ] teaser SKILL.md
- [ ] screen SKILL.md
- [ ] 文档完善

**总计**: ~14 天

---

## 6. 部署

### 6.1 本地开发

```bash
openclaw plugins install -l ./financial-plugin
openclaw plugins enable financial-analysis
openclaw gateway restart
```

### 6.2 发布

```bash
npm publish
openclaw plugins install @openclaw/financial-analysis
```

---

## 7. 迭代扩展 (无限)

**扩展能力无上限**，可根据需求持续添加新命令和 Skills。

### 扩展方式

```markdown
# 在 SKILL.md 添加新触发词
trigger:
  - "comps"        # 已有
  - "dcf"          # 已有
  - "新命令"       # 随时添加
```

### 扩展步骤

1. 在 `skills/` 创建新目录 + SKILL.md
2. 在 `scripts/builders/` 添加 builder
3. 在主 SKILL.md trigger 添加关键词

### 示例扩展 (未来可选)

| 新增命令 | 功能 |
|----------|------|
| buyer-list | 买方清单 |
| deal-tracker | 交易追踪 |
| process-letter | 流程函 |
| morning-note | 早间报告 |
| dd-checklist | 尽职调查清单 |
| ic-memo | IC 备忘录 |
| client-review | 客户回顾 |
| rebalance | 组合调仓 |
| ... | ... (无限)

---

## 8. 参考

- Anthropic Financial Services Plugins: https://github.com/anthropics/financial-services-plugins
- OpenClaw Plugin: https://docs.openclaw.ai/tools/plugin
- OpenClaw Manifest: https://docs.openclaw.ai/plugins/manifest

---

## 9. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-03-20 | 初始设计 - 选项B核心版 (10命令+7Skills) |

---

*最后更新: 2026-03-20 20:55 (Asia/Shanghai)*