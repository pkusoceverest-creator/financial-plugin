# OpenClaw Native Plugin - 设计约束文档

> 来源: https://docs.openclaw.ai/tools/plugin
> 分析日期: 2026-03-21

---

## 一、插件能力类型

| 能力 | 注册方法 | 功能说明 |
|------|----------|----------|
| **Model Provider** | `registerProvider` | LLM 模型供应商 |
| **Channel** | `registerChannel` | 聊天通道 (Slack, Telegram, Matrix 等) |
| **Agent Tool** | `registerTool` | Agent 可调用的工具函数 |
| **Speech** | `registerSpeechProvider` | TTS/STT 语音合成与识别 |
| **Media Understanding** | `registerMediaUnderstandingProvider` | 图像/音频/视频分析 |
| **Image Generation** | `registerImageGenerationProvider` | 图像生成 |
| **Web Search** | `registerWebSearchProvider` | Web 搜索 |
| **Hook** | `registerHook` / `on(...)` | 生命周期钩子 |

---

## 二、文件结构规范

### 2.1 必需文件

```
my-plugin/
├── package.json           # npm 元数据 + openclaw 配置块
├── openclaw.plugin.json   # 插件清单 (必需)
└── index.ts               # 入口点
```

### 2.2 可选文件

```
my-plugin/
├── setup-entry.ts         # 设置向导
├── api.ts                 # 公共导出
├── runtime-api.ts         # 内部导出
└── src/
    ├── provider.ts       # 能力实现
    ├── runtime.ts        # 运行时连接
    └── *.test.ts         # 测试
```

---

## 三、package.json 结构

### 3.1 Channel 插件示例

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Short description of the channel."
    }
  }
}
```

### 3.2 Provider 插件示例

```json
{
  "name": "@myorg/openclaw-my-provider",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "providers": ["my-provider"]
  }
}
```

---

## 四、openclaw.plugin.json 结构

### 4.1 必需字段

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

**必需 keys**:
- `id` (string): 规范化的插件 ID
- `configSchema` (object): JSON Schema (即使为空也要提供)

### 4.2 可选字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `kind` | string | 插件类型 (如 "memory", "context-engine") |
| `channels` | array | 注册的 channel IDs |
| `providers` | array | 注册的 provider IDs |
| `providerAuthEnvVars` | object | 认证环境变量映射 |
| `providerAuthChoices` | array | 认证选项元数据 |
| `skills` | array | 技能目录 (相对于插件根) |
| `name` | string | 显示名称 |
| `description` | string | 简短描述 |
| `uiHints` | object | UI 渲染提示 |
| `version` | string | 插件版本 |

### 4.3 完整示例

```json
{
  "id": "my-plugin",
  "kind": "provider",
  "name": "My Plugin",
  "description": "Adds My Provider to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": { "type": "string" }
    }
  },
  "channels": ["my-channel"],
  "providers": ["my-provider"],
  "skills": ["./skills"]
}
```

---

## 五、入口点定义

### 5.1 Channel 插件

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Connects OpenClaw to My Channel",
  plugin: {
    // Channel adapter implementation
  },
});
```

### 5.2 Provider 插件

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/core";

export default definePluginEntry({
  id: "my-provider",
  name: "My Provider",
  register(api) {
    api.registerProvider({
      // Provider implementation
    });
  },
});
```

### 5.3 多能力插件

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/core";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerProvider({ /* ... */ });
    api.registerTool({ /* ... */ });
    api.registerImageGenerationProvider({ /* ... */ });
  },
});
```

---

## 六、SDK 导入约束 (重要)

### 6.1 必须使用聚焦子路径

```typescript
// ✅ 正确
import { definePluginEntry } from "openclaw/plugin-sdk/core";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import { buildOauthProviderAuthResult } from "openclaw/plugin-sdk/provider-oauth";

// ❌ 错误: 单体 root 导入
import { ... } from "openclaw/plugin-sdk";
```

### 6.2 常用子路径参考

| 子路径 | 用途 |
|--------|------|
| `plugin-sdk/core` | 插件入口定义和基础类型 |
| `plugin-sdk/channel-setup` | 设置向导适配器 |
| `plugin-sdk/channel-pairing` | DM 配对原语 |
| `plugin-sdk/channel-reply-pipeline` | 回复前缀 + 输入接线 |
| `plugin-sdk/channel-config-schema` | 配置模式构建器 |
| `plugin-sdk/channel-policy` | 群组/DM 策略助手 |
| `plugin-sdk/secret-input` | 密钥输入解析 |
| `plugin-sdk/webhook-ingress` | Webhook 请求助手 |
| `plugin-sdk/runtime-store` | 持久化插件存储 |
| `plugin-sdk/allow-from` | 允许列表解析 |
| `plugin-sdk/reply-payload` | 消息回复类型 |
| `plugin-sdk/provider-oauth` | OAuth 登录 + PKCE 助手 |
| `plugin-sdk/provider-onboard` | Provider 上线配置补丁 |
| `plugin-sdk/testing` | 测试工具 |

### 6.3 内部导入约束

```typescript
// ✅ 正确: 本地模块
// api.ts
export { MyConfig } from "./src/config.js";

// ❌ 错误: 不能自我导入
import { ... } from "@myorg/openclaw-my-plugin";
```

---

## 七、Agent Tool 注册

### 7.1 必需工具

```typescript
import { Type } from "@sinclair/typebox";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerTool({
      name: "my_tool",
      description: "Do a thing",
      parameters: Type.Object({ input: Type.String() }),
      async execute(_id, params) {
        return { content: [{ type: "text", text: params.input }] };
      },
    });
  },
});
```

### 7.2 可选工具

```typescript
api.registerTool(
  {
    name: "workflow_tool",
    description: "Run a workflow",
    parameters: Type.Object({ pipeline: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.pipeline }] };
    },
  },
  { optional: true },  // 用户需在 allowlist 中启用
);
```

启用配置:
```json5
{
  tools: { allow: ["workflow_tool"] }
}
```

---

## 八、部署要求

### 8.1 发现顺序 (first match wins)

1. **Config paths**: `plugins.load.paths`
2. **Workspace extensions**: `<workspace>/.openclaw/extensions/*.ts`
3. **Global extensions**: `~/.openclaw/extensions/*.ts`
4. **Bundled plugins**: 内置插件

### 8.2 安装方式

```bash
# 从 npm 安装
openclaw plugins install @openclaw/voice-call

# 从本地目录安装
openclaw plugins install ./my-plugin

# 链接开发 (不复制)
openclaw plugins install -l ./my-plugin

# 搜索社区插件
openclaw plugins search <query>
```

### 8.3 配置变更

**配置修改需要重启 Gateway**:

```bash
openclaw gateway restart
```

---

## 九、配置结构

### 9.1 完整配置示例

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: {
      paths: ["~/Projects/oss/voice-call-extension"]
    },
    entries: {
      "voice-call": {
        enabled: true,
        config: { provider: "twilio" }
      },
    },
    slots: {
      memory: "memory-lancedb",
      contextEngine: "legacy"
    }
  }
}
```

### 9.2 字段说明

| 字段 | 说明 |
|------|------|
| `enabled` | 主开关 (默认: true) |
| `allow` | 允许列表 (可选) |
| `deny` | 拒绝列表 (优先于 allow) |
| `load.paths` | 额外插件路径 |
| `slots` | 独占类别选择器 |
| `entries.<id>` | 每个插件的配置 |

---

## 十、独占 Slot

| Slot | 控制内容 | 默认值 |
|------|----------|--------|
| `memory` | 活跃的 memory 插件 | `memory-core` |
| `contextEngine` | 活跃的 context engine | `legacy` (内置) |

---

## 十一、设计约束总结

### 11.1 强制约束

| 约束 | 说明 |
|------|------|
| **必须提供 configSchema** | 即使为空对象 `{}` |
| **必须使用聚焦子路径导入** | 禁止 `openclaw/plugin-sdk` root |
| **禁止自我导入** | 不能通过 SDK 路径导入自己 |
| **Tool 名称唯一** | 不能与核心工具冲突 |
| **配置变更需重启** | 修改配置后必须 `gateway restart` |

### 11.2 推荐约束

| 约束 | 说明 |
|------|------|
| **Workspace 插件默认禁用** | 需显式启用 |
| **可选工具使用 optional: true** | 触发副作用或需要额外二进制时 |
| **使用 Typebox 定义参数** | 类型安全的工具参数 |

---

## 十二、与 Anthropic FSI Plugin 的架构对比

| 维度 | OpenClaw Native Plugin | Anthropic FSI Plugin (Bundle) |
|------|------------------------|------------------------------|
| **格式** | TypeScript + JSON Manifest | 纯 Markdown |
| **能力类型** | Provider/Channel/Tool/Speech 等 | Skills + Commands + MCP |
| **触发机制** | 自动注册 + 显式调用 | 自动 Skill + 手动 Command |
| **数据源** | 自定义 MCP 配置 | 11 个 MCP 集成 |
| **部署方式** | npm / 本地路径 | Claude Plugin CLI |
| **验证方式** | JSON Schema | Markdown 结构 |

**关键差异**:
- **OpenClaw** 是运行时扩展框架，支持多维度能力
- **Anthropic FSI** 是垂直领域 Skill 包，专注于金融服务工作流
- 如果要构建**跨平台 AI Agent 系统**，选择 OpenClaw Native
- 如果要构建**特定领域技能包**，参考 Anthropic FSI 的 Bundle 模式

---

## 十三、CLI 参考

```bash
# 查看和管理
openclaw plugins list                    # 列出已加载插件
openclaw plugins inspect <id>            # 详细信息
openclaw plugins inspect <id> --json      # 机器可读格式
openclaw plugins status                   # 运行状态
openclaw plugins doctor                   # 诊断

# 安装和更新
openclaw plugins install <npm-spec>      # 从 npm 安装
openclaw plugins install <path>           # 从本地路径安装
openclaw plugins install -l <path>       # 链接 (开发模式)
openclaw plugins update <id>              # 更新单个插件
openclaw plugins update --all             # 更新全部

# 启用/禁用
openclaw plugins enable <id>
openclaw plugins disable <id>
```

---

## 十四、Pre-submission 检查清单

<Check>**package.json** 包含正确的 `openclaw` 元数据</Check>
<Check>入口点使用 `defineChannelPluginEntry` 或 `definePluginEntry`</Check>
<Check>所有导入使用聚焦的 `plugin-sdk/<subpath>` 路径</Check>
<Check>内部导入使用本地模块，不是 SDK 自我导入</Check>
<Check>`openclaw.plugin.json` 清单存在且有效</Check>
<Check>测试通过</Check>
<Check>`pnpm check` 通过 (仓库内插件)</Check>

---

*文档版本: 1.0*
*最后更新: 2026-03-21*
*类型: 设计约束文档*