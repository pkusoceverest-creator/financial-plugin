# Financial Plugin 路线图

> 版本: 1.0
> 创建日期: 2026-03-24
> 状态: 规划中

---

## 一、项目愿景

基于 Anthropic FSI 架构，构建金融分析插件生态：
- **Core Plugin**: financial-analysis (基础财务分析)
- **Add-on Plugins**: 量化交易 + 深度研究 (扩展方向)

---

## 二、Add-on Plugin 开发规划

### 2.1 量化 Plugin

**目标**: 提供量化交易策略开发与回测能力

| 维度 | 说明 |
|------|------|
| **可行性** | ✅ 高 |
| **核心依赖** | OpenBB + FinRL/Qlib |
| **开发周期** | 2-3 周 |

**关键 Skill 设计**:

| Skill | 输入 | 输出 | 技术栈 |
|-------|------|------|--------|
| **backtest** | 策略 + 参数 | 回测报告 + 风险指标 | OpenBB + Backtrader |
| **factor-mine** | 股票池 + 时间范围 | 因子列表 + IC 值 | RD-Agent / Qlib |
| **rl-trading** | 环境配置 | 训练模型 + 策略 | FinRL |
| **portfolio-opt** | 股票池 + 约束 | 最优权重 | CVXPY |

**框架选择**:

| 框架 | 定位 | 优势 | 适用场景 |
|------|------|------|---------|
| **FinRL** | 强化学习交易 | 5 种 DRL 算法、快速原型 | 研究/教学、RL 策略验证 |
| **Qlib** | 全栈量化平台 | 20+ SOTA 模型、RD-Agent、生产级 | 机构级量化研究 |

**集成方式**:
- OpenBB 作为数据层 (复用 Core 配置)
- FinRL/Qlib 作为策略引擎
- 输出统一为研究报告 + 可执行策略

---

### 2.2 深度研究 Plugin

**目标**: 提供个股/行业深度研究能力

| 维度 | 说明 |
|------|------|
| **可行性** | ✅ 高 |
| **核心依赖** | Open Deep Research + OpenBB |
| **开发周期** | 1-2 周 |

**关键 Skill 设计**:

| Skill | 触发场景 | 输出 |
|-------|---------|------|
| **deep-research** | 研究某公司 | 研究报告 (Markdown/PDF) |
| **sector-analysis** | 分析某行业 | 行业研究报告 |
| **investment-thesis** | 写投资论点 | 投资论点文档 + 追踪清单 |
| **risk-assessment** | 风险评估 | 风险矩阵 + 缓解建议 |

**工作流设计** (基于 Open Deep Research):

```
┌─────────────────────────────────────┐
│   Data Gathering                    │  → OpenBB 数据收集
├─────────────────────────────────────┤
│   Financial Analysis                │  → 调用 Core Skills
├─────────────────────────────────────┤
│   Insight Synthesis                 │  → 投资洞察综合
├─────────────────────────────────────┤
│   Research Report                   │  → 生成研究报告
└─────────────────────────────────────┘
```

**集成方式**:
- Open Deep Research 工作流框架
- OpenBB 数据源 + Core Skills 分析能力
- 结构化输出研究报告

---

## 三、可行性矩阵

| 维度 | 量化 Plugin | 深度研究 Plugin |
|------|:-----------:|:---------------:|
| **架构支持** | ✅ | ✅ |
| **数据层** | ✅ OpenBB | ✅ OpenBB |
| **框架集成** | ✅ FinRL/Qlib | ✅ LangGraph |
| **Skill 复用** | ✅ Core Skills | ✅ Core Skills |
| **技术门槛** | ⚠️ 中等 | ✅ 低 |

---

## 四、开发路线图

### Phase 1: 深度研究 Plugin (优先)

| 阶段 | 内容 | 时间 |
|------|------|------|
| 1.1 | 创建 plugin.json + 目录结构 | 1 天 |
| 1.2 | 实现 deep-research Skill | 3-5 天 |
| 1.3 | 实现 sector-analysis Skill | 3 天 |
| 1.4 | 实现 investment-thesis Skill | 2 天 |
| 1.5 | 测试 + 文档 | 2 天 |

**里程碑**: 深度研究 Plugin v1.0 发布

---

### Phase 2: 量化 Plugin

| 阶段 | 内容 | 时间 |
|------|------|------|
| 2.1 | 框架选型 (FinRL vs Qlib) | 1 天 |
| 2.2 | 创建 plugin.json + 目录结构 | 1 天 |
| 2.3 | 实现数据层集成 (OpenBB) | 2 天 |
| 2.4 | 实现 backtest Skill | 3-5 天 |
| 2.5 | 实现 factor-mine Skill (可选) | 5 天 |
| 2.6 | 测试 + 文档 | 2 天 |

**里程碑**: 量化 Plugin v1.0 发布

---

## 五、风险与缓解

| 风险 | 影响 | Plugin | 缓解措施 |
|------|:----:|:------:|---------|
| **回测数据质量** | 中 | 量化 | 验证 OpenBB 历史数据，必要时接入付费数据源 |
| **LLM 幻觉** | 高 | 研究 | 数据引用 + Checklist + 人工审核 |
| **模型训练耗时** | 中 | 量化 | 异步执行 + 进度通知 |
| **多 LLM 成本** | 中 | 研究 | 使用本地模型或低成本 API |
| **框架依赖** | 低 | 量化 | 选择成熟框架，预留替换空间 |

---

## 六、与 ECOSYSTEM 框架集成

| 框架 | 集成 Plugin | 集成点 |
|------|------------|--------|
| **FinRL** | 量化 | RL 策略训练后端 |
| **Qlib** | 量化 | 因子挖掘、模型训练、回测引擎 |
| **RD-Agent** | 量化 | 自动化因子发现 (Qlib 集成) |
| **Open Deep Research** | 研究 | 工作流框架、多模型编排 |

---

## 七、目录结构规划

### 量化 Plugin

```
quant-plugin/
├── .claude-plugin/
│   └── plugin.json          # 依赖 financial-analysis
├── commands/
│   ├── backtest.md
│   ├── factor-mine.md
│   └── strategy-opt.md
├── skills/
│   ├── backtest/
│   │   └── SKILL.md
│   ├── factor-analysis/
│   │   └── SKILL.md
│   └── rl-trading/
│       └── SKILL.md
└── scripts/
    ├── backtest_runner.py
    └── factor_lib.py
```

### 深度研究 Plugin

```
research-plugin/
├── .claude-plugin/
│   └── plugin.json          # 依赖 financial-analysis
├── commands/
│   ├── research.md
│   ├── sector.md
│   └── thesis.md
├── skills/
│   ├── deep-research/
│   │   └── SKILL.md
│   ├── sector-analysis/
│   │   └── SKILL.md
│   ├── investment-thesis/
│   │   └── SKILL.md
│   └── risk-assessment/
│       └── SKILL.md
└── references/
    ├── research-frameworks.md
    └── industry-metrics.md
```

---

## 八、结论

| 结论 | 说明 |
|------|------|
| ✅ **可行性高** | 架构、数据、框架均已就绪 |
| ⚠️ **需规划** | 工作流设计、输出格式、质量控制 |
| 📅 **建议顺序** | 先深度研究 (低门槛)，后量化 (高价值) |
| 🔄 **可并行** | 两个 Plugin 可独立开发，共享 Core |

---

## 九、下一步行动

1. [ ] 完成深度研究 Plugin plugin.json 设计
2. [ ] 设计 deep-research Skill 工作流
3. [ ] 验证 Open Deep Research 与 OpenBB 集成
4. [ ] 创建 quant-plugin 原型目录结构
5. [ ] 调研 FinRL vs Qlib 选型

---

*最后更新: 2026-03-24*