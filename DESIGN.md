# Financial Analysis Plugin - 项目设计

**项目名称**: OpenClaw Financial Analysis Plugin  
**版本**: 2.0.0  
**状态**: 规划中  
**创建日期**: 2026-03-20  
**更新日期**: 2026-03-20  
**方案**: Claude Bundle (纯数据，无运行时代码)

---

## 1. 项目概述

### 1.1 目标

借鉴 **Anthropic Financial Services Plugins** (GitHub: anthropics/financial-services-plugins, 6,433 stars)，使用 Claude Bundle 格式开发财务分析 Plugin。

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

### 3.3 命令格式

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

## 4. .claude-plugin/plugin.json

```json
{
  "name": "financial-analysis",
  "description": "投资银行级财务分析工具：可比公司分析、DCF估值、LBO模型、并购模型",
  "version": "1.0.0"
}
```

---

## 5. 触发词配置

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

## 6. 开发计划

### Phase 1: 基础框架 (1天)

- [ ] 创建目录结构
- [ ] 创建 .claude-plugin/plugin.json
- [ ] 创建 config.json

### Phase 2: 核心 Skills (8天)

#### Phase 2a: Comps (2天)
- [ ] commands/comps.md
- [ ] skills/comps-analysis/SKILL.md

#### Phase 2b: DCF (2天)
- [ ] commands/dcf.md
- [ ] skills/dcf-model/SKILL.md

#### Phase 2c: LBO (2天)
- [ ] commands/lbo.md
- [ ] skills/lbo-model/SKILL.md

#### Phase 2d: Merger (2天)
- [ ] commands/merger-model.md
- [ ] skills/merger-model/SKILL.md

### Phase 3: 扩展 Skills (3天)

- [ ] 3-statement-model (commands + skill)
- [ ] earnings (commands + skill)
- [ ] one-pager (commands + skill)

### Phase 4: 完善 (2天)

- [ ] cim, teaser, screen
- [ ] hooks.json
- [ ] 文档

**总计**: ~14 天

---

## 7. 迭代扩展 (无限)

扩展方式: 在 `commands/` 和 `skills/` 添加新目录即可。

---

## 8. 部署

### 8.1 本地开发

```bash
openclaw plugins install -l ./financial-plugin
openclaw plugins enable financial-analysis
openclaw gateway restart
```

### 8.2 Claude Bundle 安装

```bash
# 从本地目录安装
openclaw plugins install ./financial-plugin
```

---

## 9. 参考

- Anthropic Financial Services Plugins: https://github.com/anthropics/financial-services-plugins
- OpenClaw Plugin Bundles: https://docs.openclaw.ai/plugins/bundles

---

## 10. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-03-20 | 初始设计 (Native Plugin) |
| 2.0.0 | 2026-03-20 | 修正为 Claude Bundle 格式 |

---

*最后更新: 2026-03-20 22:20 (Asia/Shanghai)*