# OpenClaw vs OpenCode (anomalyco) 深度架构对比

**分析日期**：2026年2月2日  
**项目来源**：
- OpenClaw: https://github.com/openclaw/openclaw
- OpenCode: https://github.com/anomalyco/opencode

---

## 一、项目概述

### OpenCode (anomalyco) 是什么

**定位**：开源 AI 编程助手（AI Coding Agent）  
**口号**："The open source AI coding agent"  
**核心功能**：
- 终端交互式编程助手（TUI）
- 桌面应用（Electron/Tauri）
- Web 界面
- 支持多 LLM Provider
- 代码编辑、重构、调试

**与 OpenClaw 的本质区别**：
- **OpenClaw**：服务端常驻进程，多平台通讯集成（微信/钉钉等）
- **OpenCode**：客户端工具，专注代码开发场景

---

## 二、技术栈对比

| 维度 | **OpenClaw** | **OpenCode (anomalyco)** |
|------|--------------|--------------------------|
| **语言** | TypeScript | TypeScript |
| **运行时** | Node.js 22+ | Bun 1.3.5 |
| **包管理** | npm/pnpm | Bun（workspaces） |
| **架构** | 多进程服务 | Monorepo（多包架构） |
| **UI 框架** | 无（通过通讯软件交互） | SolidJS + OpenTUI |
| **AI SDK** | 直接调用 API | Vercel AI SDK |
| **存储** | SQLite + JSON | SQLite（Bun原生支持） |
| **构建工具** | 无（直接运行） | Turbo + SST |

### OpenCode 技术亮点

**Bun 运行时**：
- 比 Node.js 更快的启动速度
- 内置 TypeScript 支持
- 原生 SQLite 支持

**Vercel AI SDK**：
- 统一的 AI 调用接口
- 自动流式输出处理
- 内置 Tool Calling 支持

**Monorepo 结构**：
```
packages/
├── opencode/      # 核心 CLI（TUI应用）
├── app/           # 应用逻辑
├── console/       # 控制台界面
├── desktop/       # 桌面应用
├── web/           # Web 界面
├── plugin/        # 插件系统
├── ui/            # UI 组件库
├── sdk/           # SDK
└── ...
```

---

## 三、架构深度对比

### 3.1 整体架构

**OpenClaw 架构**：
```
┌─────────────────────────────────────────────┐
│                  用户层                      │
│  微信/钉钉/WhatsApp/Discord/邮件              │
└─────────────────────┬───────────────────────┘
                      │ Webhook/WebSocket
┌─────────────────────▼───────────────────────┐
│                Gateway层                   │
│         （独立进程，端口18789）              │
└─────────────────────┬───────────────────────┘
                      │ IPC
┌─────────────────────▼───────────────────────┐
│                Agent层                     │
│  Session │ Memory │ AI Provider │ Tools    │
└─────────────────────────────────────────────┘
```

**OpenCode 架构**：
```
┌─────────────────────────────────────────────┐
│                用户层                        │
│  终端(TUI) / 桌面应用 / Web 界面             │
└─────────────────────┬───────────────────────┘
                      │
┌─────────────────────▼───────────────────────┐
│              核心层（Monorepo）              │
│  ┌─────────┐ ┌─────────┐ ┌─────────────┐  │
│  │opencode │ │   app   │ │   plugin    │  │
│  │ (CLI)   │ │ (逻辑)  │ │  (扩展)     │  │
│  └────┬────┘ └────┬────┘ └──────┬──────┘  │
│       │           │             │         │
│       └───────────┴──────┬──────┘         │
│                          │                │
│  ┌───────────────────────▼────────────┐  │
│  │         Vercel AI SDK              │  │
│  │  Anthropic/OpenAI/Google/Azure...  │  │
│  └────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 3.2 核心差异

| 特性 | OpenClaw | OpenCode |
|------|----------|----------|
| **进程模型** | 多进程（Gateway+Agent分离） | 单进程（CLI工具） |
| **运行模式** | 服务端常驻（daemon） | 客户端即用即走 |
| **交互方式** | 异步消息（聊天软件） | 同步交互（TUI） |
| **部署方式** | 服务器部署 | 本地安装（npm/brew） |
| **持久连接** | 是（WebSocket长连接） | 否（运行即连，退出即断） |

### 3.3 AI 集成对比

**OpenClaw**：
```typescript
// 直接调用 Provider API
const providers = {
  kimi: new KimiProvider({ apiKey }),
  claude: new ClaudeProvider({ apiKey }),
  gpt: new GPTProvider({ apiKey })
};

// 手动处理流式输出
const stream = await provider.sendMessages(messages);
for await (const chunk of stream) {
  yield chunk.content;
}
```

**OpenCode**（使用 Vercel AI SDK）：
```typescript
// 使用 Vercel AI SDK
import { generateText, streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

// 自动处理流式输出
const { textStream } = await streamText({
  model: anthropic('claude-3-5-sonnet-20241022'),
  messages,
  tools: { /* tools */ }
});

for await (const chunk of textStream) {
  console.log(chunk);
}
```

**Vercel AI SDK 优势**：
- 统一的 Provider 接口
- 自动流式处理
- 内置 Tool Calling 解析
- 类型安全

### 3.4 UI 架构对比

**OpenClaw**：
- 无自有 UI
- 通过通讯软件（微信/钉钉）交互
- 消息驱动（异步）

**OpenCode**：
- **TUI**（Terminal UI）：使用 OpenTUI + SolidJS
- **桌面应用**：基于 Electron/Tauri
- **Web 界面**：SolidJS + TailwindCSS

**OpenCode TUI 架构**：
```
┌─────────────────────────────────────────────┐
│                OpenTUI Layer               │
│         （基于 SolidJS 的响应式 UI）          │
├─────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ 聊天面板  │  │ 代码编辑  │  │ 文件树   │  │
│  │          │  │          │  │          │  │
│  └──────────┘  └──────────┘  └──────────┘  │
├─────────────────────────────────────────────┤
│              输入框 + 命令行                 │
└─────────────────────────────────────────────┘
```

### 3.5 工具系统对比

**OpenClaw Tools**：
- 文件操作（read_file, write_file）
- 浏览器自动化（browser）
- 数据库查询（query_database）
- 邮件发送（send_email）
- **通讯集成**（send_whatsapp, send_dingtalk）

**OpenCode Tools**：
- 文件操作（read_file, edit_file）
- 代码搜索（search_files）
- 命令执行（execute_command）
- Git 操作（git_diff, git_commit）
- **代码编辑**（ specialized code editing tools）

**差异总结**：
- OpenClaw：面向**办公自动化**和**通讯**
- OpenCode：面向**代码开发**和**工程化**

---

## 四、代码结构分析

### 4.1 OpenCode Monorepo 结构

```
opencode/
├── packages/
│   ├── opencode/          # ⭐ 核心 CLI 包
│   │   ├── src/
│   │   │   ├── index.ts   # 入口
│   │   │   ├── agent/     # Agent 逻辑
│   │   │   ├── tools/     # 工具实现
│   │   │   └── ui/        # TUI 组件
│   │   └── package.json
│   │
│   ├── app/               # 应用逻辑
│   ├── console/           # 控制台 UI
│   ├── desktop/           # 桌面应用
│   ├── web/               # Web 界面
│   ├── plugin/            # 插件系统
│   ├── ui/                # 共享 UI 组件
│   ├── sdk/               # SDK
│   └── util/              # 工具函数
│
├── infra/                 # SST 基础设施配置
├── sdks/                  # VSCode 插件等
└── package.json           # 根配置（workspaces）
```

### 4.2 核心包：opencode

**依赖分析**：
- `@ai-sdk/*`：Vercel AI SDK 多 Provider 支持
- `@opentui/*`：终端 UI 框架
- `solid-js`：响应式 UI
- `tree-sitter`：代码解析
- `vscode-jsonrpc`：LSP 支持

**架构模式**：
- **Agent Pattern**：Build Agent + Plan Agent（双模式）
- **Tool Use**：AI 自动决定调用工具
- **Streaming**：流式响应，实时显示

### 4.3 与 OpenClaw 目录结构对比

| OpenClaw | OpenCode | 说明 |
|----------|----------|------|
| `src/gateway/` | 无 | OpenClaw 有独立 Gateway |
| `src/agents/` | `packages/opencode/src/agent/` | Agent 逻辑 |
| `src/channels/` | 无 | OpenClaw 多平台接入 |
| `src/tools/` | `packages/opencode/src/tools/` | 工具实现 |
| `src/skills/` | `packages/plugin/` | 插件/技能系统 |
| `src/memory/` | 内置 | 记忆系统 |
| 无 | `packages/ui/` | OpenCode 有 UI 组件库 |
| 无 | `packages/web/` | OpenCode 有 Web 界面 |
| 无 | `packages/desktop/` | OpenCode 有桌面应用 |

---

## 五、关键实现差异

### 5.1 会话管理

**OpenClaw**：
```typescript
// 多用户会话管理
class SessionManager {
  sessions: Map<string, Session>;
  
  async getSession(userId: string): Promise<Session> {
    // 从 SQLite 加载
    // 维护多用户状态
  }
}
```

**OpenCode**：
```typescript
// 单用户当前会话
class Session {
  messages: Message[];
  
  addMessage(msg: Message) {
    // 仅维护当前对话
    // 退出后可选保存
  }
}
```

**差异**：
- OpenClaw：**服务端多用户**会话管理
- OpenCode：**客户端单用户**当前会话

### 5.2 存储方案

**OpenClaw**：
- SQLite：会话、记忆、审计日志
- JSON：配置
- 向量库：语义记忆

**OpenCode**：
- SQLite：对话历史（Bun 原生支持）
- 文件系统：配置、代码缓存

### 5.3 部署方式

**OpenClaw**：
```bash
# 服务器部署
git clone https://github.com/openclaw/openclaw.git
npm install
npm start  # 常驻进程
```

**OpenCode**：
```bash
# 客户端安装
curl -fsSL https://opencode.ai/install | bash
# 或
npm i -g opencode-ai

# 运行
opencode  # 即用即走
```

---

## 六、适用场景总结

### 选择 OpenClaw，如果：

- ✅ 需要**服务端常驻**，24小时在线
- ✅ 需要**多平台通讯**集成（微信/钉钉/邮件）
- ✅ 需要**团队协作**，多人使用
- ✅ 需要**定时任务**和自动化
- ✅ 需要**数据留存在服务器**

**典型场景**：
- 企业智能客服
- 数据监控报警
- 自动化办公流程
- 团队知识库问答

### 选择 OpenCode，如果：

- ✅ **个人开发者**使用
- ✅ 专注**代码编写**和调试
- ✅ 需要**IDE 级别的交互**
- ✅ 追求**轻量快速启动**
- ✅ 需要**桌面/Web 界面**

**典型场景**：
- 日常编码开发
- 代码重构和审查
- 学习新技术
- 快速原型开发

### 两者都用：

- **OpenClaw** 部署在服务器：处理团队自动化、监控、客服
- **OpenCode** 安装在开发机：辅助日常编码工作

---

## 七、架构决策建议

| 需求 | 推荐选择 | 理由 |
|------|---------|------|
| 企业级部署 | OpenClaw | 多进程、多用户、服务端架构 |
| 个人开发 | OpenCode | 轻量、IDE体验、即用即走 |
| 代码助手 | OpenCode | TUI界面、代码工具丰富 |
| 自动化平台 | OpenClaw | 定时任务、多平台接入 |
| 数据隐私 | 两者皆可 | 都支持本地运行 |
| 扩展定制 | OpenClaw | Skills系统更灵活 |

---

## 八、总结

| 维度 | OpenClaw | OpenCode (anomalyco) |
|------|----------|----------------------|
| **定位** | 企业级 AI 助手平台 | 个人开发者 AI 工具 |
| **架构** | 多进程服务 | Monorepo 客户端 |
| **交互** | 消息异步 | TUI同步 |
| **部署** | 服务器 | 本地安装 |
| **运行时** | Node.js | Bun |
| **AI SDK** | 直接调用 | Vercel AI SDK |
| **UI** | 无（借第三方） | SolidJS + OpenTUI |
| **场景** | 办公自动化 | 代码开发 |

**核心洞察**：

虽然都叫 "OpenX"，但两者是**完全不同的产品**：
- **OpenClaw** = 企业自动化平台（类似 RPA + AI）
- **OpenCode** = 开发者生产力工具（类似 Cursor + Claude Code）

根据你的需求选择，而不是技术栈相似度！

---

**文档结束**

*基于实际代码分析（OpenClaw v2026.1.24 / OpenCode v1.1.48）*
