# OpenCode 项目学习指南

## 项目概述

**OpenCode** 是一个开源的 AI 编程代理工具，类似于 Claude Code，但具有以下核心特点：

- 100% 开源
- 不绑定特定 AI 提供商（支持 Claude、OpenAI、Google、本地模型等）
- 内置 LSP (Language Server Protocol) 支持
- 专注于 TUI (Terminal User Interface)
- Client/Server 架构

---

## 目录

- [阶段一：项目架构总览](#阶段一项目架构总览)
- [阶段二：核心模块深入](#阶段二核心模块深入)
- [阶段三：工具系统](#阶段三工具系统)
- [阶段四：AI Provider 集成](#阶段四ai-provider-集成)
- [阶段五：TUI 与 UI 层](#阶段五tui-与-ui-层)
- [阶段六：扩展与插件系统](#阶段六扩展与插件系统)
- [阶段七：实战与贡献](#阶段七实战与贡献)

---

## 阶段一：项目架构总览

### 1.1 技术栈

| 技术 | 用途 |
|------|------|
| **Bun** | JavaScript/TypeScript 运行时 (v1.3+) |
| **TypeScript** | 主要开发语言 |
| **SolidJS** | TUI 和 Web UI 框架 |
| **Hono** | HTTP 服务器框架 |
| **Zod** | Schema 验证 |
| **ai-sdk** | AI 模型集成 (Vercel AI SDK) |
| **Turbo** | Monorepo 构建工具 |
| **Tauri** | 桌面应用框架 |

### 1.2 Monorepo 结构

```
opencode/
├── packages/
│   ├── opencode/        # 核心业务逻辑 & 服务器 (最重要!)
│   ├── app/             # 共享 Web UI 组件 (SolidJS)
│   ├── desktop/         # Tauri 桌面应用
│   ├── plugin/          # @opencode-ai/plugin 源码
│   ├── sdk/js/          # JavaScript/TypeScript SDK
│   ├── ui/              # UI 组件库
│   ├── util/            # 工具函数库
│   ├── console/         # 控制台相关
│   ├── web/             # 网站/Landing page
│   ├── docs/            # 文档
│   └── ...
├── infra/               # 基础设施配置 (SST)
├── github/              # GitHub Action
├── script/              # 构建脚本
└── themes/              # 主题配置
```

### 1.3 学习任务

1. **阅读文件**：
   - `file://README.md` - 项目介绍
   - `file://CONTRIBUTING.md` - 贡献指南
   - `file://STYLE_GUIDE.md` - 代码风格指南
   - `file://package.json` - 依赖和工作区配置

2. **运行项目**：
   ```bash
   bun install       # 安装依赖
   bun dev           # 启动开发模式
   bun dev .         # 在 opencode 仓库根目录运行
   ```

3. **理解入口**：
   - 主入口：`file://packages/opencode/src/index.ts`
   - 使用 yargs 构建 CLI

---

## 阶段二：核心模块深入

### 2.1 packages/opencode/src 目录结构

```
src/
├── index.ts          # CLI 入口
├── agent/            # Agent 定义 (build, plan, explore 等)
├── session/          # 会话管理 (核心!)
├── provider/         # AI Provider 集成
├── tool/             # 工具定义 (bash, read, edit 等)
├── server/           # HTTP API 服务器
├── cli/              # CLI 命令实现
│   └── cmd/
│       └── tui/      # TUI 界面 (SolidJS)
├── lsp/              # Language Server Protocol
├── mcp/              # Model Context Protocol
├── config/           # 配置管理
├── permission/       # 权限系统
├── storage/          # 数据持久化
├── bus/              # 事件总线
├── project/          # 项目实例管理
└── util/             # 工具函数
```

### 2.2 核心模块详解

#### Session (会话)
**文件**: `file://packages/opencode/src/session/index.ts`

Session 是 OpenCode 的核心概念，管理用户与 AI 的对话：

```typescript
export namespace Session {
  // Session 信息结构
  export const Info = z.object({
    id: Identifier.schema("session"),
    projectID: z.string(),
    directory: z.string(),
    title: z.string(),
    // ...
  })

  // 核心方法
  export const create = fn(...)      // 创建会话
  export const get = fn(...)         // 获取会话
  export const update = fn(...)      // 更新会话
  export const messages = fn(...)    // 获取消息列表
}
```

**关键文件**:
- `file://packages/opencode/src/session/index.ts` - 会话管理
- `file://packages/opencode/src/session/message-v2.ts` - 消息模型
- `file://packages/opencode/src/session/prompt.ts` - Prompt 构建
- `file://packages/opencode/src/session/llm.ts` - LLM 调用

#### Agent (代理)
**文件**: `file://packages/opencode/src/agent/agent.ts`

OpenCode 内置多种 Agent：

| Agent | 用途 |
|-------|------|
| **build** | 默认代理，完整开发权限 |
| **plan** | 只读代理，用于分析和规划 |
| **general** | 子代理，用于复杂搜索和多步骤任务 |
| **explore** | 快速代码探索 |
| **compaction** | 上下文压缩 |
| **title** | 生成会话标题 |
| **summary** | 生成摘要 |

### 2.3 学习任务

1. **阅读核心文件**：
   - `file://packages/opencode/src/session/index.ts`
   - `file://packages/opencode/src/agent/agent.ts`
   - `file://packages/opencode/src/session/prompt.ts`

2. **理解事件驱动**：
   - 查看 `file://packages/opencode/src/bus/` - 事件总线实现
   - 了解 BusEvent 如何协调各模块

3. **理解存储层**：
   - `file://packages/opencode/src/storage/storage.ts`

---

## 阶段三：工具系统

### 3.1 Tool 架构

**文件**: `file://packages/opencode/src/tool/tool.ts`

Tool 是 AI 代理可以调用的能力单元：

```typescript
export namespace Tool {
  export interface Info<Parameters, Metadata> {
    id: string
    init: (ctx?) => Promise<{
      description: string
      parameters: Parameters    // Zod schema
      execute(args, ctx): Promise<{
        title: string
        metadata: Metadata
        output: string
      }>
    }>
  }

  // 定义工具的便捷方法
  export function define<P, M>(id: string, init: ...): Info<P, M>
}
```

### 3.2 内置工具

| 工具文件 | 功能 |
|----------|------|
| `bash.ts` | 执行 Shell 命令 |
| `read.ts` | 读取文件 |
| `write.ts` | 写入文件 |
| `edit.ts` | 编辑文件 |
| `glob.ts` | 文件模式匹配 |
| `grep.ts` | 内容搜索 |
| `lsp.ts` | LSP 操作 |
| `task.ts` | 子任务代理 |
| `question.ts` | 向用户提问 |
| `webfetch.ts` | 获取网页内容 |
| `websearch.ts` | 网络搜索 |
| `codesearch.ts` | 代码搜索 |
| `skill.ts` | 技能调用 |
| `todo.ts` | TODO 管理 |

### 3.3 工具示例：Read Tool

**文件**: `file://packages/opencode/src/tool/read.ts`

```typescript
export const ReadTool = Tool.define(
  "read",
  async (initCtx) => ({
    description: PROMPT,  // 从 read.txt 加载
    parameters: z.object({
      filePath: z.string(),
      offset: z.number().optional(),
      limit: z.number().optional(),
    }),
    async execute(args, ctx) {
      // 实现文件读取逻辑
      return {
        title: `Read ${args.filePath}`,
        metadata: { ... },
        output: content,
      }
    }
  })
)
```

### 3.4 学习任务

1. **阅读工具定义**：
   - `file://packages/opencode/src/tool/tool.ts` - 工具接口
   - `file://packages/opencode/src/tool/registry.ts` - 工具注册
   - `file://packages/opencode/src/tool/bash.ts` - Bash 工具实现

2. **理解工具 Prompt**：
   - 每个工具有对应的 `.txt` 文件描述其用途
   - 如 `file://packages/opencode/src/tool/bash.txt`

3. **动手实践**：
   - 尝试添加一个简单的自定义工具

---

## 阶段四：AI Provider 集成

### 4.1 Provider 架构

**文件**: `file://packages/opencode/src/provider/provider.ts`

OpenCode 支持多个 AI 提供商：

```typescript
const BUNDLED_PROVIDERS = {
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  // ... 更多
}
```

### 4.2 模型管理

**文件**: `file://packages/opencode/src/provider/models.ts`

```typescript
export namespace Provider {
  export interface Model {
    id: string
    name: string
    provider: string
    cost?: {
      input: number
      output: number
      cache?: { read: number; write: number }
    }
  }

  // 获取模型
  export async function getModel(providerID: string, modelID: string)
  
  // 默认模型
  export async function defaultModel()
}
```

### 4.3 学习任务

1. **阅读 Provider 实现**：
   - `file://packages/opencode/src/provider/provider.ts`
   - `file://packages/opencode/src/provider/transform.ts` - 提供商转换

2. **理解认证**：
   - `file://packages/opencode/src/provider/auth.ts`
   - `file://packages/opencode/src/auth/index.ts`

3. **配置自定义 Provider**：
   - 学习如何通过配置文件添加新的 Provider

---

## 阶段五：TUI 与 UI 层

### 5.1 TUI 架构

**目录**: `file://packages/opencode/src/cli/cmd/tui/`

TUI 使用 SolidJS + OpenTUI 构建：

```
tui/
├── app.tsx           # 主应用组件
├── component/        # UI 组件
├── context/          # SolidJS Context
├── routes/           # 路由
├── ui/               # 基础 UI 组件
├── util/             # 工具函数
├── worker.ts         # Worker 线程
└── thread.ts         # 线程命令
```

### 5.2 Server/Client 架构

```
┌─────────────────┐     HTTP/WebSocket     ┌─────────────────┐
│   TUI Client    │ ◄──────────────────► │   Server        │
│  (SolidJS)      │                        │  (Hono)         │
└─────────────────┘                        └─────────────────┘
        │                                          │
        │                                          ▼
        │                                  ┌─────────────────┐
        │                                  │  Session/Agent  │
        │                                  │  Tool/Provider  │
        │                                  └─────────────────┘
        │
        ▼
┌─────────────────┐
│ @opencode-ai/sdk│  (生成的 TypeScript SDK)
└─────────────────┘
```

### 5.3 Web App

**目录**: `file://packages/app/`

共享的 Web UI 组件，供 TUI 和桌面应用使用。

```bash
# 运行 Web 开发服务器
bun run --cwd packages/app dev
```

### 5.4 学习任务

1. **阅读 TUI 代码**：
   - `file://packages/opencode/src/cli/cmd/tui/app.tsx`
   - `file://packages/opencode/src/cli/cmd/tui/component/`

2. **理解 Server 路由**：
   - `file://packages/opencode/src/server/server.ts`
   - `file://packages/opencode/src/server/routes/`

3. **学习 SDK 生成**：
   - 运行 `./script/generate.ts` 生成 SDK
   - 查看 `file://packages/sdk/js/`

---

## 阶段六：扩展与插件系统

### 6.1 Plugin 系统

**目录**: `file://packages/plugin/`

```typescript
// @opencode-ai/plugin 提供扩展能力
import { Tool } from "@opencode-ai/plugin/tool"

export const MyTool = Tool.define("my-tool", {
  description: "...",
  parameters: z.object({...}),
  async execute(args, ctx) {
    // 实现逻辑
  }
})
```

### 6.2 MCP (Model Context Protocol)

**目录**: `file://packages/opencode/src/mcp/`

支持 MCP 协议扩展 AI 能力：

```typescript
// 配置 MCP 服务器
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["./my-mcp-server.js"]
      }
    }
  }
}
```

### 6.3 Skill 系统

**目录**: `file://packages/opencode/src/skill/`

Skill 是可复用的提示词和工具组合。

### 6.4 学习任务

1. **理解 Plugin 架构**：
   - `file://packages/plugin/src/index.ts`
   - `file://packages/plugin/src/tool.ts`

2. **学习 MCP 集成**：
   - `file://packages/opencode/src/mcp/`

3. **查看配置示例**：
   - 研究 `opencode.json` 配置格式

---

## 阶段七：实战与贡献

### 7.1 开发工作流

```bash
# 1. 安装依赖
bun install

# 2. 启动开发模式
bun dev

# 3. 运行类型检查
bun run typecheck

# 4. 运行测试
bun test

# 5. 构建
./packages/opencode/script/build.ts --single
```

### 7.2 调试技巧

```bash
# 使用 inspector 调试
bun run --inspect=ws://localhost:6499/ dev

# 调试服务器
bun run --inspect=ws://localhost:6499/ ./src/index.ts serve --port 4096

# 附加 TUI
opencode attach http://localhost:4096
```

### 7.3 贡献指南

1. **Issue First**: 先创建 Issue 讨论
2. **PR 规范**: 使用 Conventional Commits
   - `feat:` 新功能
   - `fix:` Bug 修复
   - `docs:` 文档
   - `chore:` 维护任务
   - `refactor:` 重构

3. **代码风格**:
   - 避免 `let`，优先使用 `const`
   - 避免 `else` 语句
   - 避免 `try/catch`，使用 `.catch()`
   - 避免 `any` 类型
   - 单词变量名优先

### 7.4 推荐学习路径

| 阶段 | 目标 | 时间 |
|------|------|------|
| 1 | 理解项目结构，能运行开发环境 | 1-2 天 |
| 2 | 深入 Session/Agent 核心模块 | 2-3 天 |
| 3 | 掌握 Tool 系统，能添加简单工具 | 2-3 天 |
| 4 | 理解 Provider 集成机制 | 1-2 天 |
| 5 | 熟悉 TUI/Server 架构 | 2-3 天 |
| 6 | 学习扩展系统 | 1-2 天 |
| 7 | 实战贡献 | 持续 |

---

## 附录

### A. 关键文件索引

| 文件 | 描述 |
|------|------|
| `packages/opencode/src/index.ts` | CLI 入口 |
| `packages/opencode/src/session/index.ts` | 会话管理 |
| `packages/opencode/src/agent/agent.ts` | Agent 定义 |
| `packages/opencode/src/tool/tool.ts` | Tool 接口 |
| `packages/opencode/src/provider/provider.ts` | AI Provider |
| `packages/opencode/src/server/server.ts` | HTTP Server |
| `packages/opencode/src/cli/cmd/tui/app.tsx` | TUI 主组件 |

### B. 常用命令

```bash
bun dev                    # 启动开发
bun dev .                  # 在当前目录开发
bun run typecheck          # 类型检查
bun test                   # 运行测试
./script/generate.ts       # 生成 SDK
```

### C. 相关资源

- 官方文档: https://opencode.ai/docs
- Discord: https://discord.gg/opencode
- GitHub: https://github.com/anomalyco/opencode

---

**祝你学习愉快！** 🎉
